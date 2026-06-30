# Dashboard Runtime Cleanliness

P2 separates tracked dashboard assets from runtime state.

## Tracked Static Assets

These files are tracked and should change only when dashboard source changes:

```text
control-plane/dashboard/public/index.html
control-plane/dashboard/public/style.css
control-plane/dashboard/public/app.js
```

`control-plane/dashboard/generate_dashboard.py` writes a static shell. It does not read audit logs, metrics, action previews, Cloudflare activation evidence, or runtime health snapshots.

## Runtime State

Dynamic dashboard state is provided by:

```text
GET /api/dashboard-state
```

The same sanitized state can be written for inspection or refresh jobs with:

```bash
scripts/oreo-dashboard-state
```

The script writes:

```text
runtime/dashboard-state.json
```

That file is ignored by Git.

## Included State

The runtime state contains:

- workload registry data
- privacy state
- desired and effective access state
- route and exposure state
- manifest-derived operation and backup state
- last sanitized audit event per workload
- recent sanitized audit event summaries
- monitoring configuration

Secret-like audit fields are redacted by shared audit sanitization before they reach the dashboard state.

## Cleanliness Check

`scripts/smoke-test` verifies the invariant by:

1. Reading tracked dashboard asset bytes.
2. Running `scripts/oreo-dashboard-state`.
3. Running `python3 control-plane/dashboard/generate_dashboard.py`.
4. Comparing tracked asset bytes after regeneration.
5. Confirming `runtime/dashboard-state.json` is ignored.

Expected smoke output includes:

```text
PASS dashboard generation clean
PASS runtime dashboard state ignored
```
