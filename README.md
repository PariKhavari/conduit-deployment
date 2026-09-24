# Conduit Container

A fully containerized version of the **Conduit** application (a Medium.com clone), consisting of a Django REST Framework backend and an Angular frontend, orchestrated together with PostgreSQL via Docker Compose.

## Table of Contents

- [Description](#description)
- [Tech Stack](#tech-stack)
- [Quickstart](#quickstart)
- [Services & Images](#services--images)
- [Docker Architecture](#docker-architecture)
- [Usage](#usage)
  - [Environment Variables](#environment-variables)
  - [Configuration & Customization](#configuration--customization)
  - [Superuser Creation](#superuser-creation)
  - [Static Files](#static-files)
- [Testing the Setup](#testing-the-setup)
- [Logs](#logs)
- [Known Limitations](#known-limitations)
- [Notable Fixes](#notable-fixes)

## Description

This repository contains the full setup required to run the Conduit application entirely inside Docker containers. It bundles:

- `conduit-backend/` – a Django + Django REST Framework API server (Python), serving the application's data and the Django admin panel.
- `conduit-frontend/` – an Angular single-page application, built and served through an nginx-unprivileged container, which also acts as a reverse proxy to the backend.
- A PostgreSQL database container for persistent data storage.

The purpose of this repository is to demonstrate how an older, previously non-containerized full-stack project can be "modernized" into a reproducible, portable setup that can be started with a single command (`docker compose up -d`), without requiring any manual installation of Python, Node, or PostgreSQL on the host machine.

## Tech Stack

- **Frontend:** Angular 17, Node.js (build stage), nginx-unprivileged (runtime)
- **Backend:** Python 3.6, Django, Django REST Framework, Gunicorn (WSGI server)
- **Database:** PostgreSQL 15 (Alpine)
- **Orchestration:** Docker, Docker Compose

## Quickstart

**Prerequisites:**
- Docker and Docker Compose installed (Docker Engine 20+ recommended)
- Git

**Steps:**

```bash
# 1. Clone the repository
git clone https://github.com/<your-username>/conduit-container-v2.git
cd conduit-container-v2

# 2. Create your local environment file from the example
cp .env.example .env

# 3. Edit .env and fill in real values (see "Environment Variables" below)
nano .env

# 4. Build and start all containers
docker compose up -d --build

# 5. Check that everything is running
docker compose ps
```

Once all three containers (`database`, `backend`, `frontend`) report as `Up` (and `healthy` once their healthchecks pass), the application is reachable at:

```
http://<host-ip>:8282
```

A Django superuser is created automatically on first startup — see [Superuser Creation](#superuser-creation) below.

## Services & Images

| Service | Image | Tag | Notes |
|---|---|---|---|
| `database` | `postgres` | `15-alpine` | Official image, unmodified |
| `backend` | `conduit-backend` | `latest` (locally built) | Built from `conduit-backend/Dockerfile` |
| `frontend` | `conduit-frontend` | `latest` (locally built) | Built from `conduit-frontend/Dockerfile`, based on `nginxinc/nginx-unprivileged:1.25-alpine` |

## Docker Architecture

Both `Dockerfile`s use **multi-stage builds** to keep the final image size (and attack surface) as small as possible:

- **Backend** (`conduit-backend/Dockerfile`):
  - A shared `base` stage applies a one-time fix (redirecting Debian's package sources to the archive, since Debian Buster is EOL), used by both stages below so the fix isn't duplicated.
  - Stage `builder` installs build tools (`build-essential`, `libpq-dev`) and compiles the Python dependencies from `requirements.txt` into an isolated `/install` folder.
  - Stage `runtime` starts from `base`, copies **only** the already-installed packages (`COPY --from=builder /install /usr/local`) and the application code, and runs everything as a non-root user (`appuser`). None of the compiler tools from the `builder` stage end up in the final image.
- **Frontend** (`conduit-frontend/Dockerfile`):
  - Stage `build` uses Node.js to install dependencies (`npm ci`) and compile the Angular application into static production files.
  - Stage `runtime` copies **only** the resulting static files (`COPY --from=build /app/dist/angular-conduit /usr/share/nginx/html`) into a lightweight, unprivileged nginx image. Node.js, `node_modules`, and the TypeScript source code never end up in the final image.

This means the images shipped to production contain only what's needed to *run* the application, not what was needed to *build* it, resulting in smaller images and a reduced surface for potential vulnerabilities.

## Usage

### Environment Variables

All configuration is provided via a `.env` file at the project root (not committed to Git — see `.env.example` for the required keys). The most relevant variables:

| Variable | Used by | Purpose |
|---|---|---|
| `FRONTEND_PORT` | frontend | Host port the application is published on (default `8282`) |
| `POSTGRES_DB` | database, backend | Name of the Postgres database |
| `POSTGRES_USER` | database, backend | Postgres username |
| `POSTGRES_PASSWORD` | database, backend | Postgres password |
| `POSTGRES_HOST` | backend | Hostname of the database service (default `database`, i.e. the Compose service name — only change this if connecting to an external database) |
| `POSTGRES_PORT` | backend | Port the database listens on (default `5432`) |
| `DJANGO_ALLOWED_HOSTS` | backend | Comma-separated list of hostnames/IPs Django will accept requests for |
| `DJANGO_SUPERUSER_USERNAME` | backend | Username for the automatically created Django admin superuser |
| `DJANGO_SUPERUSER_EMAIL` | backend | Email for the automatically created Django admin superuser |
| `DJANGO_SUPERUSER_PASSWORD` | backend | Password for the automatically created Django admin superuser — see [Superuser Creation](#superuser-creation) below, this variable has special behavior |

> [!WARNING]
> Avoid special characters such as `$`, `&`, or `*` in `POSTGRES_PASSWORD` or `DJANGO_SUPERUSER_PASSWORD`. Docker Compose interprets `$` as the start of a variable substitution, which can silently produce a different password than the one you intended (leading to authentication failures that are hard to diagnose). Stick to alphanumeric characters for these values.

### Configuration & Customization

- **Changing the exposed port:** The frontend is published on port `8282` by default. This is configurable via the `FRONTEND_PORT` variable in `.env` — change it and recreate the frontend container:
  ```bash
  docker compose up -d --force-recreate frontend
  ```
- **Allowing a different host/IP:** If you deploy this to a different server, update `DJANGO_ALLOWED_HOSTS` in `.env` to include that server's IP address or domain name, then recreate the backend container:
  ```bash
  docker compose up -d --force-recreate backend
  ```
- **Database credentials:** Changing `POSTGRES_USER`, `POSTGRES_PASSWORD`, or `POSTGRES_DB` in `.env` only takes effect on a **fresh** database volume. If the `db-data` volume already exists, either remove it (`docker compose down -v`, which deletes all data) or manually update the credentials inside the running Postgres container.
- **Rebuilding after code changes:** Since the backend and frontend are built from local Dockerfiles, any code change requires a rebuild:
  ```bash
  docker compose build backend frontend
  docker compose up -d
  ```

### Superuser Creation

A Django admin superuser is created **automatically** on container startup, based on the `DJANGO_SUPERUSER_USERNAME`, `DJANGO_SUPERUSER_EMAIL`, and `DJANGO_SUPERUSER_PASSWORD` values in `.env` — no manual command required.

> [!IMPORTANT]
> The custom `UserManager.create_superuser()` method in this codebase (`conduit/apps/authentication/models.py`) always sets the password from the `DJANGO_SUPERUSER_PASSWORD` environment variable (if set and at least 4 characters long), falling back to the hardcoded password `securepass` otherwise. This is pre-existing application behavior, not something introduced by containerization.

> [!TIP]
> Login to the Django admin requires the **email address**, not the username, since the custom `User` model uses email as its `USERNAME_FIELD`.

If you ever need to create an **additional** superuser manually:

```bash
docker compose exec backend python manage.py createsuperuser
```

### Static Files

Backend static files (e.g. Django REST Framework's browsable API assets, admin panel CSS) are collected into a shared Docker volume (`backend-static`) at container startup and served by nginx under `/static/`. This is why the admin panel and browsable API render with correct styling even though Django itself doesn't serve static files in production.

## Testing the Setup

Before considering the deployment complete, the following was verified:

- The frontend is reachable at `http://<host-ip>:8282`.
- The backend runs via Gunicorn (a WSGI server), **not** Django's development server (`manage.py runserver`).
- Navigating through the app (articles, tags, profiles) loads data correctly from the API.
- Creating, favoriting, and viewing articles works correctly.
- The Django admin panel is reachable and the superuser can log in.
- Containers automatically restart after an internal crash, thanks to `restart: unless-stopped`.

> [!NOTE]
> This restart policy does **not** trigger on a manual `docker compose kill`/`stop`, since Docker treats that as an intentional stop, not a crash. A true crash (e.g. the main process inside the container dying unexpectedly) does trigger an automatic restart.
- Logs can be inspected via the CLI and optionally saved to a file for later use:
  ```bash
  docker compose logs backend > backend-logs.txt
  ```

## Logs

To view live logs for a specific service:

```bash
docker compose logs -f backend
```

To persist the current logs of a container to a file:

```bash
docker logs <container-name> > my-container-logs.txt
```

## Known Limitations

- **psycopg2 version pin:** `psycopg2-binary` is pinned to `2.8.6` instead of a newer release. Versions `>= 2.9` introduced a change in how timezone offsets are returned, which is incompatible with this project's older Django version and causes Django admin pages to fail with a database-timezone assertion error at runtime, even though the database itself is correctly configured for UTC.
- **Legacy dependency versions:** This project intentionally runs on older versions of Python, Django, and related packages to match the original (pre-Docker) codebase. As a result, some dependencies may carry known CVEs. This is a tradeoff made to keep the original application runnable rather than rewriting it against current dependency versions.
- **Chrome address bar autocomplete on `/admin`:** Typing `/admin` (without a trailing slash) directly into Chrome's address bar can trigger Chrome's own autocomplete behavior before the request is even sent, which may not reflect the server's actual (correct) redirect behavior. This is a browser-specific quirk, not a server misconfiguration — verified via `curl`, the server always returns a relative redirect to `/admin/`. Use the full path with a trailing slash (`/admin/`) to avoid this.
- **Article deletion returns 405:** Deleting an article (even as its own author) currently returns `405 Method Not Allowed`. This appears to be pre-existing backend behavior unrelated to containerization and is out of scope for this project, which focuses on the Docker/Compose setup rather than application-level bugfixing.
- **Article deletion returns 405:** Deleting an article (even as its own author) currently returns `405 Method Not Allowed`. This appears to be pre-existing backend behavior unrelated to containerization and is out of scope for this project (which focuses on the Docker/Compose setup, not application-level bugfixing).

> [!CAUTION]
> As described in [Superuser Creation](#superuser-creation), if `DJANGO_SUPERUSER_PASSWORD` is not set (or too short), Django silently falls back to a hardcoded default password (`securepass`) for any superuser created via `createsuperuser`. Always set this variable explicitly in your `.env`.

## Notable Fixes

A few non-obvious issues were found and fixed while containerizing this project:

- **Article publishing failed (404):** The frontend sent `POST /articles/` with a trailing slash, but the backend router uses `trailing_slash=False`. Fixed by removing the trailing slash in `articles.service.ts`.
- **Admin panel redirected to the wrong port:** Behind the reverse proxy, `/admin` (no trailing slash) triggered a 301 redirect using nginx's internal port (`8080`) instead of the public port (`8282`). Fixed with `absolute_redirect off;` in `nginx.conf`, so nginx returns a relative redirect instead.
- **`entrypoint.sh` had Windows (CRLF) line endings:** Caused the container to fail on startup (`set: Illegal option -`). Fixed with a `.gitattributes` file (`*.sh text eol=lf`) enforcing Unix line endings for shell scripts.
