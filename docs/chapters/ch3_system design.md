## 3.4 IAM Security Architecture

### 3.4.1 Design Rationale

Three distinct AWS identities exist in this pipeline, each scoped
to the principle of least privilege, with zero long-lived
credentials stored in GitHub or on EC2.

| Identity | Type | Lifetime | Used By |
|----------|------|----------|---------|
| IAM User | Permanent | Indefinite | Developer (WSL CLI) |
| GitHub Actions Role | Temporary (OIDC) | 60 minutes | CI/CD pipeline |
| EC2 Instance Role | Temporary (IMDSv2) | 6 hours, auto-rotated | CloudWatch Agent |

### 3.4.2 GitHub OIDC Authentication Flow

The complete authentication sequence when the GitHub Actions
workflow executes:

**Step 1 — Workflow trigger:** GitHub Actions Runner triggers
the deployment workflow and requests cryptographic proof of
its identity.

**Step 2 — JWT issuance:** GitHub's Token Authority (OIDC
provider) issues a short-lived JSON Web Token, signed using
RS256 (RSA signature with SHA-256), containing claims about
the repository, branch, and environment.

**Step 3 — Token presentation:** The runner presents this JWT
to AWS IAM via the `AssumeRoleWithWebIdentity` API call.

**Step 4 — Cryptographic verification (the critical security
check):**
- 4a. AWS fetches GitHub's public signing keys from GitHub'ss
  JWKS endpoint (`/.well-known/jwks`)
- 4b. GitHub returns its current public keys, matched by the
  `kid` (Key ID) claim in the JWT header
- 4c. AWS IAM mathematically verifies the JWT signature against
  these public keys, AND checks the IAM role's Trust Policy
  conditions (repository, branch, environment must match exactly)

This step prevents JWT forgery — only GitHub's actual private
key can produce a signature that verifies against GitHub's
published public keys.

**Step 5 — Temporary credential issuance:** AWS STS returns
temporary credentials: AccessKeyId, SecretAccessKey, and
SessionToken, valid for 60 minutes only.

**Step 6 — Deployment execution:** The runner uses these
temporary credentials to call AWS APIs (EC2 describe, CloudWatch)
for the duration of the deployment.

### 3.4.3 EC2 Instance Metadata Security — IMDSv2

The EC2 instance is configured with `HttpTokens: required`,
enforcing Instance Metadata Service Version 2 (IMDSv2) exclusively.

**Step 7 — Token request:** Any process on EC2 requesting
credentials (e.g., CloudWatch Agent) must first send a PUT
request to `169.254.169.254/latest/api/token` to obtain a
session token.

**Step 8 — Credential retrieval:** Only with this session token
in the request header can the process then GET temporary
credentials from `/iam/security-credentials/`. These credentials
auto-rotate every 6 hours.

**Security rationale:** IMDSv1 permitted a single GET request to
retrieve credentials directly — vulnerable to Server-Side Request
Forgery (SSRF) attacks, where a compromised application could be
tricked into requesting the metadata endpoint and exfiltrating
credentials. IMDSv2's two-step PUT-then-GET pattern with custom
headers cannot be replicated by typical SSRF payloads, which are
generally limited to simple GET requests.

### 3.4.4 Metrics Pipeline and Per-Call Authorization

**Step 9:** The Observability Compute Instance streams server
performance metrics and Nginx logs to the AWS CloudWatch Metrics
Engine using the IMDSv2-issued temporary credentials.

**Step 10:** Critically, IAM does not authorize access only once
at role assumption — every individual API call (each
`PutMetricData` request) is independently evaluated against the
attached IAM policy by AWS IAM's per-call evaluator.

**Step 11:** Only after this per-call authorization succeeds is
the metric ingestion permitted into CloudWatch.

### 3.4.5 Audit Trail

All authorization decisions — both allow and deny — and every
`AssumeRoleWithWebIdentity` call are logged to AWS CloudTrail,
providing an immutable audit log. This satisfies the
accountability principle: any access to AWS resources can be
traced to a specific workflow run, repository, and timestamp.

### 3.4.6 Threat Model Summary

| Threat | Mitigation |
|--------|-----------|
| Leaked GitHub Secret containing AWS keys | OIDC — no static keys exist anywhere |
| Forged identity claiming to be GitHub Actions | JWKS cryptographic signature verification |
| Workflow run from unauthorized branch/repo | Trust policy condition checks (4c) |
| SSRF attack stealing EC2 credentials | IMDSv2 enforced — PUT+header required |
| Stolen temporary credentials | Expire in 60 min (GitHub) / 6hr (EC2) |
| Untraceable access | CloudTrail immutable audit logging |
| Overly broad permissions | Per-call evaluation against least-privilege policy |

**Diagram:** Figure 3.x — `iam_oidc_security_architecture_detailed.png`