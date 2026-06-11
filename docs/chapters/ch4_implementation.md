
# Chapter 4: Implementation

## 4.1 Application Layer — HTML Dashboard

The application consists of a static HTML dashboard served by Nginx.
The file `app/index.html` displays six key observability metrics:
Service Status, Average MTTR, Uptime percentage, Failures Simulated,
Alerts Triggered, and Average MTTD.

These values are hardcoded as placeholder data during initial UI development.
Final values will be updated after completing all five controlled failure
simulations in Chapter 5.

The dashboard uses no external dependencies or JavaScript frameworks —
intentionally kept lightweight so Nginx can serve it as a pure static file
with minimal resource consumption on the t2.micro instance.

**Screenshot:** `docs/screenshots/dashboard_initial_ui.png`

## 4.2 Nginx Configuration

The file `app/nginx.conf` configures Nginx as a static file server
running on port 80 inside the Docker container.

Two design decisions are critical to the observability experiment.
First, a dedicated `/health` endpoint returns HTTP 200 with the
body "healthy". The `health-check.sh` script polls this endpoint
every 30 seconds — when it stops responding, the failure detection
timestamp is recorded, starting the MTTD measurement.

Second, the log format is structured with labelled fields including
response time (`rt=$request_time`) and status code. This structure
allows the CloudWatch Agent to parse access logs into queryable
metrics without additional transformation.

Health check requests are excluded from access logs (`access_log off`)
to prevent polling noise from inflating log volume and obscuring
genuine traffic patterns.


## 4.2.1 Implementation Challenges — Nginx Configuration

Initial attempts used Nginx's default configuration, which lacked
a `/health` endpoint required for automated failure detection.
Understanding why the log format needed structured labelling
(rather than free-form text) required studying how CloudWatch
metric filters parse log streams.

The `access_log off` directive for the `/health` location was
added after recognising that health-check polling at 30-second
intervals would generate approximately 2,880 log entries per day
— noise that would obscure genuine error events in CloudWatch
log analysis.

Security hardening was applied following the principle of least
privilege: `server_tokens off` removes version disclosure,
method restriction limits the attack surface to GET/HEAD only,
and timeout values were set to mitigate slow loris attacks — a
relevant concern for a single t2.micro instance with limited
worker connections.


## 4.3 Dockerfile

The `app/Dockerfile` packages Nginx, the custom configuration,
and the HTML dashboard into a single deployable image using
`nginx:alpine` as the base — chosen for its minimal attack
surface (~5MB) compared to full distributions.

The default Nginx configuration is removed at build time to
prevent conflicts with the custom `nginx.conf`. A Docker
`HEALTHCHECK` directive polls the `/health` endpoint every
30 seconds with a 5-second timeout, marking the container
unhealthy after three consecutive failures. This provides
a second layer of failure detection alongside the external
`health-check.sh` script.

Nginx runs with `daemon off` so it remains as PID 1 in the
container. Docker monitors PID 1 — if Nginx crashes, PID 1
exits, the container stops, and CloudWatch detects the
resulting metric drop.


## 4.3.1 Docker Security Practices

The base image is pinned to `nginx:1.27-alpine` rather than
the floating `nginx:alpine` tag, ensuring reproducible builds
regardless of upstream changes. Alpine Linux reduces the attack
surface significantly — the image is approximately 5MB compared
to 70MB+ for Ubuntu-based alternatives, with fewer installed
packages that could contain vulnerabilities.

A `.dockerignore` file prevents documentation, scripts, git
history, and environment files from being copied into the image
during build. This is critical in pipelines where images may
be pushed to container registries — accidental inclusion of
`.env` files or AWS credentials would constitute a significant
security breach.

Image metadata is documented via `LABEL` directives, enabling
traceability via `docker inspect`.



## 4.3.3 Image Versioning Strategy

Docker images follow an immutable versioning strategy — once
a version tag is built, it is never overwritten. Changes always
produce a new version tag:

- `v1` — initial build using `nginx:1.27-alpine` (minor version pinned)
- `v2` — improved build using `nginx:1.27.5-alpine` (exact patch version pinned)

This practice ensures full traceability — any running container
can be traced back to its exact build configuration. Overwriting
existing tags creates silent inconsistencies where the same tag
produces different behaviour over time, a common cause of
production incidents.

The `latest` tag is maintained as a pointer to the most recent
stable version (`v2`), following standard Docker Hub convention.

## 😎😎 why add .dockerignore 
Without .dockerignore:
  Someone writes COPY . . in Dockerfile
  → .env file with AWS keys gets copied into image
  → image pushed to Docker Hub (public)
  → AWS keys exposed to internet
  → AWS bill: $47,000 next morning (real incident, happens frequently)

With .dockerignore:
  .env is blocked before it even reaches Docker daemon
  → impossible to accidentally copy it

###If you change only index.html and rebuild, Docker reuses all layers above it unchanged. Only re-runs from the changed layer down. Builds become very fast after the first time.
This is why order in Dockerfile matters — put things that change rarely at the top, things that change often at the bottom.###


## WHY I CHOSE ALPINE BASE :
Our image:     48.2MB   (nginx:alpine base)
nginx:latest:  161MB    (Ubuntu base)

Difference: 161MB - 48.2MB = 112.8MB smaller
That's 70% reduction in image size


## 4.3.4 Container Vulnerability Scanning and Remediation

### Scanning Methodology

Docker Scout was used to scan all image versions against the
CVE (Common Vulnerabilities and Exposures) database before
any deployment to AWS EC2. This follows the DevSecOps principle
of "shift left" security — identifying vulnerabilities as early
as possible in the pipeline rather than discovering them in
production.

### Initial Scan Results — v1 and v2

Both v1 and v2 returned 87 total vulnerabilities, all originating
from the nginx:1.27.5-alpine base image layers (layers 0–18).

| Severity | Count | Fixable |
|----------|-------|---------|
| Critical | 4     | Yes — severity 9.1–9.8 |
| High     | 26    | Yes — severity 8.1–8.8 |
| Medium   | 45    | Partial |
| Low      | 11    | Partial |
| **Total** | **87** | |

The application layers contributed by this project (layers 19–25)
showed zero new vulnerabilities, confirming the custom Dockerfile
instructions introduced no security issues.

### Root Cause

The base image nginx:1.27.5-alpine is built at a fixed point in
time. Alpine Linux packages within it (openssl, musl-libc, libssl)
carry version numbers from that build date. CVEs are discovered
continuously — Docker Scout cross-references installed package
versions against the current CVE database, identifying
vulnerabilities discovered after the image was originally published.
This is expected behaviour — no base image remains
vulnerability-free indefinitely.

### Remediation Applied

The following layer was added to the Dockerfile:

`RUN apk update && apk upgrade --no-cache && rm -rf /var/cache/apk/*`

This instructs Alpine's package manager to upgrade all installed
packages to their latest patched versions at build time, then
removes temporary cache files to minimise image size.

### Post-Remediation Scan — v4

| Severity | Before (v1) | After (v4) | Reduction |
|----------|-------------|------------|-----------|
| Critical | 4           | 0          | 100%      |
| High     | 26          | 3          | 88%       |
| Medium   | 45          | 21         | 53%       |
| Low      | 11          | 1          | 90%       |
| **Total** | **87**     | **25**     | **71%**   |

### Accepted Risk

The remaining 25 vulnerabilities carry no available fixes in
Alpine's current package repository — confirmed by the absence
of checkmarks in Docker Scout's Fixable column. These represent
known but currently unpatched upstream vulnerabilities.

In production environments this is documented as accepted risk,
monitored continuously, and addressed when patches become
available. Attempting to eliminate all CVEs is not achievable
in practice — the goal is to fix what is fixable and document
what is not.

### Impact on Image Size

| Version | Size    | CVEs | Change |
|---------|---------|------|--------|
| v1      | 48.2 MB | 87   | Baseline |
| v2      | 48.2 MB | 87   | Version pinned only |
| v3      | 62.5 MB | Reduced | apk upgrade, no cleanup |
| v4      | 60.1 MB | 25   | apk upgrade + cache removed |

The 11.9MB increase from v1 to v4 represents updated package
versions containing security patches. The cache cleanup in v4
recovered 2.4MB compared to v3.

**Screenshot evidence:**
- Figure 4.1: docker_scout_v1_87_vulnerabilities.png
- Figure 4.2: docker_scout_v4_after_fix_25_remaining.png
- Figure 4.3: docker_images_size_comparison.png


## 4.4.1 Docker Compose — Local Development

A `docker-compose.yml` file is maintained at the project root
for local development convenience. It encodes the full
`docker run` configuration as a declarative file, including
port mapping, restart policy, and healthcheck parameters.

The `restart: unless-stopped` policy ensures the container
automatically recovers from crashes and EC2 instance reboots,
while remaining stopped when manually halted — important for
controlled failure simulations where deliberate container
termination must not trigger automatic recovery.

Note: Docker Compose is used for local development only.
The production deployment pipeline uses GitHub Actions with
direct `docker build` and `docker run` commands via SSH.


## 4.5.1 Debugging — Container Health Check Failure

During local testing, `docker ps` reported the container status
as `(unhealthy)` despite the `/health` endpoint returning HTTP 200
when accessed externally via `curl http://localhost:8080/health`.

**Root cause investigation:**

The HEALTHCHECK directive runs inside the container. When executed
internally using `wget -qO- http://localhost/health`, the command
returned "Connection refused". However, using the explicit IPv4
address resolved the issue:
`wget -qO- http://127.0.0.1/health` returned `healthy`.

This confirmed the issue was DNS resolution: inside the Alpine
container, `localhost` resolves to `::1` (IPv6) rather than
`127.0.0.1` (IPv4). Our nginx.conf only contained `listen 80;`
which binds to IPv4 only, so IPv6 connection attempts were refused.

Evidence was also visible in the container startup logs:
The nginx:alpine entrypoint script attempts to add IPv6 support
to the default configuration file — but since we replaced that
file with our custom nginx.conf, the script had no effect.

**Fix applied in v5:**

Two changes resolved the issue:
1. Added `listen [::]:80;` to the nginx.conf server block,
   enabling Nginx to accept both IPv4 and IPv6 connections
2. Updated the HEALTHCHECK to use `127.0.0.1` explicitly with
   more robust wget flags: `--no-verbose --tries=1 --spider`

Image v5 was rebuilt and `docker ps` confirmed `(healthy)` status
after 35 seconds.

**Lesson:** Internal container health checks behave differently
from external connectivity checks. IPv4/IPv6 dual-stack
configuration is essential in Alpine-based containers where
localhost resolves to IPv6 by default.

Figure 4.x was not captured during the unhealthy state.
The `(unhealthy)` status was observed in `docker ps` output
prior to the IPv6 fix. Post-fix status is shown in Figure 4.x.
