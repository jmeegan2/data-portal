# Data Portal

Browser-based data environment with RStudio and VS Code, backed by the Dart PostGIS database.

## Quick Start

```bash
docker compose up --build
```

Go to http://localhost

## Login

Credentials are in `.env`:
- **Username:** `admin`
- **Password:** `admin`

To change credentials, update `.env` and regenerate the password file:

```bash
htpasswd -nbB $AUTH_USER $AUTH_PASS > nginx/.htpasswd
```

## DB Connection

Both environments connect to the Dart backend DB. Credentials are in `.env` and available as environment variables inside the containers:

| Variable    | Value                  |
|-------------|------------------------|
| DB_HOST     | host.docker.internal   |
| DB_PORT     | 5551                   |
| DB_NAME     | dart                   |
| DB_USER     | dart                   |
| DB_PASSWORD | DART                   |

Test scripts are pre-loaded in each environment:
- **VS Code:** `python test_db_connection.py`
- **RStudio:** `source("test_db_connection.R")`

## Docker Images

Three custom images, all hardened (OS patches, non-root, minimal attack surface).

### `nginx/` — Reverse Proxy

Based on `nginxinc/nginx-unprivileged` (Alpine). The Dockerfile just applies `apk upgrade` for CVE patches. All config (reverse proxy routes, basic auth) is mounted at runtime via `default.conf`. Runs as UID 101.

#### Proxy Headers

The reverse proxy config (`default.conf`) sets several headers for the RStudio and VS Code locations:

| Directive | Purpose |
|-----------|---------|
| `proxy_pass` | Forwards requests to the upstream container (e.g., `vscode:8080`, `rstudio:8787`) |
| `proxy_http_version 1.1` | Required for WebSocket support |
| `proxy_set_header Upgrade` | Passes the client's WebSocket upgrade request to the upstream |
| `proxy_set_header Connection "upgrade"` | Tells the upstream to perform the HTTP → WebSocket protocol switch |
| `proxy_set_header Host` | Preserves the original hostname so the upstream knows the request's domain |
| `proxy_set_header X-Real-IP` | Passes the user's real IP address (otherwise upstream only sees nginx's internal IP) |
| `proxy_read_timeout 3600` | Keeps connections alive for up to 1 hour so long-running interactive sessions aren't killed |

The WebSocket lines (Upgrade, Connection, HTTP 1.1) are critical — both VS Code and RStudio use persistent WebSocket connections for their interactive UIs. Without these, the sessions would fail to load.

Nginx listens on plain HTTP (port 8080) because TLS termination is handled upstream by the Kubernetes Ingress or load balancer. Internal cluster traffic between the ingress and nginx does not need to be encrypted.

### `rstudio/` — R Environment

Based on `rocker/rstudio` (Ubuntu). Installs `libpq-dev` and the `RPostgres` package so R sessions can connect to the Dart PostGIS database. The rest of the Dockerfile fixes file permissions so `rserver` can run as the `rstudio` user (UID 1000) without root — the base image assumes root by default. Purges `linux-libc-dev` to reduce CVE surface.

### `vscode/` — Python Environment

Based on `linuxserver/code-server` (Ubuntu). Installs Python 3 and `psycopg2-binary` for DB access. Also patches the bundled `undici` module to 7.24.0 to fix several CVEs (request smuggling, WebSocket DoS, CRLF injection), and renames VS Code extension `package.json` fields to work around Trivy false positives where extension metadata gets misidentified as vulnerable npm packages. Purges `npm` and `linux-libc-dev` after build. Runs as the `abc` user.

## TODOs

- In NGSC, replace `host.docker.internal` with RDS endpoint
