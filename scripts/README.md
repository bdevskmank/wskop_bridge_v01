# scripts/

Install + manifest-generation helpers.

The implementation phase fills this directory. Anticipated
contents:

- `install.sh` — copies the built daemon to `/usr/sbin/`, the
  systemd unit into `/etc/systemd/system/`, the manifest into
  `/etc/wskop-wsi/<instance>.bridge_manifest.json`. Hooks into
  `/opt/wskop-wsi/install.sh` (the host-onboarding workflow per
  [`task_info/tasks/dpi_config_ingest_discovery/08_open_questions.md §10`](https://github.com/bdevskmank/task_info/blob/main/tasks/dpi_config_ingest_discovery/08_open_questions.md)).
- `generate_manifest.py` (or `.sh`) — per
  [`../docs/DESIGN.md`](../docs/DESIGN.md) §05 §6 (hybrid manifest
  generation). Reads `wskop.c` for `metrics_register` call sites,
  merges with hand-written semantics, emits the per-instance JSON.
