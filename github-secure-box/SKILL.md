---
name: github-secure-box
description: Transform a GitHub repository URL into a safe, containerized local environment with security auditing first. Use when Codex is given a GitHub URL or repository and must clone it, inspect it without host-level package installs, run containerized secret scanning, SCA, static analysis, and outbound network review, then generate or harden Dockerfiles, docker-compose.yml, .env handling, and a Secure-Box Makefile.
---

# GitHub Secure Box

## Overview

Turn a GitHub repository into a local Docker-first sandbox without installing project dependencies on the host. Audit first, then generate or harden container assets, then produce a `Makefile` that manages the environment lifecycle and cleanup.

## Operating Rules

- Keep the host clean. Do not recommend `npm install`, `pnpm install`, `yarn install`, `pip install`, `bundle install`, `go install`, or any other host-level dependency setup.
- Prefer `docker run`, `docker build`, and `docker compose` for every audit and runtime action.
- Treat outbound network behavior as a security control, not a convenience feature.
- If a repo shows suspicious or unexplained external calls, warn the user and tighten runtime networking with `network_mode: "none"` or a strictly scoped internal bridge.
- Preserve user changes in the repo. Add or update generated files without reverting unrelated work.

## Workflow

### 1. Acquire the Repository

- If the user provides only a GitHub URL, clone it locally with Git.
- Work inside the cloned repository or an existing local checkout.
- Inspect files that reveal the stack and services:
  - `Dockerfile*`
  - `docker-compose*.yml`
  - `package.json`
  - `requirements.txt`, `pyproject.toml`, `Pipfile`
  - `go.mod`
  - `Gemfile`
  - `Cargo.toml`
  - `.env.example`
  - infrastructure files such as `helm`, `k8s`, `terraform`, or `compose` manifests

### 2. Run the Containerized Audit First

Run the audit in this order and keep it containerized. Use the command templates in [references/dockerized-audit.md](references/dockerized-audit.md).

1. Secret scanning with TruffleHog.
2. Filesystem SCA with Trivy.
3. Static analysis with Semgrep using `p/security-audit`.
4. Network analysis by reviewing source for:
   - hardcoded IPv4 and IPv6 addresses
   - `fetch`, `axios`, `XMLHttpRequest`, `requests`, `httpx`, `urllib`, `net`, `dns`, `socket`, `grpc`, `WebSocket`
   - domains outside known first-party endpoints
   - DNS beaconing, telemetry, exfiltration, or callback logic

Report findings before generating infrastructure. If scans fail because the tool image or flags need adjustment, keep the host clean and fix the container command instead of switching to local installs.

### 3. Decide the Runtime Shape

Infer required services from the repository:

- Add Redis if `package.json`, Python dependencies, or config mention `redis`.
- Add Postgres if dependencies, env vars, or config mention `postgres`, `postgresql`, or `psycopg`.
- Add MinIO for S3-compatible local storage if the app expects S3 or object storage.
- Add RabbitMQ when the code or env references AMQP, RabbitMQ, or queue workers.
- Add only the supporting services the app actually needs.

Use an internal bridge network by default. If the code appears to call untrusted hosts, restrict the application container further and document the reason.

### 4. Generate or Harden Dockerfiles

Create Dockerfiles when missing. Harden existing ones when insecure.

Enforce these rules:

- Use multi-stage builds.
- Prefer Alpine or Distroless runtime images when practical.
- Set a non-root `USER`.
- Minimize the runtime image to production artifacts only.
- Copy only required files.
- Avoid package manager caches and unnecessary shells in the final stage.
- Keep secrets out of image layers.

If the project has multiple services, split Dockerfiles by service only when the repo structure clearly supports it.

### 5. Generate or Harden `docker-compose.yml`

Ensure Compose includes:

- the application service
- detected supporting services such as Redis, Postgres, MinIO, RabbitMQ
- an internal bridge network
- sensible named volumes for stateful services
- `env_file` wiring or equivalent so `.env.example` becomes a generated local `.env`

For `.env` handling:

- If `.env.example` exists and `.env` is missing, create `.env` from `.env.example`.
- Replace obvious placeholders only when the correct local value is clear; otherwise keep safe defaults and mark them for the user.
- Never commit real secrets.

If outbound activity is suspicious:

- Prefer `network_mode: "none"` for the app service when it can still function for static analysis or local-only execution.
- Otherwise keep the service on a dedicated internal network and expose only the minimum required ports.

### 6. Generate the Secure-Box `Makefile`

Create a `Makefile` with these mandatory targets:

- `up`: build and start all services in detached mode
- `down`: stop services
- `clean`: remove containers, volumes, and project-specific images
- `nuke`: run `clean`, then delete the local project directory

Treat `nuke` as destructive:

- Implement it only when explicitly requested or clearly appropriate for the local sandbox use case.
- Call out that it deletes the checked-out local directory.
- Scope deletion to the current project directory only.

### 7. Summarize the Result

Provide:

- audit findings by severity
- generated or changed infrastructure files
- detected services and why they were added
- any network restrictions that were applied
- any remaining manual decisions, such as real secrets or external service credentials

## Decision Rules

### Use stricter networking when:

- unknown domains appear in source or config
- telemetry or analytics endpoints are unexplained
- callback URLs are assembled dynamically
- DNS libraries or raw socket calls appear outside expected infrastructure code

### Keep networking normal when:

- external calls are limited to clearly legitimate first-party APIs the app requires
- local development genuinely needs service-to-service networking inside Compose

## References

- Use [references/dockerized-audit.md](references/dockerized-audit.md) for command templates and search patterns.
- Use [references/security-sources.md](references/security-sources.md) for the mandatory documentation URLs that informed this skill.
