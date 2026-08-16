# ollama-serve

A self-hosted, TLS-only Ollama serving stack for a single GPU host (NVIDIA DGX
Spark / GB10). It runs two Ollama instances behind an nginx reverse proxy with
dynamic, health-checked upstreams and session stickiness, plus Open WebUI for a
browser interface. Every path into the stack is HTTPS; nothing is published as
plain HTTP.

## Highlights

- **2 GPU Ollama instances** on the same host, each with full GPU access via
  NVIDIA CDI (`nvidia.com/gpu=all`).
- **Shared model cache** — all instances mount the same `var/lib/ollama` host
  directory, so each model is downloaded exactly once.
- **Keep-alive pinned** (`OLLAMA_KEEP_ALIVE=-1`) so models stay resident and
  never reload between requests; large context (`OLLAMA_CONTEXT_LENGTH=64000`).
- **Dynamic upstream pool** — `healthcheck.sh` probes each instance's socket
  every 5 s and regenerates the nginx upstream, reloading only on change. Failed
  instances are dropped from the pool automatically.
- **Session stickiness** (`ip_hash`) so long conversations stay on one instance.
- **HTTPS-only entry points** on `:11435` (Ollama API) and `:11436` (Open WebUI),
  served by a self-signed local CA.
- **Hardened containers** — non-root user (`1000:1000`), `no-new-privileges`,
  and `cap_drop: ALL` on every service.
- **No TCP ports published** except through nginx: containers talk over unix
  sockets bridged by socat.

## Architecture

![Architecture](docs/diagrams/architecture.svg)

### Services and ports

| Service       | Container port | Host exposure                                  | Purpose                          |
| ------------- | -------------- | ---------------------------------------------- | -------------------------------- |
| `nginx`       | —              | `:11435` (HTTPS), `:11436` (HTTPS)             | TLS front door for the whole stack |
| `ollama-1..2` | `:11434`       | none (reached via unix sockets)                | GPU inference instances          |
| `socat-1..2`  | —              | `var/run/sockets/ollama-N.sock` (unix)         | Bridge TCP `:11434` to sockets   |
| `webui`       | `:8080`        | none (reached via unix socket)                 | Open WebUI                       |
| `socat-webui` | —              | `var/run/sockets/webui.sock` (unix)            | Bridge TCP `:8080` to a socket   |

### Request flow (Ollama API)

![Ollama API request flow](docs/diagrams/request-flow.svg)

A client calls `https://localhost:11435/v1`. nginx terminates TLS, selects a
healthy socket for the client IP via `ip_hash`, and streams the response back
with `proxy_buffering off` so SSE works end to end.

### Request flow (Open WebUI)

![WebUI request flow](docs/diagrams/webui-flow.svg)

The browser loads the UI over `https://localhost:11436`. Open WebUI then calls
back out through the host bridge gateway (`172.30.0.1`) to the same nginx pool
over TLS, using the CA baked into its image. This is why the compose bridge
subnet is pinned: the gateway `172.30.0.1` must be deterministic and present in
the server certificate's SANs.

### Health checks and automatic failover

![Health check loop](docs/diagrams/healthcheck.svg)

```sh
healthcheck.sh --watch` runs inside the nginx container. Every 5 s it curls
`/api/version` over each `ollama-N.sock`. If the healthy set changed, it writes
`var/run/nginx/upstream.conf` and reloads nginx — a crashed or restarting
instance is removed from the pool with zero downtime.
```

## Repository layout
```
├── docker-compose.yml      # The whole stack
├── nginx/
│   ├── nginx.conf          # :11435 pool + :11436 webui, TLS-only
│   ├── healthcheck.sh      # Dynamic upstream watcher
│   └── certs/              # Local CA + server cert (gitignored)
├── webui/
│   └── Dockerfile          # Open WebUI + baked-in CA trust
├── var/                    # Runtime data (gitignored)
│   ├── lib/ollama          # Shared model cache
│   ├── lib/webui           # SQLite DB + uploads
│   ├── run/sockets         # unix sockets
│   └── run/nginx           # generated upstream.conf
├── docs/diagrams/          # Mermaid sources + rendered SVG/PNG diagrams
└── opencode.json           # Example: point opencode at the local pool
```

## Prerequisites

- Docker + Docker Compose (v2).
- NVIDIA Container Toolkit with CDI enabled (used for `driver: cdi`).
- Run `./setup.sh` to create the required directories and self-signed certificates before starting the stack.

## Getting started

```sh
# Run the setup script to create directories and generate self-signed certificates
./setup.sh

# Start the stack (builds the webui image first time)
docker compose up --build -d

# Pull a model on one instance — it lands in the shared cache
docker compose exec ollama-1 ollama pull glm-4.7-flash

# Use it
curl -k https://localhost/11435/api/tags            # trust the CA to drop -k
curl -k https://localhost/11435/v1/chat/completions -d '{
  "model": "glm-4.7-flash",
  "messages": [{"role": "user", "content": "hello"}]
}'
```

Open the UI at <https://localhost:11436> (first-run: create the admin account).

## Configuration knobs

| Setting                              | Where                     | Effect                                   |
| ------------------------------------ | ------------------------- | ---------------------------------------- |
| `OLLAMA_CONTEXT_LENGTH=32768`        | compose `x-ollama-base`   | Context window for large models          |
| `OLLAMA_NUM_PARALLEL=2`             | compose `x-ollama-base`   | Two parallel requests per model (2x context size per request) |
| Number of instances (`1..2`)         | compose `services`        | Scale the pool (and `INSTANCES` in `healthcheck.sh`) |
| `ip_hash` / `keepalive 32`           | generated `upstream.conf` | Stickiness + connection reuse            |
| `OLLAMA_BASE_URL` / `WEBUI_URL`      | compose `webui`           | WebUI backend URL + public UI URL        |
| Probe interval (`INTERVAL=5`)        | `nginx/healthcheck.sh`    | Health-check cadence                     |

## Trusting the CA

- **Host / browsers**: add `nginx/certs/ca.crt` to your system/browser trust
  store, then `https://localhost:11435` and `https://localhost:11436` validate
  without warnings.
- **WebUI container**: done automatically via `webui/Dockerfile`
  (`update-ca-certificates` + `SSL_CERT_FILE`/`REQUESTS_CA_BUNDLE`).
- **opencode**: see `opencode.json`, which points `baseURL` at
  `https://localhost:11435/v1` with the local CA trusted by the system store.

## Troubleshooting

- **Upstream empty / all instances down** — check `var/run/nginx/upstream.conf`;
  the health checker only lists sockets whose `/api/version` probe succeeds.
  Verify the sockets exist (`ls var/run/sockets`).
- **WebUI can't reach the pool** — confirm the bridge gateway is `172.30.0.1`
  (it is pinned to subnet `172.30.0.0/24`) and that the server certificate
  includes it in its SANs.
- **TLS verification failures from the webui** — the baked-in CA must match
  `nginx/certs/ca.crt`; rebuild the image after rotating the CA
  (`docker compose build webui`).
- **Permission errors on writes** — the stack runs as `1000:1000`; make sure
  `var/` (and the sockets dir) are owned by that user.
- **Model download "just sits"** — large blobs stream in; watch
  `docker compose logs -f ollama-1` rather than the partial files in
  `var/lib/ollama/models/blobs`.

## Security notes

- TLS-only everywhere; no HTTP listeners are published.
- Every service runs unprivileged with `no-new-privileges` and no capabilities.
- The certs are self-signed and the CA is private to this host — nothing leaves
  the machine.
- Data mounts (`var/lib/ollama`, `var/lib/webui`) are plain host directories and
  contain user chat data and model weights; back them up accordingly.
