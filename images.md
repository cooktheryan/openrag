# OpenRAG Container Images - UBI9 Migration

All container images have been converted to use Red Hat UBI9 base images.
Built for `linux/amd64` and pushed to `quay.io/opendatahub/odh-openrag`.

All components live under a single repository using component-prefixed tags:
`quay.io/opendatahub/odh-openrag:<component>-latest`

## Local Dockerfiles (this repo — openrag)

| Dockerfile | Image Tag | Original Base | UBI9 Base |
|---|---|---|---|
| `Dockerfile` | `odh-openrag:opensearch-latest` | `opensearchproject/opensearch:3.2.0` (AlmaLinux) | `ubi9/ubi` (both stages, tarball install) |
| `Dockerfile.backend` | `odh-openrag:backend-latest` | `python:3.13-slim` (Debian) | `ubi9/ubi-minimal` (build + runtime) |
| `Dockerfile.frontend` | `odh-openrag:frontend-latest` | `node:20.20.0-slim` (Debian) | `ubi9/nodejs-20` (build + runtime) |
| `Dockerfile.langflow` | `odh-openrag:langflow-latest` | `langflowai/langflow:1.8.0` (Debian) | `ubi9/ubi` (build) + `ubi9/ubi-minimal` (runtime) |

---

## Forked Upstream Repos — Changes Applied

### 1. OpenSearch

| | |
|---|---|
| **Upstream** | https://github.com/opensearch-project/OpenSearch |
| **Fork** | https://github.com/cooktheryan/OpenSearch |
| **Original base** | `almalinux:10` |
| **UBI9 base** | `registry.access.redhat.com/ubi9/ubi:latest` |

#### Files modified

**`buildSrc/src/main/java/org/opensearch/gradle/DockerBase.java`**

```java
// BEFORE
ALMALINUX("almalinux:10");

// AFTER
ALMALINUX("registry.access.redhat.com/ubi9/ubi:latest");
```

The enum name `ALMALINUX` is kept to avoid cascading changes in `distribution/docker/build.gradle`.

#### Dockerfile template (no changes needed)

`distribution/docker/src/docker/Dockerfile` is a Groovy template that uses `${base_image}`
(injected from `DockerBase.getImage()`). It already uses `dnf` as the package manager, which
is compatible with UBI9. The packages it installs (`nmap-ncat`, `shadow-utils`, `zip`, `unzip`)
are all available in UBI9 repos.

#### Maintenance notes

- The Gradle build system injects the base image via `DockerBase.getImage()` into the template.
- The bundled JDK is OS-independent — no Java compatibility concerns.
- When syncing with upstream, only `DockerBase.java` needs to be re-applied if upstream
  changes the base image.
- `curl-minimal` on UBI9 may conflict with `curl` — if the upstream template adds `curl` to
  the `dnf install` list, use `--allowerasing` or remove it.

---

### 2. Langflow

| | |
|---|---|
| **Upstream** | https://github.com/langflow-ai/langflow |
| **Fork** | https://github.com/cooktheryan/langflow |
| **Original base (builder)** | `ghcr.io/astral-sh/uv:python3.12-bookworm-slim` (Debian) |
| **Original base (runtime)** | `python:3.12.12-slim-trixie` (Debian) |
| **UBI9 base (builder)** | `registry.access.redhat.com/ubi9/ubi:latest` |
| **UBI9 base (runtime)** | `registry.access.redhat.com/ubi9/ubi-minimal:latest` |

#### Files modified (7 Dockerfiles)

| File | Purpose | Status |
|---|---|---|
| `docker/build_and_push.Dockerfile` | Full langflow image | Converted |
| `docker/build_and_push_base.Dockerfile` | Base langflow image | Converted |
| `docker/build_and_push_backend.Dockerfile` | Backend-only (no frontend) | Converted |
| `docker/build_and_push_ep.Dockerfile` | Enterprise edition | Converted |
| `docker/build_and_push_with_extras.Dockerfile` | Image with all extras | Converted |
| `docker/dev.Dockerfile` | Development image | Converted |
| `docker/cdk.Dockerfile` | AWS CDK deployment | Converted |
| `docker/render.Dockerfile` | Render.com deployment | No change needed (references `langflowai/langflow:latest`) |
| `docker/render.pre-release.Dockerfile` | Render.com pre-release | No change needed (references `langflowai/langflow:1.0-alpha`) |

#### Common changes applied across all Dockerfiles

**Builder stage:**
- `FROM ghcr.io/astral-sh/uv:python3.12-bookworm-slim` → `FROM registry.access.redhat.com/ubi9/ubi:latest`
- `apt-get` → `dnf` with `--nodocs`
- `build-essential` → `gcc gcc-c++ make`
- Node.js via `dnf module enable nodejs:20` instead of nodesource
- `uv` installed via `pip3.12 install uv` instead of pre-installed in base image
- Python 3.12 installed via `dnf` with symlinks for `python3` and `python`
- Added `rust`, `cargo`, `openssl-devel`, `pkg-config` for native extension builds

**Runtime stage:**
- `FROM python:3.12.12-slim-trixie` → `FROM registry.access.redhat.com/ubi9/ubi-minimal:latest`
- `apt-get` → `microdnf` with `--nodocs`
- `libpq5` → `libpq`
- `xz-utils` → `xz`
- `shadow-utils` added for `useradd`
- `dpkg --print-architecture` → `uname -m` for arch detection
- `grep -oP` (Perl regex) → `grep -oE` + `sed` (PCRE not available on UBI9 minimal)
- Node.js 22 installed from official tarball (same approach, different arch detection)

**CDK Dockerfile:**
- `FROM python:3.10-slim` → `FROM registry.access.redhat.com/ubi9/ubi:latest`
- `apt-get` → `dnf`
- `postgresql-server-dev-all` → `libpq-devel`

#### Maintenance notes

- When syncing with upstream, re-apply the base image and package manager changes.
- The `uv` extras flags (`--extra postgresql`, `--extra nv-ingest`, etc.) are preserved
  exactly as upstream — only infrastructure changes were made.
- Playwright `install-deps` may not fully support RHEL — test browser automation and
  install missing system libraries manually if needed.
- Python 3.12 is used (latest in UBI9 repos), matching upstream's Python version.

---

## Build Notes

- **Cross-compilation**: Build stages that run heavy workloads (npm, uv sync, vite) use
  `--platform=$BUILDPLATFORM` to run natively on ARM and avoid qemu segfaults. Runtime stages
  use the target platform specified at `podman build --platform`.
- **OpenSearch exception**: The OpenSearch build stage in the openrag `Dockerfile` cannot use
  `$BUILDPLATFORM` because `opensearch-plugin` requires the target-arch JDK bundled in the tarball.
- **Podman machine memory**: The langflow build requires at least 6GB memory
  (`podman machine set --memory 6144`). The current machine is set to 6144MB.
- **curl-minimal**: UBI9 and UBI9-minimal ship `curl-minimal` which conflicts with the `curl`
  package. Do not install `curl` — use `--allowerasing` if full `curl` is required, or rely
  on `curl-minimal` which provides the `curl` command.
- **Python version**: UBI9 provides Python 3.12 (not 3.13). The openrag backend Dockerfile
  was updated from Python 3.13 to 3.12 accordingly.
- **Node.js**: UBI9 repos provide Node.js 16 by default. Use `dnf module enable nodejs:20`
  for Node.js 20, or install from the official tarball for Node.js 22.
- **tar/gzip**: Not included in `ubi-minimal` by default — must be explicitly installed.
