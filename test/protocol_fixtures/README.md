# test/protocol_fixtures/

Golden-fixture protocol exchanges.

Per [`../../docs/DESIGN.md`](../../docs/DESIGN.md) §07 §8 — every
request/response pair lives as a small JSON file. The bridge
daemon runs against them via a mock-stdin / loopback-socket
harness; the UI's protocol-writer code uses the same fixtures
as test inputs.

## File layout

```
test/protocol_fixtures/
├── ping_pong.json         - kind="ping" → kind="pong"
├── get_basic.json         - kind="get" → kind="values"
├── manifest.json          - kind="manifest" → kind="manifest"
├── list_workers.json      - kind="list" prefix="workers/" → kind="list"
├── subscribe_basic.json   - subscribe → subscribed → values events
├── error_unknown_kind.json - kind="frobnicate" → kind="error"
├── error_proto_v.json     - v=99 → kind="error" code=protocol_version_mismatch
└── ...
```

Each fixture file is itself JSON:

```json
{
  "name": "ping_pong",
  "description": "Basic liveness check.",
  "exchanges": [
    { "direction": "client_to_server",
      "frame": { "v": 1, "id": "h", "kind": "ping" } },
    { "direction": "server_to_client",
      "frame": { "v": 1, "id": "h", "kind": "pong",
                 "epoch_ms": "<any uint64>",
                 "schema_version": "<any string>",
                 "wskop_version": "<any string>" } }
  ]
}
```

The harness wildcard-matches on `<any …>` fields so the fixtures
do not need exact-equality against time-varying values.

## When to add a fixture

- Every new request kind gets one.
- Every documented error code gets one.
- Every wire-format edge case (>2^53 uint, null value, missing
  path) gets one.
