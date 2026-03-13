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

## TODOs

- In NGSC, replace `host.docker.internal` with RDS endpoint
