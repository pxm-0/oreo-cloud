# Oreo Cloud P0 Package

This package is the P0 handoff for building **Oreo Cloud**: a private, Git-tracked, personal ECS-style control plane for `oreochiserver`.

P0 is not a public hosting platform. P0 is the minimum safe foundation:

- one private dashboard reachable from phone/laptop over Tailscale
- Git-tracked platform configuration
- neutral workload registry
- dashboard-visible privacy and access states
- authenticated dashboard toggles for privacy and desired access
- read-only workload health/status commands
- btop-style monitoring panel behind a dashboard toggle
- workload migration planning into `/srv/oreo-cloud/workloads/<id>`
- Cloudflare exposure planning, disabled by default
- no public exposure unless a future task explicitly enables it

## Document Map

| File | Purpose |
|---|---|
| `docs/PRD.md` | Product requirements for P0 |
| `docs/ARCHITECTURE.md` | System architecture and trust boundaries |
| `docs/IMPLEMENTATION_PLAN.md` | Step-by-step build plan |
| `docs/CONFIG_MODEL.md` | Workload/access/privacy/policy schema model |
| `docs/DASHBOARD_AND_API.md` | Dashboard behavior and local control API spec |
| `docs/MONITORING.md` | btop-style metrics collector and UI spec |
| `docs/MIGRATION.md` | Safe workload move plan |
| `docs/CLOUDFLARE.md` | Cloudflare provisioning model, disabled by default |
| `docs/SECURITY.md` | Safety rules, token handling, exposure guardrails |
| `docs/OPERATIONS.md` | Commands and operator workflow |
| `docs/CODEX_TASKS.md` | Task prompts to paste into Codex |
| `docs/ACCEPTANCE.md` | P0 definition of done and smoke tests |
| `docs/REFERENCES.md` | External references used for implementation assumptions |
| `templates/` | Starter config, systemd, Caddy, Cloudflare, and script templates |

## P0 North Star

From a phone on Tailscale, open:

```text
http://oreochiserver:8088
```

See all workloads, health, routes, privacy state, desired access state, effective access state, and a toggleable monitor panel.

Admin mode can change privacy and desired access, but P0 should only apply safe states automatically. Cloudflare exposure remains planned/provisioned, not live, unless explicitly enabled in a later phase.


---

# PRD: Oreo Cloud P0

## Product Summary

Oreo Cloud is a private personal ECS-style server control plane. It organizes local Docker/Docker Compose workloads, tracks them in Git-backed configuration, exposes a private dashboard over Tailscale, and provides safe controls for visibility and access state.

P0 focuses on the platform foundation, not full deployment automation.

## User Problem

The server currently risks becoming a pile of ports, folders, and mental notes. The owner wants a single access point from phone/laptop anywhere, a clear model for workloads, and a path toward private/public workload access without accidentally exposing sensitive services.

## Target User

Primary user: the server owner/admin.

Secondary users: future collaborators or testers who may access specific workloads through a controlled access method, such as Tailscale or Cloudflare Access.

## Goals

1. Create `/srv/oreo-cloud` as the canonical platform root.
2. Track platform configuration and docs in Git.
3. Model workloads as agnostic units by default.
4. Separate workload identity from privacy and access policy.
5. Provide a private dashboard accessible from phone/laptop over Tailscale.
6. Provide dashboard toggles for privacy and desired access in authenticated admin mode.
7. Keep desired access separate from effective access.
8. Add a btop-style monitoring panel hidden behind a toggle.
9. Provide safe workload migration planning into `/srv/oreo-cloud/workloads/<id>/source`.
10. Provision Cloudflare access planning without enabling public exposure by default.

## Non-Goals for P0

P0 will not implement:

- Kubernetes
- full CI/CD
- one-click browser deploys
- browser terminal
- public dashboard
- automatic Cloudflare tunnel activation
- automatic DNS creation
- automatic destructive workload migration
- public database access
- direct Docker socket exposure
- multi-user RBAC

## Key Concept: Workloads Are Agnostic

A workload should not be permanently classified as company, demo, private, or public inside `workloads.json`.

Instead:

- `workloads.json` describes what the app is and how it runs.
- `privacy.json` stores current privacy classification.
- `access.json` stores desired and effective access states.
- `policy.json` stores guardrails.
- `exposure.json` stores provider capability/planning.

This lets the dashboard toggle privacy and access without changing the workload identity model.

## P0 Access States

```text
none
local
tailnet
cloudflare-protected
cloudflare-public
```

Definitions:

- `none`: no managed access.
- `local`: server-local URL only, usually `127.0.0.1`.
- `tailnet`: private access through Tailscale.
- `cloudflare-protected`: externally reachable but requires Cloudflare Access or equivalent auth.
- `cloudflare-public`: externally reachable without tailnet access or auth.

## P0 Privacy States

```text
unclassified
personal
internal
sensitive
restricted
```

Privacy does not equal access. A workload may be `sensitive` and still have `tailnet` access. A workload may be `unclassified` and still be kept local until deliberately exposed.

## Functional Requirements

### Workload Registry

The system must maintain a neutral workload registry with runtime, health, and paths.

### Privacy Registry

The system must maintain privacy state per workload.

### Access Registry

The system must maintain desired and effective access state per workload.

### Policy Engine

The system must block unsafe transitions by default, especially restricted/sensitive workloads moving to public exposure.

### Dashboard

The dashboard must show workload cards, privacy, desired access, effective access, health, routes, migration state, and monitoring.

### Dashboard Mutations

Unauthenticated dashboard mode is view-only. Authenticated admin mode can change privacy and request/apply access changes.

### Control API

The control API must bind only to `127.0.0.1:8099`. Caddy may proxy `/api/*` only through the Tailscale-bound dashboard route.

### Monitoring

The dashboard must include a btop-style monitor panel that is hidden by default and only fetches metrics while open.

### Cloudflare Planning

The system must generate Cloudflare exposure plans but not start tunnels, create DNS, or publish anything in P0.

### Git

The platform root must be a local Git repo. Secrets, tokens, workload source, `.env`, runtime data, metrics, logs, and backups must not be tracked.

## Non-Functional Requirements

- Safe by default.
- Idempotent scripts.
- No secret printing.
- No public exposure by default.
- All config JSON validates with `python3 -m json.tool`.
- Works without `jq`.
- Python scripts use the standard library where practical.
- Caddy config is validated before reload.
- Workload migration uses copy-validate-cutover-rollback flow.

## Success Criteria

P0 is successful when the owner can open a private dashboard from phone over Tailscale, see all workloads and system state, toggle monitor mode, toggle privacy/desired access in admin mode, and run smoke tests showing no accidental public exposure.


---

# Architecture: Oreo Cloud P0

## High-Level Architecture

```text
Phone / Laptop
      |
      | Tailscale private network
      v
oreochiserver Tailscale IP
      |
      v
Caddy bound to Tailscale IP on :8088
      |
      +--> static dashboard files
      |
      +--> /api/* -> 127.0.0.1:8099
                         |
                         v
                    Oreo Control API
                         |
                         +--> config/*.json
                         +--> scripts/oreo-access-*
                         +--> scripts/oreo-privacy-set
                         +--> runtime/audit.log

Metrics timer
      |
      v
collect_metrics.py -> dashboard/public/metrics.json

Cloudflare planning
      |
      v
generate_cloudflare_config.py -> cloudflare/planned-ingress.yml
```

## Trust Boundaries

### Tailscale Boundary

The dashboard should be reachable only from the tailnet in P0. Caddy must bind the dashboard listener to the Tailscale IP, not to all interfaces.

### Local API Boundary

The control API must bind only to `127.0.0.1:8099`. It is not a public API. It is only reachable through the Tailscale-bound Caddy route.

### Admin Mutation Boundary

View-only dashboard access does not require a token. Any mutation requires admin mode with a bearer token from `/etc/oreo-cloud/control-token`.

### Cloudflare Boundary

Cloudflare configuration is planned only in P0. No tunnel should be started and no DNS should be created by P0 scripts.

## Core Components

### 1. Platform Root

```text
/srv/oreo-cloud
```

The platform root contains tracked docs, config, templates, dashboard, API, scripts, systemd unit templates, and workload manifests.

### 2. Git Repo

`/srv/oreo-cloud` is a local Git repo. Git tracks platform definitions, not secrets or workload source code.

### 3. Neutral Workload Registry

`config/workloads.json` contains workload identity and runtime facts only.

Examples:

- workload id
- name
- lifecycle
- runtime type
- Compose path
- Compose project name
- local URL
- health check URL
- canonical source path
- migration state

It should not permanently mark a workload as private/public/company/demo.

### 4. Privacy Registry

`config/privacy.json` stores privacy state per workload.

### 5. Access Registry

`config/access.json` stores desired and effective access state per workload.

### 6. Policy Registry

`config/policy.json` stores allowed/blocked transitions and confirmation requirements.

### 7. Exposure Registry

`config/exposure.json` stores provider capability and Cloudflare provisioning state.

### 8. Dashboard

Static HTML/CSS/JS generated from config and served by Caddy.

The dashboard has:

- workload cards
- privacy controls
- access controls
- access preview/apply flow
- monitoring toggle
- Cloudflare plan panel
- audit/event panel

### 9. Control API

A tiny Python standard-library HTTP API for authenticated mutations.

### 10. Monitoring Collector

A timer-driven Python script that writes safe metrics to `metrics.json`.

### 11. Cloudflare Planner

A generator that reads workload/access/privacy/policy config and writes a planned Cloudflare ingress file only.

## P0 Network Model

```text
Private dashboard:
http://oreochiserver:8088

Fallback:
http://<tailscale-ip>:8088

Control API backend:
http://127.0.0.1:8099

Cloudflare:
planned only, no active tunnel in P0
```

## Access State Machine

```text
none -> local -> tailnet -> cloudflare-protected -> cloudflare-public
```

Not every transition is allowed. Policy decides.

Recommended P0 behavior:

- `none`, `local`, and `tailnet` can be applied by P0.
- `cloudflare-protected` can be requested and planned.
- `cloudflare-public` can be requested only if policy allows, but should not become effective until Cloudflare activation is explicitly implemented.

## Desired vs Effective Access

`desired` means what the admin requested.

`effective` means what is actually active.

This distinction avoids lying in the dashboard. Example:

```json
{
  "desired": "cloudflare-protected",
  "effective": "tailnet",
  "lastError": "Cloudflare is provisioned but disabled in P0."
}
```

## Workload Migration Architecture

Canonical workload layout:

```text
/srv/oreo-cloud/workloads/<id>/
├── README.md
├── manifest.json
└── source/      # ignored by Oreo Cloud Git
```

Migration should use copy-first, validate, cut over, symlink old path, health check, and rollback if needed.

## Security Posture

Default posture:

```text
private first, public never by accident
```

P0 must not expose:

- PostgreSQL
- Docker socket
- SSH
- dashboard publicly
- control API publicly
- workload source or env files
- Cloudflare tokens
- `.env` files


---

# Implementation Plan: Oreo Cloud P0

## Phase 0: Discovery Only

Run read-only discovery. Do not edit files, install packages, restart services, start tunnels, or expose anything.

Collect:

- hostname and user
- Tailscale IP/status
- Docker containers/networks/volumes
- Docker Compose projects
- Compose labels and working directories
- current listening ports
- Caddy status
- Git version and existing repo state
- cloudflared version/service/config hints
- existing workload directories
- current health URLs where known

Deliverable:

```text
docs/DISCOVERY.md
```

## Phase 1: Git-Tracked Base Layout

Create `/srv/oreo-cloud` and initialize a local Git repo.

Create directories:

```text
config/
config/schemas/
workloads/
control-plane/dashboard/public/
control-plane/api/
control-plane/monitoring/
cloudflare/
scripts/
systemd/
runtime/
runtime/metrics-history/
runtime/migration-backups/
docs/
```

Create:

- `.gitignore`
- `.gitattributes`
- `README.md`
- `AGENTS.md`
- `docs/GIT.md`
- `docs/SECURITY.md`
- `docs/MIGRATION.md`

Commit:

```text
Initialize Oreo Cloud repo
```

## Phase 2: Neutral Registries

Create:

```text
config/workloads.json
config/access.json
config/privacy.json
config/policy.json
config/routes.json
config/exposure.json
config/monitoring.json
```

Validate each file:

```bash
python3 -m json.tool <file> >/dev/null
```

Commit:

```text
Add neutral workload and access registries
```

## Phase 3: Workload Migration Planning

Create:

```text
scripts/oreo-migrate-workload-plan
```

This command prints a safe migration plan. It must not move files or restart workloads.

Commit:

```text
Add workload migration planning
```

## Phase 4: Optional Manual Workload Organization

Move selected workloads one at a time into:

```text
/srv/oreo-cloud/workloads/<id>/source
```

Only after:

1. current Compose project name is known
2. bind mounts and named volumes are known
3. source is copied
4. Compose validates from the new path
5. rollback path exists

Commit only safe files:

- workload `README.md`
- workload `manifest.json`
- registry updates
- docs

Do not commit source, `.env`, runtime data, backups, or secrets.

Commit:

```text
Organize workloads under Oreo Cloud
```

## Phase 5: Read-Only CLI

Create:

- `oreo-inventory`
- `oreo-workloads`
- `oreo-health`
- `oreo-open`
- `oreo-doctor`
- `oreo-git-checkpoint`
- `oreo-events`

Symlink them into `/usr/local/bin`.

Commit:

```text
Add read-only Oreo Cloud CLI
```

## Phase 6: Dashboard Generator

Create:

```text
control-plane/dashboard/generate_dashboard.py
```

Generate:

```text
control-plane/dashboard/public/index.html
control-plane/dashboard/public/style.css
control-plane/dashboard/public/app.js
```

Dashboard must show:

- workload cards
- health
- privacy state
- desired access
- effective access
- routes
- migration state
- Cloudflare plan state
- monitor toggle

Commit:

```text
Add private dashboard generator
```

## Phase 7: Monitoring

Create:

```text
control-plane/monitoring/collect_metrics.py
systemd/oreo-metrics.service
systemd/oreo-metrics.timer
scripts/oreo-monitor
```

Collector writes:

```text
control-plane/dashboard/public/metrics.json
```

The dashboard fetches `metrics.json` only when the monitor panel is open.

Commit:

```text
Add btop-style monitoring
```

## Phase 8: Local Control API

Create:

```text
control-plane/api/server.py
systemd/oreo-control-api.service
```

Bind only to:

```text
127.0.0.1:8099
```

Token file:

```text
/etc/oreo-cloud/control-token
```

Never commit or print the token.

Commit:

```text
Add dashboard control API
```

## Phase 9: Access Controller

Create:

```text
scripts/oreo-access-preview
scripts/oreo-access-apply
scripts/oreo-privacy-set
```

Behavior:

- preview policy decisions before applying
- mutate `privacy.json` only for privacy changes
- mutate `access.json.desired` first for access changes
- update `effective` only after apply succeeds
- write audit events
- regenerate dashboard
- never start Cloudflare in P0 unless explicitly overridden by a future task

Commit:

```text
Add policy-driven access controller
```

## Phase 10: Cloudflare Planning

Create:

```text
cloudflare/README.md
cloudflare/cloudflared-config.template.yml
cloudflare/planned-ingress.yml
cloudflare/generate_cloudflare_config.py
cloudflare/quick-tunnel-notes.md
scripts/oreo-cloudflare-plan
docs/CLOUDFLARE.md
```

P0 generates plans only.

Commit:

```text
Add Cloudflare exposure planning
```

## Phase 11: Private Caddy Route

Configure Caddy to serve the dashboard on port `8088`, bound to the Tailscale IP.

Back up Caddyfile first.

Validate:

```bash
sudo caddy validate --config /etc/caddy/Caddyfile
```

Reload only if validation passes:

```bash
sudo systemctl reload caddy
```

Commit docs/config example:

```text
Add private Caddy dashboard route
```

## Phase 12: Smoke Test

Create:

```text
scripts/smoke-test
```

Symlink:

```text
/usr/local/bin/oreo-cloud-smoke-test
```

Smoke test checks:

- Git repo exists
- JSON configs valid
- dashboard generated
- API binds locally
- Caddy validates
- metrics collector works
- Cloudflare plan generated
- no Tailscale Funnel detected
- no cloudflared service active unless explicitly expected
- no tracked secrets
- CLI commands work

Commit:

```text
Add Oreo Cloud smoke test
```


---

# Config Model

P0 separates workload identity from privacy and access policy.

## Files

```text
config/workloads.json   # app/runtime identity
config/privacy.json     # current privacy classification
config/access.json      # desired/effective access state
config/policy.json      # guardrails and transition rules
config/routes.json      # current route definitions
config/exposure.json    # provider capabilities and Cloudflare planning
config/monitoring.json  # monitoring behavior
```

## `workloads.json`

Purpose: describe the workload as a neutral unit.

Must contain:

- id
- name
- description
- lifecycle
- kind
- paths
- runtime
- network
- health
- actions
- migration

Must not contain permanent public/private/company/demo policy.

Example:

```json
{
  "version": 1,
  "workloads": [
    {
      "id": "intake-os",
      "name": "Intake OS",
      "description": "Application workload.",
      "lifecycle": "active",
      "kind": "web-app",
      "paths": {
        "root": "/srv/oreo-cloud/workloads/intake-os",
        "source": "/srv/oreo-cloud/workloads/intake-os/source",
        "manifest": "/srv/oreo-cloud/workloads/intake-os/manifest.json",
        "legacy": "/home/oreo/intake-os"
      },
      "runtime": {
        "type": "docker-compose",
        "composePath": "/srv/oreo-cloud/workloads/intake-os/source/docker-compose.server.yml",
        "composeProject": "intake-os",
        "service": ""
      },
      "network": {
        "localUrl": "http://127.0.0.1:8080",
        "internalPort": 8080
      },
      "health": {
        "enabled": true,
        "url": "http://127.0.0.1:8080/health",
        "expectedStatus": 200,
        "timeoutSeconds": 3
      },
      "actions": {
        "open": true,
        "health": true,
        "logs": false,
        "restart": false,
        "deploy": false,
        "accessToggle": true,
        "privacyToggle": true
      },
      "migration": {
        "status": "planned",
        "originalPath": "/home/oreo/intake-os",
        "compatibilitySymlink": false
      }
    }
  ]
}
```

## `privacy.json`

Purpose: current privacy classification.

Allowed states:

```text
unclassified
personal
internal
sensitive
restricted
```

Example:

```json
{
  "version": 1,
  "defaultPrivacy": "unclassified",
  "states": ["unclassified", "personal", "internal", "sensitive", "restricted"],
  "workloads": {
    "intake-os": {
      "privacy": "sensitive",
      "reason": "Initial operator classification.",
      "updatedAt": "",
      "updatedBy": "oreo-cloud"
    },
    "uptime-kuma": {
      "privacy": "restricted",
      "reason": "Infrastructure/admin dashboard.",
      "updatedAt": "",
      "updatedBy": "oreo-cloud"
    }
  }
}
```

## `access.json`

Purpose: desired and effective access state.

Allowed states:

```text
none
local
tailnet
cloudflare-protected
cloudflare-public
```

Example:

```json
{
  "version": 1,
  "defaultAccess": "tailnet",
  "states": ["none", "local", "tailnet", "cloudflare-protected", "cloudflare-public"],
  "workloads": {
    "intake-os": {
      "desired": "tailnet",
      "effective": "tailnet",
      "urls": {
        "local": "http://127.0.0.1:8080",
        "tailnet": "http://oreochiserver:8080",
        "cloudflare": ""
      },
      "lastAppliedAt": "",
      "lastError": ""
    }
  }
}
```

## `policy.json`

Purpose: guardrails.

Example:

```json
{
  "version": 1,
  "defaultDecision": "deny-public",
  "rules": {
    "allowDashboardPrivacyToggle": true,
    "allowDashboardAccessToggle": true,
    "requireConfirmationForCloudflareProtected": true,
    "requireConfirmationForCloudflarePublic": true,
    "requireSecondConfirmationForRestrictedPublic": true,
    "requireAuthForDashboardMutations": true,
    "allowRestrictedToCloudflarePublic": false,
    "allowSensitiveToCloudflarePublic": false,
    "allowRestrictedToCloudflareProtected": true,
    "allowSensitiveToCloudflareProtected": true,
    "allowUnclassifiedToCloudflarePublic": true,
    "allowTailnetForAll": true
  },
  "blockedTargets": ["postgres", "docker-socket", "host-ssh", "control-api"],
  "confirmationPhrases": {
    "cloudflare-public": "publish publicly",
    "restricted-cloudflare-public": "override restricted public exposure"
  }
}
```

## `routes.json`

Purpose: record dashboard and workload routes.

Example:

```json
{
  "version": 1,
  "dashboard": {
    "bind": "tailscale",
    "port": 8088,
    "url": "http://oreochiserver:8088"
  },
  "api": {
    "bind": "127.0.0.1",
    "port": 8099,
    "public": false
  },
  "workloadRoutes": {
    "intake-os": {
      "tailnet": {
        "enabled": true,
        "mode": "direct",
        "url": "http://oreochiserver:8080"
      },
      "cloudflare": {
        "enabled": false,
        "hostname": "",
        "mode": "disabled",
        "requiresAuth": true
      }
    }
  }
}
```

## `exposure.json`

Purpose: provider capability/planning.

Example:

```json
{
  "version": 1,
  "defaultProvider": "tailscale",
  "providers": {
    "tailscale": {
      "enabled": true,
      "default": true,
      "notes": "Primary private access path."
    },
    "cloudflare": {
      "enabled": false,
      "provisioned": true,
      "default": false,
      "allowQuickTunnels": false,
      "allowNamedTunnels": false,
      "requireAccessPolicy": true,
      "configPath": "/srv/oreo-cloud/cloudflare/planned-ingress.yml"
    }
  }
}
```

## `monitoring.json`

Purpose: metrics behavior.

Example:

```json
{
  "version": 1,
  "enabled": true,
  "mode": "snapshot",
  "refreshSeconds": 3,
  "dashboardToggle": true,
  "publicMetricsPath": "/srv/oreo-cloud/control-plane/dashboard/public/metrics.json",
  "historyEnabled": true,
  "historyPath": "/srv/oreo-cloud/runtime/metrics-history/metrics.jsonl",
  "include": {
    "cpu": true,
    "load": true,
    "memory": true,
    "disk": true,
    "network": true,
    "docker": true,
    "processes": true,
    "temperatures": true
  },
  "processes": {
    "limit": 8,
    "showCommandArgs": false
  },
  "security": {
    "redactSecrets": true,
    "doNotExposeEnv": true,
    "doNotExposeCommandArgs": true
  }
}
```


---

# Dashboard and Control API

## Dashboard Purpose

The dashboard is the single private access point for Oreo Cloud.

It should be mobile-first and reachable from Tailscale-connected devices:

```text
http://oreochiserver:8088
```

## Dashboard Sections

P0 dashboard sections:

1. Header / system identity
2. Workload cards
3. Monitor toggle
4. Cloudflare exposure plan
5. Events/audit summary
6. Admin mode panel

## Workload Card Fields

Each workload card should show:

```text
Name
ID
Lifecycle
Runtime type
Migration status
Health
Privacy state
Desired access
Effective access
Local URL
Tailnet URL
Cloudflare URL or planned state
Actions
```

## View-Only Mode

Unauthenticated dashboard mode can:

- view workloads
- open allowed URLs
- view health
- view desired/effective access
- view monitor panel
- view Cloudflare plan

It cannot mutate config.

## Admin Mode

Admin mode asks for a control token. The token is stored in browser `sessionStorage`, not `localStorage`.

Admin mode can:

- update privacy state
- preview access changes
- apply allowed access changes
- regenerate dashboard indirectly through API actions

Admin mode cannot:

- bypass policy
- expose blocked targets
- print token
- write token into HTML
- expose the control API publicly

## Control API

The control API must bind only to:

```text
127.0.0.1:8099
```

Caddy may proxy `/api/*` through the Tailscale-bound dashboard route.

## Endpoints

### `GET /api/workloads`

Returns merged workload view from:

- `workloads.json`
- `privacy.json`
- `access.json`
- `routes.json`
- `exposure.json`

Auth: optional for read-only.

### `GET /api/access`

Returns access state.

Auth: optional for read-only.

### `GET /api/privacy`

Returns privacy state.

Auth: optional for read-only.

### `POST /api/workloads/<id>/privacy`

Body:

```json
{
  "privacy": "sensitive",
  "reason": "Operator change"
}
```

Auth: required.

Behavior:

1. validate workload id
2. validate privacy state
3. update `privacy.json`
4. write audit event
5. regenerate dashboard
6. optionally Git checkpoint

### `POST /api/workloads/<id>/access/preview`

Body:

```json
{
  "desired": "cloudflare-protected"
}
```

Auth: required.

Behavior:

1. read workload, privacy, access, policy
2. return allowed/blocked decision
3. list files/routes that would change
4. do not modify anything

### `POST /api/workloads/<id>/access/apply`

Body:

```json
{
  "desired": "tailnet",
  "confirmation": ""
}
```

Auth: required.

Behavior:

1. validate transition
2. verify confirmation phrase when required
3. update `access.json.desired`
4. apply safe states
5. update `access.json.effective` only after success
6. write audit event
7. regenerate dashboard
8. optionally Git checkpoint

For P0, `cloudflare-protected` and `cloudflare-public` should generally update desired state and generate a plan, but not become effective unless Cloudflare activation has been explicitly implemented and enabled.

### `GET /api/metrics`

Returns `metrics.json`.

Auth: optional if dashboard is private, but still safe metrics only.

### `GET /api/events`

Returns sanitized recent audit events.

Auth: optional for read-only or required depending on final preference.

## API Auth

Token file:

```text
/etc/oreo-cloud/control-token
```

Permissions:

```text
root:oreo
0640
```

Requests use:

```text
Authorization: Bearer <token>
```

The API must never log the token.

## Audit Events

Audit file:

```text
/srv/oreo-cloud/runtime/audit.log
```

JSONL format:

```json
{"timestamp":"2026-06-30T00:00:00Z","actor":"admin-token","action":"privacy.set","workloadId":"intake-os","from":"internal","to":"sensitive","result":"ok"}
```

Do not track audit logs in Git.

## Dashboard Mutation Flow

Privacy change:

```text
select new privacy -> POST privacy -> update config -> audit -> regenerate dashboard
```

Access change:

```text
select new access -> preview -> confirmation if needed -> apply -> desired -> route/provider work -> effective -> audit -> regenerate dashboard
```

## Control API Systemd Template

See:

```text
templates/systemd/oreo-control-api.service
```


---

# Monitoring: btop-Style Toggle

## Goal

Provide a btop-inspired monitor panel in the private dashboard without embedding a terminal or exposing sensitive host data.

## Architecture

```text
systemd timer
      |
      v
collect_metrics.py
      |
      v
control-plane/dashboard/public/metrics.json
      |
      v
dashboard monitor toggle
```

## Dashboard Behavior

The monitor panel is hidden by default.

Button states:

```text
Show Monitor
Hide Monitor
```

When open, dashboard fetches `/metrics.json` every `refreshSeconds` from `monitoring.json`.

When closed, dashboard stops fetching metrics.

## Metrics to Show

P0 metrics:

- timestamp
- hostname
- uptime seconds
- load average
- CPU percent
- memory total/used/percent
- swap total/used/percent
- disk total/used/percent for `/`
- network rx/tx by interface
- Docker container status
- Docker stats if available
- safe top processes by process name only
- thermal temperature if available

## Forbidden Data

The collector must not read or expose:

- environment variables
- process command-line arguments
- container env
- Docker inspect output
- secret files
- `.env` contents
- tokens
- database connection strings

Allowed commands:

```bash
docker ps --format '{{json .}}'
docker stats --no-stream --format '{{json .}}'
ps -eo pid,comm,%cpu,%mem --sort=-%cpu
```

Disallowed commands:

```bash
env
printenv
docker inspect
cat /proc/*/environ
ps aux
ps -ef
```

## Metrics JSON Shape

```json
{
  "timestamp": "2026-06-30T00:00:00Z",
  "hostname": "oreochiserver",
  "uptimeSeconds": 123456,
  "load": {
    "one": 0.42,
    "five": 0.36,
    "fifteen": 0.31
  },
  "cpu": {
    "percent": 18.4
  },
  "memory": {
    "totalBytes": 16777216000,
    "usedBytes": 6291456000,
    "percent": 37.5
  },
  "disk": {
    "mount": "/",
    "totalBytes": 1000000000000,
    "usedBytes": 380000000000,
    "percent": 38.0
  },
  "docker": {
    "containers": [
      {
        "name": "uptime-kuma",
        "image": "louislam/uptime-kuma",
        "status": "Up 2 days"
      }
    ]
  },
  "processes": [
    {
      "pid": 1234,
      "name": "caddy",
      "cpuPercent": 1.2,
      "memoryPercent": 0.8
    }
  ]
}
```

## UI Layout

```text
+--------------------------------+
| CPU      ########..   34%       |
| RAM      #####.....   51%       |
| DISK     ###.......   28%       |
| LOAD     0.42 0.36 0.31         |
+--------------------------------+
| Containers                     |
| uptime-kuma       Up           |
| intake-os         Up           |
+--------------------------------+
| Top Processes                  |
| caddy             1.2%          |
| dockerd           0.8%          |
+--------------------------------+
```

Use plain CSS. No CDN. No external JS. No remote fonts.

## Systemd

Templates:

```text
templates/systemd/oreo-metrics.service
templates/systemd/oreo-metrics.timer
```

P0 install command:

```bash
sudo cp /srv/oreo-cloud/systemd/oreo-metrics.service /etc/systemd/system/oreo-metrics.service
sudo cp /srv/oreo-cloud/systemd/oreo-metrics.timer /etc/systemd/system/oreo-metrics.timer
sudo systemctl daemon-reload
sudo systemctl enable --now oreo-metrics.timer
```

## Acceptance

```bash
python3 /srv/oreo-cloud/control-plane/monitoring/collect_metrics.py
python3 -m json.tool /srv/oreo-cloud/control-plane/dashboard/public/metrics.json >/dev/null
```

Then dashboard must show monitor only after toggle.


---

# Workload Migration Plan

## Goal

Allow existing workload files to move into Oreo Cloud's canonical organization:

```text
/srv/oreo-cloud/workloads/<workload-id>/source
```

while preserving runtime behavior, volumes, paths, and rollback options.

## Canonical Layout

```text
/srv/oreo-cloud/workloads/<id>/
├── README.md
├── manifest.json
└── source/          # ignored by Oreo Cloud Git
```

Optional ignored directories:

```text
data/
env/
backups/
runtime/
logs/
```

## Migration Rules

1. Move one workload at a time.
2. Discover Compose project name before moving.
3. Discover Compose file path before moving.
4. Discover bind mounts and named volumes before moving.
5. Copy first, validate, then cut over.
6. Preserve old path with a symlink when useful.
7. Preserve Compose project name using `-p` or `COMPOSE_PROJECT_NAME`.
8. Preserve named volumes.
9. Do not commit workload source, `.env`, secrets, runtime data, database files, or backups.
10. Run health checks after migration.
11. Roll back if health fails.

## Discovery Commands

```bash
echo "## Compose labels"
docker ps --format 'table {{.Names}}\t{{.Label "com.docker.compose.project"}}\t{{.Label "com.docker.compose.project.working_dir"}}\t{{.Label "com.docker.compose.project.config_files"}}' || true

echo "## Container mounts"
docker inspect \
  --format '{{.Name}} {{range .Mounts}}{{.Type}}:{{.Source}}->{{.Destination}}; {{end}}' \
  $(docker ps -q) 2>/dev/null || true

echo "## Existing compose files"
find /home /srv -maxdepth 5 \( -iname 'docker-compose*.yml' -o -iname 'docker-compose*.yaml' -o -iname 'compose.yml' -o -iname 'compose.yaml' \) 2>/dev/null | sort || true

echo "## Existing env files, names only"
find /home /srv -maxdepth 5 \( -iname '.env' -o -iname '.env.*' \) 2>/dev/null | sort || true
```

Do not print `.env` contents.

## Migration Plan Command

Create:

```text
scripts/oreo-migrate-workload-plan
```

Usage:

```bash
oreo-migrate-workload-plan intake-os
```

Output must include:

- current root
- target root
- source path
- Compose file
- Compose project name
- health URL
- legacy path
- bind mount risks
- named volume risks
- suggested copy command
- suggested validation command
- suggested cutover command
- rollback command

The planner must not move files or restart services.

## Copy-Validate-Cutover Pattern

Example variables:

```bash
WORKLOAD_ID="intake-os"
OLD_ROOT="/home/oreo/intake-os"
NEW_ROOT="/srv/oreo-cloud/workloads/intake-os"
NEW_SOURCE="/srv/oreo-cloud/workloads/intake-os/source"
TIMESTAMP="$(date +%Y%m%d-%H%M%S)"
BACKUP_ROOT="/srv/oreo-cloud/runtime/migration-backups/${WORKLOAD_ID}.${TIMESTAMP}"
```

Copy:

```bash
sudo install -d -o oreo -g oreo "$NEW_ROOT"
rsync -aH --numeric-ids "$OLD_ROOT/" "$NEW_SOURCE/"
```

Validate Compose:

```bash
docker compose \
  -f "$NEW_SOURCE/docker-compose.server.yml" \
  -p intake-os \
  config >/tmp/intake-os.compose.validated.yml
```

Cut over only after validation.

```bash
docker compose \
  -f "$OLD_ROOT/docker-compose.server.yml" \
  -p intake-os \
  down

sudo mv "$OLD_ROOT" "$BACKUP_ROOT"
sudo ln -s "$NEW_SOURCE" "$OLD_ROOT"
sudo chown -h oreo:oreo "$OLD_ROOT"

docker compose \
  -f "$NEW_SOURCE/docker-compose.server.yml" \
  -p intake-os \
  up -d
```

Health check:

```bash
oreo-health
docker compose -f "$NEW_SOURCE/docker-compose.server.yml" -p intake-os ps
```

## Rollback Pattern

```bash
docker compose \
  -f "$NEW_SOURCE/docker-compose.server.yml" \
  -p intake-os \
  down || true

sudo rm "$OLD_ROOT"
sudo mv "$BACKUP_ROOT" "$OLD_ROOT"

docker compose \
  -f "$OLD_ROOT/docker-compose.server.yml" \
  -p intake-os \
  up -d
```

## Manifest

Each workload should have:

```text
/srv/oreo-cloud/workloads/<id>/manifest.json
```

Example:

```json
{
  "id": "intake-os",
  "name": "Intake OS",
  "canonicalRoot": "/srv/oreo-cloud/workloads/intake-os",
  "sourcePath": "/srv/oreo-cloud/workloads/intake-os/source",
  "legacyPaths": ["/home/oreo/intake-os"],
  "compose": {
    "enabled": true,
    "path": "/srv/oreo-cloud/workloads/intake-os/source/docker-compose.server.yml",
    "project": "intake-os",
    "service": ""
  },
  "migration": {
    "status": "planned",
    "originalPath": "/home/oreo/intake-os",
    "backupPath": "",
    "symlinkCreated": false,
    "migratedAt": "",
    "rollback": "Restore original directory from backup path and restart compose with preserved project name."
  },
  "git": {
    "trackedByOreoCloudGit": false,
    "notes": ["Do not commit source, secrets, env, data, logs, or backups."]
  }
}
```


---

# Cloudflare Provisioning Plan

## Goal

Provision for future workload access through Cloudflare without enabling exposure in P0.

Cloudflare should be a per-workload access provider, not the default platform access method.

## P0 Rules

- Do not start `cloudflared`.
- Do not create a tunnel.
- Do not create DNS records.
- Do not run Quick Tunnel automatically.
- Do not store tokens or credentials in Git.
- Do not expose the dashboard.
- Do not expose the control API.
- Do not expose PostgreSQL.
- Do not expose Docker socket.
- Generate plans only.

## Access States

Cloudflare relates to two access states:

```text
cloudflare-protected
cloudflare-public
```

For P0:

- `cloudflare-protected` can be requested and planned.
- `cloudflare-public` can be requested only if policy allows.
- Neither state becomes effective automatically unless Cloudflare activation is explicitly implemented later.

## Quick Tunnel

Quick Tunnel is useful for short demos and development previews.

P0 behavior:

```text
allowQuickTunnels: false
```

Future behavior can allow Quick Tunnel only after explicit confirmation.

## Named Tunnel

Named Tunnel is the intended future production-grade option.

Future named tunnel requirements:

- hostname configured
- workload policy allows Cloudflare
- Cloudflare Access required unless explicitly public
- credentials stored outside Git
- ingress validated before reload
- no database/admin/control targets exposed

## Files

```text
cloudflare/README.md
cloudflare/cloudflared-config.template.yml
cloudflare/planned-ingress.yml
cloudflare/generate_cloudflare_config.py
cloudflare/quick-tunnel-notes.md
scripts/oreo-cloudflare-plan
```

## Planned Ingress Generator

`generate_cloudflare_config.py` should:

1. read `workloads.json`
2. read `privacy.json`
3. read `access.json`
4. read `policy.json`
5. read `exposure.json`
6. include only workloads where desired access is Cloudflare-related and policy allows planning
7. require hostname for named routes
8. append a final `http_status:404` catch-all rule
9. write `cloudflare/planned-ingress.yml`
10. never write credentials
11. never start `cloudflared`

Example generated file:

```yaml
# Generated plan only. No tunnel is enabled by this file.
ingress:
  - hostname: demo.example.com
    service: http://127.0.0.1:8090
  - service: http_status:404
```

## CLI

Create:

```text
scripts/oreo-cloudflare-plan
```

Usage:

```bash
oreo-cloudflare-plan
```

Output:

```text
Cloudflare Exposure Plan

Provider enabled: false

Requested:
- demo-app: cloudflare-protected, missing hostname

Blocked:
- uptime-kuma: restricted/admin target
- dashboard: control-plane target

Generated:
cloudflare/planned-ingress.yml
```

## Policy Requirements

Recommended P0 policy:

- `restricted` -> `cloudflare-public`: blocked
- `sensitive` -> `cloudflare-public`: blocked
- `restricted` -> `cloudflare-protected`: allowed only with confirmation and Access requirement
- `sensitive` -> `cloudflare-protected`: allowed only with confirmation and Access requirement
- `unclassified` -> `cloudflare-public`: allowed only with confirmation phrase, but still planned only in P0

## Future Activation Phase

Only after P0:

1. Install/configure `cloudflared` if needed.
2. Store credentials under `/etc/cloudflared`, not Git.
3. Generate ingress from Oreo Cloud.
4. Validate ingress.
5. Start/reload `cloudflared`.
6. Verify Cloudflare Access policy.
7. Update `access.json.effective`.
8. Write audit event.


---

# Security Requirements

## Default Posture

```text
private first, public never by accident
```

## Hard Blocks in P0

P0 must not expose:

- PostgreSQL
- Docker socket
- SSH
- host filesystem browser
- control API directly
- dashboard publicly
- workload source code
- `.env` files
- secrets
- tokens
- database files
- runtime backups

## Dashboard Exposure

Dashboard must be served by Caddy on port `8088`, bound to the Tailscale IP only.

Do not bind dashboard to `0.0.0.0`.

## Control API Exposure

Control API must bind only to:

```text
127.0.0.1:8099
```

Caddy may proxy it only through the Tailscale-bound dashboard route.

## Token Handling

Control token path:

```text
/etc/oreo-cloud/control-token
```

Permissions:

```text
root:oreo
0640
```

Rules:

- never commit token
- never print token
- never include token in generated HTML
- never write token to logs
- dashboard stores token in `sessionStorage` only
- token expires when browser tab closes

## Git Hygiene

Git should track platform config and docs only.

Must not track:

- workload source
- `.env`
- runtime data
- logs
- metrics output
- backups
- tokens
- Cloudflare credentials
- SSH keys
- TLS private keys
- database files/dumps

## Metrics Safety

Metrics collector must not expose:

- env vars
- process command-line args
- container inspect output
- secrets

Use `ps -eo pid,comm,%cpu,%mem` rather than `ps aux` or `ps -ef`.

## Access Policy Safety

P0 rules:

- `none`, `local`, `tailnet` can be safe effective states.
- Cloudflare states can be desired/planned.
- Public Cloudflare exposure requires confirmation.
- Restricted/sensitive public exposure is blocked by default.
- Effective access updates only after successful apply.

## Audit Logging

Write audit events to:

```text
/srv/oreo-cloud/runtime/audit.log
```

Do not track audit logs in Git.

Audit events must not contain secrets.

## Caddy Changes

Before editing Caddyfile:

```bash
sudo cp /etc/caddy/Caddyfile "/etc/caddy/Caddyfile.backup.$(date +%Y%m%d-%H%M%S)"
```

Validate:

```bash
sudo caddy validate --config /etc/caddy/Caddyfile
```

Reload only on success:

```bash
sudo systemctl reload caddy
```

## Cloudflare Safety

P0 Cloudflare is plan-only.

Do not:

- start cloudflared
- run Quick Tunnel
- create named tunnel
- create DNS records
- store token in repo

## Doctor Checks

`oreo-doctor` must warn if:

- dashboard listener is not bound to Tailscale IP
- control API binds to non-localhost
- Tailscale Funnel appears enabled
- cloudflared service is active unexpectedly
- `.env` is tracked
- Cloudflare credentials are tracked
- workload source is tracked
- policy allows restricted public exposure
- `access.json.effective` says Cloudflare but Cloudflare provider is disabled


---

# Operations Guide

## Open Dashboard

From a Tailscale-connected device:

```text
http://oreochiserver:8088
```

Fallback:

```text
http://<tailscale-ip>:8088
```

## List Workloads

```bash
oreo-workloads
```

## Open a Workload

```bash
oreo-open intake-os
```

Expected output:

```text
Intake OS
Desired: tailnet
Effective: tailnet
URL: http://oreochiserver:8080
```

## Check Health

```bash
oreo-health
```

## Run Inventory

```bash
oreo-inventory
```

## Run Doctor

```bash
oreo-doctor
```

## Run Smoke Test

```bash
oreo-cloud-smoke-test
```

## Metrics

Run collector once:

```bash
oreo-monitor
```

Install timer after testing:

```bash
sudo cp /srv/oreo-cloud/systemd/oreo-metrics.service /etc/systemd/system/oreo-metrics.service
sudo cp /srv/oreo-cloud/systemd/oreo-metrics.timer /etc/systemd/system/oreo-metrics.timer
sudo systemctl daemon-reload
sudo systemctl enable --now oreo-metrics.timer
```

## Control API

Install service after testing:

```bash
sudo cp /srv/oreo-cloud/systemd/oreo-control-api.service /etc/systemd/system/oreo-control-api.service
sudo systemctl daemon-reload
sudo systemctl enable --now oreo-control-api.service
```

Create token:

```bash
sudo install -d -o root -g oreo -m 0750 /etc/oreo-cloud
openssl rand -base64 32 | sudo tee /etc/oreo-cloud/control-token >/dev/null
sudo chown root:oreo /etc/oreo-cloud/control-token
sudo chmod 0640 /etc/oreo-cloud/control-token
```

Do not paste the token into logs or commits.

## Preview Access Change

```bash
oreo-access-preview intake-os cloudflare-protected
```

## Apply Access Change

```bash
oreo-access-apply intake-os tailnet
```

For P0, Cloudflare-related states should generally update desired state and generate plans only.

## Set Privacy

```bash
oreo-privacy-set intake-os sensitive --reason "Operator classification"
```

## Cloudflare Plan

```bash
oreo-cloudflare-plan
```

This should generate:

```text
/srv/oreo-cloud/cloudflare/planned-ingress.yml
```

and not start any tunnel.

## Git Checkpoint

```bash
oreo-git-checkpoint "Add dashboard monitoring"
```

This should not push.

## Workload Migration Plan

```bash
oreo-migrate-workload-plan intake-os
```

Review before doing any move.

## Caddy Route

Dashboard route should look like:

```caddyfile
http://oreochiserver:8088, http://100.x.y.z:8088 {
    bind 100.x.y.z
    root * /srv/oreo-cloud/control-plane/dashboard/public
    handle /api/* {
        reverse_proxy 127.0.0.1:8099
    }
    file_server
}
```

Replace `100.x.y.z` with the actual Tailscale IP.


---

# Codex Task Prompts

Use these as separate Codex tasks. Do not give Codex the whole project as one vague instruction.

## Task 1: Discovery Only

```text
Perform read-only discovery for Oreo Cloud P0.

Collect:
- hostname and user
- Tailscale IP/status
- Docker containers/networks/volumes
- Docker Compose projects
- Compose labels, working directories, config files
- container bind mounts and named volumes
- listening ports
- Caddy status
- Git version and existing /srv/oreo-cloud state
- cloudflared version/service/config hints
- existing workload paths
- existing health URLs where inferable

Do not install packages.
Do not edit files.
Do not restart services.
Do not start tunnels.
Do not expose anything.
Do not print .env contents.
Write docs/DISCOVERY.md or /tmp/oreo-cloud-discovery.md if the project root does not exist yet.
```

## Task 2: Create Git-Tracked Base Layout

```text
Create /srv/oreo-cloud as a local Git repo on main.

Create directories:
config, config/schemas, workloads, control-plane/dashboard/public, control-plane/api, control-plane/monitoring, cloudflare, scripts, systemd, runtime, runtime/metrics-history, runtime/migration-backups, docs.

Create:
- .gitignore
- .gitattributes
- README.md
- AGENTS.md
- docs/GIT.md
- docs/SECURITY.md
- docs/MIGRATION.md

Do not configure Caddy.
Do not move workloads.
Do not expose anything.
Commit: Initialize Oreo Cloud repo
```

## Task 3: Create Neutral Registries

```text
Create config files:
- workloads.json
- access.json
- privacy.json
- policy.json
- routes.json
- exposure.json
- monitoring.json

Workloads are agnostic by default.
workloads.json describes app identity/runtime only.
privacy.json stores classification.
access.json stores desired/effective access.
policy.json stores guardrails.
exposure.json stores provider capability/planning.

Use discovery results for real ports, Compose paths, and project names where known.
Validate all JSON with python3 -m json.tool.
Commit: Add neutral workload and access registries
```

## Task 4: Add Workload Migration Planner

```text
Create scripts/oreo-migrate-workload-plan.

It should read workloads.json and print a safe plan to move one workload to /srv/oreo-cloud/workloads/<id>/source.

It must show:
- current path
- target path
- compose file
- compose project name
- health URL
- legacy path
- bind mount risks
- named volume risks
- copy command
- validation command
- cutover command
- rollback command

It must not move files.
It must not restart services.
It must not edit configs.
Symlink it into /usr/local/bin.
Commit: Add workload migration planning
```

## Task 5: Optional Workload Organization

```text
Move selected workloads one at a time into /srv/oreo-cloud/workloads/<id>/source.

Use copy-validate-cutover-rollback.
Preserve Compose project name with -p or COMPOSE_PROJECT_NAME.
Preserve named volumes.
Preserve old path as a symlink when useful.
Do not commit source, .env, secrets, runtime data, backups, or database files.
Run health checks after each move.
Rollback if health fails.
Commit only manifests, registry updates, and docs.
Commit: Organize workloads under Oreo Cloud
```

## Task 6: Read-Only CLI

```text
Create scripts:
- oreo-inventory
- oreo-workloads
- oreo-health
- oreo-open
- oreo-doctor
- oreo-git-checkpoint
- oreo-events

Symlink them into /usr/local/bin.
Use Python standard library where practical.
Do not require jq.
Do not print secrets.
Do not mutate services except git checkpoint when explicitly called.
Commit: Add read-only Oreo Cloud CLI
```

## Task 7: Dashboard Generator

```text
Create control-plane/dashboard/generate_dashboard.py.

Generate:
- control-plane/dashboard/public/index.html
- control-plane/dashboard/public/style.css
- control-plane/dashboard/public/app.js

Dashboard must show workload cards, privacy state, desired access, effective access, health, routes, migration state, Cloudflare plan state, and monitor toggle.
No CDN. No remote fonts. No external JS.
Commit: Add private dashboard generator
```

## Task 8: Monitoring

```text
Create:
- control-plane/monitoring/collect_metrics.py
- systemd/oreo-metrics.service
- systemd/oreo-metrics.timer
- scripts/oreo-monitor

Collector writes valid JSON to control-plane/dashboard/public/metrics.json.
Dashboard monitor panel is hidden by default and fetches metrics only while open.
Do not expose env vars, command-line args, docker inspect, tokens, or secrets.
Commit: Add btop-style monitoring
```

## Task 9: Control API

```text
Create local-only control API at control-plane/api/server.py.
Bind only to 127.0.0.1:8099.
Use token file /etc/oreo-cloud/control-token.
Never commit or print the token.

Implement:
- GET /api/workloads
- GET /api/access
- GET /api/privacy
- POST /api/workloads/<id>/privacy
- POST /api/workloads/<id>/access/preview
- POST /api/workloads/<id>/access/apply
- GET /api/metrics
- GET /api/events

Create systemd/oreo-control-api.service.
Commit: Add dashboard control API
```

## Task 10: Access Controller

```text
Create:
- scripts/oreo-access-preview
- scripts/oreo-access-apply
- scripts/oreo-privacy-set

Behavior:
- preview policy decisions before apply
- update privacy.json for privacy changes
- update access.json desired first for access changes
- update effective only after successful apply
- write runtime/audit.log
- regenerate dashboard
- do not start Cloudflare in P0
- do not enable Tailscale Funnel
- block restricted/sensitive cloudflare-public unless policy explicitly allows
Commit: Add policy-driven access controller
```

## Task 11: Cloudflare Planning

```text
Create:
- cloudflare/README.md
- cloudflare/cloudflared-config.template.yml
- cloudflare/planned-ingress.yml
- cloudflare/generate_cloudflare_config.py
- cloudflare/quick-tunnel-notes.md
- scripts/oreo-cloudflare-plan
- docs/CLOUDFLARE.md

Generate plans only.
Do not start cloudflared.
Do not create tunnels.
Do not create DNS.
Do not store tokens.
Commit: Add Cloudflare exposure planning
```

## Task 12: Private Caddy Dashboard Route

```text
Configure Caddy to serve dashboard on port 8088 bound to the Tailscale IP only.
Proxy /api/* to 127.0.0.1:8099.
Back up Caddyfile before editing.
Validate Caddy before reload.
Reload only if validation passes.
Do not enable Cloudflare.
Do not enable Funnel.
Do not expose public routes.
Commit docs/template change: Add private Caddy dashboard route
```

## Task 13: Smoke Test

```text
Create scripts/smoke-test and symlink as /usr/local/bin/oreo-cloud-smoke-test.

Check:
- Git repo exists
- JSON configs valid
- dashboard generated
- API binds localhost only
- metrics collector works
- metrics.json valid
- Cloudflare plan generated
- no Funnel detected
- no unexpected cloudflared active
- no tracked secrets
- workload CLI works
- health CLI works
- Caddy validates

Commit: Add Oreo Cloud smoke test
```


---

# Acceptance Criteria and Smoke Tests

## P0 Definition of Done

```text
[ ] /srv/oreo-cloud exists
[ ] /srv/oreo-cloud is a Git repo
[ ] .gitignore blocks secrets/runtime/workload source
[ ] .gitattributes exists
[ ] README.md exists
[ ] AGENTS.md exists
[ ] docs exist
[ ] workloads.json exists and validates
[ ] access.json exists and validates
[ ] privacy.json exists and validates
[ ] policy.json exists and validates
[ ] routes.json exists and validates
[ ] exposure.json exists and validates
[ ] monitoring.json exists and validates
[ ] workloads are neutral by default
[ ] privacy state lives outside workloads.json
[ ] access state lives outside workloads.json
[ ] desired and effective access are separate
[ ] dashboard is generated
[ ] dashboard opens from phone over Tailscale
[ ] dashboard shows workload cards
[ ] dashboard shows privacy controls
[ ] dashboard shows desired/effective access
[ ] unauthenticated dashboard is view-only
[ ] admin mode can update privacy
[ ] admin mode can preview access changes
[ ] admin mode can apply allowed access changes
[ ] monitor toggle exists
[ ] monitor toggle shows btop-style stats
[ ] metrics collector writes valid metrics.json
[ ] control API binds only to 127.0.0.1:8099
[ ] Caddy exposes API only through Tailscale-bound dashboard
[ ] Cloudflare plan exists
[ ] Cloudflare is disabled by default
[ ] no cloudflared tunnel started by P0
[ ] no Tailscale Funnel enabled
[ ] no public router ports required
[ ] no PostgreSQL exposure
[ ] no Docker socket exposure
[ ] no control token tracked by Git
[ ] no .env files tracked by Git
[ ] no workload source tracked by Oreo Cloud Git
[ ] oreo-workloads works
[ ] oreo-health works
[ ] oreo-open works
[ ] oreo-monitor works
[ ] oreo-cloudflare-plan works
[ ] oreo-doctor works
[ ] oreo-cloud-smoke-test works
```

## Manual Smoke Tests

### JSON Validation

```bash
for f in /srv/oreo-cloud/config/*.json; do
  python3 -m json.tool "$f" >/dev/null || exit 1
done
```

### Git Secret Check

```bash
cd /srv/oreo-cloud
git ls-files | grep -E '(^|/)\.env($|\.)|token|credential|secret|\.key$|\.pem$' && exit 1 || true
```

### Dashboard Generated

```bash
test -f /srv/oreo-cloud/control-plane/dashboard/public/index.html
test -f /srv/oreo-cloud/control-plane/dashboard/public/style.css
test -f /srv/oreo-cloud/control-plane/dashboard/public/app.js
```

### Metrics Valid

```bash
python3 /srv/oreo-cloud/control-plane/monitoring/collect_metrics.py
python3 -m json.tool /srv/oreo-cloud/control-plane/dashboard/public/metrics.json >/dev/null
```

### Control API Local Only

```bash
ss -tulpen | grep ':8099' | grep '127.0.0.1'
```

Fail if bound to `0.0.0.0`.

### Caddy Validation

```bash
sudo caddy validate --config /etc/caddy/Caddyfile
```

### Tailscale Dashboard Reachability

```bash
TS_IP="$(tailscale ip -4 | head -n 1)"
curl -I "http://$TS_IP:8088"
```

### Funnel Not Enabled

```bash
tailscale funnel status || true
```

Review output. P0 should not show Oreo Cloud dashboard or workloads exposed publicly.

### Cloudflare Not Active Unexpectedly

```bash
systemctl is-active cloudflared || true
```

P0 expects inactive unless a future phase explicitly changes this.

### Cloudflare Plan Only

```bash
oreo-cloudflare-plan
cat /srv/oreo-cloud/cloudflare/planned-ingress.yml
```

This must not start tunnels.

### CLI Checks

```bash
oreo-workloads
oreo-health
oreo-open intake-os || true
oreo-doctor
oreo-cloud-smoke-test
```

## Policy Tests

### Restricted Public Block

```bash
oreo-privacy-set uptime-kuma restricted --reason "admin dashboard"
oreo-access-preview uptime-kuma cloudflare-public
```

Expected: blocked.

### Tailnet Allowed

```bash
oreo-access-preview uptime-kuma tailnet
```

Expected: allowed.

### Cloudflare Protected Planned

```bash
oreo-access-preview intake-os cloudflare-protected
```

Expected: allowed or confirmation-required by policy, but P0 should not make it effective unless Cloudflare activation is implemented.

## Dashboard Tests

From phone on Tailscale:

1. Open dashboard.
2. Verify workloads appear.
3. Toggle monitor on.
4. Verify metrics load.
5. Toggle monitor off.
6. Enter admin mode.
7. Change a workload privacy state.
8. Verify audit event.
9. Preview an access change.
10. Verify unsafe public change is blocked.


---

# References

Implementation assumptions should be checked against current official docs during implementation.

## Tailscale

- Tailscale Serve docs: https://tailscale.com/docs/features/tailscale-serve
- Tailscale Funnel docs: https://tailscale.com/kb/1223/funnel

Important P0 assumption:

- Serve is private to the tailnet.
- Funnel is public.
- P0 must not enable Funnel.

## Caddy

- Caddy `bind` directive: https://caddyserver.com/docs/caddyfile/directives/bind

Important P0 assumption:

- `bind` controls which interface the listener uses.
- The site address controls the port.

## Docker Compose

- Compose project names: https://docs.docker.com/compose/how-tos/project-name/
- Docker Compose CLI reference: https://docs.docker.com/reference/cli/docker/compose/

Important P0 assumption:

- Moving Compose files can change default project naming.
- Preserve project names with `-p` or `COMPOSE_PROJECT_NAME`.

## Cloudflare Tunnel

- Cloudflare Tunnel setup: https://developers.cloudflare.com/tunnel/setup/
- Cloudflare Tunnel local configuration: https://developers.cloudflare.com/tunnel/advanced/local-management/configuration-file/
- Quick Tunnels: https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/do-more-with-tunnels/trycloudflare/

Important P0 assumption:

- Quick Tunnels are for testing/development.
- Named tunnels are the better future shape for stable workload hostnames.
- P0 generates Cloudflare plans only and does not start a tunnel.

## Git

- `git init`: https://git-scm.com/docs/git-init
- `.gitignore`: https://git-scm.com/docs/gitignore

Important P0 assumption:

- Git tracks the platform, not secrets or workload source by default.


---
