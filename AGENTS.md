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

## Agent-specific notes

- This repo is **not a git repository** — do not run git commands expecting a
  working tree.
- `opencode.json` points opencode at the local Ollama pool
  (`https://localhost:11435/v1`, model `glm-4.7-flash:latest`). If the stack is
  down, that provider is unavailable; mention it rather than editing the config
  to point elsewhere.
- Bash commands that touch the stack are slow (model loads, GPU work). Prefer
  read-only inspection (`docker compose config`, `nginx -t`, reading
  `var/run/nginx/upstream.conf`) before starting/stopping services.
