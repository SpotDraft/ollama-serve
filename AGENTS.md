# AGENTS.md

Guidance for AI agents working in this repository.

## Read the README first

[`README.md`](README.md) is the source of truth: it documents what the stack is,
the full architecture, setup, configuration knobs, and troubleshooting. Read it
before working here. This file only adds the operational details agents need
that the README does not cover.

## Verification commands

There is no test suite and no linter. Verification = validating the Compose
file, the nginx config, and the rendered diagrams:

```sh
docker compose config                 # lint the compose file (main check)
docker compose exec nginx nginx -t    # lint nginx config inside the container
docker compose exec nginx cat /etc/nginx/runtime/upstream.conf  # healthy pool
ls var/run/sockets                  # sockets must exist (ollama-N.sock, webui.sock)
```

After changing behavior, start the stack (`docker compose up --build -d`) and run
the smoke tests from the README.

## The OpenTelemetry overlay

`docker-compose.otel.yml` is an optional, purely additive overlay (see the
README's "eBPF telemetry" section). It is not loaded unless `COMPOSE_FILE` is
set in `.env`, so `docker compose config` alone does **not** check it. Validate
both shapes after touching either compose file:

```sh
docker compose config --quiet                                          # base
docker compose -f docker-compose.yml -f docker-compose.otel.yml config --quiet
```

The overlay's two config files have their own validators, and both catch real
errors that Compose does not:

```sh
docker run --rm -e OTELCOL_LOG_LEVEL=info -e OTELCOL_DEBUG_VERBOSITY=basic \
  -v "$(pwd)/otel/otelcol:/etc/otelcol:ro" \
  otel/opentelemetry-collector-contrib:0.158.0 validate --config=/etc/otelcol/local.yaml

docker run --rm --pid=host --privileged \
  -v "$(pwd)/otel/obi/config.yaml:/etc/obi/config.yaml:ro" \
  otel/ebpf-instrument:v0.10.0 --config=/etc/obi/config.yaml
```

OBI has no `validate` subcommand: it exits immediately on a bad config and
otherwise logs `starting Application Observability mode` and keeps running, so
treat "still running after a few seconds" as the pass condition and stop it.

Keep the overlay additive — it must never require an edit to
`docker-compose.yml`, `nginx/nginx.conf`, or `nginx/healthcheck.sh`. OBI
discovers its targets through the kernel by listening port, not over the
network, which is what allows the base stack to stay untouched.

## Never commit

The README's security notes cover why secrets stay local; `.gitignore` excludes
the following, so keep it that way:

- `var/` — runtime data (model blobs, SQLite DB, sockets) and user chat data.
- `nginx/certs/` — private CA keys and certificates.
- Any `*.key`, `*.crt`, `*.pem`, `*.sock`, `*.log`.

## Diagrams

Diagram sources are Mermaid `.mmd` files in `docs/diagrams/`; the README
references the rendered `*.svg`. When you change a `.mmd`, regenerate both
formats:

```sh
docker run --rm --user 1000:1000 \
  -v "$(pwd):/data" minlag/mermaid-cli \
  -i /data/docs/diagrams/NAME.mmd -o /data/docs/diagrams/NAME.svg
docker run --rm --user 1000:1000 \
  -v "$(pwd):/data" minlag/mermaid-cli \
  -i /data/docs/diagrams/NAME.mmd -o /data/docs/diagrams/NAME.png -b white
```

Note: the mermaid-cli image runs as uid 1001 by default; the `--user 1000:1000`
flag is required so it can write into the bind-mounted `docs/` tree.

