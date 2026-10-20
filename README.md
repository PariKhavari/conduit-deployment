# Conduit Deployment

Automated CI/CD deployment of the Conduit full-stack application (Django backend, Angular frontend, PostgreSQL) to a remote VM using GitHub Actions, GitHub Container Registry (GHCR) and Docker Compose.

> [!NOTE]
> This guide covers the automated deployment. For running the containers manually, see [README.md](README.md).

## Table of Contents

- [Summary](#summary)
- [Description](#description)
  - [Repository Structure](#repository-structure)
  - [How the Pipeline Works](#how-the-pipeline-works)
- [Quickstart](#quickstart)
  - [Prerequisites](#prerequisites)
  - [Quickstart Guide](#quickstart-guide)
- [Usage](#usage)
  - [1. Get Your Own Copy of the Repository](#1-get-your-own-copy-of-the-repository)
  - [2. Prepare the VM](#2-prepare-the-vm)
  - [3. Set Up SSH Access for GitHub Actions](#3-set-up-ssh-access-for-github-actions)
  - [4. Create the Environment File](#4-create-the-environment-file)
  - [5. Add GitHub Secrets](#5-add-github-secrets)
  - [6. Trigger the Deployment](#6-trigger-the-deployment)
  - [7. Verify the Deployment](#7-verify-the-deployment)
  - [8. Configuration Reference](#8-configuration-reference)
  - [9. Customizing the Setup](#9-customizing-the-setup)
  - [10. Operating the Containers](#10-operating-the-containers)
  - [11. Logs](#11-logs)
  - [12. Rollback](#12-rollback)
  - [13. Troubleshooting](#13-troubleshooting)
- [Security Notes](#security-notes)
- [Extras](#extras)

---

## Summary

| What | How |
|------|-----|
| Build | GitHub Actions builds the backend and frontend images on a GitHub runner, never on the VM |
| Registry | Images are pushed to GHCR, tagged with the commit SHA and `latest` |
| Deploy | The workflow connects to the VM via SSH, copies `docker-compose.yaml` and `.env`, pulls the new images and starts them with `docker compose up -d` |
| Trigger | Every push to `main`, or manually via **Run workflow** |
| Result | The frontend is available at `http://<VM_IP>:8282` |
| Failure | Any failing step stops the workflow with an error (`set -euo pipefail`) |

---

## Description

This repository contains a containerized version of the Conduit application and a GitHub Actions workflow that automates the full release process: build, push, deploy. It is meant as a reproducible template for deploying a multi-container application to a single VM without building anything on the server.

### Repository Structure

```
.
├── .github/
│   └── workflows/
│       └── deployment.yaml     # CI/CD workflow (build + deploy jobs)
├── conduit-backend/            # Django REST API
│   ├── Dockerfile              # Multi-stage build based on python:3.6-slim-buster
│   ├── .dockerignore
│   └── entrypoint.sh           # Prepares the Django app before Gunicorn starts
├── conduit-frontend/           # Angular client
│   ├── Dockerfile              # Multi-stage build: node:20-alpine -> nginx-unprivileged
│   ├── .dockerignore
│   └── nginx.conf
├── docker-compose.yaml         # Services: database, backend, frontend
├── .env.example                # Template for the runtime configuration
├── .gitattributes              # Enforces LF line endings for shell scripts
├── .gitignore
├── DEPLOYMENT.md               # This guide (automated CI/CD deployment)
└── README.md                   # Guide for running the containers
```

### How the Pipeline Works

```
 push to main ──► build job (GitHub runner) ──► GHCR ──► deploy job ──SSH──► VM
                  - build backend image                  - copy compose + .env
                  - build frontend image                 - docker compose pull
                  - push :<sha> and :latest              - docker compose up -d
```

1. **Build job:** checks out the code, logs in to GHCR with the automatic `GITHUB_TOKEN`, builds both images and pushes them.
2. **Deploy job:** starts only if the build job succeeded (`needs: build`). It loads the SSH key into an `ssh-agent` ([webfactory/ssh-agent](https://github.com/marketplace/actions/webfactory-ssh-agent)), adds the VM to `known_hosts`, copies the Compose file and the `.env` file to the VM via `scp`/`ssh`, logs in to GHCR on the VM, pulls the images for the current commit SHA and starts the containers in detached mode. Finally it removes unused images on the VM.

| Deploy step | Purpose |
|-------------|---------|
| Set up SSH agent | Loads `SSH_PRIVATE_KEY` into memory; no key file is written to the runner |
| Add VM to known hosts | Writes `SSH_KNOWN_HOSTS` so the VM identity is verified |
| Copy compose file and .env to VM | Creates `DEPLOY_DIR`, copies `docker-compose.yaml` via `scp` and writes `.env` with permissions `600` |
| Pull images and start containers | Logs in to GHCR, `docker compose pull`, `docker compose up -d`, logs out, prunes old images |

> [!NOTE]
> The VM only pulls and runs ready-made images. All builds happen on GitHub runners.

---

## Quickstart

### Prerequisites

- GitHub account with your own fork of this repository (Actions and GHCR enabled)
- Linux VM reachable via SSH with **Docker** and the **Docker Compose plugin**
- Port `8282` open in the VM firewall
- SSH key pair **without passphrase** (public key on the VM, private key in GitHub)

### Quickstart Guide

1. Fork this repository on GitHub and clone your fork:

   ```bash
   git clone https://github.com/PariKhavari/conduit-deployment.git
   cd conduit-deployment
   ```

2. Create a deploy key and install it on the VM:

   ```bash
   ssh-keygen -t ed25519 -f ~/.ssh/conduit_deploy -C "github-actions-deploy"
   ssh-copy-id -i ~/.ssh/conduit_deploy.pub <USER>@<VM_IP>
   ```

3. Read the VM host key for the `SSH_KNOWN_HOSTS` secret:

   ```bash
   ssh-keyscan <VM_IP>
   ```

4. Create your runtime configuration and edit the values:

   ```bash
   cp .env.example .env
   ```

5. Add the secrets `SSH_HOST`, `SSH_USER`, `SSH_PRIVATE_KEY`, `SSH_KNOWN_HOSTS` and `ENV_FILE` in GitHub (see [5. Add GitHub Secrets](#5-add-github-secrets)).
6. Push to `main` or click **Actions → Build and Deploy → Run workflow**.
7. Open `http://<VM_IP>:8282`.

---

## Usage

The following sections describe the complete setup in detail. Follow steps 1 to 7 in order when setting up a new VM. Sections 8 to 13 are reference material for configuration and operation.

### 1. Get Your Own Copy of the Repository

#### 1.1 Fork the repository

Click **Fork** at the top right of the repository page on GitHub.

#### 1.2 Clone your fork

```bash
git clone https://github.com/PariKhavari/conduit-deployment.git
cd conduit-deployment
```

> [!IMPORTANT]
> A plain clone of someone else's repository is not enough. Secrets, Actions runs and GHCR images belong to a GitHub repository, so you need your **own** repository (fork) to deploy.

### 2. Prepare the VM

#### 2.1 Install Docker and the Compose plugin

```bash
curl -fsSL https://get.docker.com | sh
```

#### 2.2 Allow your user to run Docker without `sudo`

```bash
sudo usermod -aG docker ${USER}
```

Log out and in again afterwards.

#### 2.3 Check the installation

```bash
docker --version
docker compose version
```

> [!IMPORTANT]
> Port `8282` must be open in the firewall of your VM or cloud provider, otherwise the frontend is not reachable from the internet.

### 3. Set Up SSH Access for GitHub Actions

GitHub Actions logs in to the VM like a user with an SSH key.

> [!WARNING]
> Password login is not possible in the pipeline. The key must be created **without a passphrase**, otherwise the deploy job fails with `Permission denied`.

#### 3.1 Create a key pair on your local machine

Press Enter twice when asked for a passphrase:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/conduit_deploy -C "github-actions-deploy"
```

#### 3.2 Copy the public key to the VM

```bash
ssh-copy-id -i ~/.ssh/conduit_deploy.pub <USER>@<VM_IP>
```

> [!TIP]
> Without `ssh-copy-id` (e.g. on Windows), log in to the VM and append the content of `conduit_deploy.pub` manually:
>
> ```bash
> mkdir -p ~/.ssh && chmod 700 ~/.ssh
> echo "<CONTENT_OF_PUBLIC_KEY>" >> ~/.ssh/authorized_keys
> chmod 600 ~/.ssh/authorized_keys
> ```

#### 3.3 Test the login

It must work without a password:

```bash
ssh -i ~/.ssh/conduit_deploy <USER>@<VM_IP>
```

#### 3.4 Read the host key of the VM

Needed for the `SSH_KNOWN_HOSTS` secret:

```bash
ssh-keyscan <VM_IP>
```

#### 3.5 Overview: where each key goes

| Key | Where it goes |
|-----|---------------|
| Public key (`conduit_deploy.pub`) | VM: `~/.ssh/authorized_keys` of `SSH_USER` |
| Private key (`conduit_deploy`) | GitHub secret `SSH_PRIVATE_KEY` |
| Host key (output of `ssh-keyscan`) | GitHub secret `SSH_KNOWN_HOSTS` |

> [!CAUTION]
> Never commit the private key to the repository and never share it. It grants full access to the VM.

### 4. Create the Environment File

#### 4.1 Copy the template

The file is never committed (`.gitignore`).

```bash
cp .env.example .env
```

#### 4.2 Fill in your values

See [8.1 Environment Variables](#81-environment-variables-env) for all variables.

> [!IMPORTANT]
> Add the VM IP to `DJANGO_ALLOWED_HOSTS` and `CORS_ORIGIN_WHITELIST` (e.g. `http://<VM_IP>:8282`), otherwise the frontend cannot load data.

> [!WARNING]
> Avoid `$`, `&` and `*` in passwords. They can break Docker Compose variable interpolation.

#### 4.3 Generate a Django secret key

```bash
python -c "import secrets; print(secrets.token_urlsafe(50))"
```

### 5. Add GitHub Secrets

#### 5.1 Open the secrets page

In your repository: **Settings → Secrets and variables → Actions → New repository secret**

#### 5.2 Create the following secrets

| Secret | Value |
|--------|-------|
| `SSH_HOST` | IP address or hostname of the VM (same as used for `ssh-keyscan`) |
| `SSH_USER` | User on the VM that owns the public key |
| `SSH_PRIVATE_KEY` | Full content of the private key, including the `-----BEGIN ...-----` and `-----END ...-----` lines |
| `SSH_KNOWN_HOSTS` | Complete output of `ssh-keyscan <VM_IP>` |
| `ENV_FILE` | Full content of your `.env` file |

> [!IMPORTANT]
> Paste the **content** of the private key, not its file name or path. GitHub has no access to your local machine.

> [!NOTE]
> `GITHUB_TOKEN` is created automatically by GitHub for every run and does not need to be added.

### 6. Trigger the Deployment

#### 6.1 Start the workflow

- **Automatic:** push or merge into `main`.
- **Manual:** **Actions → Build and Deploy → Run workflow**.

> [!NOTE]
> The **Run workflow** button only appears once the workflow file exists on `main`.

#### 6.2 Follow the run

Open the **Actions** tab. First the `build` job, then the `deploy` job must turn green.

### 7. Verify the Deployment

| Check | Command / Action |
|-------|------------------|
| Frontend reachable | Open `http://<VM_IP>:8282` in the browser |
| Data is loaded | Navigate through articles, profiles and tags |
| Containers running | `docker compose ps` on the VM (in `~/conduit`) |
| Auto restart works | `sudo kill -9 $(docker inspect -f '{{.State.Pid}}' <CONTAINER_NAME>)`, then `docker compose ps` again |
| Logs available | `docker compose logs -f backend` |

### 8. Configuration Reference

#### 8.1 Environment Variables (`.env`)

| Variable | Description | Default / Example |
|----------|-------------|-------------------|
| `POSTGRES_DB` | Database name | `conduit` |
| `POSTGRES_USER` | Database user | `conduit_user` |
| `POSTGRES_PASSWORD` | Database password | `change_me` |
| `POSTGRES_HOST` | Database host (Compose service name) | `database` |
| `POSTGRES_PORT` | Database port | `5432` |
| `DJANGO_SECRET_KEY` | Django secret key | `change_me` |
| `DJANGO_DEBUG` | Debug mode | `false` |
| `DJANGO_ALLOWED_HOSTS` | Allowed hostnames | `localhost,127.0.0.1,<VM_IP>` |
| `DJANGO_SUPERUSER_USERNAME` | Admin username | `admin` |
| `DJANGO_SUPERUSER_EMAIL` | Admin email | `admin@example.com` |
| `DJANGO_SUPERUSER_PASSWORD` | Admin password | `change_me` |
| `CORS_ORIGIN_ALLOW_ALL` | Allow all origins | `false` |
| `CORS_ORIGIN_WHITELIST` | Allowed frontend origins | `http://<VM_IP>:8282` |
| `FRONTEND_PORT` | Public port of the frontend | `8282` |
| `GHCR_OWNER` | Lowercase GitHub owner of the images | `<github-user>` |
| `IMAGE_TAG` | Image version to run | `latest` |

#### 8.2 Workflow (`.github/workflows/deployment.yaml`)

| Setting | Meaning | Default |
|---------|---------|---------|
| `on.push.branches` | Branches that trigger a deployment | `main` |
| `REGISTRY` | Container registry | `ghcr.io` |
| `DEPLOY_DIR` | Target folder on the VM (in the home of `SSH_USER`) | `conduit` |
| `IMAGE_TAG` | Version deployed to the VM | `${{ github.sha }}` |

#### 8.3 Services (`docker-compose.yaml`)

| Service | Image | Port | Volume |
|---------|-------|------|--------|
| `database` | `postgres:15-alpine` | internal | `db-data` |
| `backend` | `ghcr.io/${GHCR_OWNER}/conduit-backend:${IMAGE_TAG:-latest}` | `8000` (internal) | `backend-static` |
| `frontend` | `ghcr.io/${GHCR_OWNER}/conduit-frontend:${IMAGE_TAG:-latest}` | `${FRONTEND_PORT:-8282}` → `8080` | `backend-static` (read-only) |

> [!NOTE]
> All services use `restart: unless-stopped`. The backend waits until the database health check (`pg_isready`) succeeds.

### 9. Customizing the Setup

| Goal | Change |
|------|--------|
| Use another public port | Set `FRONTEND_PORT` in `ENV_FILE` and update `CORS_ORIGIN_WHITELIST` |
| Deploy to a new VM | Update `SSH_HOST`, `SSH_KNOWN_HOSTS`, the VM IP in `ENV_FILE` and install the public key on the new VM |
| Deploy from another branch (e.g. for testing) | Add the branch to `on.push.branches` |
| Change the deploy folder | Change `DEPLOY_DIR` in the workflow |
| Change the number of Gunicorn workers | Edit `--workers 3` in the `CMD` of `conduit-backend/Dockerfile` |
| Use another registry | Change `REGISTRY` and the login step in the workflow, and the image names in `docker-compose.yaml` |

> [!TIP]
> If you add a test branch to `on.push.branches`, remove it again before merging into `main`, so that only `main` deploys to the VM.

### 10. Operating the Containers

On the VM:

```bash
cd ~/conduit
docker compose ps                # status of all services
docker compose restart backend   # restart one service
docker compose down              # stop all containers (data in volumes is kept)
docker compose up -d             # start all containers in detached mode
```

> [!CAUTION]
> `docker compose down -v` also deletes the volumes, including the database. Use it only if you really want to reset all data.

### 11. Logs

> [!NOTE]
> The **Actions logs** show whether the deployment succeeded. The **container logs** on the VM show whether the application works at runtime (e.g. database errors, CORS errors, crashes).

#### 11.1 View logs

```bash
docker compose logs                       # all services
docker compose logs -f backend            # follow backend logs live
docker compose logs --tail=100 frontend   # last 100 lines of the frontend
```

#### 11.2 Save logs to a file

```bash
docker compose logs --no-color > conduit-logs-$(date +%F).txt
```

### 12. Rollback

Every build stays in GHCR under its commit SHA. To run an older version:

```bash
cd ~/conduit
IMAGE_TAG=<COMMIT_SHA> docker compose pull
IMAGE_TAG=<COMMIT_SHA> docker compose up -d
```

> [!TIP]
> You find the commit SHA of every deployment in the **Actions** tab or with `git log --oneline`.

### 13. Troubleshooting

| Error | Cause | Solution |
|-------|-------|----------|
| `Host key verification failed` | `SSH_KNOWN_HOSTS` belongs to another VM or hostname | Run `ssh-keyscan <VM_IP>` again and replace the secret |
| `Permission denied (publickey,password)` | Private key in the secret does not match the public key on the VM, key has a passphrase, or wrong `SSH_USER` | Store the matching private key without passphrase, check `authorized_keys` of `SSH_USER` |
| Frontend loads but shows no data | VM IP missing in `DJANGO_ALLOWED_HOSTS` or `CORS_ORIGIN_WHITELIST` | Update `ENV_FILE` and re-run the workflow |
| `docker: command not found` in deploy job | Docker not installed on the VM | See [2. Prepare the VM](#2-prepare-the-vm) |
| `/bin/sh^M: not found` in backend container | `entrypoint.sh` saved with Windows line endings (CRLF) | Keep `.gitattributes` (`*.sh text eol=lf`) and re-commit the file |
| **Run workflow** button missing | Workflow file is not on `main` yet | Merge the workflow into `main` or push to a branch listed in `on.push.branches` |

> [!TIP]
> After fixing a secret, use **Re-run failed jobs** in the Actions tab. The build job does not need to run again.

---

## Security Notes

> [!CAUTION]
> Never store SSH keys, passwords, tokens, usernames or IP addresses in the repository.

- All sensitive values are GitHub secrets.
- `.env` is ignored by Git; only `.env.example` with placeholders is committed.
- The `.env` file on the VM is written with restricted permissions (`umask 077`).
- Secrets are passed to shell steps via `env:` instead of inline expressions to prevent script injection.
- The SSH private key is only held in memory by `ssh-agent` and never written to the runner's disk; the agent is stopped automatically after the job.
- The host identity of the VM is verified via `known_hosts` (`SSH_KNOWN_HOSTS`).
- The deploy job only has read permissions for packages; only the build job may push to GHCR.
- The backend runs as a non-root user, the frontend uses the unprivileged Nginx image.

---

## Extras

- Separate Docker networks: the database is only reachable from the backend (`backend-net`).
- Health checks for the database and the frontend container.
- Images are tagged with the commit SHA, which allows exact deployments and easy rollbacks.
- Unused images are removed on the VM after each deployment (`docker image prune -f`).
- SSH key handling via the Marketplace action `webfactory/ssh-agent` instead of a self-written key file setup.
- `.gitattributes` enforces LF line endings for shell scripts, so the containers also start correctly when developing on Windows.
