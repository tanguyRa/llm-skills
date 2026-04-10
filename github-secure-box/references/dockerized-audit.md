# Dockerized Audit

Use these patterns to keep analysis containerized and avoid host-level dependency installs.

## Mounting Pattern

- Mount the repository read-only when a scan does not need write access.
- Set the container working directory to the mounted repository path.
- Prefer explicit image tags when reproducibility matters.

Generic pattern:

```bash
docker run --rm -v "$PWD:/repo:ro" -w /repo <image> <args>
```

## Secret Scanning

Use TruffleHog first.

Example:

```bash
docker run --rm -v "$PWD:/repo:ro" ghcr.io/trufflesecurity/trufflehog:latest filesystem /repo --only-verified
```

If verified-only is too narrow for the repo, rerun without that flag and separate likely secrets from low-signal matches.

## SCA

Use Trivy against the filesystem.

Example:

```bash
docker run --rm -v "$PWD:/repo:ro" aquasec/trivy:latest fs /repo
```

Focus on vulnerable dependencies, lockfiles, OS packages in Docker contexts, and misconfiguration hints that affect the generated container setup.

## Static Analysis

Use Semgrep with the security audit ruleset.

Example:

```bash
docker run --rm -v "$PWD:/repo:ro" semgrep/semgrep:latest semgrep --config p/security-audit /repo
```

## Network Analysis Search Patterns

Search source and config for:

- hardcoded IPs:
  - `([0-9]{1,3}\.){3}[0-9]{1,3}`
  - common IPv6 literals
- HTTP clients and outbound calls:
  - `fetch\(`
  - `axios\.`
  - `requests\.`
  - `httpx\.`
  - `urllib`
  - `XMLHttpRequest`
  - `WebSocket`
  - `grpc`
- lower-level networking:
  - `socket`
  - `dns`
  - `dgram`
  - `net\.`
- suspicious behavior:
  - dynamic domain construction
  - base64-decoded URLs
  - retry loops to unknown hosts
  - beacon, telemetry, analytics, exfiltration, callback, webhook logic

Prefer `rg` for local code search and summarize whether each outbound destination is first-party, third-party but expected, or unexplained.

## Compose Hardening Notes

- Use an internal bridge network for app-to-service traffic.
- Expose only ports required for local development.
- If suspicious outbound traffic is present, prefer `network_mode: "none"` where feasible.
- Keep stateful services on named volumes.
- Wire `.env` from `.env.example` and keep secrets local.
