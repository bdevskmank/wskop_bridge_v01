# wskop_bridge_v01

The wskop **runtime-data bridge daemon**.

This is a satellite of [`wskop_v01`](https://github.com/bdevskmank/wskop_v01)
that ships the consumer side of the **DPI runtime data contract**:

- Reads `/mnt/tmp_ramfs/run/<pci_id>/...` produced by wskop's
  `ssg_src/metrics/` subsystem and (in v2) the broker's streaming
  output under `streams/`.
- Listens on a UNIX socket at `/run/wskop-wsi/<instance>/bridge.sock`
  that a UI container bind-mounts.
- Speaks the wire protocol documented in
  [`docs/DESIGN.md`](docs/DESIGN.md) (length-prefixed JSON).

The producer side (wskop's `ssg_src/metrics/` subsystem and
upcoming counter additions) lives in
[`wskop_v01`](https://github.com/bdevskmank/wskop_v01).
This repo does **not** vendor wskop_v01 as a submodule — the
bridge has no symbol-link dependency on wskop. It consumes the
on-disk format that wskop already produces and the manifest the
implementation phase ships at install time.

## Status

**Design** — see [`docs/DESIGN.md`](docs/DESIGN.md). Implementation
follows in a subsequent task.

The design is the same nine files as
[`task_info/tasks/dpi_runtime_data_contract_design/`](https://github.com/bdevskmank/task_info/tree/main/tasks/dpi_runtime_data_contract_design),
copied here as a single concatenated document under
[`docs/DESIGN.md`](docs/DESIGN.md) so contributors to this repo
have everything in one place.

## Layout

```
wskop_bridge_v01/
├── README.md             - this file
├── docs/
│   └── DESIGN.md         - concatenated design from task_info, kept in sync
├── daemon/               - bridge daemon source (TBD; see DESIGN.md §02)
├── proto/                - wire-protocol schema files (TBD; see DESIGN.md §03)
├── include/              - shared headers between daemon/ and test/
├── manifests/            - example bridge_manifest.json (per DESIGN.md §05)
├── systemd/              - systemd template unit (TBD; see DESIGN.md §02 §4)
├── scripts/              - install.sh helpers + manifest generator
└── test/
    └── protocol_fixtures/ - golden JSON fixtures for the wire protocol
```

## Building

(TBD — implementation phase chooses C / Rust / Go per
[`docs/DESIGN.md`](docs/DESIGN.md) "open questions" §Q2.)

## License

See [`LICENSE`](LICENSE) (BSD-3-Clause, matching wskop_v01).

## Cross-references

- Producer side: https://github.com/bdevskmank/wskop_v01 (HEAD
  `3ff7479` at design time)
- Discovery report:
  https://github.com/bdevskmank/task_info/tree/main/tasks/dpi_runtime_data_contract_discovery
- Design report:
  https://github.com/bdevskmank/task_info/tree/main/tasks/dpi_runtime_data_contract_design
- Cross-task coordination:
  https://github.com/bdevskmank/task_info
