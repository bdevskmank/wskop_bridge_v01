# manifests/

Example / template per-instance manifests.

The runtime contract publishes `/etc/wskop-wsi/<instance>.bridge_manifest.json`
per [`../docs/DESIGN.md`](../docs/DESIGN.md) §05. This directory
holds:

- `example.bridge_manifest.json` — illustrative manifest with the
  counter set from §05 §7 of the design. Useful for tests and as
  a reference for the implementation phase's manifest generator.
- (When (c) hybrid generation lands per design §05 §6) the
  hand-written portions of the manifest live here; the generator
  reads them, fills in the path list from `wskop.c`, and produces
  the final file under `/etc/wskop-wsi/`.
