# agana-plugins-waiteraid

An [Agana](https://github.com/oliverhkraft/agana-plugin-runtime) plugin that
lets the AI phone agent talk to **WaiterAid / BokaBord** on behalf of a
restaurant: book tables, look up and change the caller's bookings, cancel,
leave notes for the staff, and sync restaurant data (hours, menus, rules)
into the agent's knowledge base. No bridge code changes per restaurant —
install this plugin, upload one secret, set the restid, you're live.

## Tools

| Identifier | What it does | How it runs |
|---|---|---|
| `find_booking_by_phone` | Looks up active bookings for the caller's phone at this restaurant. Returns only booking id, datetime, party size, and edit link — guest name, phone, and email are stripped before the agent sees them. | HTTP — `GET /wa-caller-api/getBookings` |
| `send_message_to_restaurant` | Posts a free-text note (max 500 chars) attached to an existing booking — allergies, late arrival, special requests. | HTTP — `POST /wa-caller-api/writeMessageToRestaurant` |
| `reserve_table` | Books a table: guest lookup, guest creation, and booking creation. Phone number is taken from the caller automatically — the agent never asks for it. Supports children count, seating preference, occasion, accessibility needs, dietary notes, and company bookings. | Bridge handler |
| `update_booking` | Changes an existing booking: date, time, party size, allergies, notes. | Bridge handler |
| `cancel_booking` | Cancels a booking by `booking_id`. | Bridge handler |

HTTP tools call WaiterAid's `/wa-caller-api/` endpoints directly with
`Bearer` auth and JMESPath-extracted responses. Bridge-handler tools
(`bridge_handler: true`) are dispatched to the Agana bridge, which
orchestrates the multi-step WaiterAid `/wa-api/` flows.

`find_booking_by_phone` falls back to the caller's number when no phone
is given, treats an empty booking list as success, and returns empty on
vendor errors so the conversation degrades gracefully.

## Actions

| Identifier | What it does |
|---|---|
| `refresh_restaurant_data` | Fetches opening hours, menus, and rules from WaiterAid's `/api/ai-instructor/getData` and rewrites the company's RAG documents (wipes previous `waiteraid_instructor` docs first). Run it after the restaurant changes hours or menus. |

## Per-restaurant configuration

| What | Where | Source |
|---|---|---|
| `WAITERAID_CALLER_BEARER` | Encrypted secret on the company | The token your WaiterAid contact (BokaBord) issues for the `wa-caller-api` and `ai-instructor` surfaces. |
| `waiteraid_restid` | `company.metadata` in the Agana brain | The restaurant's numeric WaiterAid id (e.g. `1240`). |

## Quickstart (vibe-code locally)

```bash
# Install the Agana CLI runtime
pip install "git+https://github.com/oliverhkraft/agana-plugin-runtime.git"

# Clone this plugin
git clone https://github.com/oliverhkraft/agana-plugins-waiteraid.git
cd agana-plugins-waiteraid

# Validate the manifest shape
agana-plugin validate

# Run the offline golden tests (7 cases — should all pass)
agana-plugin test

# Try it against the real WaiterAid API
cp .env.dev.example .env.dev
# Fill in WAITERAID_CALLER_BEARER + WAITERAID_RESTID
agana-plugin dry-run find_booking_by_phone --input '{"phone":"+46707766558"}'
```

`agana-plugin dry-run` prints the rendered HTTP request and the
JMESPath-extracted result — that's what the agent will see. Bridge-handler
tools (`reserve_table`, `update_booking`, `cancel_booking`) can't be
dry-run locally; they need a running bridge.

CI (`.github/workflows/validate.yml`) runs `validate` + `test` on every
push and pull request.

## Install into a restaurant (production)

The Agana brain handles per-company install + encrypted secret storage:

```bash
# 1. Install the manifest for a company
curl -X POST "$AGANA_BRAIN_BASE_URL/v1/plugins/install" \
  -H "x-api-key: $AGANA_BRAIN_API_KEY" \
  -H "Content-Type: application/json" \
  -d "$(jq -n --arg cid "$COMPANY_ID" \
        --slurpfile m manifest.json --slurpfile t plugin.test.json \
        '{company_id: $cid, manifest: $m[0], tests: $t[0],
          repo_url: "https://github.com/oliverhkraft/agana-plugins-waiteraid",
          commit_sha: "main"}')"

# 2. Upload the bearer token (encrypted at rest with Fernet)
curl -X POST "$AGANA_BRAIN_BASE_URL/v1/secrets/upsert" \
  -H "x-api-key: $AGANA_BRAIN_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"company_id":"'"$COMPANY_ID"'","name":"WAITERAID_CALLER_BEARER","value":"..."}'

# 3. Set the restaurant's WaiterAid id on the company metadata
#    (key: waiteraid_restid — the plugin reads it via
#    context.company.metadata.waiteraid_restid)
```

Then run the `refresh_restaurant_data` action once so the agent knows the
restaurant's hours, menus, and rules. Next inbound call to that
restaurant's number dispatches through this plugin.

## Privacy

`find_booking_by_phone` never exposes guest PII to the agent: the response
extraction picks only `booking_id`, `datetime`, `party_size`, and
`shortcode_url`, dropping name, phone, and email returned by WaiterAid.
This is covered by a dedicated golden test.

## License

MIT.
