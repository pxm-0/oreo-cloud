# Codex Instructions for Oreo Cloud

This project manages private self-hosted infrastructure.

## Hard Rules

- Do not expose services publicly by default.
- Do not enable Tailscale Funnel.
- Do not start Cloudflare tunnels in P0.
- Do not create DNS records in P0.
- Do not open router ports.
- Do not expose PostgreSQL.
- Do not expose Docker socket.
- Do not expose the control API publicly.
- Do not commit secrets, tokens, `.env`, runtime files, workload source, or backups.
- Prefer read-only discovery before changes.
- Back up config files before editing.
- Validate Caddy before reload.
- Keep scripts idempotent.
- Preserve Docker Compose project names during migration.

## Project Model

Workloads are agnostic by default.

- `workloads.json` describes app/runtime identity.
- `privacy.json` stores classification.
- `access.json` stores desired/effective access.
- `policy.json` stores guardrails.
- `exposure.json` stores provider capability/planning.

## Coding Style

- Use Python standard library where practical.
- Do not require `jq`.
- Do not print secrets.
- Shell scripts should use safe defaults.
- JSON files must validate with `python3 -m json.tool`.
