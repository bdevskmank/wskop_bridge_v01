# daemon/

Bridge daemon source.

The implementation phase fills this directory. The design is in
[`../docs/DESIGN.md`](../docs/DESIGN.md) §02 (bridge daemon) and
§03 (wire protocol).

## Expected entry points (per DESIGN.md §02)

- A `main.c` (or `main.rs` / `main.go`) that:
  1. Parses CLI args: `--instance`, `--metrics-base`, `--socket`,
     `--manifest`.
  2. Loads the manifest file.
  3. Binds the UNIX socket at the given path with mode 0660.
  4. Runs the single-threaded `epoll` loop.
- Helpers for: reading the metrics tree, parsing wire-protocol
  frames, formatting responses, scheduling subscribes via
  `timerfd`.

## Non-goals

- No writes to the metrics tree.
- No DPDK linkage.
- No HTTP server, no Redis client.
- No authentication beyond filesystem perms.

## Implementation language

TBD per [`../docs/DESIGN.md`](../docs/DESIGN.md) Open Questions
§Q2. C, Rust, or Go.
