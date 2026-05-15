# proto/

Wire-protocol schema files.

Defined in [`../docs/DESIGN.md`](../docs/DESIGN.md) §03.

## Suggested contents

- `wire-protocol.v1.md` — narrative spec (copy of §03 above).
- `bridge_manifest.schema.json` — JSON Schema for the
  per-instance manifest file (
  `/etc/wskop-wsi/<instance>.bridge_manifest.json` per
  DESIGN.md §05).
- `protocol-v1.schema.json` — JSON Schema describing each
  request / response shape, importable by both the bridge daemon
  and the UI's TypeScript / Python / etc. consumer code.

The protocol versions and the manifest schema versions are
**independent** per DESIGN.md §05 §5 — bumping a counter
description doesn't affect `proto/protocol-v1.schema.json`.
