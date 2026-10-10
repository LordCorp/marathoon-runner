# Marathoon Connected Runner

Run your [Marathoon](https://marathoon.fr) jobs on your own infrastructure: a Docker host or a
Kubernetes cluster. The runner only makes **outbound HTTPS** calls (to the Marathoon API and to
your image registries) and needs no inbound port. Compute that runs on your runner is not billed
in Marathoon credits.

- Image: `ghcr.io/lordinaire/marathoon-runner` (linux/amd64, linux/arm64)
- Helm chart: `oci://ghcr.io/lordinaire/charts/marathoon-runner`

## 1. Get an enrollment token

In the Marathoon web app: **Settings → Infrastructure → Connect infrastructure → Runner**.
The wizard creates the runner and shows a one-time enrollment token (`mre_…`, valid 15 minutes)
together with the exact commands below, already filled in.

## 2. Install

### Docker

The runner runs as the non-root user `app` (uid/gid **1654**). It needs a staging directory it
owns, at the **same path** on the host and in the container, and the group that owns the Docker
socket.

```bash
# One-time: staging directory owned by the runner user
sudo install -d -o 1654 -g 1654 /var/lib/marathoon/runner-tmp

docker run -d --name marathoon-runner --restart unless-stopped \
  --group-add "$(stat -c %g /var/run/docker.sock)" \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /var/lib/marathoon/runner-tmp:/var/lib/marathoon/runner-tmp \
  -v marathoon-runner-config:/etc/marathoon-agent \
  -e MARATHOON_API_URL='<api url shown by the wizard>' \
  -e MARATHOON_AGENT_NAME='build-box-01' \
  -e MARATHOON_ENROLL_TOKEN='mre_…' \
  ghcr.io/lordinaire/marathoon-runner:latest
```

The token is exchanged once for runner credentials, stored in the `marathoon-runner-config`
volume: keep that volume when you recreate the container.

### Docker Compose

```yaml
# One-time, on the host:
#   sudo install -d -o 1654 -g 1654 /var/lib/marathoon/runner-tmp
#   export DOCKER_GID=$(stat -c %g /var/run/docker.sock)
services:
  marathoon-runner:
    image: ghcr.io/lordinaire/marathoon-runner:latest
    restart: unless-stopped
    group_add:
      - "${DOCKER_GID}"
    environment:
      MARATHOON_API_URL: "<api url shown by the wizard>"
      MARATHOON_AGENT_NAME: "build-box-01"
      MARATHOON_ENROLL_TOKEN: "mre_…"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /var/lib/marathoon/runner-tmp:/var/lib/marathoon/runner-tmp
      - marathoon-runner-config:/etc/marathoon-agent
volumes:
  marathoon-runner-config:
```

### Kubernetes (Helm)

Jobs run as Kubernetes Jobs in your cluster: no Docker socket, no privileged pod, no persistent
volume.

```bash
helm upgrade --install marathoon-runner oci://ghcr.io/lordinaire/charts/marathoon-runner \
  --namespace marathoon --create-namespace \
  --set-string marathoon.apiUrl='<api url shown by the wizard>' \
  --set-string marathoon.enrollToken='mre_…'
```

## Configuration

| Variable | Purpose |
|---|---|
| `MARATHOON_API_URL` | Marathoon API origin (without `/api/v1`) |
| `MARATHOON_ENROLL_TOKEN` | One-time enrollment token (never stored) |
| `MARATHOON_AGENT_NAME` | Name shown in the web app |
| `MARATHOON_RUNNER_CPUS` / `MARATHOON_RUNNER_MEMORY_MB` | CPU (cores) and memory (MiB) ceilings for job containers; `0` = unlimited |
| `MARATHOON_RUNNER_NETWORK_MODE` | Job container network: `bridge` (default) or `none` |
| `MARATHOON_RUNNER_ALLOWED_REGISTRIES` | Comma-separated registries (or `registry/namespace` prefixes) jobs may pull from; empty = any |
| `MARATHOON_RUNNER_READONLY_ROOTFS` | Read-only root filesystem for job containers (`/outputs` stays writable) |
| `DOCKER_HOST` | Docker endpoint, if not the local socket |

Job containers run with all capabilities dropped and `no-new-privileges` by default.

## Troubleshooting

| Symptom | Fix |
|---|---|
| The web app says *the local Docker daemon is unreachable* | The container is missing `--group-add "$(stat -c %g /var/run/docker.sock)"` |
| Jobs fail while staging inputs or collecting outputs | `/var/lib/marathoon/runner-tmp` on the host is not owned by uid 1654, or is not mounted at the same path |
| The container exits with code 2 at start | Enrollment failed: the token expired (15 min) or was already used; generate a new one |
| *Agent not online* | Check outbound HTTPS to the API URL; `docker logs marathoon-runner` |

---

This repository only hosts the runner's documentation. Questions and issues: use the support
channel of the Marathoon web app.
