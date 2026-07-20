# CodeSentry — Deployment & Integration Questions for the Organization

Purpose: CodeSentry needs to move from local/dev setup to an enterprise deployment
on **Azure**, with source control and CI/CD on **GitLab**. Below are the open
questions to align with the platform/infra/security teams before finalizing the
architecture. Answers can be filled in inline under each question.

---

## 1. Azure Landing Zone & Subscription

1. Which Azure subscription(s) / resource group(s) is this workload assigned to
   (dev, staging, prod — separate subscriptions or one with resource groups)?
2. Is there an existing Azure Landing Zone / enterprise-scale architecture we
   must conform to (management groups, naming conventions, tagging policy)?
3. Who owns subscription-level access, and how do we request new resources
   (self-service via Terraform/Bicep, or a ticket to a platform team)?
4. Are there mandated Azure Policy guardrails (allowed regions, allowed SKUs,
   required tags, encryption requirements) we need to design against?
5. Which Azure region(s) should we deploy to, and is multi-region /
   active-active required, or single-region with DR failover?

## 2. Compute Hosting Model

CodeSentry has three runtime pieces: a **FastAPI web API**, **Celery background
workers**, and **CLI-invoked scanner binaries/subprocesses** (Semgrep, Gitleaks,
Trivy, OSV-Scanner).

6. Preferred compute target: **Azure Container Apps**, **AKS**, **App Service
   (Linux container)**, or **Azure VMs**? Is there an existing standard the org
   uses for containerized Python services?
7. Since scanners are invoked as subprocesses that run against untrusted/
   uploaded code and shell out to external binaries, is there a requirement for
   **sandboxed/isolated execution** (e.g., ephemeral containers, gVisor,
   Kubernetes jobs per scan) rather than a long-lived shared worker process?
8. What are the expected concurrency / scan-volume requirements (scans/day,
   peak concurrent scans)? This drives worker autoscaling design.
9. Do we need GPU or special compute for the optional LLM enrichment step, or
   is that purely an outbound API call (see Section 6)?
10. Is Windows or Linux the target container OS? (Current dev setup is Windows,
    but scanner binaries — Semgrep/Trivy/Gitleaks/OSV — are more standard on
    Linux containers.)

## 3. Networking & Security

11. Should the API be internet-facing, or internal-only behind a VPN /
    ExpressRoute / private endpoint?
12. Is Azure Front Door / Application Gateway / API Management the required
    ingress layer for TLS termination, WAF, and auth?
13. What are the network segmentation requirements between the web API,
    Celery workers, Redis, and the database (VNet, subnets, NSGs, private
    endpoints)?
14. Since users can submit a **git URL** to be cloned, or upload a **ZIP**
    containing arbitrary source code — is outbound internet access from the
    worker/scanner tier restricted or must it go through an egress
    proxy/firewall allowlist (e.g., only allow git clone to approved hosts)?
15. Is a Web Application Firewall (WAF) mandatory in front of the upload
    endpoint given it accepts file uploads?

## 4. Identity & Access Management

16. What authentication is required for the API — Azure AD (Entra ID)
    OAuth2/OIDC, API keys, mutual TLS, or something else? Is there an existing
    internal auth gateway/SSO pattern to plug into?
17. Should we use **Managed Identity** for the app/workers to access Azure
    resources (Key Vault, Storage, Database) instead of connection strings/
    secrets?
18. Are there role-based access requirements (who can trigger scans, who can
    view reports/findings, who can see raw secrets even if masked)?
19. Do findings/reports need to be scoped per team/project with tenant
    isolation, or is this a single shared internal tool?

## 5. Data, Storage & Persistence

20. Target database: **Azure Database for PostgreSQL Flexible Server**, or
    is SQLite acceptable for any environment (current default is SQLite)?
21. Where should uploaded ZIPs, extracted workspaces, and generated
    `report.html` files live — **Azure Blob Storage**, Azure Files, or local
    ephemeral disk on the compute tier?
22. What are the retention requirements for scan reports, uploaded source
    code, and findings data (e.g., 30/90/365 days, indefinite, compliance-
    driven)?
23. Findings can contain **secrets, vulnerable code snippets, and dependency
    info** — is there a data classification requirement (Confidential/
    Restricted) that dictates encryption-at-rest, access logging, or a
    specific storage tier?
24. Is Redis required for production (Celery broker/result backend) —
    **Azure Cache for Redis**, and what tier/SLA?
25. Do we need geo-redundant backup for the database and stored reports, and
    what's the target RPO/RTO?

## 6. LLM / Azure OpenAI Integration

CodeSentry has an optional LLM enrichment step (`LLM_PROVIDER=azure` is
already supported in config) that sends only normalized findings + short
snippets (never full source) to the LLM.

26. Should we use **Azure OpenAI Service** (in-tenant, private) instead of
    public OpenAI, given source-derived snippets are sent to it?
27. Which Azure OpenAI deployment/model, region, and quota should we target,
    and who provisions the resource + API keys?
28. Is there a requirement to keep the LLM call fully within the Azure
    tenant boundary (private endpoint to Azure OpenAI, no public internet
    egress) for compliance reasons?
29. Even with snippets capped at ~10 lines and no full source sent, does this
    require a data-processing/security review given code fragments and
    secrets-adjacent content could be included?
30. Should the LLM step be **disabled by default** (`LLM_PROVIDER=none`) in
    production until a review is complete, with opt-in per scan?

## 7. Secrets Management

31. Should all secrets (DB credentials, LLM API keys, Redis connection
    strings) be stored in **Azure Key Vault** and injected via Managed
    Identity, replacing the current `.env` file approach?
32. Is there an existing Key Vault / secret-rotation policy we must follow?
33. How should GitLab CI/CD authenticate to Azure to deploy (OIDC federation
    with Entra ID / workload identity federation, or a stored service
    principal secret in GitLab CI/CD variables)?

## 8. GitLab Repository & CI/CD

34. Is this project moving to a GitLab **group/subgroup** with existing
    org-wide CI templates, security scanning policies, or protected branch
    rules we must adopt?
35. Should GitLab's **own built-in SAST/Secret Detection/Dependency
    Scanning** be enabled alongside CodeSentry, or does CodeSentry replace/
    supplement that pipeline stage?
36. What's the expected CI/CD flow — build & push container image to
    **Azure Container Registry (ACR)**, then deploy via GitLab CI to
    Container Apps/AKS/App Service? Or is Azure DevOps/Pipelines used
    instead for the deploy step (GitLab for source, Azure for release)?
37. Are we required to use **GitLab Environments** with manual approval
    gates for production deploys?
38. Is a self-hosted GitLab Runner (inside the Azure VNet, for private
    resource access) required, or are SaaS GitLab runners acceptable?
39. Do we need GitLab Container/Package Registry, or exclusively ACR for
    image storage?

## 9. Environments & Release Strategy

40. How many environments are required (dev / test / staging / prod), and do
    they map to separate Azure resource groups or subscriptions?
41. What's the expected deployment cadence and strategy — blue/green,
    canary, or simple rolling update?
42. Is Infrastructure-as-Code required (Bicep or Terraform), and does the
    org have a preferred one + existing modules to reuse?

## 10. Observability & Compliance

43. What's the required logging/monitoring stack — **Azure Monitor / Log
    Analytics / Application Insights**, or a third-party tool (Datadog,
    Splunk)?
44. Are there audit-logging requirements for who ran which scan, on what
    code, and when (relevant since this tool processes potentially
    sensitive source code)?
45. Does this system fall under any compliance scope (SOC 2, ISO 27001,
    internal AppSec policy) that dictates specific controls for a tool that
    scans and stores source code and secrets-detection results?
46. What are the alerting/on-call expectations for scan failures or scanner
    binary unavailability?

## 11. Scanner Binaries & Third-Party Tooling

47. Semgrep/Gitleaks/Trivy/OSV-Scanner are currently dropped into a local
    `tools/` folder — for enterprise deployment, should these be baked into
    the container image at build time, and who owns keeping them patched/
    updated?
48. Are there approved/blocked open-source tool lists we need to check these
    scanners against before enterprise use?
49. Trivy and OSV-Scanner may reach out to public vulnerability databases
    (NVD, OSV.dev) — is that outbound traffic acceptable, or do we need an
    internal mirror/proxy for vulnerability feeds?

## 12. Cost & Ownership

50. Is there a budget/cost-center this workload should be tagged against for
    Azure cost tracking?
51. Who is the long-term operational owner (platform team, security team, or
    the application team) once this is live?

---

_Prepared as a discussion checklist — fill in answers inline or link to the
relevant ADRs/design docs as they're finalized._
