# Marathoon Connected Runner

Run your [Marathoon](https://marathoon.fr) jobs on your own infrastructure: a Docker host or a
Kubernetes cluster. The runner only makes **outbound HTTPS** calls (to the Marathoon API and to
your image registries) and needs no inbound port. Compute that runs on your runner is not billed
in Marathoon credits.

- Image: `ghcr.io/lordcorp/marathoon-runner` (linux/amd64, linux/arm64)
- Helm chart: `oci://ghcr.io/lordcorp/charts/marathoon-runner`

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
  ghcr.io/lordcorp/marathoon-runner:latest
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
    image: ghcr.io/lordcorp/marathoon-runner:latest
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

### Docker Swarm

One runner task per worker node labelled `marathoon.runner=true`, all under one identity kept in a
Swarm secret. Each node runs jobs on its own Docker and takes as many as it can hold.

```bash
# Once, on a manager: the API key goes straight into the secret (never printed, never on disk)
docker run --rm --group-add "$(stat -c %g /var/run/docker.sock)" \
  -v /var/run/docker.sock:/var/run/docker.sock \
  ghcr.io/lordcorp/marathoon-runner:latest enroll \
  --api-url '<api url shown by the wizard>' --token 'mre_…' \
  --swarm-secret marathoon-runner-credentials
# Then deploy the stack file shown by the wizard (Docker Swarm tab)
docker stack deploy -c marathoon-runner.stack.yml marathoon
```

The stack runs read-only, without capabilities, on workers only, with the secret mounted `0400`.
Draining a node lets running jobs finish (10 minutes). Revoking the machine in the web app stops
every node.

### Kubernetes (Helm)

Jobs run as Kubernetes Jobs in your cluster: no Docker socket, no privileged pod, no persistent
volume.

```bash
helm upgrade --install marathoon-runner oci://ghcr.io/lordcorp/charts/marathoon-runner \
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
| `MARATHOON_RUNNER_SHUTDOWN_GRACE_SECONDS` | On stop, how long running jobs may finish before being cancelled (default 0) |
| `MARATHOON_CREDENTIALS_FILE` | Credentials file (Swarm secret); authoritative, never copied to disk |
| `MARATHOON_INSTANCE_ID` | Instance name when several processes share one identity (Swarm: `{{.Node.Hostname}}`) |
| `MARATHOON_RUNNER_MAX_CONCURRENCY` | Jobs run in parallel: `auto` (default) or a fixed number — see below |
| `MARATHOON_RUNNER_CPUS` / `MARATHOON_RUNNER_MEMORY_MB` | CPU (cores) and memory (MiB) ceilings for job containers; `0` = unlimited |
| `MARATHOON_RUNNER_NETWORK_MODE` | Job container network: `bridge` (default) or `none` |
| `MARATHOON_RUNNER_ALLOWED_REGISTRIES` | Comma-separated registries (or `registry/namespace` prefixes) jobs may pull from; empty = any |
| `MARATHOON_RUNNER_READONLY_ROOTFS` | Read-only root filesystem for job containers (`/outputs` stays writable) |
| `DOCKER_HOST` | Docker endpoint, if not the local socket |

Job containers run with all capabilities dropped and `no-new-privileges` by default.

### Parallel jobs

The runner executes several jobs at once. With `auto` it sizes that from where it runs, and
re-checks every minute:

- **Docker / Docker Desktop** — the daemon's CPUs and memory divided by one job's share
  (`MARATHOON_RUNNER_CPUS` / `MARATHOON_RUNNER_MEMORY_MB`, else 1 CPU / 1 GiB), at least 1.
- **Docker Swarm** — the same, per node: the stack above runs one runner task per node. A node set
  to `drain` or `pause` takes no new job.
- **Kubernetes** — what the namespace's ResourceQuotas still admit at the job pod's resources,
  bounded by the schedulable nodes when the chart's `rbac.readNodes` is on; 4 otherwise.

Automatic sizing stays at 16 or below; a fixed number may go higher.

## Verify the agent downloads

The installers on `dl.marathoon.fr` come with a `SHA256SUMS` file signed by the Marathoon release
key, [`marathoon-agent-release.asc`](marathoon-agent-release.asc) in this repository:

```
Marathoon agent releases <security@marathoon.fr>
B4E7 599A 27EC 0CDA 2686  F84D 095E D59D 01DC 60F6
```

```bash
curl -fsSL https://raw.githubusercontent.com/LordCorp/marathoon-runner/main/marathoon-agent-release.asc | gpg --import
curl -fsSLO https://dl.marathoon.fr/latest/SHA256SUMS -O https://dl.marathoon.fr/latest/SHA256SUMS.asc
gpg --verify SHA256SUMS.asc SHA256SUMS          # "Good signature", fingerprint as above
sha256sum -c SHA256SUMS --ignore-missing        # in the folder holding the downloaded files
```

## Example pipeline images

The example pipelines of the web app's **Examples** page run public images, so a runner pulls
them with no login and nothing to build:

| Image | What it does |
|---|---|
| `ghcr.io/lordcorp/marathoon-analyzer-simple:1.0` | Simulated analysis, no input file |
| `ghcr.io/lordcorp/marathoon-analyzer-file:1.0` | Simulated analysis of the input files |
| `ghcr.io/lordcorp/marathoon-csv-json:1.0` | Converts CSV inputs to JSON, with a report |
| `ghcr.io/lordcorp/marathoon-sample-report:1.0` | Writes a visual report (charts, table, image) |

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
