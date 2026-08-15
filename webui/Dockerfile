FROM ghcr.io/open-webui/open-webui:main

# Bake the local ollama-serve CA into the container's trust store so the
# webui can verify the TLS connection to the nginx pool (https://172.30.0.1:11435)
# without disabling verification or breaking other TLS (HF, registries, ...).
COPY nginx/certs/ca.crt /usr/local/share/ca-certificates/ollama-serve-ca.crt
RUN update-ca-certificates

# The app rebuilds the bundled frontend into open_webui/static at every
# startup (deletes + copies), so it must be writable by the non-root user
# (uid 1000) at runtime.
RUN chown -R 1000:1000 /app/backend/open_webui/static
