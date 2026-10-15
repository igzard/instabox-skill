# instabox HTTP API

Base URL: `https://api.instabox.sh` (or `$INSTABOX_API_URL`). Full reference: https://instabox.sh/docs/api

## Rules

- Auth: `X-API-Key: ibk_...` (or `Authorization: Bearer ibk_...`).
- Lifecycle calls (create, start, stop, suspend, resume, delete) return an **operation**. Poll it until `succeeded` or `failed` before the next change to that machine — one operation per machine at a time (`409` otherwise).
- Lifecycle calls require an `Idempotency-Key` header (missing → `400`). Reuse a key only to retry the same request.
- `exec` takes no Idempotency-Key: a retry runs the command again.

## Examples

```sh
H="X-API-Key: $INSTABOX_API_KEY"
API="${INSTABOX_API_URL:-https://api.instabox.sh}"

# create (templateId or snapshotId)
curl -s -H "$H" -H "Idempotency-Key: $(uuidgen)" -H 'Content-Type: application/json' \
  -d '{"name": "box", "templateId": "universal", "timeoutSeconds": 1800, "vault": false}' \
  "$API/v1/machines"

# long-poll the returned operation
curl -s -H "$H" "$API/v1/operations/op_...?wait=25s"

# run a command (≤ 10 min); returns stdout, stderr, exitCode
curl -s -H "$H" -H 'Content-Type: application/json' \
  -d '{"cmd": "uname -a", "timeoutSeconds": 60}' "$API/v1/machines/m_.../exec"

# delete
curl -s -X DELETE -H "$H" -H "Idempotency-Key: $(uuidgen)" "$API/v1/machines/m_..."
```

## Endpoints

| Purpose | Method + path |
|---|---|
| Create machine | `POST /v1/machines` |
| List (filter by label) | `GET /v1/machines?metadata=key:value` |
| Pause / resume | `POST /v1/machines/{id}/suspend` / `.../resume` |
| Delete | `DELETE /v1/machines/{id}` |
| Run command | `POST /v1/machines/{id}/exec` |
| SSH certificate (send public key) | `POST /v1/machines/{id}/connections` |
| Start desktop | `POST /v1/machines/{id}/desktop` body `{"action":"start","width":1280,"height":800}` |
| Desktop stream link / revoke | `POST` / `DELETE /v1/machines/{id}/desktop/stream` body `{"viewOnly":true,"ttlSeconds":3600}` |
| Save snapshot | `POST /v1/machines/{id}/snapshots` |
| List snapshots | `GET /v1/snapshots` |
| Metrics | `GET /v1/machines/{id}/metrics` |
| Machine usage | `GET /v1/machines/{id}/usage` |
| Account usage & limits | `GET /v1/usage` |
| Operation status | `GET /v1/operations/{op}?wait=25s` |

Start from a snapshot: `POST /v1/machines` with `"snapshotId"`.

## Error codes

`missing_api_key`, `insufficient_scope` (403), `not_found`, `operation_conflict` (409), `quota_exceeded` (429, retryable for exec concurrency), `spec_not_allowed` (400: size fields on the hosted service), `no_capacity` (failed create operation).
