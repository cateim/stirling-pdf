# cateim/stirling-pdf — Repository Guide (for AI agents & humans)

This repo is a **CI-only project**: it automatically builds an **unofficial, 100% MIT
(`core` flavor) Docker image of [Stirling-PDF](https://github.com/Stirling-Tools/Stirling-PDF)**
and publishes it to Docker Hub as `cateim/stirling-pdf`. There is **no application source code
here** — the app is Stirling-PDF's; this repo only orchestrates the build.

## Why this exists

The official Stirling-PDF image bundles the commercial `app/proprietary` module, which enforces a
**5-user limit** and gates login/SSO behind a paid license. Stirling ships a build **flavor**
selector (`STIRLING_FLAVOR = core | proprietary | saas`). The **`core`** flavor compiles WITHOUT
the proprietary module → pure **MIT**, **no user limit**.

### Critical trade-off (read before changing anything)

`core` removes the ENTIRE proprietary module — which is where **all login / SSO / user management /
email** lives. So the resulting image has **NO built-in authentication**. It MUST run behind an
external auth layer:

- **Cloudflare Access** (Zero Trust) — the intended solution (Google login at the edge);
- or Authelia / Authentik, a reverse proxy with auth, or LAN-only.

### Licensing stance (do NOT cross this line)

- `core` is **MIT** (© Stirling PDF Inc.). This build contains **none** of `app/proprietary`.
- We only select the MIT flavor the project itself exposes via `STIRLING_FLAVOR`. We do **NOT**
  patch, strip, or circumvent any license check — that is explicitly forbidden by the proprietary
  license and is off-limits. Keep upstream copyright/license notices intact.

## How the build works

`.github/workflows/build.yml` runs weekly (Sun 04:00 UTC) + manual `workflow_dispatch` + `push` to
`master` that touches the workflow. It has **3 jobs**:

1. **prepare** (no secrets) — reads the latest release tag (GitHub API), validates it, and decides
   whether to build (skips if the version tag already exists on Docker Hub, unless manually dispatched).
2. **build** (matrix `amd64` + `arm64`) — `actions/checkout` of Stirling-Tools/Stirling-PDF **at that
   tag** and builds their **official** `docker/embedded/Dockerfile` with
   `--build-arg STIRLING_FLAVOR=core --build-arg VERSION_TAG=<version>`; each arch is pushed **by digest**.
3. **merge** — combines the per-arch digests into a multi-arch manifest tagged `:<version>` + `:latest`.

We ship **no Dockerfile of our own** — reusing upstream's means base-image bumps and dependency
changes are inherited automatically; we only flip the flavor.

- **Runners:** `ubuntu-24.04-arm` (arm64) + `ubuntu-latest` (amd64) — both free for **public** repos.
  Native per-arch builds avoid slow QEMU (the build is heavy: JDK 25 + Gradle + React frontend).
- **Target arch:** **linux/amd64 + linux/arm64** (multi-arch). Deploy host is an Orange Pi 5 (arm64).
- **Pipeline security:** third-party actions are **pinned to commit SHA** (Dependabot keeps them
  current via `.github/dependabot.yml`); `permissions: contents: read`; `persist-credentials: false`;
  `push` scoped to `master`. Accepted trust boundary: we build arbitrary upstream code weekly — a
  compromised upstream yields a compromised image. The Dockerfile `RUN` steps run in the BuildKit
  sandbox (no access to the runner's Docker Hub token); scope the token to this single repo (Read+Write)
  to bound blast radius.

## Setup (one-time)

- Repo must be **public** (for the free arm64 + amd64 runners).
- Actions secrets: `DOCKERHUB_USERNAME`, `DOCKERHUB_TOKEN`. Scope the token to the **single repo
  `cateim/stirling-pdf`, Read & Write** (no Delete/Admin).
- Trigger the first build via **Actions → Docker Publish → Run workflow** (or push the workflow).

## Deploy target (the consumer of this image)

The consumer is the sibling **DIY** repo (`.../GitHub/DIY`): guide `docs/stirling-pdf.md`, stack
`assets/stacks/stirling-pdf.yml`, deployed on an Orange Pi 5 via **Portainer**, data under
`/srv/pdf`, port `8082`, exposed via **Cloudflare Tunnel** at `pdf.selflabs.org`.

This repo also ships a ready [`docker-compose.yml`](docker-compose.yml) (loopback port, `/srv/pdf`
data, resource limits, hardening, all core-safe env vars documented). To switch the deploy to THIS
image (via that compose or the DIY stack):

1. Point the stack `image:` to `cateim/stirling-pdf:latest`.
2. Put **Cloudflare Access (Google)** in front — the `core` image has no login.
3. Drop proprietary-only env (`SECURITY_ENABLELOGIN`, `SECURITY_OAUTH2_*`, `MAIL_*`,
   `DISABLE_ADDITIONAL_FEATURES`) — no effect without the proprietary module.
4. The container runs as **root then drops to UID/GID 1000** (PUID/PGID); `chown -R 1000:1000
   /srv/pdf`. OCR language data path is `/usr/share/tesseract-ocr/5/tessdata`.

## Conventions

- **Commits:** Conventional Commits, in English (e.g. `ci: ...`, `docs: ...`). Never commit secrets.
- **Never** `git commit`/`git push` automatically — only when the owner explicitly asks.
- This repo lives inside **Google Drive**. A git repo in a synced folder can corrupt `.git` because
  the sync client locks files during git maintenance — that is the real cause of any
  `could not write ... Permission denied` / `geometric-repack failed` error, **NOT** Google Drive
  "re-materializing" files. Prefer keeping `.git` out of the sync path (see the `git-drive-sync`
  skill) or clone outside Drive.

## Background / decision history

Native Stirling OAuth2/SSO login is a **paid** feature: `isOAuthEligible()` returns true only with a
paid license or the per-user `oauthGrandfathered` flag, and grandfathering is applied **only** during
a V1→V2 upgrade — a fresh V2 install can never unlock it for free (confirmed in
`app/proprietary/.../UserLicenseSettingsService.java`). Building `core` was chosen to get unlimited
users under MIT, accepting that authentication moves entirely to Cloudflare Access.

> Note: functionally, `DISABLE_ADDITIONAL_FEATURES=true` on the **official** image gives the same
> result (no limit, no login) with zero build. This repo exists to produce a clean 100%-MIT artifact
> (no dormant proprietary code) as an ops exercise, not because a build is strictly required.
