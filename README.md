# agana-plugins-waiteraid

An [Agana](https://github.com/oliverhkraft/agana-plugin-runtime) plugin that
lets the AI phone agent talk to **WaiterAid / BokaBord** on behalf of a
restaurant: identify the caller's existing bookings and leave notes for
the staff. No bridge code changes per restaurant — install this plugin,
upload two secrets, you're live.

## Tools

| Identifier | What it does | When the agent uses it |
|---|---|---|
| `find_booking_by_phone` | Looks up active future bookings for the caller's phone at this restaurant. | When the caller says they've already booked and wants to change, confirm, or ask about it. |
| `send_message_to_restaurant` | Posts a free-text note attached to an existing booking. | When the caller wants the restaurant to know something — allergies, late arrival, special request. |

Both tools call WaiterAid's `/wa-caller-api/` endpoints with `Bearer`
auth. Responses come back to the agent as structured data so it can
answer naturally in the conversation.

## Per-restaurant configuration

Each restaurant needs two secrets uploaded to its company in the Agana
brain — see "Install into a company" below.

| Secret name | Source |
|---|---|
| `WAITERAID_CALLER_BEARER` | The token your WaiterAid contact (BokaBord) issues for the `wa-caller-api` surface. |
| `WAITERAID_RESTID` | The restaurant's numeric WaiterAid id (e.g. `1240`). |

## Quickstart (vibe-code locally)

```bash
# Install the Agana CLI runtime
pip install "git+https://github.com/oliverhkraft/agana-plugin-runtime.git"

# Clone this plugin
git clone https://github.com/oliverhkraft/agana-plugins-waiteraid.git
cd agana-plugins-waiteraid

# Validate the manifest shape
agana-plugin validate

# Run the offline golden tests (5 cases — should all pass)
agana-plugin test

# Try it against the real WaiterAid API
cp .env.dev.example .env.dev
# Fill in WAITERAID_CALLER_BEARER + WAITERAID_RESTID
agana-plugin dry-run find_booking_by_phone --input '{"phone":"+46707766558"}'
```

`agana-plugin dry-run` prints the rendered HTTP request and the
JMESPath-extracted result — that's what the agent will see.

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

# 3. Upload the restid
curl -X POST "$AGANA_BRAIN_BASE_URL/v1/secrets/upsert" \
  -H "x-api-key: $AGANA_BRAIN_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"company_id":"'"$COMPANY_ID"'","name":"WAITERAID_RESTID","value":"339"}'
```

That's it. Next inbound call to that restaurant's number dispatches
through this plugin.

## Scope

This plugin currently wires the two `/wa-caller-api/` (JSON-bodied) tools.
WaiterAid also has form-encoded `/wa-api/` endpoints for creating, editing,
and cancelling bookings (`addBooking`, `editBooking`, `setBookingStatus`).
They'll be added here as soon as the Agana runtime supports
`body_encoding: form` — at which point the agent will be able to take new
reservations, change existing ones, and cancel without operator help.

## License

MIT.
