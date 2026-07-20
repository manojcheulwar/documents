# CodeSentry — Application Functionality

CodeSentry is a token-efficient source code security scanner. It scans a
project (ZIP upload, local directory, or git URL) with open-source security
tools, normalizes and deduplicates the results, scores risk deterministically,
optionally enriches findings with an LLM, and produces a self-contained
`report.html`. Full pipeline: **ingest → scan → normalize → dedup → risk
score → enrich (optional) → render → persist**.

## 1. Input / Ingestion

- **ZIP upload** — via CLI (`python cli.py project.zip`) or API
  (`POST /scans/upload`, multipart).
- **Local directory path** — via CLI or API (`POST /scans/path`).
- **Git URL** — `https://` URLs only, shallow-cloned (`--depth 1`).
- **Safe ZIP extraction** (`app/utils/file_utils.py`):
  - Rejects path traversal (absolute paths, drive letters, `..` segments).
  - Zip-bomb guard (compression-ratio cap).
  - Per-file size cap, total extracted size cap, max file count cap.
  - Blocks executable/binary extensions (`.exe`, `.dll`, `.so`, `.jar`, …) from
    ever being written to disk.
  - Skips noise directories (`.git`, `node_modules`, `venv`, `dist`, …).
- **Upload validation** — enforces `.zip` extension, non-empty, and a
  configurable max upload size.

## 2. Project Profiling

- Detects **primary language** and a breakdown of languages by file-extension
  counts (Python, JS/TS, Java, Go, Ruby, PHP, C/C++, Rust, Swift, Terraform,
  etc.).
- Detects **frameworks/ecosystems** from marker files (`requirements.txt`,
  `package.json`, `pom.xml`, `go.mod`, `Gemfile`, `Dockerfile`, `manage.py`,
  `angular.json`, …).

## 3. Security Scanners (pluggable, each individually toggle-able)

| Scanner | Purpose |
|---|---|
| **Semgrep** | Static application security testing (SAST) — code-level vulnerability rules |
| **Gitleaks** | Hardcoded secret / credential detection |
| **Trivy** | Dependency vulnerabilities + container/IaC misconfigurations |
| **OSV-Scanner** | Open-source dependency vulnerability database lookups |

- Scanner binaries dropped into the project's `tools/` folder are
  auto-detected (no PATH setup needed).
- Each scanner runs with a hard timeout, `shell=False` subprocess execution,
  resolved executables only, and capped output size.
- Missing/failing scanners are skipped gracefully and reported in the report
  appendix (per-scanner status, duration, error).

## 4. Normalization & Classification (`app/normalizer/normalize.py`)

- Converts each scanner's native JSON output into a common `Finding` schema.
- **OWASP Top 10 (2021) mapping** — CWE IDs are mapped to OWASP categories
  (A01–A10) via an extensive CWE→OWASP lookup table.
- Extracts/derives: severity, confidence, CWE, CVSS score (when available),
  fixed-version info (for dependency findings), references, and a ≤10-line
  code snippet around the finding location.
- **Secrets are never exposed** — Gitleaks findings store only a masked
  value (`abc1**...*** (32 chars)`) plus rule/entropy metadata, never the raw
  secret.
- **Deduplication** — identical findings (same rule + file + line) are merged
  and counted as occurrences; results are sorted by severity/risk.

## 5. Deterministic Risk Scoring (`app/risk/risk_score.py`)

- **Per-finding score (0–100)**: severity base score, upgraded by real CVSS
  score when present, discounted by scanner confidence (HIGH/MEDIUM/LOW).
- **Project-level score (0–100)**: exponential saturation over
  severity-weighted finding counts — one critical dominates a pile of lows;
  score never quite reaches 100.
- Guardrails: any confirmed CRITICAL finding floors the project score at 70;
  any HIGH floors it at 50.
- **Risk level labels**: CLEAN / LOW / MEDIUM / HIGH / CRITICAL, derived from
  the numeric score.
- Severity-count aggregation for reporting/stats.

## 6. LLM Enrichment (optional, `app/llm/`)

- **The LLM never sees your source code.** Only normalized findings with
  ≤10-line snippets are ever sent.
- **Fix templates first** — common CWEs (SQLi, XSS, command injection, path
  traversal, hardcoded secrets, weak crypto, insecure deserialization, XXE)
  get free, zero-token deterministic impact/recommendation/secure-code-example
  text before any LLM call.
- **Grouping & batching** — duplicate findings collapse to one representative
  + count; findings batched (configurable per-batch/max-batches caps) to put a
  hard ceiling on LLM cost.
- **Response caching** — cached to disk (`.llm_cache/`) keyed by a hash of
  finding IDs + model, so re-scans cost nothing.
- **Provider-agnostic** — works with any OpenAI-compatible chat completions
  API: OpenAI, Azure OpenAI, Ollama, LM Studio, vLLM. `LLM_PROVIDER=none` runs
  fully offline (templates only).
- Produces, per scan: executive summary, overall risk narrative, prioritized
  action list, and per-finding impact / recommendation / secure code example
  / remediation steps.
- Strict JSON-only parsing of LLM responses with graceful degradation on
  malformed output (best-effort, non-fatal).

## 7. HTML Report (`app/report/`, single self-contained `report.html`)

1. Title, project name, scan date/time, detected stack.
2. Executive summary + overall risk score (0–100) with level badge.
3. Severity summary — stat tiles + bar chart (Critical/High/Medium/Low/Info).
4. OWASP Top 10 (2021) mapping chart.
5. Sortable vulnerability table + collapsible per-finding details (snippet,
   impact, fix, secure code example, remediation steps, references).
6. Findings grouped by file.
7. Dependency vulnerabilities (with fixed versions).
8. Secret findings (values always masked).
9. Container / IaC misconfigurations.
10. Prioritized action plan.
11. False-positive notes (low-confidence findings flagged for review).
12. Appendix: per-scanner execution status, durations, and errors.

- No CDN, no external requests — fully self-contained.
- Light/dark mode support.
- Jinja2 autoescaping — scanned code cannot inject script into the report.

## 8. Interfaces

### CLI (`cli.py`)
```
python cli.py path/to/project
python cli.py project.zip --output report.html
python cli.py path/to/project --no-llm
```
Prints a scan summary (risk score/level, severity counts, report path).

### REST API (`app/main.py`, FastAPI)
- `GET /health` — liveness check.
- `POST /scans/upload` — multipart ZIP upload (`file`, `project_name`, `use_llm`).
- `POST /scans/path` — JSON `{"target": "<dir or https git url>"}`.
- `GET /scans/{scan_id}` — status, risk score/level, stats.
- `GET /scans/{scan_id}/report` — download the generated `report.html`.

### Background jobs (`app/tasks.py`, Celery)
- `CELERY_TASK_ALWAYS_EAGER=true` (default) runs scans inline — no Redis
  needed for development.
- Set to `false` + run a Celery worker (`--pool=solo` on Windows) for real
  async job processing backed by Redis.

## 9. Persistence (`app/db.py`, SQLAlchemy — SQLite or PostgreSQL)

- **`Scan`** table: id, project name, status (queued/running/done/failed),
  error, created/finished timestamps, risk score/level, report path, stats
  JSON.
- **`FindingRecord`** table: per-scan findings (id, severity, tool, full
  finding payload as JSON) for later querying/auditing.

## 10. Configuration (`app/config.py`, `.env`-driven)

- App/storage paths (uploads, reports, workspace directories, DB URL).
- Upload/extraction limits (max upload MB, max extracted MB, max file count).
- Per-scanner enable flags + timeout.
- Celery broker/backend URLs and eager-mode toggle.
- LLM provider, model, API key, base URL, batch/cache limits, timeout.

## 11. Security Hardening Built In

- ZIP extraction: path-traversal rejection, zip-bomb ratio check, file-count
  and size caps, blocked executable extensions.
- Scanned code is **never executed** — static analysis only.
- Subprocess calls use argument lists only (`shell=False`), resolved
  executables, hard timeouts, output size caps.
- Secrets are masked everywhere: findings, snippets, database records.
- Scanner-reported file paths are re-sanitized (traversal-safe, relative)
  before rendering.
- Jinja2 autoescape on — scanned code cannot inject script into the report.
- Git cloning restricted to `https://` URLs with `--depth 1`.

## 12. Project Layout

```
app/
  main.py                 FastAPI endpoints
  pipeline.py             end-to-end scan orchestration
  tasks.py                Celery background jobs
  db.py                   SQLAlchemy models (SQLite/PostgreSQL)
  config.py               .env-driven settings
  scanner/                semgrep / gitleaks / trivy / osv wrappers
  normalizer/normalize.py common format + OWASP mapping + dedup
  risk/risk_score.py      deterministic scoring
  llm/                    prompt batching, templates, caching, client
  report/                 Jinja2 HTML report
  models/finding.py       common Finding schema
  utils/                  safe subprocess + file handling
samples/                  example scanner outputs
tests/                    normalizer + risk scoring tests
tools/                    bundled scanner binaries (gitleaks, trivy, osv-scanner)
cli.py                    command-line entry point
```
