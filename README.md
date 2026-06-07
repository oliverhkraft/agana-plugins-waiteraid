# agana-plugins-waiteraid

Agana plugin for WaiterAid's `/wa-caller-api/` (Bearer-authed) endpoints.

## Tools

| Identifier | Description |
|---|---|
| `find_booking_by_phone` | Look up active future bookings for the caller's phone. |
| `send_message_to_restaurant` | Post a free-text note about a known booking. |

## Note on `agana-plugin` runtime availability

The `agana-plugin` CLI used by the quickstart/CI below isn't on PyPI yet — it
lives in the Agana monorepo at `agana-plugin-runtime/`. For local development
install it via:

    pip install -e <path-to-monorepo>/agana-plugin-runtime

The GitHub Actions `validate` workflow is set to `workflow_dispatch` (manual)
until the runtime is published. Switch it back to `on: [push, pull_request]`
once `pip install agana-plugin` works against PyPI.

## Quickstart

```
pip install agana-plugin   # see note above — until published, install from monorepo
git clone https://github.com/oliverhkraft/agana-plugins-waiteraid.git
cd agana-plugins-waiteraid

# Validate the manifest
agana-plugin validate

# Run offline golden tests
agana-plugin test

# Set real credentials and try against the real WaiterAid API
cp .env.dev.example .env.dev
# fill in WAITERAID_CALLER_BEARER and WAITERAID_RESTID
agana-plugin dry-run find_booking_by_phone --input '{"phone":"+46707766558"}'
```

## Install into a company

Currently via the brain HTTP API directly (the portal "Plugins" page lands in Plan 5):

```
curl -X POST https://brain.agana.ohk.sh/v1/plugins/install \
  -H "x-api-key: $AGANA_BRAIN_API_KEY" \
  -H "Content-Type: application/json" \
  -d @<(jq -n --arg cid "$COMPANY_ID" --slurpfile manifest manifest.json --slurpfile tests plugin.test.json \
    '{company_id: $cid, manifest: $manifest[0], tests: $tests[0], commit_sha: "manual"}')
```

Then upload secrets (per company):

```
curl -X POST https://brain.agana.ohk.sh/v1/secrets/upsert \
  -H "x-api-key: $AGANA_BRAIN_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"company_id":"'"$COMPANY_ID"'","name":"WAITERAID_CALLER_BEARER","value":"..."}'

curl -X POST https://brain.agana.ohk.sh/v1/secrets/upsert \
  -H "x-api-key: $AGANA_BRAIN_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"company_id":"'"$COMPANY_ID"'","name":"WAITERAID_RESTID","value":"1240"}'
```

Then flip the bridge `AGANA_BRAIN_PLUGINS_ENABLED=true` and the next call will dispatch through this plugin.

## Not yet covered (deferred to Plan 4b)

The WaiterAid `/wa-api/` endpoints for `reserve_table` / `update_booking` / `cancel_booking` use `application/x-www-form-urlencoded` POST bodies. The runtime's invoker currently only sends JSON. These three tools stay in the bridge's legacy `if/elif` chain until the runtime adds a `body_encoding: "form"` option.
