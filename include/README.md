# include/

Shared headers between `daemon/` and `test/`.

The implementation phase fills this with:

- `bridge_protocol.h` — request/response struct shapes, error
  code enum, schema-version constant.
- `bridge_paths.h` — canonical path-string constants
  (`workers/lcore_%d/in_pkts_total`, etc.) so the daemon and the
  test harness agree.
- `manifest.h` — manifest-loading interface.
