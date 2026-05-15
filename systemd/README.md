# systemd/

Template unit files for the bridge.

Anticipated:

- `wskop-bridge@.service` — per
  [`../docs/DESIGN.md`](../docs/DESIGN.md) §02 §4. Template
  instance unit; one per wskop-wsi instance.
- `wskop-bridge.path` (optional) — if the bridge wants to
  socket-activate; not in v1.

The unit is `PartOf=wskop-wsi@%i.service`, `After=` and
`Requires=` the same. See §02 §4 of the design for the exact
shape.
