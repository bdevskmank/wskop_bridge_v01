# wskop_bridge_v01 — Design (concatenated)

This document is the same nine-file design that lives in the
[`task_info` repo](https://github.com/bdevskmank/task_info/tree/main/tasks/dpi_runtime_data_contract_design),
concatenated into a single file for contributors of this satellite
repo.

The canonical source remains the per-file design in `task_info`.
When the two get out of sync, `task_info` wins; update this file
to match.

---


---

# DPI Runtime Data Contract — Design report

How wskop exposes its polled runtime data to the UI today (**v1**),
and what the streaming-watch layer looks like once the broker
lcore lands (**v2**).

## Status

In progress / review-ready. See
[`../dpi_runtime_data_contract_design.md`](../dpi_runtime_data_contract_design.md)
for live snapshot and
[`../../logs/dpi_runtime_data_contract_design.log.md`](../../logs/dpi_runtime_data_contract_design.log.md)
for the narrative.

## How to read this directory

Files **01–06 are descriptive design.** File **07 is opinion.**
File **08 is open questions** for the implementation phase or the
operator. Every nontrivial code claim has a `wskop/<path>:<line>`
citation against `wskop_v01@3ff7479` (current HEAD at the time of
writing). Pseudocode is in markdown only and is clearly marked as
pseudocode — never as quoted code.

| # | File | Topic |
|---|------|-------|
| 00 | this file | Index / reading order |
| 01 | [01_v1_producer_additions.md](01_v1_producer_additions.md) | Counters to add to the metrics ramfs tree |
| 02 | [02_bridge_daemon_design.md](02_bridge_daemon_design.md) | Sibling-systemd bridge daemon — lifetime, concurrency, failure modes |
| 03 | [03_wire_protocol.md](03_wire_protocol.md) | UNIX-socket wire protocol — grammar, versioning, examples |
| 04 | [04_watched_flag_and_propagation.md](04_watched_flag_and_propagation.md) | `policy_info` bit 38 claim, writer/reader, hazard posture |
| 05 | [05_counter_semantics_manifest.md](05_counter_semantics_manifest.md) | Counter meanings, known anomalies, manifest.json |
| 06 | [06_v2_streaming_layer.md](06_v2_streaming_layer.md) | Per-worker watch-emit ring, broker, v1↔v2 bridge |
| 07 | [07_observations.md](07_observations.md) | Agent's opinions, clearly labelled |
| 08 | [08_open_questions.md](08_open_questions.md) | For implementation phase / operator |
| — | [refs/](refs/) | Verbatim code excerpts cited above (beyond what discovery pinned) |

## Reading order

1. **First-time reader:** start with the discovery
   [`00_README.md`](../dpi_runtime_data_contract_discovery/00_README.md)
   for context; then come back here and read 01 → 02 → 03 → 06 →
   04 → 05 → 09. (The hot-path impact statement is inside file 06
   §6 and re-summarised across 01 and 04.)
2. **Implementation handoff:** 01 (what to write), 02–03 (the new
   daemon and its protocol), 04 (the bit), 05 (what to publish in
   the manifest), 06 (v2 staging), then 08 (questions to resolve
   before coding).
3. **Operator review:** 02 (systemd shape), 05 (consumer-facing
   manifest), 07 (opinion).

## Repos and hosts referenced

- `wskop_v01` — main repo. Local read-only clone at
  `/home/sekom/dev/wskop_v01_reference`, HEAD `3ff7479` on `main`.
- `wskop_bridge_v01` — **new satellite repo** at
  `github.com/bdevskmank/wskop_bridge_v01`, created by this task.
  The bridge daemon and its tests live there; the design report
  in this directory is duplicated into the satellite as
  `docs/DESIGN.md`.
- `10.106.60.72` (dev rig, identical to production `.111`) — used
  for read-only observation. No service restart, no config change.

## What this report does and does not do

- **Does** specify what counters to add and where, the bridge
  daemon's process shape and wire protocol, the v1↔v2 split, the
  watched-bit choice and propagation, and a staging plan for
  implementation.
- **Does not** write `.c` or `.h` files in `wskop_v01`. Pseudocode
  only. The satellite `wskop_bridge_v01` skeleton ships with
  directory structure and the design doc; first runnable code is
  the implementation phase.
- **Does not** re-design `ssg_src/metrics/` — extends it. The
  producer-side `metrics_set` / `metrics_increment` calls land in
  the existing per-role registration blocks (see
  [01](01_v1_producer_additions.md)).
- **Does not** propose a Redis mirror, an in-wskop HTTP server,
  or anything that requires the UI container to mount
  `/mnt/tmp_ramfs`. Consumer-side surface is a UNIX socket bind-
  mounted into the container (operator A5 in discovery 08).
- **Does not** clone packets. Counters only, per operator A6.

---

# 01 — v1 producer-side counter additions

The producer side of v1 is a small extension of
[`ssg_src/metrics/`](https://github.com/bdevskmank/wskop_v01/tree/main/ssg_src/metrics).
No new mechanism — every addition is a one-line
`metrics_register` at lcore entry plus a one-line `metrics_set` or
`metrics_increment` at the existing cold-path call site. None of
the additions land on the worker hot path.

All citations are against `wskop_v01@3ff7479`. Where the discovery
report cited `13348af`, the equivalent HEAD line is given inline.

## 0. Counter ordering and where calls go

The producer pattern at HEAD looks identical for all four roles —
worker, shaper, watch, system — and is established by the existing
registrations:

```
/* PSEUDOCODE — pattern, not a copy of code */
metrics_handle_t *h = metrics_open(pci_id, ROLE, lcore_id);
int idx_X = metrics_register(h, "X");          // once at lcore entry
int idx_Y = metrics_register(h, "Y");          // once at lcore entry
...
/* On periodic / event-driven cold-path site: */
metrics_set(h, idx_X, value);                  // ramfs write
metrics_increment(h, idx_Y, delta);            // ramfs write
...
metrics_close(h);                              // lcore shutdown
```

The `_open` / `_register` / `_close` calls already exist at:

| Role | `metrics_open` | `metrics_register` block | `metrics_close` |
|---|---|---|---|
| worker | [wskop.c:5515](https://github.com/bdevskmank/wskop_v01/blob/3ff7479/wskop.c#L5515) | [wskop.c:5517-5520](https://github.com/bdevskmank/wskop_v01/blob/3ff7479/wskop.c#L5517-L5520) | [wskop.c:5600](https://github.com/bdevskmank/wskop_v01/blob/3ff7479/wskop.c#L5600) |
| shaper | [wskop.c:5845](https://github.com/bdevskmank/wskop_v01/blob/3ff7479/wskop.c#L5845) | [wskop.c:5847-5850](https://github.com/bdevskmank/wskop_v01/blob/3ff7479/wskop.c#L5847-L5850) | [wskop.c:6172](https://github.com/bdevskmank/wskop_v01/blob/3ff7479/wskop.c#L6172) |
| watch | [wskop.c:6591](https://github.com/bdevskmank/wskop_v01/blob/3ff7479/wskop.c#L6591) | [wskop.c:6593-6595](https://github.com/bdevskmank/wskop_v01/blob/3ff7479/wskop.c#L6593-L6595) | [wskop.c:6898](https://github.com/bdevskmank/wskop_v01/blob/3ff7479/wskop.c#L6898) |
| system (main lcore) | [wskop.c:7628](https://github.com/bdevskmank/wskop_v01/blob/3ff7479/wskop.c#L7628) | [wskop.c:7630-7631](https://github.com/bdevskmank/wskop_v01/blob/3ff7479/wskop.c#L7630-L7631) | [wskop.c:7930](https://github.com/bdevskmank/wskop_v01/blob/3ff7479/wskop.c#L7930) |

Every new counter slots into one of these four registration blocks
and reuses the existing handle. `METRICS_MAX_COUNTERS_PER_HANDLE`
is 32 ([metrics.h:30](https://github.com/bdevskmank/wskop_v01/blob/3ff7479/ssg_src/metrics/metrics.h#L30));
worker has 4 / shaper 4 / watch 3 / system 2 registered today, so
there is room for the additions in §1 below without raising the
cap.

Transmit is **not** registered for metrics today (verified —
[02 §3](../dpi_runtime_data_contract_discovery/02_lcore_observability_responsibilities.md) of the discovery,
re-verified at HEAD by searching `transmit_core_function_dpi_3`'s
body for `metrics_` — no hits between `wskop.c:6200` and
`wskop.c:6440`). Two of the additions below require opening a
`transmit` handle for the first time; see §3.

## 1. Counters to add — list

Closes the gaps in
[discovery 06 §4](../dpi_runtime_data_contract_discovery/06_ui_data_needs_vs_current_exposure.md)
and folds in the easy wins of
[discovery 07 §8](../dpi_runtime_data_contract_discovery/07_observations.md).
Each row: lcore (always cold), path under `/mnt/tmp_ramfs/run/<pci_id>/`,
units, cadence, source, increment-site pseudocode.

### 1.1 Per-lcore traffic stats (closes discovery 06 §1.3 / §1.4)

Source is the existing per-lcore-owned `core_stats[]` array
([wskop.c:5305](https://github.com/bdevskmank/wskop_v01/blob/3ff7479/wskop.c#L5305)).
Worker/shaper/transmit lcores already write their own slots on the
hot path (per-pkt and per-burst — see
[discovery 04 §1](../dpi_runtime_data_contract_discovery/04_metrics_and_counters.md)).
The new counters **read** `core_stats[self]` on a cold-path tick
and copy the value into ramfs. **No additional hot-path work.**

| Path (per role) | Source (cold-path read) | Units | Cadence | Notes |
|---|---|---|---|---|
| `workers/lcore_<N>/in_pkts_total` | `core_stats[self].in_pkts` | packets | 1 s | self-read on each lcore's existing 1 s branch (see §2 below) |
| `workers/lcore_<N>/out_pkts_total` | `core_stats[self].out_pkts` | packets | 1 s | same |
| `workers/lcore_<N>/out_drop_pkts_total` | `core_stats[self].out_drop_pkts` | packets | 1 s | same |
| `shapers/lcore_<N>/in_pkts_total` | `core_stats[self].in_pkts` | packets | 1 s | new shaper-side periodic branch (§2) |
| `shapers/lcore_<N>/out_pkts_total` | `core_stats[self].out_pkts` | packets | 1 s | same |
| `shapers/lcore_<N>/out_drop_pkts_total` | `core_stats[self].out_drop_pkts` | packets | 1 s | same |
| `transmit/lcore_<N>/in_pkts_total` | `core_stats[self].in_pkts` | packets | 1 s | requires opening a `transmit` handle (§3) |
| `transmit/lcore_<N>/out_pkts_total` | `core_stats[self].out_pkts` | packets | 1 s | same |
| `transmit/lcore_<N>/out_drop_pkts_total` | `core_stats[self].out_drop_pkts` | packets | 1 s | same |

**Cost.** Cold path only. Each role reads its own `core_stats`
slot, formats decimal, `pwrite`+`ftruncate`. No load on the
per-packet path.

### 1.2 Per-port DPDK NIC stats (the easy win from discovery 07 §8)

DPDK already maintains these in `rte_eth_stats`. Wskop calls
`rte_eth_stats_get` nowhere on a periodic path today (verified by
grep — only a stale `struct rte_eth_stats stats;` declaration at
[wskop.c:7819](https://github.com/bdevskmank/wskop_v01/blob/3ff7479/wskop.c#L7819)).
Watch lcore reads each port's stats on its 1 s cold branch and
publishes them under `system/port_<N>/`.

| Path | Source | Units | Cadence |
|---|---|---|---|
| `system/port_<P>/rx_pkts` | `rte_eth_stats.ipackets` | packets | 1 s (watch) |
| `system/port_<P>/tx_pkts` | `rte_eth_stats.opackets` | packets | 1 s (watch) |
| `system/port_<P>/rx_bytes` | `rte_eth_stats.ibytes` | bytes | 1 s (watch) |
| `system/port_<P>/tx_bytes` | `rte_eth_stats.obytes` | bytes | 1 s (watch) |
| `system/port_<P>/rx_drop_nic` | `rte_eth_stats.imissed + rte_eth_stats.ierrors` | packets | 1 s (watch) |
| `system/port_<P>/tx_drop_nic` | `rte_eth_stats.oerrors` | packets | 1 s (watch) |

**Why `system/port_<P>/`** and not `system/<counter>`: the system
handle already owns ports (`pci_id` plus its peer subscriber/wan
port). Two ports are configured on the dev rig (see
[dpi_config_ingest_discovery/01_config_file_inventory.md §nic](../dpi_config_ingest_discovery/01_config_file_inventory.md)).
A flat `system/<counter>` namespace conflates them; per-port
subdirectories keep the surface obvious.

**Where the watch lcore opens the handles.** The system handle is
opened on the main lcore at startup
([wskop.c:7628](https://github.com/bdevskmank/wskop_v01/blob/3ff7479/wskop.c#L7628)).
That handle's fds are owned by the main lcore — wrong for watch
to write to per `metrics.h:42-44`. The implementation has two
choices (decided in [08 Q3](08_open_questions.md#q3)): either
(a) extend `metrics_open` to accept a sub-role
(`system/port_<P>`) and have the watch lcore open its own handle
at watch-entry, or (b) put the per-port counters on a fresh
top-level role `"ports"` (path `/mnt/tmp_ramfs/run/<pci_id>/ports/port_<P>/`)
that watch owns alone. **(b) is the cleaner default;** it keeps
`system/` as "info written once by main" and `ports/` as "stats
published periodically by watch."

For the rest of this document, the chosen path is
`ports/port_<P>/<counter>` and the role string is `"ports"`. If
the implementation picks (a), the manifest's path strings update
accordingly; no protocol change.

**Cost.** Cold path. `rte_eth_stats_get` is documented as racy-
relaxed; the call itself is a few hundred cycles per port. Two
ports × six counters × 1 Hz = trivial on the watch lcore.

### 1.3 Mempool fill (closes discovery 06 §1.2)

Already accessible via `rte_mempool_avail_count` and
`rte_mempool_in_use_count`. Two existing call sites use them for
stderr diagnostics:
[flow_mng_aging.c:365-366](https://github.com/bdevskmank/wskop_v01/blob/3ff7479/ssg_src/ffunctions/flow_mng_aging.c#L365-L366),
[flow_mng.c:425-427](https://github.com/bdevskmank/wskop_v01/blob/3ff7479/ssg_src/ffunctions/flow_mng.c#L425-L427),
[subsman_2.c:537-548](https://github.com/bdevskmank/wskop_v01/blob/3ff7479/ssg_src/subsman/subsman_2.c#L537-L548).
The watch lcore reads them on a 1 s cold branch and publishes them
under `pools/<pool_name>/`.

The pool names known at HEAD (from those call sites): the
per-worker `flow_info_pool` family, `subscriber_info_pool`,
`flow_ref_pool`, and the various aging-side pools. The exact list
must be enumerated by the implementation phase against
`pools[]` arrays in `flow_mng.c`. The contract publishes a
single per-pool path scheme; pool naming follows whatever the
binary already names them.

| Path | Source | Units | Cadence |
|---|---|---|---|
| `pools/<pool_name>/avail` | `rte_mempool_avail_count(pool)` | objects | 1 s (watch) |
| `pools/<pool_name>/in_use` | `rte_mempool_in_use_count(pool)` | objects | 1 s (watch) |
| `pools/<pool_name>/capacity` | `avail + in_use` (computed once at startup; static) | objects | once (main) |

**Hot path cost.** None. Same `core_stats`-style cold publish.

### 1.4 Heartbeat / loop EMA (closes discovery 06 §1.1)

Per-lcore liveness today reaches only `cmon_dump_all` →
stdout/journal
([core_monitor.h:597](https://github.com/bdevskmank/wskop_v01/blob/3ff7479/ssg_src/profiling/core_monitor.h#L597)).
Watch lcore reads `g_core_status[lcore_id]` for every registered
lcore on its 1 s branch and publishes them.

| Path | Source | Units | Cadence |
|---|---|---|---|
| `workers/lcore_<N>/heartbeat` | `g_core_status[N].heartbeat` | counter | 1 s (watch) |
| `workers/lcore_<N>/ema_loop_cycles_fp` | `g_core_status[N].ema_loop_cycles_fp` | fp-cycles | 1 s (watch) |
| `shapers/lcore_<N>/heartbeat` | same | — | 1 s (watch) — **needs `cmon_register_core` on shaper entry** (see §4) |
| `shapers/lcore_<N>/ema_loop_cycles_fp` | same | — | 1 s (watch) |
| `transmit/lcore_<N>/heartbeat` | same | — | 1 s (watch) — needs `cmon_register_core` on transmit entry |
| `transmit/lcore_<N>/ema_loop_cycles_fp` | same | — | 1 s (watch) |
| `watch/heartbeat` | `g_core_status[<watch lcore>].heartbeat` | counter | 1 s (watch self) |
| `watch/ema_loop_cycles_fp` | `g_core_status[<watch lcore>].ema_loop_cycles_fp` | fp-cycles | 1 s (watch self) |

**Why these are cross-lcore safe to read.** `cmon` heartbeat is
atomic-relaxed
([core_monitor.h:213](https://github.com/bdevskmank/wskop_v01/blob/3ff7479/ssg_src/profiling/core_monitor.h#L213));
EMA is a plain store but the watch lcore tolerates a 1-tick-old or
mid-update read (same posture as `cmon_dump_all` already takes).

**Hot path cost.** None on workers/shapers/transmit — they already
do `CMON_LOOP_TOP`/`_BOTTOM` per loop iteration; nothing
changes per packet. The watch lcore picks up the values cross-
lcore.

### 1.5 Subscriber count (closes a small piece of discovery 06 §3)

UI's "subscriber count on this node" needs a single integer:
`rte_hash_count(subs_w_prbs)`. Watch lcore reads on 1 s branch.
Path: `system/subscriber_count`. The handle to use is the
`system` handle, but per the cross-lcore-fd rule we instead
register `subscriber_count` on the **watch** handle and place it
at `watch/subscriber_count`. (The handles do not collide with
existing names.)

| Path | Source | Units | Cadence |
|---|---|---|---|
| `watch/subscriber_count` | `rte_hash_count(subs_w_prbs)` | subscribers | 1 s (watch) |
| `watch/flow_count_total` | `sum(rte_hash_count(ff_ht_<wid>))` over workers | flows | 1 s (watch) |

### 1.6 Watched-flow count (preparing for the watched bit — see [04](04_watched_flag_and_propagation.md))

Once the watched bit lands, the UI wants to know how many flows
are currently watched. Watch lcore walks `subs_w_prbs` plus the
per-worker collector tables on its existing 1 s consolidation
tick and counts matches.

| Path | Source | Units | Cadence |
|---|---|---|---|
| `watch/watched_flow_count` | walk over collector tables, count entries whose `policy_info & (1 << WATCHED_BIT)` | flows | 1 s (watch) |
| `watch/watched_subscriber_count` | walk over `subs_w_prbs`, count entries with `watched=true` | subscribers | 1 s (watch) |

If [04](04_watched_flag_and_propagation.md)'s subscriber-side
"watched" flag lives on a new field on `subsc_w_prb_ref` rather
than a bit on `flow_info_d->policy_info`, the second source path
changes; the contract path string stays the same.

## 2. Where the cold publish hooks land in HEAD

### 2.1 Worker (no hot path)

The worker already has a 1 s branch inside its main loop at
[wskop.c:5547-5552](https://github.com/bdevskmank/wskop_v01/blob/3ff7479/wskop.c#L5547-L5552)
where `metrics_set(worker_metrics, m_flow_table_occ, …)` is
called. New worker counters in §1.1 land in the same `if (
1_second_tick )` block; pseudocode:

```
/* PSEUDOCODE — placed inside the existing 1s tick at wskop.c:5547+ */
metrics_set(worker_metrics, m_in_pkts_total,        core_stats[self].in_pkts);
metrics_set(worker_metrics, m_out_pkts_total,       core_stats[self].out_pkts);
metrics_set(worker_metrics, m_out_drop_pkts_total,  core_stats[self].out_drop_pkts);
metrics_set(worker_metrics, m_flow_table_occ,       (uint64_t)rte_hash_count(ff_ht_self));  /* existing call */
```

**Per-packet cost: zero.** All inside the cold branch.

### 2.2 Shaper (no hot path)

The shaper main loop has no 1 s tick today. Add one alongside the
existing `metrics_increment` sites at
[wskop.c:6036-6042](https://github.com/bdevskmank/wskop_v01/blob/3ff7479/wskop.c#L6036-L6042).
Pseudocode: maintain a `last_publish_tsc` local variable; on each
loop iteration, branch only if
`(rte_get_timer_cycles() - last_publish_tsc) > tsc_hz`; this is
the same pattern the watch lcore uses with
`check_time_interval_precise_ts`. The branch is hit at most once
per second; mispredict cost amortises to zero. Single load + one
predicted-not-taken branch on the loop iteration boundary, not on
the per-packet path.

```
/* PSEUDOCODE — added near top of shaper main loop */
if (unlikely(rte_get_timer_cycles() - last_publish_tsc > tsc_hz)) {
    metrics_set(shaper_metrics, m_in_pkts_total,       core_stats[self].in_pkts);
    metrics_set(shaper_metrics, m_out_pkts_total,      core_stats[self].out_pkts);
    metrics_set(shaper_metrics, m_out_drop_pkts_total, core_stats[self].out_drop_pkts);
    last_publish_tsc = rte_get_timer_cycles();
}
```

**Per-packet cost: ~1 cycle** (`unlikely` branch on TSC delta;
TSC read is a single rdtscp / rdtsc, ~10-20 cycles, but only on
the branch-not-taken outer loop turn, not per packet). The B2.1
budget per
[discovery 05 §B.1](../dpi_runtime_data_contract_discovery/05_flow_watch_cost_analysis.md)
absorbs this.

### 2.3 Transmit (no hot path; new handle)

Same shape as shaper §2.2 but requires opening a `transmit`
handle in the lcore entry. Pseudocode:

```
/* PSEUDOCODE — at top of transmit_core_function_dpi_3 (wskop.c:6200) */
metrics_handle_t *transmit_metrics = metrics_open(g_redis_config.pci_id,
                                                  "transmit", rte_lcore_id());
int m_in_pkts_total       = metrics_register(transmit_metrics, "in_pkts_total");
int m_out_pkts_total      = metrics_register(transmit_metrics, "out_pkts_total");
int m_out_drop_pkts_total = metrics_register(transmit_metrics, "out_drop_pkts_total");
/* … (eventually heartbeat + ema_loop_cycles_fp via cmon, see §4 below) */

/* In the main loop, 1s-gated branch as in §2.2 */

/* At lcore shutdown */
metrics_close(transmit_metrics);
```

`metrics_init` will need a `"transmit"` directory at startup. One-
line addition to the `roles[]` array in
[metrics.c:103](https://github.com/bdevskmank/wskop_v01/blob/3ff7479/ssg_src/metrics/metrics.c#L103):
`{ "workers", "shapers", "transmit", "watch", "system" }`. (Or
`"ports"` + `"pools"` per §1.2 / §1.3.) Idempotent `mkdir_p`
already in place.

### 2.4 Watch (no hot path; cross-lcore consumer)

The watch loop already has 1 s / 5 s / 10 s branches via
`check_time_interval_precise_ts`. Add the new per-port / pool /
liveness publishes to its existing 1 s branch. Pseudocode:

```
/* PSEUDOCODE — placed inside the watch lcore's 1s tick branch (near wskop.c:6824+) */
if (_1_seconds_timer) {
    /* per-port */
    for (int p = 0; p < nb_ports; p++) {
        struct rte_eth_stats s; rte_eth_stats_get(p, &s);
        metrics_set(ports_metrics_per_port[p], m_rx_pkts,     s.ipackets);
        metrics_set(ports_metrics_per_port[p], m_tx_pkts,     s.opackets);
        metrics_set(ports_metrics_per_port[p], m_rx_bytes,    s.ibytes);
        metrics_set(ports_metrics_per_port[p], m_tx_bytes,    s.obytes);
        metrics_set(ports_metrics_per_port[p], m_rx_drop_nic, s.imissed + s.ierrors);
        metrics_set(ports_metrics_per_port[p], m_tx_drop_nic, s.oerrors);
    }
    /* per-pool */
    for (each pool) {
        metrics_set(pools_metrics_per_pool[p], m_avail,  rte_mempool_avail_count(pool));
        metrics_set(pools_metrics_per_pool[p], m_in_use, rte_mempool_in_use_count(pool));
    }
    /* per-lcore liveness */
    for (each registered lcore N) {
        metrics_set(per_lcore_liveness_handle[N], m_heartbeat,     g_core_status[N].heartbeat);
        metrics_set(per_lcore_liveness_handle[N], m_ema_loop_fp,   g_core_status[N].ema_loop_cycles_fp);
    }
    /* aggregate counters */
    metrics_set(watch_metrics, m_watched_flow_count,       count_watched_flows());
    metrics_set(watch_metrics, m_watched_subscriber_count, count_watched_subscribers());
    metrics_set(watch_metrics, m_subscriber_count,         rte_hash_count(subs_w_prbs));
    metrics_set(watch_metrics, m_flow_count_total,         sum_of_flow_tables());
}
```

Per `metrics.h:42-44` (the "must be called from the lcore that
will write to it" rule), watch lcore opens each `per_lcore_liveness_handle[N]`
itself at watch-entry. The implementation thus opens
`workers × N + shapers × N + transmit × N` watch-owned handles —
fewer than 32 in practice on the dev rig (8 workers × 2 = 16
handles; cap is per-handle, not per-lcore, so this is fine).

## 3. The two structural changes (small)

1. **`metrics_init` learns two more role directories.** One-line
   addition to the `roles[]` array at
   [metrics.c:103](https://github.com/bdevskmank/wskop_v01/blob/3ff7479/ssg_src/metrics/metrics.c#L103):
   add `"transmit"`, `"ports"`, `"pools"`. Idempotent `mkdir_p`
   handles the additions.
2. **Transmit lcore opens a `metrics_handle_t`.** New
   `metrics_open` + `metrics_register` block at the top of
   `transmit_core_function_dpi_3`
   ([wskop.c:6200](https://github.com/bdevskmank/wskop_v01/blob/3ff7479/wskop.c#L6200))
   and a matching `metrics_close` before the function returns.

That is the entirety of structural change in v1's producer side.
Everything else is `metrics_set`/`metrics_increment` calls slotted
into already-cold lcore branches.

## 4. The shaper + transmit `cmon` registration gap

Per [discovery 02 §2](../dpi_runtime_data_contract_discovery/02_lcore_observability_responsibilities.md)
the shaper does **not** call `cmon_register_core` and therefore
its `g_core_status` slot is uninitialised; the same is true for
transmit. Re-verified at HEAD by searching
`shaper_core_function_dpi_3` and `transmit_core_function_dpi_3`
bodies for `cmon_register_core` — no hits.

The §1.4 entries `shapers/lcore_<N>/heartbeat`,
`transmit/lcore_<N>/heartbeat`, etc. **depend on** the missing
registration. Implementation has two options:

- **(a)** Add `cmon_register_core(rte_lcore_id(), CMON_ROLE_SHAPER, …)`
  at the top of `shaper_core_function_dpi_3` and likewise for
  transmit. This is the same shape as the existing worker
  registration ([wskop.c near 5460s](https://github.com/bdevskmank/wskop_v01/blob/3ff7479/wskop.c#L5460-L5500)).
  Per-loop cost is the existing `CMON_LOOP_TOP` macro path that
  worker already pays; on a hyperthread that is ~3-5 cycles per
  loop iteration (not per packet). Acceptable under the B2.1
  budget if the shaper main loop is not so tight that a single
  atomic-relaxed write per iteration shows up — measurable, not
  obvious.
- **(b)** Skip the liveness counters for shaper / transmit until
  someone fixes `cmon`. The contract publishes
  `shapers/lcore_<N>/heartbeat` as a path that the bridge daemon
  returns with `value=null` and a manifest entry saying
  `status="not_wired"`.

**(b) is the v1 default.** The contract should not ship a counter
that reads stale-zero (i.e. the uninitialised slot value), because
a UI consumer would treat that as "shaper is stuck." Wire-up of
shaper/transmit `cmon` is a separate workstream — flagged in
[08 Q4](08_open_questions.md#q4).

## 5. Counters that stay as-is but get a documented anomaly

Per [discovery 04 §7](../dpi_runtime_data_contract_discovery/04_metrics_and_counters.md)
several already-registered handles are not incremented in HEAD:
`packets_unattributed_table_full`, `packets_dropped_tx_queue_full`,
`ring_polls_empty`, `ring_polls_full`, `ndpi_pool_high_water`.
Re-verified at HEAD by searching for `metrics_increment` calls on
each handle ID — no hits found in `wskop.c` for any of these names.

Operator A7 in discovery 08 says **"zero for now, may be used
later"** — placeholders. The contract therefore continues to
publish their paths but the **manifest documents each with
`status="expected_zero"`** so the UI does not invite alarm on
them. Details and full text in [05 §3](05_counter_semantics_manifest.md).

## 6. Naming policy

All new counter file names use `snake_case`, lowercase, no spaces.
Where a counter is a monotonic count of events, the name ends in
`_total` (the Prometheus-conventional suffix; matches what a
future scraper would expect and reads naturally inside the bridge
daemon's wire output too). The existing counters
(`flow_table_occupancy`, `fall_through_count`, etc.) do **not**
follow this rule. v1 does not rename them — backward-compat with
whichever observer might already be reading the ramfs path is
more valuable than a uniform naming convention. The manifest in
[05](05_counter_semantics_manifest.md) flags each existing counter
that does not follow `_total` with a `legacy_name=true` field.

## 7. Staging — what lands when

Per the "stage independently, no stacking on a regressed stage"
brief item:

| Stage | Adds | Wskop-side change | Hot-path? | Acceptance |
|---|---|---|---|---|
| **S1** | `metrics_init` roles array (one line); empty directories appear on rig | one-line edit to [metrics.c:103](https://github.com/bdevskmank/wskop_v01/blob/3ff7479/ssg_src/metrics/metrics.c#L103) | no | rig shows `transmit/`, `ports/`, `pools/` dirs created at startup |
| **S2** | per-lcore `core_stats` publishes for worker + shaper | shaper gets a 1 s gated branch; both use existing handle | no (1 s gated branch is `unlikely`-marked) | `workers/lcore_<N>/in_pkts_total` populates on rig; throughput regression A/B vs. `pre-S2` baseline ≤ noise floor |
| **S3** | transmit handle + per-transmit-lcore counters | new `metrics_open`/`_register`/`_close` block in transmit lcore | no (same 1 s branch pattern) | `transmit/lcore_<N>/...` populates; throughput A/B vs. `pre-S3` ≤ noise |
| **S4** | per-port DPDK stats + per-pool mempool stats | additions to watch 1 s branch only | no (cold path) | `ports/port_<P>/...`, `pools/<name>/...` populate |
| **S5** | cross-lcore liveness publishes via watch | additions to watch 1 s branch | no | `workers/lcore_<N>/heartbeat`, etc. tick monotonically |
| **S6** | aggregate counters `watched_flow_count`, `watched_subscriber_count`, `subscriber_count`, `flow_count_total` | additions to watch 1 s branch | no | counts match independent verification (e.g. `redis-cli HLEN`) |

Each stage independently committable in `wskop_v01`. None depends
on the watched-bit work in [04](04_watched_flag_and_propagation.md)
except S6's `watched_*` rows — those land *after* the watched bit
lands. The bridge daemon ([02](02_bridge_daemon_design.md)) and
its protocol ([03](03_wire_protocol.md)) live in
`wskop_bridge_v01` and are independently committable.

The implementation phase must run a throughput A/B against the
`pre-b1-cleanup-complete` baseline tag per the brief; see
[06 §6](06_v2_streaming_layer.md) for the cross-cutting hot-path
impact statement and acceptance criterion.

---

# 02 — Bridge daemon design

The bridge daemon serves the consumer side of v1: it listens on a
UNIX socket that the UI container bind-mounts, reads
`/mnt/tmp_ramfs/run/<pci_id>/...` on demand, and answers requests
in the wire protocol defined in
[03](03_wire_protocol.md).

Per operator A5 in
[discovery 08 Q5](../dpi_runtime_data_contract_discovery/08_open_questions.md#q5):
the UI is a container-wrapped OS service that **cannot** mount
tmp_ramfs directly. UNIX socket is the chosen surface. No Redis
mirror, no HTTP server, no ramfs mount into the container.

## 0. Implementation language

**Decision (operator A2, 2026-05-15): C.** Same toolchain as
wskop_v01. The bridge has no DPDK dependency (it links neither
`librte_*` nor any wskop symbol), but a C codebase lets the same
build chain produce the satellite binary and reuses operator
familiarity with the project's idioms. The satellite repo's
`daemon/` directory is structured as a C source tree.

The contract itself is language-agnostic; if a future v2 of the
bridge wants to switch to Rust or Go for the smaller bug surface
on JSON parsing, the wire protocol's externally-observable
behaviour stays the same. v1 is C.

## 1. Lifetime: sibling systemd service

Two candidate shapes:

- **(A)** In-process inside `wskop` on a cold lcore. New lcore
  function alongside `watch_core_function_3`, registered the same
  way; opens the socket, reads the metrics tree from the same
  process address space.
- **(B)** Sibling systemd service unit running outside the wskop
  process. Reads the same `/mnt/tmp_ramfs/run/<pci_id>/...` from
  the host filesystem; the wskop process is the only writer.

**v1 picks (B).** Three reasons:

1. **Decoupled lifecycles.** wskop crashes and restarts under
   `wskop-wsi@.service` should not take down the UI's data
   plumbing, and a bridge bug should not affect data-plane
   correctness. (B) makes both true by construction; the bridge
   daemon is a separate `Restart=always` unit.
2. **No new lcore consumed.** Per
   [discovery 02 §0](../dpi_runtime_data_contract_discovery/02_lcore_observability_responsibilities.md)
   wskop already runs worker + shaper + transmit + watch + redis_cli.
   Adding a cold "bridge" lcore burns a core for serving a few
   hundred bytes per second over a UNIX socket.
3. **Symmetry with the existing IPDR pipeline.** The IPFIX scanner
   and sender already follow the same pattern: wskop writes to a
   ramfs path, sister `*.service` units read it and ship it on
   ([discovery 01 §2.5](../dpi_runtime_data_contract_discovery/01_existing_exposure_surfaces.md)).
   The bridge is a third sibling reading the third ramfs surface.

**The cost is a copy.** The bridge process reads files instead of
sharing memory with the producer; for the polled layer that is a
few `read()`s per request per counter — well within the latency
budget the UI cares about (≤100 ms for a "give me the whole tree"
request) but worth flagging because it pushes us toward batched
reads rather than per-counter requests on the wire (see
[03 §4](03_wire_protocol.md)).

**v2 streaming-layer caveat.** A streaming consumer that drains a
per-worker SP/SC ring on the broker has to live inside the wskop
process (DPDK rings cross processes only via cross-process rings,
which add cost and complexity). When v2 lands the streaming
producer is on a cold lcore inside wskop; the bridge daemon stays
out-of-process and **receives** streamed counters from wskop via a
second UNIX socket (or, simpler, the broker writes a streaming
JSON-Lines file under the ramfs tree that the bridge tails). See
[06 §3](06_v2_streaming_layer.md).

## 2. Process shape

```
┌─────────────────────────────────────────────────────────────────┐
│ wskop-wsi@<instance>.service                                    │
│   (existing, owns producer side of ramfs)                       │
│                                                                 │
│   wskop binary                                                  │
│   └─ writes /mnt/tmp_ramfs/run/<pci_id>/... via ssg_src/metrics │
└─────────────────────────────────────────────────────────────────┘
                              │ ramfs (host-local)
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ wskop-bridge@<instance>.service                                 │
│   (new, owns consumer side)                                     │
│                                                                 │
│   wskop-bridge binary                                           │
│   ├─ listens on UNIX socket /run/wskop-wsi/<instance>/bridge.sock
│   ├─ reads /mnt/tmp_ramfs/run/<pci_id>/... per request          │
│   └─ writes nothing else                                        │
└─────────────────────────────────────────────────────────────────┘
                              │ host bind-mount
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ UI container                                                    │
│   - mounts /run/wskop-wsi/<instance>/bridge.sock                │
│   - speaks the wire protocol (see 03_wire_protocol.md)          │
└─────────────────────────────────────────────────────────────────┘
```

The bridge binary lives in `wskop_bridge_v01` (the satellite repo
this design creates). It does **not** link against any wskop
internal — it consumes only the on-disk format the metrics tree
already produces and the manifest from
[05](05_counter_semantics_manifest.md).

## 3. Socket path convention

```
/run/wskop-wsi/<instance>/bridge.sock
```

- `/run/wskop-wsi/<instance>/` already exists per
  [dpi_config_ingest_discovery 01 §1](../dpi_config_ingest_discovery/01_config_file_inventory.md);
  the `.launch` files live in it. Reusing that directory keeps the
  per-instance grouping aligned with how the rest of the deployment
  thinks about instances.
- Tmpfs-backed (`/run` is `tmpfs` on the dev rig per `mount`); no
  cleanup needed across reboots.
- File mode `0660`, owner `root`, group `wskop` (the existing
  group on the rig per `db.env`'s mode). The UI container's bind-
  mount inherits the host's perms; the operator can choose to
  mount it read-only on the container side.
- One socket **per instance** (i.e. per wskop process / per
  `pci_id`). A multi-instance host has one bridge process per
  instance.

The bridge daemon's command line takes one argument: the `pci_id`
(matching the wskop binary's `--nic-port-config=` derivation in
[dpi_config_ingest 01 §nic](../dpi_config_ingest_discovery/01_config_file_inventory.md)).
The systemd template unit derives the socket path and the
metrics-tree path from `pci_id`.

## 4. Systemd unit (pseudocode skeleton)

`/etc/systemd/system/wskop-bridge@.service`:

```ini
# PSEUDOCODE — actual unit ships in wskop_bridge_v01/systemd/
[Unit]
Description=wskop runtime-data bridge for %i
PartOf=wskop-wsi@%i.service
After=wskop-wsi@%i.service
Requires=wskop-wsi@%i.service

[Service]
Type=notify
EnvironmentFile=-/etc/wskop-wsi/%i.json
ExecStartPre=/usr/bin/mkdir -p /run/wskop-wsi/%i
ExecStart=/usr/sbin/wskop-bridge \
          --instance %i \
          --metrics-base /mnt/tmp_ramfs/run \
          --socket /run/wskop-wsi/%i/bridge.sock \
          --manifest /etc/wskop-wsi/%i.bridge_manifest.json
Restart=always
RestartSec=2
NotifyAccess=main
User=root
Group=wskop
UMask=0007

[Install]
WantedBy=multi-user.target
```

- `PartOf=` + `After=` makes the bridge follow wskop-wsi's
  lifecycle: when the operator stops wskop-wsi the bridge stops
  too. `Requires=` means the bridge won't start without wskop-wsi.
- `Restart=always` plus `RestartSec=2` keeps the bridge alive
  through its own bugs without flapping.
- `--metrics-base` is overridable per-instance (matches the
  existing `WSKOP_METRICS_BASE_DIR` env on the wskop side) so the
  smoke test can run on a host without `/mnt/tmp_ramfs`.

## 5. Restart and reconnect semantics

### 5.1 wskop restarts

When `wskop-wsi@<i>.service` restarts:

- The metrics tree under `/mnt/tmp_ramfs/run/<pci_id>/` is **not**
  removed by the wskop binary (it uses `mkdir_p` idempotent +
  re-open at startup per
  [metrics.c:55-85](https://github.com/bdevskmank/wskop_v01/blob/3ff7479/ssg_src/metrics/metrics.c#L55-L85)).
  But the **values** are reset by the next `metrics_register` call
  which initialises each file to `0\n`
  ([metrics.c:198](https://github.com/bdevskmank/wskop_v01/blob/3ff7479/ssg_src/metrics/metrics.c#L198)).
- `system/start_time` is re-written to the new boot epoch
  ([wskop.c:7632](https://github.com/bdevskmank/wskop_v01/blob/3ff7479/wskop.c#L7632)).
  The bridge daemon publishes this value untouched; the UI
  detects a wskop restart by observing `start_time` move.
- The bridge daemon does not need to restart. Its existing
  filesystem reads continue to work. The `PartOf=` shape means
  systemd does restart it; that is for clean lifecycle, not
  correctness.

### 5.2 Bridge daemon restarts

- All connected clients see EOF. They reconnect.
- Bridge re-binds the socket. Since the socket lives in `/run`
  which is tmpfs, the stale inode goes away with the previous
  process; the bridge `bind()`s a fresh socket. (Defensive
  `unlink(sock_path)` before `bind()` covers the edge case where
  the daemon was SIGKILLed mid-shutdown.)
- The UI's container bind-mount **persists** across daemon
  restarts because the bind-mount is to the directory, not to the
  inode. The UI's reconnect logic is "open the same path; loop on
  ECONNREFUSED for up to N seconds; fail soft."

### 5.3 Client connect / disconnect

- Clients can disconnect without notice; the bridge sees EPIPE on
  next write and drops the connection. No state kept per-client
  beyond the open fd and an optional subscribe set
  ([03 §3](03_wire_protocol.md) details subscribe semantics).
- New clients announce a protocol version on connect; mismatch is
  an error response ([03 §2](03_wire_protocol.md)).

### 5.4 Bridge auto-detects wskop restarts (per operator Q7)

**Decision (operator A7, 2026-05-15):** the bridge actively
detects wskop restarts and notifies subscribed clients. This was
flagged as observation in [07 §11](07_observations.md); the
operator promoted it to spec.

**Mechanism.** On startup, the bridge reads
`system/start_time` and stores the value as
`last_known_start_time_seconds`. The bridge re-reads
`system/start_time` on every polling tick (1 Hz default for
v1 polled subscribes; on demand for one-shot `get`). When the
new value > stored, the bridge:

1. Updates `last_known_start_time_seconds`.
2. **Pushes an unsolicited event** to every connected client
   (regardless of subscribe set):
   ```json
   { "v":1, "kind":"event",
     "event": {
       "type":"wskop_restart",
       "previous_start_time": 1778751156,
       "new_start_time":      1778834567 } }
   ```
   The frame omits `id` because it is unsolicited (per the
   protocol's optional-on-events rule in
   [03 §2](03_wire_protocol.md)).
3. **Discards any in-flight per-subscription rate state** (e.g.
   "last `last_seen_tsc` we saw for flow N") so the next push
   doesn't claim a synthetic delta.

**On the client (UI) side.** On receiving `event.type=="wskop_restart"`:

- Refresh the manifest via `kind="manifest"` — schema may or may
  not have changed; safe to assume it did and refetch.
- Reset any local "previous sample" caches the UI uses to compute
  rates. The counters reset to 0 on wskop restart per
  [05 §3.2](05_counter_semantics_manifest.md) (`monotonic_nondecreasing`
  resets are a known restart side-effect).
- Re-issue any active `subscribe` calls if the protocol has been
  decided to drop them on a restart event. **v1 leaves them
  alive** — the bridge does not invalidate client subscribes on
  detection; the next pushed `values` frame just carries the new
  (reset) values. UI computes rates from the next two samples
  onward.

**Edge case — start_time decreases.** If
`system/start_time` ever returns a value less than the stored
one (impossible under normal operation; would indicate clock
skew or a corrupted file), the bridge treats it as a restart and
re-emits the event. The clock-skew case is rare on a wskop host;
the corrupted-file case is caught by `value_parse_failure`
elsewhere ([03 §4](03_wire_protocol.md)).

**Implementation cost.** One `read()` per polling tick on the
existing `system/start_time` file — already in the polled set
for clients that subscribe to `system/`. No new resource;
bridge keeps the value cached.

## 6. Ramfs reads: on-demand vs cached

**v1: on-demand, no in-memory cache.** Three reasons:

1. The polled layer's largest payload is the entire metrics tree
   for one `pci_id` — at most a few hundred small files (≤32 per
   handle × ~10 handles in the staged build of
   [01 §7](01_v1_producer_additions.md)). Walking that with
   `openat`+`read`+`close` is ~tens of microseconds on a ramfs;
   the file contents are at most a 20-byte decimal each. The
   end-to-end request latency is well under any UI's perceived
   threshold.
2. **Freshness wins over speed.** The metrics tree updates at most
   once per second (per
   [01 §2](01_v1_producer_additions.md)); caching for less than
   that is wasted work, caching for more risks the UI lagging the
   running state by an unbounded amount on the next polled tick.
3. **No invalidation logic** to get wrong. The producer writes
   atomically (`pwrite`+`ftruncate` per metrics.h); a reader at
   any moment sees a torn value only if the producer wrote a
   shorter value (e.g. `1000000` → `100` leaves no trailing
   bytes — confirmed in
   [metrics.c:151-156](https://github.com/bdevskmank/wskop_v01/blob/3ff7479/ssg_src/metrics/metrics.c#L151-L156)).
   No locking needed.

**v2 streaming caveat.** Per-flow watch emits change the picture —
the bridge daemon then tails an append-only stream rather than
polling a directory. See [06](06_v2_streaming_layer.md).

## 7. Concurrency model

**v1: single-threaded `epoll` loop.** The listener socket plus
each accepted client lives in one `epoll_wait`. Each request is
processed inline.

Why not thread-per-connection: max concurrent client count is on
the order of "one UI process per box" + "one or two debug shells"
= ≤5 clients. The per-request work is filesystem reads on ramfs,
which never block. Thread overhead and the
synchronisation it would require are net cost.

Why not async non-blocking I/O across multiple cores: the request
rate is ~1 Hz per client at the polled cadence. A 3 GHz core can
satisfy thousands of `GET /workers/lcore_0/*` requests per
second; CPU is not the bottleneck.

Pseudocode:

```
/* PSEUDOCODE — wskop_bridge_v01/daemon/main.c skeleton */

int listen_fd = bind_unix_socket(SOCKET_PATH, /* mode= */ 0660);
int epfd = epoll_create1(EPOLL_CLOEXEC);
epoll_add(epfd, listen_fd, EPOLLIN);

struct client {
    int  fd;
    char rbuf[REQ_MAX];
    size_t rbuf_len;
    /* subscription set for v2 — empty in v1 */
};

while (!shutdown_requested) {
    struct epoll_event evs[64];
    int n = epoll_wait(epfd, evs, 64, /* ms timeout = */ -1);
    for (int i = 0; i < n; i++) {
        if (evs[i].data.fd == listen_fd) {
            int cfd = accept4(listen_fd, NULL, NULL, SOCK_CLOEXEC | SOCK_NONBLOCK);
            register_client(cfd);
        } else {
            struct client *c = clients_lookup(evs[i].data.fd);
            handle_client_event(c, evs[i].events);
        }
    }
}
```

`handle_client_event` parses one request at a time (request grammar
in [03](03_wire_protocol.md)), reads the relevant ramfs paths,
formats the response, writes. On any error (parse failure,
filesystem error) it returns a structured error response on the
same connection and keeps the connection open.

## 8. Failure modes

### 8.1 Ramfs missing entirely

`metrics-base` directory not found. Bridge logs to stderr (journal
via systemd) and returns `error: metrics_unavailable` to every
request until the directory appears. Polls for the directory's
existence every 1 s while in this state. Does **not** exit —
systemd would respawn it forever, which is noise; the bridge
self-heals on first successful `stat`.

This corresponds to the "wskop not yet started; bridge raced ahead
on boot" case. The `After=wskop-wsi@%i.service` directive minimises
this window but does not eliminate it.

### 8.2 wskop stopped while bridge running

Same on the bridge side: ramfs still exists, files still readable,
but values stop advancing. `system/start_time` stops updating.
**The bridge does not detect this** — that is a UI concern. The
manifest documents `system/start_time` as the wskop-liveness
indicator; the UI's freshness check is `(now - start_time) > X`
plus a parallel check that some other counter is advancing.

(A future enhancement could have the bridge check the
`wskop-wsi@.service` `ActiveState` via D-Bus and reflect it in a
status field. Out of v1 scope.)

### 8.3 Partial files

Producer atomic write semantics ([metrics.c:151-156](https://github.com/bdevskmank/wskop_v01/blob/3ff7479/ssg_src/metrics/metrics.c#L151-L156))
guarantee a file's contents are always a complete value-plus-
newline once visible. The single edge case is a freshly-created
file before its first `write_value_at` — but the producer
initialises each file to `0\n` inside `metrics_register`
([metrics.c:198](https://github.com/bdevskmank/wskop_v01/blob/3ff7479/ssg_src/metrics/metrics.c#L198))
before any other code can race. So **partial-file reads are
not a concern** at HEAD.

Bridge-side defensive: if a `read()` returns 0 bytes, retry once;
on second zero-byte read return `error: empty_value` with the
counter path. This catches a hypothetical regression where a future
counter is left empty between `open` and first `write`.

### 8.4 File parse error

A counter file containing something that doesn't parse as a `uint64`
or a short ASCII string. Returns `error: parse_failure` with the
path. The bridge does **not** crash or skip the rest of the
response.

### 8.5 Socket-side errors

| Condition | Response |
|---|---|
| Client sends malformed request | error response on the same connection; keep connection open |
| Client closes mid-response | `EPIPE` on write; drop connection; no log spam |
| Listen-side `accept` fails | log, sleep 100 ms, retry |
| `epoll_wait` returns `EINTR` | normal — re-enter loop |

### 8.6 Bridge OOM

Stay-alive plan: bridge memory is bounded — no per-flow state in
v1, no large in-memory cache. The largest allocation is the
response buffer (a few kilobytes). OOM is not a realistic failure
mode in v1; if it ever becomes one in v2 (streaming), the failure
is "drop a streamed event with a counter, log, continue." Same
posture as
[discovery 05 §B.5](../dpi_runtime_data_contract_discovery/05_flow_watch_cost_analysis.md)'s
ring-drop on overflow.

## 9. What the bridge does NOT do

- Does not write to the metrics tree. **Read-only by construction.**
- Does not validate counter semantics — that is the manifest's job
  ([05](05_counter_semantics_manifest.md)).
- Does not perform aggregation (e.g. "sum in_pkts_total across
  workers"). UI does that.
- Does not authenticate clients. The bind-mount permission + the
  group ownership on the socket are the access-control surface.
  This matches the existing pattern for `db.env` and the rest of
  the wskop-wsi deployment.
- Does not speak HTTP. The wire protocol is line-/length-based on
  the UNIX socket only.

## 10. Binary placement

| Path | Purpose |
|---|---|
| `/usr/sbin/wskop-bridge` | the bridge daemon binary (built from `wskop_bridge_v01/`) |
| `/etc/systemd/system/wskop-bridge@.service` | the systemd template |
| `/etc/wskop-wsi/<instance>.bridge_manifest.json` | per-instance manifest (see [05](05_counter_semantics_manifest.md)) — generated at install or wskop-build time |
| `/run/wskop-wsi/<instance>/bridge.sock` | the UNIX socket (created by bridge on start) |

`install.sh` (per
[dpi_config_ingest_discovery 08 §10](../dpi_config_ingest_discovery/08_open_questions.md))
acquires three new files; the UI's host-onboarding path picks them
up as part of its existing "install wskop" workflow.

## 11. Why this matches the operator's deployment shape

- Container UI **bind-mounts** a host socket. UNIX socket
  bind-mount is the standard container interop pattern; the host
  serves, the container speaks.
- The UI does **not** need filesystem access to `/mnt/tmp_ramfs`
  (operator A5 says it cannot have it). The bridge has the
  filesystem access; the UI has the socket.
- No new ports opened, no new network service exposed.
- One bridge per wskop instance keeps the failure domain tight.
  The UI side handles per-instance discovery via the existing
  `/etc/wskop-wsi/<instance>.json` files
  ([dpi_config_ingest 01 §3](../dpi_config_ingest_discovery/01_config_file_inventory.md)).

---

# 03 — Wire protocol

The bridge daemon ([02](02_bridge_daemon_design.md)) serves a
UNIX socket. This file picks the wire protocol on it.

## 1. Format choice: length-prefixed JSON

**Decision.** Each message — request or response — is **a JSON
object, prefixed with a 4-byte big-endian unsigned length** giving
the payload size in bytes. UTF-8 encoded. No trailing newline; the
length is authoritative.

```
+--------+--------+--------+--------+--------+--------+
|  L3    |  L2    |  L1    |  L0    |  { ... JSON ... }
+--------+--------+--------+--------+--------+--------+
  uint32 big-endian length            payload (length bytes)
```

Maximum payload size in v1: **1 MiB**. Larger responses use the
pagination scheme in §5; cleaner than relying on TCP-style
chunking on a UNIX socket where most clients are happier reading
one full message at a time.

### 1.1 Why not line-delimited "path = value"

It was the simplest option and the brief lists it first. But:

- The UI needs **typed** values (`uint64`, string, `null` for
  unwired counters per [discovery 08 Q7](../dpi_runtime_data_contract_discovery/08_open_questions.md#q7)).
  `key = value` text either reinvents type tagging (`key: type =
  value`) or forces every value through `parseInt`. JSON gives the
  UI the types directly.
- The contract has structured error responses, capability
  negotiation (schema version, [05](05_counter_semantics_manifest.md)),
  and a subscribe semantics with multi-counter payloads (§3). All
  three are natural in JSON, awkward in line-delimited text.
- A `path = value` format would not extend cleanly to v2's
  streaming records (5-tuple + counters + classification per
  [discovery 05 §B.5](../dpi_runtime_data_contract_discovery/05_flow_watch_cost_analysis.md)).
  JSON-Lines (newline-delimited JSON) is the alternative streaming
  format and shares grammar with v1's responses, easing the v1→v2
  bridge.

### 1.2 Why not JSON-Lines for v1 too

Length-prefixed framing makes the request/response model easier:
the bridge reads exactly `length` bytes per message, never has to
scan for a newline, never has to handle a `\n` inside a string
value. The v2 streaming layer
([06 §3](06_v2_streaming_layer.md)) **does** use JSON-Lines for
exactly the reason JSON-Lines is good — open-ended streaming with
no length known up front. Two formats over the same socket,
disambiguated by message direction (server-push for v2 stream is
explicit in the subscribe response).

### 1.3 Why not Protocol Buffers / FlatBuffers / etc.

The UI is a JavaScript-ish runtime per the operator's
"container-wrapped OS service" answer. Adding a protobuf compiler
to the UI's build path for ~30 counters is overhead with no
payoff. JSON is the line of least resistance. If a future v3
needs binary efficiency (it won't — payload is hundreds of bytes
per second per client) the schema-version field §2.4 lets a v3
protocol coexist with v1 on the same socket.

## 2. Top-level message shape

Every message — request or response — has the same root fields:

```json
{
  "v": 1,
  "id": "<opaque request id, echoed in response>",
  "kind": "<request kind | response kind | error | event>",
  ...
}
```

| Field | Type | Meaning |
|---|---|---|
| `v` | uint | Protocol version. **1** for v1. The bridge accepts only messages whose `v` matches the bridge's supported version; mismatch → error response with `kind="error"`, `error.code="protocol_version_mismatch"`. |
| `id` | string (≤64 chars) | Request correlator. The server echoes the same value in every response and every event tied to a subscribe. Clients pick. Optional on events that are unsolicited. |
| `kind` | string | Discriminator. See §3 / §4. |

Additional fields are kind-specific (§3 below).

### 2.4 Schema version vs protocol version

- `v` (this section) is the **wire protocol version** — message
  shape, length-prefixing, field names. Bumping `v` is breaking.
- A **schema version** for the counter set ships separately, in
  the manifest published by `GET /manifest` (§3.2 and
  [05](05_counter_semantics_manifest.md)). Adding new counters
  bumps the schema version but does not bump `v`.

## 3. Request kinds

### 3.1 `kind="get"` — one-shot read

```json
/* request */
{ "v": 1, "id": "r1", "kind": "get", "paths": ["workers/lcore_0/in_pkts_total",
                                                "system/start_time"] }

/* response (success) */
{ "v": 1, "id": "r1", "kind": "values",
  "values": [
    { "path": "workers/lcore_0/in_pkts_total", "type": "uint64", "value": 12345678 },
    { "path": "system/start_time",             "type": "uint64", "value": 1778751156 }
  ] }
```

Paths are relative to `/mnt/tmp_ramfs/run/<pci_id>/`. A single
`get` may carry up to 256 paths. Glob support
(`workers/lcore_*/in_pkts_total`) is **not** in v1; the UI calls
`GET /list` first then issues a single `get` with the resolved
path list. (Avoids server-side fnmatch implementation in v1; the
list endpoint already enumerates everything.)

For paths that don't exist, the response carries a
`"value": null` entry plus `"status": "missing"`:

```json
{ "path": "workers/lcore_99/in_pkts_total",
  "type": "uint64", "value": null, "status": "missing" }
```

For paths that the manifest marks as `expected_zero` (the
unincremented counters per
[discovery 08 Q7](../dpi_runtime_data_contract_discovery/08_open_questions.md#q7)),
the response carries the literal current value (`0`) plus
`"status": "expected_zero"`. The UI displays them without alarm.

### 3.2 `kind="manifest"` — counter catalog

```json
/* request */
{ "v": 1, "id": "r2", "kind": "manifest" }

/* response */
{ "v": 1, "id": "r2", "kind": "manifest",
  "schema_version": "2026.05.15-1",
  "wskop_version":  "unknown",      /* from system/version file */
  "counters": [
    { "path": "workers/lcore_0/in_pkts_total",
      "type": "uint64", "kind": "counter", "unit": "packets",
      "description": "Packets received by worker lcore 0 from its RX queues.",
      "status": "live",
      "owner_lcore_role": "worker", "owner_lcore_id": 0,
      "publish_cadence_seconds": 1.0,
      "since_schema_version": "2026.05.15-1",
      "monotonicity": "monotonic_nondecreasing" },
    { "path": "workers/lcore_0/packets_dropped_tx_queue_full",
      "type": "uint64", "kind": "counter", "unit": "packets",
      "description": "Worker-side packets dropped because the TX queue was full at the worker→TX hop.",
      "status": "expected_zero",
      "legacy_name": true,
      "publish_cadence_seconds": 1.0,
      "since_schema_version": "2026.05.15-1",
      "notes": "Registered but no increment site at HEAD; placeholder for a future cleanup. Treat zero as 'no data', not 'no drops.'" },
    ...
  ] }
```

Full manifest contents are defined in
[05](05_counter_semantics_manifest.md). The bridge loads the
manifest from a file (`--manifest /etc/wskop-wsi/<i>.bridge_manifest.json`)
at startup and serves it; no parsing of counter files happens for
this request.

### 3.3 `kind="list"` — directory enumeration

```json
/* request */
{ "v": 1, "id": "r3", "kind": "list", "prefix": "workers/" }

/* response */
{ "v": 1, "id": "r3", "kind": "list",
  "prefix": "workers/",
  "entries": [
    { "path": "workers/lcore_0/", "type": "directory" },
    { "path": "workers/lcore_1/", "type": "directory" },
    ...
  ] }
```

Recursive listing with `"recursive": true` returns counter leaves
too. The full tree response under one `pci_id` is bounded by the
manifest size — see §5 if it exceeds 1 MiB (it won't in v1).

### 3.4 `kind="subscribe"` — periodic push (the v1 polled use case)

```json
/* request */
{ "v": 1, "id": "r4", "kind": "subscribe",
  "paths": ["workers/lcore_0/in_pkts_total",
            "workers/lcore_0/out_pkts_total",
            "system/start_time"],
  "interval_seconds": 1.0 }

/* response — ack */
{ "v": 1, "id": "r4", "kind": "subscribed",
  "interval_seconds": 1.0,
  "max_lag_seconds": 0.0 }

/* server-pushed events on the same socket, same id, repeated */
{ "v": 1, "id": "r4", "kind": "values",
  "epoch_ms": 1778834567890,
  "values": [
    { "path": "workers/lcore_0/in_pkts_total", "type": "uint64", "value": 12345700 },
    ...
  ] }
```

- `interval_seconds` is clamped server-side to `[1.0, 60.0]`. The
  producer cadence is 1 s (§01); finer is meaningless.
- The client may have multiple active subscribes (different `id`s,
  different path sets, different intervals). The bridge schedules
  them per-client.
- Unsubscribe is `{ kind: "unsubscribe", id: "r4" }`. Closing the
  connection also drops all subscribes for that client.

Subscribe replaces polling-via-`get` for clients that want a
live view; saves connection-setup churn on tight intervals. The
bridge implementation is a `timerfd` per subscribe, single-
threaded with the rest of the epoll loop.

### 3.5 `kind="ping"` — liveness

```json
/* request */
{ "v": 1, "id": "r5", "kind": "ping" }
/* response */
{ "v": 1, "id": "r5", "kind": "pong",
  "epoch_ms": 1778834567890,
  "schema_version": "2026.05.15-1",
  "wskop_version": "unknown" }
```

Useful for the UI's reconnect logic and for the bridge's own
healthcheck.

## 4. Response kinds and error model

Successful responses:

- `kind="values"` — for `get` and pushed `subscribe` events
- `kind="manifest"` — for `manifest`
- `kind="list"` — for `list`
- `kind="subscribed"` / `kind="unsubscribed"` — for subscribe ack
- `kind="pong"` — for ping
- `kind="event"` — **unsolicited** server-pushed events, no `id`.
  Currently defined event types:
  - `event.type="wskop_restart"` — bridge detected
    `system/start_time` advance. Carries `previous_start_time`
    and `new_start_time` (both `epoch_seconds`). See
    [02 §5.4](02_bridge_daemon_design.md) for the bridge-side
    detection mechanic and the UI-side reset. Pushed to **every**
    connected client regardless of subscribe set.

  Future event types are added in additive bumps of the manifest's
  `schema_version` (not the protocol `v`); the UI should ignore
  unknown event types gracefully.

Errors:

```json
{ "v": 1, "id": "r1", "kind": "error",
  "error": {
    "code": "metrics_unavailable",
    "message": "Metrics base directory does not exist: /mnt/tmp_ramfs/run/0000:18:00.0/",
    "retry_after_seconds": 1.0
  } }
```

| `error.code` | When | Recoverable? |
|---|---|---|
| `protocol_version_mismatch` | Request `v` is not the bridge's supported version | No without a client upgrade |
| `parse_failure` | Request is not valid JSON or doesn't match the schema | Client bug; not retryable |
| `unknown_kind` | Request `kind` not recognised | Client bug |
| `metrics_unavailable` | Ramfs base directory missing | Retry after `retry_after_seconds` |
| `path_invalid` | Path attempts to escape the metrics base (e.g. `../etc/passwd`) | Client bug |
| `path_missing` | Counter file does not exist (paired with per-path `status="missing"` in §3.1 — only used here if the whole `get` failed) | Manifest mismatch; refetch manifest |
| `value_parse_failure` | Counter file contents did not parse as the declared type | wskop-side bug; report-and-continue |
| `subscribe_limit` | Client has too many active subscribes (cap = 16) | Unsubscribe first |
| `internal` | Catch-all; bridge logs the actual error | Retry sparingly |

Errors are always per-request: the connection stays open after an
error response.

## 5. Pagination

For `get` and `list` responses that would exceed the 1 MiB
message cap, the server includes a `next` token and the client
re-issues with `{ "from_token": "..." }`. The token encodes the
server-side offset within the path list (for `get`) or the cursor
into the directory iteration (for `list`).

In v1 the metrics tree fits comfortably under 1 MiB (a few hundred
counters × ~150 bytes manifest entry each ≈ 50 kiB); pagination
exists in the protocol but is **not exercised** by v1 traffic. It
is in the grammar from the start so v2's streaming layer does not
have to introduce it later.

## 6. Field-naming and serialisation rules

- **Integers** larger than `2^53` (JSON's safe-integer limit)
  serialise as **strings** with `"type": "uint64_string"`. Counters
  for byte totals can blow past `2^53` in days at 100 Gbps; the UI
  consumer needs to parse them as bigint. Counters that fit
  serialise as JSON numbers with `"type": "uint64"`.

  Wire choice: a per-counter type tag (rather than guessing from
  magnitude) keeps the UI code straightforward.

  ```json
  /* both possible — type tells the UI which path to take */
  { "path": "system/start_time", "type": "uint64", "value": 1778751156 }
  { "path": "ports/port_0/rx_bytes", "type": "uint64_string", "value": "9007199255000000" }
  ```

  Default `type` for new counters is `uint64`; the manifest opts a
  counter into `uint64_string` when it is known to exceed the safe
  integer range. Cumulative byte counters (the only ones at risk)
  use `uint64_string`.

  **Python UI note (per operator A10, 2026-05-15):** Python's
  stdlib `json` parses arbitrary-precision JSON numbers into
  Python `int` without precision loss, so `type="uint64"` for
  values above `2^53` also works in Python (unlike a Node UI
  where any number above `2^53` silently rounds to `float`).
  The contract still **prefers `uint64_string` for cumulative
  byte counters** so the wire format is portable to a future
  Node frontend without schema change. Python parses both forms
  cleanly via `int(value)` when `type="uint64_string"` and
  direct on `type="uint64"`.

- **Strings** (e.g. `system/version`) serialise as JSON strings,
  trimmed of the trailing newline that
  [metrics.c:241](https://github.com/bdevskmank/wskop_v01/blob/3ff7479/ssg_src/metrics/metrics.c#L241)
  writes.
- **Null** indicates "counter not present" or "unwired" per the
  manifest's `status` field.
- **Timestamps** in events use `epoch_ms` (uint64). The
  `system/start_time` counter is unix epoch seconds; the manifest
  records its unit explicitly.

## 7. Grammar (EBNF-flavored)

```
message      = uint32_be(length) , payload

payload      = "{" , root , "}"

root         = "\"v\":" , uint
             , "," , "\"id\":" , json_string  (* optional on server events without correlator *)
             , "," , "\"kind\":" , json_string
             , kind_specific_fields

kind         = "get" | "values" | "manifest" | "list"
             | "subscribe" | "subscribed" | "unsubscribe" | "unsubscribed"
             | "ping" | "pong"
             | "error"

(* See §3 / §4 for the fields each kind requires. *)
```

The bridge implementation does not need a parser generator; a
straightforward JSON parser plus a `switch` on `kind` covers it.

## 8. Example session

```
[client] connect /run/wskop-wsi/wskop-wsi-01/bridge.sock
[client] -> { v:1, id:"h", kind:"ping" }
[bridge] <- { v:1, id:"h", kind:"pong", epoch_ms: 1778834567890,
              schema_version: "2026.05.15-1", wskop_version: "unknown" }
[client] -> { v:1, id:"m", kind:"manifest" }
[bridge] <- { v:1, id:"m", kind:"manifest", schema_version: "...", counters: [...] }
[client] -> { v:1, id:"sub1", kind:"subscribe",
              paths: ["ports/port_0/rx_pkts", "ports/port_0/tx_pkts",
                      "workers/lcore_0/in_pkts_total"],
              interval_seconds: 1.0 }
[bridge] <- { v:1, id:"sub1", kind:"subscribed", interval_seconds: 1.0, ... }
[bridge] <- (every 1s) { v:1, id:"sub1", kind:"values", epoch_ms: ..., values: [...] }
[bridge] <- { v:1, id:"sub1", kind:"values", epoch_ms: ..., values: [...] }
...
[client] -> { v:1, id:"u", kind:"unsubscribe", subscribe_id: "sub1" }
[bridge] <- { v:1, id:"u", kind:"unsubscribed", subscribe_id: "sub1" }
[client] close
```

## 9. v2 forward-compat

The protocol's headroom for v2:

- **Watched-flow events** ride on top of the existing `subscribe`
  shape with a new path namespace `watched/flow/<rss_hash>` and a
  new `kind="stream_event"` for unsolicited per-event pushes (the
  bridge transitions from "polled values" to "stream of records").
  Per [06](06_v2_streaming_layer.md).
- **Schema version field** lets v2 add new counter types and
  paths without bumping `v`.
- **Pagination token** is in v1 even though unused, so v2 events
  can carry windows of streamed records.

Bumping wire protocol `v` should be unnecessary for the
foreseeable v2 surface; the design is intentionally conservative
on what is in the grammar.

## 10. What this protocol does NOT do (v1)

- **No authentication.** Filesystem perms on the socket are the
  only gate (see [02 §3](02_bridge_daemon_design.md)).
- **No write requests.** The contract is read-only. Operator
  actions (mark a flow / subscriber as watched) flow through the
  existing Redis command channel — see
  [04 §3](04_watched_flag_and_propagation.md).
- **No global mutations.** The bridge has no state to mutate
  across requests beyond per-client subscribe sets.
- **No metrics namespaces other than the one wskop produces.**
  Host-level metrics (CPU, memory) the UI wants should come from
  a sibling agent reading `/proc/`; the bridge does **not** add
  that to its surface. (Operator A from
  [dpi_config_ingest 08 §8](../dpi_config_ingest_discovery/08_open_questions.md)
  contemplates a separate host-onboarding/inventory agent for
  exactly this; the runtime contract bridge stays narrow.)

---

# 04 — Watched-flag bit, writer, reader, hazard posture

This file claims a single bit in `flow_info_d->policy_info` for
the "watched" semantics, defines who writes it and who reads it,
and pins the cross-thread-write posture it inherits.

The bit is **needed only by v2** (the streaming layer) — the v1
polled producer additions in
[01](01_v1_producer_additions.md) do not require knowing per-flow
that a flow is watched. But the bit is **specified now** so that:

- v1 can ship the aggregate counters `watch/watched_flow_count`
  and `watch/watched_subscriber_count`
  ([01 §1.6](01_v1_producer_additions.md)) the moment the
  watched-flag write-side lands.
- The flag layout does not need to be revisited when v2 lands —
  it ships once.

## 1. Bit choice: `WATCHED_BIT = 38`

Re-verified at `wskop_v01@3ff7479` HEAD:

- `bit_operations.h` defines, in order:
  `SID 0..3`, `SID_FLAG 4`, `INSPECTION_DEPTH 5..9`,
  `AID_FLAG 10`, `AID 11..14`, `ACTION_PARAMS 15..25`,
  `FFUNCTION_FLOW_EDIT_LATCH 27`, `EXTERNAL_FLOW_EDIT_LATCH 28`,
  `POLICY_LOCK 29..32`, `SHAPER_KEY 33..37`,
  `REDIRECT_RST_SENT_BIT 53` (new — claimed by task (b) since the
  discovery), `RING_CONFIGURATION_FLAG 54`,
  `DS_RING_AVAILABLE 55`, `US_RING_AVAILABLE 56`,
  `SERVICE_KEY 64..95` (declared but unimplemented — see §5
  below). All citations:
  [bit_operations.h:15-91](https://github.com/bdevskmank/wskop_v01/blob/3ff7479/ssg_src/ffunctions/bit_operations.h#L15-L91).

- Bits **26, 38–52, and 57–63** are unclaimed in the live header.

- `bit_operations.c` does not implement `get_service_key_val` /
  `set_service_key_val` (declared at
  [bit_operations.h:166-168](https://github.com/bdevskmank/wskop_v01/blob/3ff7479/ssg_src/ffunctions/bit_operations.h#L166-L168);
  verified by `grep -n "service_key\\|SERVICE_KEY" bit_operations.c`
  returning no hits). The "overrun" concern in
  [discovery 03 §2.1](../dpi_runtime_data_contract_discovery/03_flow_state_and_per_flow_data.md)
  is therefore a header-only declaration today; no live caller
  touches bits 64-95.

**Decision.** Claim **bit 38** for the watched semantics:

```
#define WATCHED_BIT 38   /* contract: see runtime data contract design, 04 */
```

Rationale:

- **Adjacent to the contiguous SHAPER_KEY end (37).** Keeps the
  used-bit map contiguous from 0 to 38, then sparse to 53.
- **Far from task (b)'s `REDIRECT_RST_SENT_BIT = 53`.** Even if
  task (b) later decides to grow the redirect-related state into
  the 50-52 range, the watched bit doesn't collide.
- **Cleanly inside the unclaimed 38-52 range.** No collision risk
  with SERVICE_KEY's notional 64-95 even if it gets implemented as
  the `__uint128_t` interpretation the discovery hypothesised.

(Note: the discovery's
[5 §B.3](../dpi_runtime_data_contract_discovery/05_flow_watch_cost_analysis.md)
proposed bit 38 already; this file ratifies that choice now that
HEAD has been re-checked.)

## 2. Layout and naming

```
WATCHED_BIT 38
```

Lives in `bit_operations.h`. Single-bit accessors, modelled on
[task (b)'s `is_redirect_rst_sent` / `set_redirect_rst_sent`
inline pair at bit_operations.h:179-189](https://github.com/bdevskmank/wskop_v01/blob/3ff7479/ssg_src/ffunctions/bit_operations.h#L179-L189):

```c
/* PSEUDOCODE — exactly the shape task (b) used; lands in bit_operations.h */
static inline int is_watched(const uint64_t *p)
{
    return (*p >> WATCHED_BIT) & 1ULL;
}

static inline void set_watched(uint64_t *p)
{
    *p |= (1ULL << WATCHED_BIT);
}

static inline void unset_watched(uint64_t *p)
{
    *p &= ~(1ULL << WATCHED_BIT);
}
```

(Task (b) chose to keep its accessors inline; we follow the same
posture so the reader path is a single ALU op once the line is
loaded.)

## 3. Writer side — three propagation cases

The operator clicks a UI element. The UI writes a "mark watched"
intent to Redis (the existing command channel — same path as PRB
recompile per
[discovery 02 §5](../dpi_runtime_data_contract_discovery/02_lcore_observability_responsibilities.md)
and `tasks/redis_cli_discovery/`). From there, three propagation
cases land the bit:

### 3.1 Per-flow watch (operator clicks flow N)

Goes through `redis_core_function_2` →
`update_specific_subs_flow_actions_prb` → walks the subscriber's
`flow_list` and finds the matching flow → sets the bit on
`flow_d->policy_info`.

**Same cross-thread-write hazard as the existing PRB recompile
path.** Cited and accepted in
[discovery 03 §4](../dpi_runtime_data_contract_discovery/03_flow_state_and_per_flow_data.md):
single-word atomic store on x86_64, no QSBR, multi-field tears
possible but `policy_info` is a single u64 so its own bit writes
are tear-free. Setting `WATCHED_BIT` is `*p |= (1ULL << 38)` —
the only edge case is a concurrent worker `set_aid` on the same
u64 (a read-modify-write that could clobber the watched bit if
the worker holds a stale value).

**Mitigation.** Workers' own writes to `policy_info` go through
the soft-lock protocol in `policy_info` bits 27, 28, 29-32
(`FFUNCTION_FLOW_EDIT_LATCH`, `EXTERNAL_FLOW_EDIT_LATCH`,
`POLICY_LOCK` per
[bit_operations.h:30-37](https://github.com/bdevskmank/wskop_v01/blob/3ff7479/ssg_src/ffunctions/bit_operations.h#L30-L37)).
The redis_cli writer **must** take the external latch before
RMW-ing `policy_info` (call `set_policy_info_lock_external` at
[bit_operations.c:168-171](https://github.com/bdevskmank/wskop_v01/blob/3ff7479/ssg_src/ffunctions/bit_operations.c#L168-L171),
mutate, call `reset_lock_ffunction_n_external`). This is the
posture the existing PRB recompile path already uses; the watched
write joins that pattern.

The push-design refactor in
[`policy_version_refactor_design/`](../policy_version_refactor_design/01_mechanism.md)
will eventually remove cross-thread writes to `flow_info_d`
entirely. **When that refactor lands, the watched-bit write moves
into the same `pending_updates` table.** Until then, the redis_cli
writer does the lock-mediated RMW.

### 3.2 Per-subscriber watch (operator clicks subscriber S)

Same path as PRB-recompile-for-subscriber: walk
`subscriber->flow_list`, stamp the watched bit on each
`flow_d->policy_info`. Inherits the same hazards as §3.1.

**Additional state on `subscriber_info`.** A new boolean field
`watched` is added:

```c
/* PSEUDOCODE — new field on subsc_w_prb_ref, where the design phase
 * would also add 'policy_version' per
 * policy_version_refactor_design/04_subscriber_struct.md */
struct subsc_w_prb_ref {
    /* ... existing fields ... */
    uint16_t policy_version;       /* per PR #26 reserved field */
    uint8_t  watched;              /* 0 or 1 */
    /* ... */
};
```

The watched field on the subscriber state is set by `redis_cli`
on receipt of a Redis intent ("subscribe-watch S=true"). New
flows of S will inherit the bit at PRB-evaluation time (§3.3
below). Existing flows of S require the §3.2 list walk.

The existing reserved hole in `subsc_w_prb_ref` (PR #26 reserved
`policy_version`) supplies space for `watched` without growing the
struct.

### 3.3 At-flow-creation watch (PRB-evaluation-time stamp)

When a new flow gets its first packet, `pe_action_identify_prb_common`
([ffunctions.c:1793](https://github.com/bdevskmank/wskop_v01/blob/3ff7479/ssg_src/ffunctions/ffunctions.c#L1793))
runs once on the worker; it has the subscriber context in hand.
**One extra line:** if the subscriber's `watched` flag is set,
call `set_watched(&flow_d->policy_info)`.

This is **the only place** the watched bit is set on the worker
side. It runs on the **cold first-packet branch** of a flow, not
on every packet. Worker hot-path cost for this case: **zero
additional cycles**.

For pseudocode placement:

```
/* PSEUDOCODE — augmented near ffunctions.c:1500-1600 in
 * pe_action_identify_prb_common */

void pe_action_identify_prb_common(struct ft_with_metics_item_bidirection *flow_d_metics,
                                   struct flow_info_d *flow_d,
                                   struct subsc_w_prb_ref *sub_ref,
                                   /* ... */)
{
    /* ... existing AID/AID_PARAMS/SHAPER_KEY work ... */
    set_aid(&flow_d->policy_info, aid_resolved);
    set_aid_params(&flow_d->policy_info, params_resolved);
    /* ... */

    /* NEW: stamp watched bit if subscriber is watched. */
    if (sub_ref && sub_ref->watched) {
        set_watched(&flow_d->policy_info);
    }
}
```

## 4. Reader side — the hot-path consumer

**v1 reader: none on the hot path.** The polled-layer aggregate
counters in
[01 §1.6](01_v1_producer_additions.md) iterate the watch-managed
collector tables (`ff_ht_cllctr_<wid>`) on the **watch lcore**
once per second. The collector tables hold a `policy_info` copy
truncated to u16 per
[discovery 04 §3](../dpi_runtime_data_contract_discovery/04_metrics_and_counters.md);
bit 38 falls outside the u16 truncation, so the aggregate counter
cannot use the collector table — it reads
`flow_d->policy_info` from the fast table directly.

Cross-thread-read safety: single u64 load on x86 is atomic; the
read sees either pre- or post-update value, never torn. Counting
matches each tick is at most a brief tear — the count converges
on the next tick. Acceptable.

**v2 reader: the worker hot path.** Per [06](06_v2_streaming_layer.md)
the worker tests the watched bit on every packet and, if set,
SP/SC-enqueues a fixed-size summary record onto the per-worker
watch-emit ring. Cost analysis lives in [06 §6](06_v2_streaming_layer.md);
the short version is **one bit-AND on an already-loaded u64**
(the `policy_info` cache line is loaded earlier in `run_packet_function`
at [ffunctions.c:686](https://github.com/bdevskmank/wskop_v01/blob/3ff7479/ssg_src/ffunctions/ffunctions.c#L686)).
Single-digit-cycle worst case. Branch predicted not-taken for
flows that aren't watched (the typical case).

## 5. Hot-path impact statement — v1

**v1 producer-side additions do not touch the worker hot path.**
The watched-bit reads and the aggregate counter computations all
live on the watch lcore's 1 s tick.

The watched-bit **write** in `pe_action_identify_prb_common`
(§3.3) is on the worker's **cold first-packet branch** of a flow.
Per
[discovery 05 §B.2](../dpi_runtime_data_contract_discovery/05_flow_watch_cost_analysis.md)
this is the same path that already sets `aid_val`, `aid_params`,
shaper, etc. — adding one bit set is below noise.

| Site | Hot or cold | Worst-case per-packet cost |
|---|---|---|
| `pe_action_identify_prb_common` watched-stamp (§3.3) | cold (first pkt only) | 1 store amortised over the flow lifetime |
| Aggregate counter computation on watch (§4 / [01 §1.6](01_v1_producer_additions.md)) | watch-lcore cold tick | 0 on workers |
| Subscriber-list-walk RMW (§3.1, §3.2) | redis_cli lcore cold | 0 on workers (modulo soft-lock contention — see §6) |

## 6. Cross-thread-write posture (acknowledged)

Per [discovery 07 §3](../dpi_runtime_data_contract_discovery/07_observations.md),
the cross-thread-write hazard is **a chronic condition, not an
emergency**. The watched bit's writers (redis_cli for §3.1 /
§3.2, worker for §3.3) all touch `flow_d->policy_info`. The
writes are:

- **Worker-side** (`set_watched` in §3.3) — on the cold flow-
  creation branch, gated by the existing
  `FFUNCTION_FLOW_EDIT_LATCH` soft-lock protocol (the worker
  already takes this latch in
  [ffunctions.c around 1700-1772](https://github.com/bdevskmank/wskop_v01/blob/3ff7479/ssg_src/ffunctions/ffunctions.c#L1700-L1772)
  for the PRB-evaluation stage).
- **redis_cli-side** (`set_watched` in §3.1, §3.2) — must take
  `EXTERNAL_FLOW_EDIT_LATCH` (call `set_policy_info_lock_external`
  → mutate → `reset_lock_ffunction_n_external`). Same posture as
  PRB recompile today; **no fresh hazard.**

When the policy-version-push refactor in
[`policy_version_refactor_design/`](../policy_version_refactor_design/)
lands, the redis_cli-side writes to `flow_info_d` go away
entirely: writes route through the watch core's walker into
`pending_updates`, and the worker consumes them on a cold per-
packet branch. **The watched bit moves into the same
`pending_updates` value tuple at that point.** The work to add it
is one field on the pending entry and one assignment in the
walker. The v1 design does not require the refactor; the v1
design **does** require that the contract not gain new hazards on
top of the ones the refactor will eventually clean up.

Concretely: the contract design's cross-thread-write posture is
**single-bit RMW under the existing soft-lock protocol**. Same
hazard class as the existing PRB recompile path; no worse.

## 7. Reader of the bit, single-load semantics

Both the worker hot-path read (v2) and the watch-lcore aggregate
count (v1) load `flow_d->policy_info` in one u64 load. On x86_64
that's atomic; the read sees either the pre- or post-update value
of the *entire* u64. Multi-field tearing
(reading `aid_val` and `policy_info` and getting inconsistent
state) is **possible** during a PRB recompile but **not relevant
to the watched bit's own semantics** — the bit is independently
meaningful regardless of the other bits' state.

(A future enhancement: stamp a sequence counter into
`policy_info` bits 39-47 — 9 bits — so a consumer that wants
strict consistency across multiple `policy_info`-derived fields
can read-recheck-or-retry. Out of scope.)

## 8. Unwatch and retention semantics

### 8.1 Unwatch is `unset_watched` everywhere set_watched runs

- Operator unmarks a subscriber → redis_cli walks `flow_list`,
  `unset_watched(&flow_d->policy_info)` on each.
- Operator unmarks a flow → redis_cli walks the one flow,
  `unset_watched`.
- Subscriber's `watched` field flips to 0 → new flows of that
  subscriber do not get the bit at flow creation.

`unset_watched` is implemented (§2) and takes the same path as
`set_watched`. The aggregate counter
`watch/watched_flow_count` falls when the watch lcore's next
1 s walk sees fewer matches.

### 8.2 Retention across wskop restart, flow aging, subscriber unmark

**Decision (operator A13, 2026-05-15):** the three persistence
cases behave as follows. Implementation phase tests against this
table.

| Event | Per-flow `WATCHED_BIT` survives? | Subscriber `watched` survives? | Result |
|---|---|---|---|
| **wskop restart** | **no** — per-flow state is rebuilt from scratch on the new process; `flow_info_d` is reinitialised | **yes** — `subscriber_info` state is restored from Redis at startup (the standard subscriber-cache restore path); the field comes back if the Redis source carries it | New flows of a still-watched subscriber inherit the bit at flow creation per §3.3. Pre-restart explicit per-flow marks are **lost** (no Redis-backed mirror); operator re-marks if they want them back. |
| **flow aging** | **yes**, until the flow is reaped | n/a | The bit stays with the flow until the flow record dies. After aging, a new flow with the same 5-tuple gets the bit only if the **subscriber** is watched (the §3.3 path). |
| **subscriber unmark** | **yes** — flow-level bits on existing flows persist until an explicit `unset_watched` walk runs | n/a | Explicit per-flow marks outlive the subscriber-level mark. The operator's choice "watch this one flow" is honoured even after the subscriber is no longer watched globally. New flows of that subscriber do **not** inherit the bit (subscriber `watched=0`). |

The third row is the load-bearing semantic — it's also the one
that surprises most. The reading:

- Per-subscriber watch is a **rule** ("future flows of S are
  interesting"); changing the rule shouldn't retroactively undo
  explicit per-flow operator actions.
- Per-flow watch is a **fact** ("this specific flow is being
  watched right now"); only an explicit per-flow unmark clears
  it.

**Implementation implication.** The per-subscriber unmark path
(§3.2 of this file) **only modifies `subscriber_info.watched`**.
It does **not** walk `subscriber->flow_list` to unset bits. The
per-flow unmark path (§3.1) walks one flow and unsets the bit.
The per-subscriber **mark** path still walks the list and sets
the bit (so existing flows of a newly-watched subscriber become
visible immediately). Asymmetry is by design.

If an operator wants to "unmark subscriber S and clear all its
existing flow marks too," that is two UI gestures: unmark
subscriber + per-flow unmark walk. A future v2.1 might add a
combined gesture; out of scope for v1.

### 8.3 The aggregate counters track all three cases correctly

- `watch/watched_flow_count` re-counts on every watch tick, so
  wskop restart resets it (per-flow state lost), flow aging
  reduces it (matching flow gone), and per-flow unmark reduces
  it (bit cleared).
- `watch/watched_subscriber_count` counts subscribers with
  `watched==1`, which survives wskop restart via the Redis
  restore; it does **not** change on flow aging or per-flow
  marks.

UI consumers reading both counters together get a consistent
picture under all three event types.

## 9. Test plan (for the implementation phase)

- Unit-test `is_watched` / `set_watched` / `unset_watched` against
  a `uint64_t` value matrix that covers the boundary bits 37/38/39
  and confirms no other policy_info bit toggles.
- On the rig, exercise §3.1, §3.2, §3.3 paths and inspect the
  ramfs:
  - Mark subscriber S as watched via redis_cli; verify
    `watch/watched_subscriber_count` increments.
  - Trigger new flow for S; verify
    `watch/watched_flow_count` increments.
  - Mark flow N watched directly; verify count increments.
  - Unmark; verify counts decrement after next watch tick.
- Throughput A/B per the brief: stamp pass with watched=0 in all
  flows vs. baseline; expect noise-floor delta. The B2.1 budget
  is met by construction (no per-pkt write; only the cold flow-
  creation stamp).

## 10. What this does NOT do

- Does not add new fields to `flow_info_d`. The size at HEAD is
  96 bytes per the comment at
  [ffunctions.h:144-148](https://github.com/bdevskmank/wskop_v01/blob/3ff7479/ssg_src/ffunctions/ffunctions.h#L144-L148);
  the new bit lives in the existing `policy_info` u64. No new
  cache line touched on the hot path.
- Does not specify the ring shape for v2 emit. That is in
  [06](06_v2_streaming_layer.md).
- Does not fix the cross-thread-write hazard. That is the policy-
  version-push refactor in
  [`policy_version_refactor_design/`](../policy_version_refactor_design/)
  and explicitly out of scope here.
- Does not claim bits 39-52. They remain unclaimed for future
  workstreams (a sequence counter per §7, or a per-flow stream-id
  for v2 if needed).

---

# 05 — Counter semantics & the manifest

Per [discovery 08 Q8](../dpi_runtime_data_contract_discovery/08_open_questions.md#q8)
the contract publishes **per-counter semantics**, not just paths.
This file specifies how — what the manifest contains, where it
lives, how it versions, and how it documents the known semantic
anomalies surfaced by the discovery.

## 1. Where the manifest lives

```
/etc/wskop-wsi/<instance>.bridge_manifest.json
```

- Generated at build/install time (not at runtime by wskop).
- Read once by the bridge daemon at startup
  ([02 §4](02_bridge_daemon_design.md): the
  `--manifest /etc/wskop-wsi/<i>.bridge_manifest.json` flag).
- Served via the `kind="manifest"` request type
  ([03 §3.2](03_wire_protocol.md)).
- One per instance because counter paths can vary between
  instances (different worker counts produce different
  `workers/lcore_N/...` rosters). The manifest enumerates the
  concrete paths.

### 1.1 Why a static file and not "discover at runtime"

Two reasons:

1. **The manifest is the schema, not the data.** A counter file
   on the ramfs is a value. The manifest says what `workers/lcore_0/in_pkts_total`
   *means*, in what unit, with what semantic caveats. The bridge
   does not derive this from the ramfs alone — it derives it from
   the contract.
2. **The static-file model survives "ramfs not yet populated".**
   The bridge can serve `kind="manifest"` immediately after start,
   before wskop has written its first counter; the UI then knows
   what to expect once the data appears.

A dynamic discovery scheme (bridge scans ramfs and infers) would
fail this test, miss the `expected_zero` annotations, and re-
invent the "what is this counter" lookup.

### 1.2 Why per-instance vs global

Worker count is per-instance (the dev rig has 8 workers per
[dpi_config_ingest 01 §application](../dpi_config_ingest_discovery/01_config_file_inventory.md)).
Per-instance means the manifest's concrete path list matches what
the bridge will see. A global manifest with a glob (`workers/lcore_*/...`)
shifts the work to the consumer.

Per-instance also lets a future operator deploy two instances
with different counter sets without needing both consumers to
ignore each other's surfaces.

## 2. Manifest schema

```jsonc
{
  "$schema": "https://wskop.example/bridge_manifest.schema.json",
  "schema_version": "2026.05.15-1",
  "instance_id": "wskop-wsi-01",
  "pci_id":      "0000:18:00.0",
  "wskop_version_at_build": "<value of WSKOP_BUILD_VERSION at build time>",
  "generated_at_epoch_ms": 1778834567890,

  "counters": [
    {
      "path": "workers/lcore_0/in_pkts_total",
      "type": "uint64",                /* or uint64_string for >2^53 */
      "kind": "counter",               /* counter | gauge | info */
      "unit": "packets",               /* bytes | packets | seconds | objects | "" */
      "description": "Packets received by worker lcore 0 from its RX queues.",
      "status": "live",                /* live | expected_zero | not_wired | deprecated */
      "monotonicity": "monotonic_nondecreasing", /* counter | gauge:absolute | gauge:bounded */
      "owner_lcore_role": "worker",
      "owner_lcore_id": 0,
      "publish_cadence_seconds": 1.0,
      "since_schema_version": "2026.05.15-1",
      "source_file_line": "wskop.c:5566",   /* for traceability */
      "legacy_name": false              /* true if not <name>_total */
    },
    ...
  ],

  "anomalies": [
    {
      "id": "ACTION_DROP_misnomer",
      "affects_paths": ["workers/lcore_*/out_drop_pkts_total"],
      "summary": "ACTION_DROP increments out_drop_pkts but still forwards the packet."
    },
    {
      "id": "ipdr_cadence_bug",
      "affects_paths": [],
      "summary": "IPDR/IPFIX path ticks 1000x faster than design intent; orthogonal to ramfs counters."
    }
  ]
}
```

## 3. Fields, in detail

### 3.1 `status` — four values

- **`live`** — counter is read on its cadence, increment site
  exists, value is meaningful. The UI uses it directly.

- **`expected_zero`** — counter is registered and the file exists,
  but no increment site is wired at HEAD. Operator A7 from
  [discovery 08](../dpi_runtime_data_contract_discovery/08_open_questions.md#q7)
  says these are reserved placeholders to be wired later. **UI
  must not alarm on a zero value.** Per
  [discovery 04 §7](../dpi_runtime_data_contract_discovery/04_metrics_and_counters.md)
  the affected counters at HEAD are:
  - `workers/lcore_<N>/packets_unattributed_table_full`
  - `workers/lcore_<N>/packets_dropped_tx_queue_full`
  - `workers/lcore_<N>/ndpi_pool_high_water`
  - `shapers/lcore_<N>/ring_polls_empty`
  - `shapers/lcore_<N>/ring_polls_full`

  Re-verified at HEAD by searching for `metrics_increment` calls
  on each name — none found.

- **`not_wired`** — counter is in the manifest but no file is
  produced because the upstream feature (e.g. `cmon_register_core`
  on shaper / transmit per
  [01 §4](01_v1_producer_additions.md)) is missing. Bridge
  returns `value=null, status="missing"` per
  [03 §3.1](03_wire_protocol.md). UI displays as "n/a".

- **`deprecated`** — counter still produces a file but the
  contract advises the UI not to use it for new code. Empty in
  v1. Reserved for future cleanups (e.g. if the
  `flow_table_occupancy` name is one day replaced by `flow_count_total`).

### 3.2 `monotonicity` — three values

- `monotonic_nondecreasing` (default for counters) — total since
  wskop boot. Wraps at u64-max (effectively never in production).
- `gauge:absolute` — current absolute value (e.g.
  `pools/<name>/avail`, `system/start_time`, `watch/current_idle_timeout_seconds`).
- `gauge:bounded` — bounded by a known capacity, set in the
  manifest's `bounds.max` field (e.g.
  `pools/<name>/in_use` bounded by `pools/<name>/capacity`).

The UI applies the right rate computation. `monotonic_nondecreasing`
gets `(now - prev) / dt`; `gauge:absolute` is shown as the value
itself.

### 3.3 `legacy_name`

`true` if the counter name doesn't follow the `_total` /
`_<unit>` convention established for the new counters in
[01 §6](01_v1_producer_additions.md). At HEAD:

- `legacy_name=true`: `flow_table_occupancy`, `fall_through_count`,
  `misroute_count`, `ring_polls_empty`, `ring_polls_full`,
  `flows_aged_total` (this one is fine actually — it ends in
  `_total`), `flows_admitted_rejected_table_full`,
  `current_idle_timeout_seconds`, `packets_unattributed_table_full`,
  `packets_dropped_tx_queue_full`, `ndpi_pool_high_water`,
  `start_time`, `version`.

These names stay as-is for backward compatibility. The manifest
exposes them and the UI handles them; v2 may decide to rename.

### 3.4 `source_file_line`

Traceability back to the increment / set site in wskop. Lets a
human reading the manifest jump to the code that produces the
value. Optional but recommended.

### 3.5 `unit`

Free text but constrained. Allowed values in v1: `packets`,
`bytes`, `flows`, `subscribers`, `seconds`, `cycles`, `fp-cycles`,
`objects`, `epoch_seconds`, `epoch_ms`, `""` (no unit, e.g. for
`info`-kind strings like `system/version`).

## 4. Anomalies block — known semantic gotchas

The contract publishes each known anomaly explicitly so consumers
don't trip over them. Each anomaly entry has an `id` (stable
identifier the UI can match against), an `affects_paths` array
(may use the same glob shorthand as `legacy_name`'s description —
the UI consumer expands), and a `summary` (human-readable).

The v1 anomaly set (sourced from
[discovery 04](../dpi_runtime_data_contract_discovery/04_metrics_and_counters.md)
and [discovery 06](../dpi_runtime_data_contract_discovery/06_ui_data_needs_vs_current_exposure.md)):

### 4.1 `ACTION_DROP_misnomer`

```json
{
  "id": "ACTION_DROP_misnomer",
  "affects_paths": [
    "workers/lcore_*/out_drop_pkts_total"
  ],
  "summary": "ACTION_DROP increments out_drop_pkts but the packet is still forwarded; the counter overcounts 'real' drops. Cross-reference: docs/worker_core.md §10.",
  "since_schema_version": "2026.05.15-1"
}
```

### 4.2 `ipdr_cadence_1000x`

```json
{
  "id": "ipdr_cadence_1000x",
  "affects_paths": [],
  "summary": "IPDR / IPFIX timer is called with TRIM_STEP_1S=1 against check_time_interval_precise_ts(interval_ms), firing 1000x too fast. Affects the CSV export pipeline, not the ramfs counters. Cross-reference: docs/watch_core.md §10, docs/overall_architecture.md §11, discovery 01 §2.2.",
  "since_schema_version": "2026.05.15-1",
  "expected_fix": "Constants renamed to milliseconds OR interval helper switched to seconds"
}
```

This one does not affect any ramfs path; published so the UI can
display a banner if it ingests IPDR-derived data.

### 4.3 `unincremented_placeholders`

```json
{
  "id": "unincremented_placeholders",
  "affects_paths": [
    "workers/lcore_*/packets_unattributed_table_full",
    "workers/lcore_*/packets_dropped_tx_queue_full",
    "workers/lcore_*/ndpi_pool_high_water",
    "shapers/lcore_*/ring_polls_empty",
    "shapers/lcore_*/ring_polls_full"
  ],
  "summary": "These counters are registered but have no increment site at HEAD; they read zero. Marked status=expected_zero in the per-counter section. UI must not alarm on zero. Operator A7 (discovery 08 Q7).",
  "since_schema_version": "2026.05.15-1"
}
```

This duplicates the per-counter `status="expected_zero"` field;
the anomaly entry exists so the UI can show a single explanatory
banner rather than per-counter help text.

### 4.4 `policy_info_truncation_in_collector`

```json
{
  "id": "policy_info_truncation_in_collector",
  "affects_paths": [
    "watch/watched_flow_count"
  ],
  "summary": "ff_ht_cllctr_<wid>.policy_info is u16, truncated from the u64 fast-table policy_info; WATCHED_BIT=38 falls outside that u16. The watched_flow_count counter therefore reads fast-table policy_info directly. Cross-reference: discovery 04 §3, runtime contract design 04 §4.",
  "since_schema_version": "2026.05.15-1"
}
```

Surfaces a design choice the implementation must respect; not a
bug. UI doesn't act on this — listed so a reviewer sees it.

### 4.5 `shaper_transmit_cmon_unregistered`

```json
{
  "id": "shaper_transmit_cmon_unregistered",
  "affects_paths": [
    "shapers/lcore_*/heartbeat",
    "shapers/lcore_*/ema_loop_cycles_fp",
    "transmit/lcore_*/heartbeat",
    "transmit/lcore_*/ema_loop_cycles_fp"
  ],
  "summary": "Shaper and transmit lcores do not call cmon_register_core, so g_core_status entries are uninitialised. v1 omits these counters; manifest entries have status='not_wired'. Cross-reference: design 01 §4.",
  "since_schema_version": "2026.05.15-1"
}
```

## 5. Schema versioning

### 5.1 What bumps the version

`schema_version` follows `YYYY.MM.DD-N` where `N` increments per
release on the same day. Bumped when:

- a counter is added, removed, or renamed
- a counter's `status` changes (e.g. `not_wired` → `live`)
- a counter's `unit` or `monotonicity` changes
- an anomaly is added, resolved (removed), or its
  `affects_paths` change

**Not bumped** for:

- a counter's current value (obviously)
- a counter's `description` text — descriptions are advisory and
  freely improvable
- new examples added to the per-counter `notes`

### 5.2 What the UI does with the version

- On connect, the UI calls `kind="manifest"` and stashes
  `schema_version`.
- On a periodic reconnect or every N minutes, it re-queries via
  `kind="ping"` ([03 §3.5](03_wire_protocol.md)) which returns
  the current `schema_version`. If it changed, the UI re-fetches
  the manifest before trusting any value diff.
- Older UI clients that don't know how to handle a newer
  schema_version still get well-formed responses for the counters
  they know — additions are backward-compatible by construction.

### 5.3 Backward compat policy

- **Additive changes** never break old consumers. New counters,
  new anomaly entries, new optional fields — all safe.
- **Status changes** (live → expected_zero, expected_zero → live)
  are safe but the UI should refresh.
- **Renames** are not safe. Avoid in v1; if unavoidable in a
  future contract, ship both old + new for one schema release,
  mark old `deprecated`, then drop in the next.
- **Path moves** (e.g. moving `system/port_<P>/...` to
  `ports/port_<P>/...`) — equivalent to a rename. Same rules.

## 6. How the manifest gets generated

**Decision (operator A1, 2026-05-15): (c) hybrid.** The
implementation phase ships a small generator script in
`wskop_bridge_v01/scripts/generate_manifest.<py|sh>` that:

1. **Code-generated path list.** Walks `wskop_v01/wskop.c` (and
   any future contributors to `metrics_register` calls) and
   extracts the `(role, counter_name)` pairs. Maps them to
   concrete paths against the instance's worker / shaper /
   transmit count (from
   [`/etc/wskop-wsi/<instance>.json`](../dpi_config_ingest_discovery/01_config_file_inventory.md)).
2. **Hand-written semantics.** Reads
   `wskop_bridge_v01/manifests/<instance>.semantics.json`
   (versioned in git) which carries `description`, `unit`,
   `status`, `monotonicity`, `legacy_name`, `anomalies` per
   counter name (not per concrete path — the generator
   broadcasts the per-name semantics across all matching
   concrete paths).
3. **Emits** `/etc/wskop-wsi/<instance>.bridge_manifest.json`,
   merging (1) and (2). Bumps `schema_version` if a counter was
   added/removed since the last run (the generator stores a
   previous-run cache for diffing).

**CI gate.** A `check_manifest.sh` script runs in CI on
`wskop_v01` PRs that touch `metrics_register`: it runs the
generator against the touched commit and fails if a new counter
is added without a corresponding hand-written semantics entry.
This catches "added a counter, forgot to document it" before
merge.

This shape was chosen because:

- Option (a) (hand-written only) drifts silently — easy to add a
  new `metrics_register` and forget the manifest.
- Option (b) (code-generated only) can't infer `description`,
  `status` (live / expected_zero / not_wired / deprecated), or
  anomaly linkage from the source.
- (c) lets both halves do what they're best at and gates the
  combined output via the CI script.

Alternatives (a) and (b) were considered and rejected.

## 7. Example manifest excerpt (illustrative; not exhaustive)

```jsonc
{
  "schema_version": "2026.05.15-1",
  "instance_id": "wskop-wsi-01",
  "pci_id":      "0000:18:00.0",
  "wskop_version_at_build": "unknown",
  "generated_at_epoch_ms": 1778834567890,

  "counters": [
    {
      "path": "system/start_time",
      "type": "uint64", "kind": "gauge", "unit": "epoch_seconds",
      "description": "Unix epoch (seconds) when the wskop process started. Re-written on every wskop boot.",
      "status": "live",
      "monotonicity": "gauge:absolute",
      "owner_lcore_role": "system", "publish_cadence_seconds": 0.0,
      "source_file_line": "wskop.c:7632",
      "since_schema_version": "2026.05.15-1",
      "legacy_name": true,
      "notes": "Use this to detect wskop restarts: if value increases, the binary restarted between samples."
    },
    {
      "path": "system/version",
      "type": "string", "kind": "info", "unit": "",
      "description": "WSKOP_BUILD_VERSION macro at link time; 'unknown' if not provided to the build.",
      "status": "live",
      "monotonicity": "gauge:absolute",
      "owner_lcore_role": "system", "publish_cadence_seconds": 0.0,
      "source_file_line": "wskop.c:7633",
      "since_schema_version": "2026.05.15-1",
      "notes": "Trivial fix: build with -DWSKOP_BUILD_VERSION=<git describe> per discovery 08 Q13."
    },
    {
      "path": "workers/lcore_0/in_pkts_total",
      "type": "uint64_string", "kind": "counter", "unit": "packets",
      "description": "Packets received by worker lcore 0 from its RX queues (sum across queues).",
      "status": "live",
      "monotonicity": "monotonic_nondecreasing",
      "owner_lcore_role": "worker", "owner_lcore_id": 0,
      "publish_cadence_seconds": 1.0,
      "source_file_line": "wskop.c:5566",
      "since_schema_version": "2026.05.15-1",
      "legacy_name": false
    },
    {
      "path": "workers/lcore_0/out_drop_pkts_total",
      "type": "uint64", "kind": "counter", "unit": "packets",
      "description": "Packets dropped on the worker side (ring full at worker->TX hop, policy=DROP — see anomaly ACTION_DROP_misnomer).",
      "status": "live",
      "monotonicity": "monotonic_nondecreasing",
      "owner_lcore_role": "worker", "owner_lcore_id": 0,
      "publish_cadence_seconds": 1.0,
      "source_file_line": "wskop.c:5993,5998,6005,6010,6053,6057",
      "since_schema_version": "2026.05.15-1",
      "legacy_name": false,
      "anomalies": ["ACTION_DROP_misnomer"]
    },
    {
      "path": "workers/lcore_0/packets_dropped_tx_queue_full",
      "type": "uint64", "kind": "counter", "unit": "packets",
      "description": "Worker-side packets dropped because the TX queue was full. Currently registered but no increment site at HEAD.",
      "status": "expected_zero",
      "monotonicity": "monotonic_nondecreasing",
      "owner_lcore_role": "worker", "owner_lcore_id": 0,
      "publish_cadence_seconds": 1.0,
      "source_file_line": "wskop.c:5518",
      "since_schema_version": "2026.05.15-1",
      "legacy_name": true,
      "anomalies": ["unincremented_placeholders"]
    },
    {
      "path": "shapers/lcore_0/heartbeat",
      "type": "uint64", "kind": "counter", "unit": "",
      "description": "Per-lcore liveness counter from cmon — increments once per loop iteration.",
      "status": "not_wired",
      "monotonicity": "monotonic_nondecreasing",
      "owner_lcore_role": "shaper", "owner_lcore_id": 0,
      "publish_cadence_seconds": 1.0,
      "anomalies": ["shaper_transmit_cmon_unregistered"],
      "since_schema_version": "2026.05.15-1"
    },
    {
      "path": "watch/watched_flow_count",
      "type": "uint64", "kind": "gauge", "unit": "flows",
      "description": "Number of flows currently marked watched (WATCHED_BIT=38 on flow_info_d.policy_info). Computed by the watch lcore on the 1s tick.",
      "status": "live",
      "monotonicity": "gauge:absolute",
      "owner_lcore_role": "watch",
      "publish_cadence_seconds": 1.0,
      "source_file_line": "new — see runtime contract design 01 §1.6 / 04 §4",
      "since_schema_version": "2026.05.15-1",
      "anomalies": ["policy_info_truncation_in_collector"]
    }
  ],

  "anomalies": [
    { "id": "ACTION_DROP_misnomer", ... },
    { "id": "ipdr_cadence_1000x", ... },
    { "id": "unincremented_placeholders", ... },
    { "id": "policy_info_truncation_in_collector", ... },
    { "id": "shaper_transmit_cmon_unregistered", ... }
  ]
}
```

## 8. The unit-naming-convention question (resolved)

[01 §6](01_v1_producer_additions.md) introduces a `_total` suffix
convention for new counters. The manifest's `legacy_name` flag
documents which existing counters do not follow it. The contract
publishes both — the legacy names stay as-is, the new names
follow the convention. UI consumers using the path lists from the
manifest do not need to know which is which.

## 9. The version-string question (resolved)

`system/version` reads `unknown` on the rig because
`WSKOP_BUILD_VERSION` is not set at link time
([discovery 08 Q13](../dpi_runtime_data_contract_discovery/08_open_questions.md#q13)).
The manifest captures this:

- `wskop_version_at_build` field at the top — what the manifest
  expects.
- The per-counter `notes` for `system/version` flags it.
- A future build with `-DWSKOP_BUILD_VERSION=$(git describe)`
  doesn't break the contract; the value field of the published
  string just becomes meaningful.

Implementation phase note: the Makefile change is trivial and is
referenced in the bridge daemon's `install.sh` companion.

---

# 06 — v2 streaming layer

The streaming layer ships when the broker lcore lands (operator
A4 in
[discovery 08](../dpi_runtime_data_contract_discovery/08_open_questions.md#q4)).
This file specifies what gets emitted on the per-packet hot path,
what the broker does with it, and how the v1 bridge daemon
interoperates with v2.

**Scope reminder (operator A6):** **counters only, no packet
copies.** Per-flow summary records — 5-tuple + counters +
classification — drained over an SP/SC ring. No `rte_pktmbuf_clone`.

## 1. Per-worker watch-emit ring — element shape

**Ring:** one **SP/SC** `rte_ring` per worker, **drained by the
broker.** Single producer (the worker), single consumer (the
broker). DPDK SP/SC rings are lock-free with cache-friendly write
patterns; this is the same ring topology that the worker→TX path
already uses
([discovery 02 §1](../dpi_runtime_data_contract_discovery/02_lcore_observability_responsibilities.md),
[ffunctions.c:411](https://github.com/bdevskmank/wskop_v01/blob/3ff7479/ssg_src/ffunctions/ffunctions.c#L411)
pattern reference).

**Element shape — fixed-size struct, no pointers, sized to a
single cache line.**

```c
/* PSEUDOCODE — element type, lives in a new header e.g.
 * ssg_src/watch_emit/watch_emit.h that workers and broker share. */
struct watch_emit_record {
    /* 5-tuple, packed */
    uint32_t  src_ip;
    uint32_t  dst_ip;
    uint16_t  src_port;
    uint16_t  dst_port;
    uint8_t   protocol;
    uint8_t   _pad0;            /* keep word alignment */

    /* Identity */
    uint32_t  rss_hash;         /* matches m->hash.rss; key into flow tables */
    uint32_t  subs_ip;          /* from flow_d->subs_ip */

    /* Classification snapshot at emit time */
    uint32_t  svc_id;           /* from flow_d enriched cache (worker has it) */
    uint16_t  svc_cat;          /* same */

    /* Counters — instantaneous, monotonic since flow create.
     * Worker writes from flow_d after counter bump; broker computes deltas. */
    uint64_t  packet_count_fwd;
    uint64_t  byte_count_fwd;
    uint64_t  packet_count_rev;
    uint64_t  byte_count_rev;

    /* Timestamps */
    uint64_t  last_seen_tsc;    /* rte_get_timer_cycles() at emit time */

    /* Worker identity for the broker's draining bookkeeping */
    uint8_t   worker_id;
    uint8_t   _pad1[3];
};
/* sizeof = 64 (one cache line) */
```

**Why fixed-size:** no heap allocation, no copy-loop, single
`rte_ring_enqueue_sp` call. Single-cache-line copy out of `flow_d`
+ a few stack-resident scalars. Aligned with the B2.1 budget per
[discovery 05 §B.5(ii)](../dpi_runtime_data_contract_discovery/05_flow_watch_cost_analysis.md).

**Why no pointers (no mbuf reference):** operator A6 — counters
only. The record carries the values, not handles.

## 2. Per-worker ring sizing and back-pressure

**Ring depth:** **4096** entries per worker. Justification:

- Maximum sustained per-worker watched-flow rate at line rate
  (per discovery 05 §B.8) is the ring-enqueue cost; one SP/SC
  enqueue per watched packet on a 25 Gbps worker tops out at the
  ring's drain rate, not the depth.
- 4096 × 64 B = 256 KiB per worker — fits in L2 easily, no NUMA
  pressure. 8 workers × 256 KiB = 2 MiB total.
- Depth provides headroom for burst spikes when many flows of one
  subscriber start near-simultaneously (the §3 of
  [04 watched flag propagation](04_watched_flag_and_propagation.md)
  case for per-subscriber watch).

**Back-pressure: drop-on-full, count the drops.** Worker calls
`rte_ring_sp_enqueue_burst` (or `_sp_enqueue_bulk`); on failure
(ring full → broker stalled), it **does not block** and increments
a per-worker `watch_emit_drops_total` counter (published per the
metrics extension in [01](01_v1_producer_additions.md)).

```
/* PSEUDOCODE — worker hot path; placed inside run_packet_function
 * AFTER the existing counter bump and BEFORE the per-action dispatch. */

if (unlikely(is_watched(&flow_d->policy_info))) {
    struct watch_emit_record rec;
    rec.src_ip          = /* from m or flow_d */;
    rec.dst_ip          = /* ... */;
    rec.src_port        = /* ... */;
    rec.dst_port        = /* ... */;
    rec.protocol        = /* ... */;
    rec.rss_hash        = m->hash.rss;
    rec.subs_ip         = flow_d->subs_ip;
    rec.svc_id          = enriched->svc_id;       /* already-cached pointer */
    rec.svc_cat         = enriched->svc_cat;
    rec.packet_count_fwd= flow_d->packet_count_fwd;
    rec.byte_count_fwd  = flow_d->byte_count_fwd;
    rec.packet_count_rev= flow_d->packet_count_rev;
    rec.byte_count_rev  = flow_d->byte_count_rev;
    rec.last_seen_tsc   = flow_d->last_seen;
    rec.worker_id       = worker_id;

    if (rte_ring_sp_enqueue(watch_emit_ring[worker_id], &rec) < 0) {
        /* Drop-on-full; cold-path branch, not per-pkt. */
        worker_pe_counters[worker_id].watch_emit_drops_total++;
    }
}
```

The `unlikely()` keeps the no-watched-flow case at one bit-AND
plus a predicted-not-taken branch. Per [04 §4](04_watched_flag_and_propagation.md)
and [discovery 05 §B.2](../dpi_runtime_data_contract_discovery/05_flow_watch_cost_analysis.md)
this is single-digit cycles per packet on the no-watched-flow
path.

## 3. Cadence — per-event, not batched

**Default: emit one record per watched packet.** The UI gets
sub-millisecond resolution on a watched flow's counter movement.
The broker handles aggregation if the UI's display cadence is
slower; pushing aggregation into the worker would add hot-path
state (a "last emit tsc per flow") and a delay branch.

**Per-flow rate caps (broker-side):** the broker may impose a
"max 1 emit per flow per 10 ms" rate cap when forwarding to the
bridge — purely a broker-side decision; the worker emits
unconditionally on watched packets. This keeps the broker drain
loop's CPU usage bounded if a watched flow runs at 10 Mpps.

## 4. Broker — what it does with drained records

**The broker is the consumer.** A new cold lcore, scoped by task
(e) per
[INDEX.md](../../INDEX.md). When it lands the runtime contract's
v2 producer side plugs into it.

```
┌─ worker 0 ─┐        ┌─ worker 7 ─┐
│ SP enq    →│  ...   │ SP enq    →│
└────┬───────┘        └────┬───────┘
     │  SP/SC ring × 8 (one per worker)
     ▼                     ▼
   ┌───────── broker lcore (cold) ────────────┐
   │ rte_ring_sc_dequeue_burst for each ring  │
   │   ↓                                      │
   │  aggregation / rate-cap / format         │
   │   ↓                                      │
   │ write JSON-Lines to                      │
   │   /mnt/tmp_ramfs/run/<pci>/streams/watch.jsonl
   │   (rotating; tail-friendly)              │
   └──────────────────┬───────────────────────┘
                      │ tails
                      ▼
   ┌── wskop-bridge ──────────────────────────┐
   │ pushes events to subscribed UI clients   │
   │ via "kind":"stream_event" frames         │
   │ over the existing UNIX socket            │
   └──────────────────────────────────────────┘
```

**The broker writes a streaming file.** Path:
`/mnt/tmp_ramfs/run/<pci_id>/streams/watch.jsonl`.

Each line is a JSON object — the same record-shape format the
bridge will forward to UI clients. Format follows the wire-
protocol JSON conventions in [03](03_wire_protocol.md):

```jsonl
{"v":1,"kind":"stream_event","stream":"watch",
 "epoch_ms":1778834567890,
 "rss":1234567,"subs_ip":167772417,
 "src":"10.0.0.1","dst":"203.0.113.5","sport":54321,"dport":443,"proto":6,
 "svc_id":42,"svc_cat":7,
 "pkts_fwd":1023,"bytes_fwd":1500000,"pkts_rev":1024,"bytes_rev":3000000,
 "last_seen_ms_ago":2}
```

**Why a file on ramfs and not a direct broker→bridge UNIX socket
(or DPDK cross-process ring):**

- **Process boundary alignment.** The broker is in-wskop (it must
  be, to drain DPDK SP/SC rings); the bridge is out-of-wskop
  ([02 §1](02_bridge_daemon_design.md)). The shared substrate
  they both can touch is the ramfs.
- **No new cross-process IPC in wskop.** A DPDK cross-process
  ring would couple bridge lifecycle to wskop lifecycle and
  reintroduce the "if the bridge dies wskop pays" coupling the
  bridge daemon was designed to avoid.
- **Crash isolation.** If the bridge segfaults, the broker keeps
  writing to the rotating file; on bridge restart it reopens and
  resumes tailing from the latest known offset (or whatever its
  reconnect policy is — see §5).
- **JSON-Lines vs. struct-on-disk:** the broker's write rate is
  bounded — at the rate-cap of ~1 event per watched flow per
  10 ms × hundreds of watched flows × a few KiB per line ≈
  hundreds of KiB/s. Trivial on ramfs.

**Rotation:** standard log-rotation pattern. `watch.jsonl`
rotates to `watch.jsonl.<unix>` on a size threshold (e.g. 16 MiB)
or time threshold (e.g. 60 s); broker writes to the current
file. Bridge tails the current file and seamlessly follows the
rename on rotation (`inotify` for `IN_CREATE`/`IN_MOVED_FROM`).

### 4.1 Why the broker writes JSON-Lines

Same protocol family as v1 polled responses ([03](03_wire_protocol.md)).
Bridge daemon parses one line, wraps it into a wire-protocol
frame, pushes to subscribed clients. No transformation needed
beyond optional filtering (e.g. only push events for flows the
client subscribed to).

## 5. Bridge ↔ broker interaction

### 5.1 Bridge tails the broker's file

On a client's `kind="subscribe"` request with paths like
`watched/flow/<rss>` or `watched/all`, the bridge:

1. Opens (or already has open) `/mnt/tmp_ramfs/run/<pci>/streams/watch.jsonl`.
2. Sets up an `inotify_init1` watch on the directory for
   `IN_CREATE` (new file after rotation) and `IN_MODIFY` (new
   lines appended).
3. On `IN_MODIFY`, reads new bytes, splits on `\n`, parses each
   line as JSON, filters per the client's subscribe path set,
   forwards as `kind="stream_event"` frames.

The bridge does **not** seek backward through old files — it
delivers events from the moment of subscribe onward. Historical
data is the polled layer's job.

### 5.2 The wire frame on the socket

Same envelope as the polled responses:

```json
{ "v":1, "id":"<subscribe_id>", "kind":"stream_event",
  "stream":"watch",
  "epoch_ms": 1778834567890,
  "record": { "rss": 1234567, ... (fields as in §4) } }
```

The `record` object is the broker's JSON-Lines content. The
bridge does not modify it; the bridge does add the protocol
envelope and the request correlator.

### 5.3 Does the broker also speak the UNIX socket directly?

**No.** Brief explicitly asks "Does the bridge daemon also serve
v2 streams, or does the broker speak directly?" — **the bridge
serves both.** Reasons:

- One socket per instance (per
  [02 §3](02_bridge_daemon_design.md)) keeps the UI's
  connection model simple.
- The broker would have to grow socket-listener code anyway
  (accept, epoll, per-client buffering, framing) — duplicating
  half the bridge.
- The bridge already speaks the wire protocol; the broker
  speaking the same protocol from a separate listener risks the
  two implementations drifting.

The UI does **not** need to know whether it is consuming a polled
or a streamed counter — both arrive via the same socket, the
same subscribe semantics, just with different `kind` (`values`
vs `stream_event`).

## 6. Hot-path impact statement — v2

**Required section per the brief.** Every site touched, classified
hot vs cold, worst-case per-packet cost on hot sites.

### 6.1 Sites touched by v1 producer additions

| Site | File:line at HEAD | Hot or cold | Per-packet cost |
|---|---|---|---|
| Worker 1 s tick branch publishes core_stats | `wskop.c:5547+` (existing branch) | **cold** (1 s gated) | 0 cycles on hot path |
| Shaper 1 s tick branch publishes core_stats | new branch top of shaper loop | **cold** (1 s gated, unlikely-marked) | ~1 cycle on the loop iteration boundary, not per packet |
| Transmit 1 s tick branch publishes core_stats | new — top of transmit loop | **cold** (1 s gated, unlikely-marked) | ~1 cycle on loop boundary |
| Watch 1 s tick: ports/pools/liveness/aggregate | `wskop.c:6824+` (existing branch) | **cold** | 0 cycles on worker hot path |
| `metrics_init` roles[] adds 3 entries | `metrics.c:103` | **startup** | 0 cycles |
| `pe_action_identify_prb_common` watched stamp | per [04 §3.3](04_watched_flag_and_propagation.md), inside this function (`ffunctions.c:1793`) | **cold** (first-pkt-only branch) | ~1 cycle amortised over flow lifetime |
| `subscriber_info.watched` field RMW (redis_cli) | new — in subscriber update path on redis_cli lcore | **cold** | 0 cycles on worker hot path |
| Walk subscriber `flow_list` on operator mark | new — in redis_cli on Redis intent | **cold** | 0 cycles on worker hot path |

**Net v1 hot-path delta: zero.** The acceptance criterion is met
by construction.

### 6.2 Sites touched by v2 streaming producer

| Site | File:line at HEAD | Hot or cold | Per-packet cost |
|---|---|---|---|
| `run_packet_function` watched-bit test | `ffunctions.c:686` (`get_aid(policy_info)` already loads the line); new branch immediately after | **HOT** | 1 bit-AND + 1 predicted-not-taken branch ≈ **1-2 cycles** |
| `run_packet_function` SP/SC enqueue (on `is_watched`) | new — inside the `if(unlikely(is_watched))` branch | **HOT** (taken only for watched flows) | ~10-15 cycles for `rte_ring_sp_enqueue` per discovery 05 §B.5; ~50 cycles for the record's stack init + copy |
| `worker_pe_counters[].watch_emit_drops_total` increment | new — inside the ring-full branch | **HOT** (taken only on ring-full) | rare; not budget-relevant |
| Broker lcore drain loop | new — broker is task (e) | **cold** (broker lcore) | 0 cycles on worker hot path |
| Broker writes `watch.jsonl` | new | **cold** | 0 cycles on worker hot path |

**Worker hot-path worst case per packet:**

- **No-watched-flow case** (typical): 1 bit-AND on already-loaded
  `policy_info` u64 + 1 predicted-not-taken branch = **1-2 cycles**.
- **Watched-flow case** (rare): 1 bit-AND + 1 taken branch +
  ~50-65 cycles for record build + SP/SC enqueue. Aligns with
  the discovery's [05 §B.8](../dpi_runtime_data_contract_discovery/05_flow_watch_cost_analysis.md)
  envelope.

### 6.3 Acceptance criterion (cannot run, stating it)

**No throughput regression vs. `pre-b1-cleanup-complete` baseline
tag with WATCHED_BIT clear on all flows and v1 polled producer
additions live.** The criterion is identical to the one that
gated stage-2 of task (a)
([discovery 05 §B.1](../dpi_runtime_data_contract_discovery/05_flow_watch_cost_analysis.md))
and task (b) (
[task_info/INDEX.md row (b)](../../INDEX.md)): A/B throughput
delta ≤ noise floor (~5 %) on the rig's loop with no watched
flows. The watched-flow case is **not** subject to the same gate
— it is "as cheap as we can make it but cost is acknowledged."

The brief explicitly says "you can't run it, but state it." This
section states it. The implementation phase runs the A/B.

### 6.4 The B2.1 lesson, restated for v2

The B2.1 incident
([discovery 05 §B.1](../dpi_runtime_data_contract_discovery/05_flow_watch_cost_analysis.md))
showed that instrumentation cost on the shaper hot path crashed
throughput from ~97 → ~6 Gbps even though every other signal was
clean. **Only throughput is authoritative.**

The design minimises per-packet hot-path cost to a single bit-AND
plus a predicted-not-taken branch. The cost is structurally
**below** the deltas measured for tasks (a) and (b) (∆ +0.16 %
and ∆ +2.9 %) because the bit is being read from a line that is
**already loaded** by `get_aid`
([ffunctions.c:686](https://github.com/bdevskmank/wskop_v01/blob/3ff7479/ssg_src/ffunctions/ffunctions.c#L686)).

The only path that adds real cost is the watched-flow taken
branch — by construction rare, and a UI decision to mark "many"
flows as watched is an explicit choice the operator makes
knowing the cost. The §3 broker rate-cap optionally throttles
the cost of mass-marked watches, but the hot-path cost itself is
still incurred. **The UI should expose how many flows are
watched** ([01 §1.6](01_v1_producer_additions.md)'s `watched_flow_count`)
so the operator sees the choice they're making.

## 7. Staging — v2 layered onto v1

| Stage | Adds | Wskop-side change | Hot-path? | Acceptance |
|---|---|---|---|---|
| **W1** | `WATCHED_BIT` and accessors | `bit_operations.h` define + 3 inlines | none | unit tests pass |
| **W2** | `subscriber_info.watched` field + redis_cli marker writes | new field on `subsc_w_prb_ref`; new keyspace event handling on redis_cli | none on worker hot path | mark/unmark via redis_cli observable on rig |
| **W3** | `pe_action_identify_prb_common` stamps watched on first packet | one branch in cold first-pkt path | cold first-pkt only | watched_flow_count moves on flow creation |
| **W4** | v1 polled `watched_flow_count` / `watched_subscriber_count` | watch 1s walks (per [01 §1.6](01_v1_producer_additions.md)) | none | counters track marks via UI |
| **W5** | per-worker SP/SC `watch_emit_ring` + worker-side enqueue | ring creation in main; per-pkt branch in `run_packet_function` | **HOT** (1-2 cycles unwatched, +50-65 cycles per watched pkt) | throughput A/B vs. `pre-b1-cleanup-complete`: unwatched-only ∆ ≤ noise |
| **W6** | broker lcore consumes rings, writes `watch.jsonl` | broker is task (e) — design follows whatever task (e) ships | cold | `watch.jsonl` populates on rig with watched flows |
| **W7** | bridge tails `watch.jsonl` and pushes `kind="stream_event"` | bridge-side change in `wskop_bridge_v01/` | n/a | UI subscribed to `watched/flow/<rss>` receives events |

**W1-W4** can land before the broker (no per-packet hot-path
work; the watched bit is set but not read by anything beyond the
watch lcore's 1 s counter computation). The v1 polled
`watched_flow_count` is meaningful from W4 onward.

**W5-W7** require the broker (task (e)). The worker-side
enqueue (W5) **only makes sense** if a consumer exists; W5 +
W6 land together.

## 8. Stop-gap if the broker slips

Per [discovery 08 Q9](../dpi_runtime_data_contract_discovery/08_open_questions.md#q9):
if v2 is needed before the broker lands, the watch lcore is the
candidate stop-gap consumer. The cost: watch lcore is already
busy on the 1 / 5 / 10 s ticks
([discovery 02 §4](../dpi_runtime_data_contract_discovery/02_lcore_observability_responsibilities.md))
and adding "drain N SP/SC rings continuously" pushes against its
loop budget.

The recommendation: **do not ship v2 ahead of the broker.** v1
(polled counters + `watched_flow_count` / `watched_subscriber_count`)
covers the UI's "watch this flow / subscriber" gesture at 1 Hz
resolution — visibly slower than per-event, but not useless.
Stop-gap noted as a possibility, not a commitment;
[08 Q5](08_open_questions.md#q5) flags it for re-decision when
the broker timeline firms up.

## 9. The v1 ↔ v2 bridge — how the UI tells them apart

It doesn't have to. Both arrive on the same socket, both use
`kind="values"` or `kind="stream_event"`. The UI subscribes to:

- `workers/lcore_0/in_pkts_total` — polled (1 Hz, `values`).
- `watched/flow/12345/counters` — streamed (per-event, `stream_event`).

The bridge knows which is which from the path prefix in the
subscribe. The UI does not have to know which it is asking for;
it just asks. The bridge produces `values` from the ramfs tree
and `stream_event` from the broker's JSON-Lines file.

(If the UI consumer **does** want to know the source, the
`stream_event` envelope's `stream` field tells it. The `values`
response does not need an equivalent — its `path` already tells.)

## 10. What v2 does NOT do

- Does not clone packets. Operator A6.
- Does not introduce a hot-path mempool. No `init_clone_pool`
  resurrection. Per
  [discovery 05 §A.4](../dpi_runtime_data_contract_discovery/05_flow_watch_cost_analysis.md).
- Does not surface per-packet contents to the UI. Only the
  fields in §1.
- Does not bypass the bridge daemon. Even at full v2, the UI's
  socket surface is the bridge's UNIX socket — same wire
  protocol, same subscribe semantics.
- Does not require redesigning `flow_info_d`. The watched bit
  lives in `policy_info`; the record is built from existing
  `flow_d` fields plus the enriched-table cache the worker
  already has in hand.
- Does not modify `ssg_src/metrics/`. The streaming layer uses a
  parallel `streams/` subtree under `/mnt/tmp_ramfs/run/<pci>/`
  with a different write mechanic (append + rotate) than
  `metrics_set`'s atomic-overwrite-of-one-value.

---

# 07 — Observations (opinion)

Clearly labelled opinion. Descriptive design lives in 01-06. Open
questions in 08. Nothing here should be cited as a decided
contract claim without being moved into 01-06 first.

## 1. The polled layer carries the UI for surprisingly long

The aggregate set of `kind="values"` and `kind="subscribe(interval=1s)"`
responses serves every UI need that doesn't end in the phrase
"per packet."

- Per-lcore throughput, drops, heartbeat, EMA — 1 Hz.
- Per-port and per-pool — 1 Hz.
- Per-flow live counters of watched flows — at 1 Hz via
  `watched_flow_count` aggregate plus per-flow polled lookups
  through the enriched table.

The streaming layer (v2) earns its place only when the UI wants
**sub-second** resolution of per-flow change — debugging a stuck
flow, or watching a real-time speed-test for a single subscriber.
That's a real need but it's narrower than "the UI needs streaming
for everything."

My opinion: **ship v1 well, leave v2 to land when the broker
lands.** Pushing v2 ahead of the broker would compress the
schedule and force a stop-gap consumer (watch lcore drain) that
introduces hot-path coupling for marginal UI gain.

## 2. The bridge daemon is much smaller than the producer-side work

Producer additions (01) touch wskop in 6 staged commits and need
A/B throughput acceptance per stage. The bridge daemon (02-03)
is one new binary, one systemd unit, one wire-protocol JSON
parser. It has no `flow_info_d` exposure, no DPDK linkage, no
hot-path cost. **A small team of one can ship the bridge in
parallel with the wskop-side counter additions** and they
converge at install time.

The implementation phase should split labour accordingly: the
wskop changes need someone who has already built throughput
A/B on the rig; the bridge daemon can be a one-week task for
anyone comfortable with C, JSON, and `epoll`. Or in any language
the operator prefers — Go or Rust would be smaller-error-surface
without performance impact at the bridge's traffic rate.

## 3. The metric handle's `_close` semantics deserve a second look

[metrics.h:68-72](https://github.com/bdevskmank/wskop_v01/blob/3ff7479/ssg_src/metrics/metrics.h#L68-L72)
says: "Called from the owning lcore on shutdown… `fsync`s each fd
before close so final values are durable, then frees the handle."
That `fsync` is expensive on a ramfs (no-op semantically; one
syscall overhead per fd anyway). The intent reads like "flush to
disk before we lose them" but ramfs is RAM — values are durable
only as long as the process is up.

**Not a bug** — just a comment that the close path's `fsync`
could be conditionally dropped if `/mnt/tmp_ramfs` is mounted
`tmpfs`. Or kept, costing one ms total per shutdown, which is
fine. The runtime data contract doesn't change `_close`
semantics; flagged so the implementation phase doesn't trip on
it.

## 4. The cross-thread-write hazard's role in this contract

The discovery's [07 §3](../dpi_runtime_data_contract_discovery/07_observations.md)
described the hazard as "a chronic condition, not an emergency"
and recommended the contract design **acknowledge** it. This
design does. [04 §3](04_watched_flag_and_propagation.md) explicitly
lists each cross-thread write the contract introduces:

- redis_cli sets `subscriber_info.watched` (not `flow_info_d`).
- redis_cli on per-flow / per-subscriber mark walks `flow_list`
  and sets the watched bit on each `flow_d->policy_info` — the
  **only new cross-thread write to `flow_info_d`** in this contract.

That single write joins the existing PRB-recompile hazard class.
It must use the existing soft-lock protocol (
`set_policy_info_lock_external` → mutate → `reset_lock_ffunction_n_external`),
not bypass it. **If the policy_version-push refactor in
[`policy_version_refactor_design/`](../policy_version_refactor_design/)
lands first, the watched bit's write moves into `pending_updates`
and the cross-thread write goes away.** I think this is
likely — that refactor is review-ready and the runtime contract
is still in design — so the implementation phase should plan for
either ordering.

## 5. The brief's "broker is being added" answer reshaped the v2 design

Before operator A4, v1+v2 looked like "polled layer now, decide
streaming layer based on whether the broker shows up." After
A4 ("broker is coming soon"), v2 reads cleanly as "broker drains
the per-worker rings; writes a JSON-Lines file under
`streams/`; bridge tails." That structure is markedly simpler
than the alternative ("watch lcore drains rings" or "build a
new dedicated lcore for this purpose").

My opinion: **the broker should land before v2 ships, even if
the broker initially only does the watch-emit drain and nothing
else.** The broker's task brief (task (e)) presumably has a
broader scope; carving a "broker v1 = watch-emit drain only"
slice may compress the integration window between task (e) and
this contract.

## 6. The schema-version pattern is the under-celebrated part of the wire protocol

The wire-protocol `v` is the boring breaker-of-things; the
manifest `schema_version` is what actually decouples wskop
changes from UI changes. The contract pays for it once (the
manifest file) and then **every** counter addition or annotation
update is a `schema_version` bump + a per-instance manifest
regenerate, no protocol or bridge change required.

This matters because the discovery flagged five counter-
semantic anomalies (drop misnomer, IPDR cadence, unincremented
placeholders, policy_info truncation, shaper/transmit cmon) —
any of which could change as wskop evolves. Carrying the
anomaly metadata in the manifest lets the contract correct its
own documentation without code change on either side.

Side note: the `legacy_name` flag is a small thing but it solves
the "do we rename `flow_table_occupancy` to `flow_table_occupancy_total`
to follow the new convention" question by saying **no** — keep
the name, label it legacy in the manifest, move on. The
alternative is a renaming PR that doesn't make anything actually
work better.

## 7. The bridge daemon as a sibling systemd service is the right shape, but…

The `PartOf=` plus `After=` plus `Requires=` triplet on the
bridge unit ([02 §4](02_bridge_daemon_design.md)) means the
bridge dies when wskop dies and starts after wskop. That's
correct but for one edge case: **dev / debug shells.** An
operator running `wskop-shared` by hand for a strace session
doesn't have `wskop-wsi@.service` running and therefore can't
inspect via the bridge.

Workaround: a `--standalone` flag on the bridge that skips the
`Requires=` and lets the bridge serve whatever is in the
metrics tree, even if wskop is not running via systemd. For dev
only; production uses the standard systemd shape.

Not in v1. Flagged for the implementation phase to consider.

## 8. The wire protocol should be tested as much as the producer

I keep nudging this back at the design phase: the bridge's
protocol parser and the UI's protocol writer have to agree, and
neither has a wskop or rig dependency. **A golden-fixture test
suite** (one JSON file per protocol exchange — request and
expected response) belongs in the satellite repo from day one.
The bridge runs the fixtures locally and the UI side runs them
against a mock socket. Catches every regression that a protocol-
v bump or a manifest-schema-bump could introduce.

This is straightforward but the kind of thing that doesn't get
done unless it's in the design. The `wskop_bridge_v01` skeleton
should include `test/protocol_fixtures/` with the kind="ping"
exchange already written; everything else follows.

## 9. The `WSKOP_METRICS_BASE_DIR` env override is a quietly important feature

[metrics.c:37-44](https://github.com/bdevskmank/wskop_v01/blob/3ff7479/ssg_src/metrics/metrics.c#L37-L44)
honours `WSKOP_METRICS_BASE_DIR` if set. The wskop binary today
ignores it
([metrics/README.md "Mount requirement"](https://github.com/bdevskmank/wskop_v01/blob/3ff7479/ssg_src/metrics/README.md#mount-requirement))
because the smoke test uses it and the production path is meant
to be unconditional. But the **bridge daemon should accept the
same env** (via its `--metrics-base` flag in
[02 §4](02_bridge_daemon_design.md)).

Why this matters: lets the bridge and a tiny test-rig wskop
build run against `/tmp/wskop_metrics_test/` instead of
`/mnt/tmp_ramfs/`, end-to-end, without root. Useful for CI of the
satellite repo. The protocol behaviour is identical.

## 10. The contract should not become "the wskop API"

The discovery noted that wskop's external surface is sparse
(stderr, IPDR CSV, Redis-via-command). The runtime data contract
adds a UNIX socket. That is enough for the UI's read needs. It
must **not** become the inverse — operators using the bridge for
write commands ("watch this flow", "drain this pool", etc.).
Those belong on the existing Redis command channel.

If a "wskop CLI surface" is genuinely wanted, the right next
step is to revive the dead `simple_cli.c`
([discovery 02 §0](../dpi_runtime_data_contract_discovery/02_lcore_observability_responsibilities.md))
with its own UNIX socket and command grammar, **not** to grow
write semantics into the bridge. Discovery 08 Q14 flagged this
already; my opinion is that growing the bridge into a CLI surface
is a refactor everyone will regret in six months.

## 11. The `system/start_time` counter is the UI's freshness key

In [05 §7](05_counter_semantics_manifest.md)'s manifest, the
`notes` for `system/start_time` mention using it to detect wskop
restarts. I think this is **load-bearing** for the UI's
correctness story and deserves more than a `notes` mention:

- If the UI displays a stale value because the bridge cached
  something across a wskop restart, the cached value can mislead
  for arbitrary time. The UI watching `start_time` is the only
  protection.
- The bridge doesn't cache anything in v1 ([02 §6](02_bridge_daemon_design.md))
  so the issue is mostly hypothetical, but v2's streaming layer
  has a stream-rotation lifecycle that intersects with wskop
  restart in subtle ways — when wskop comes back, the old
  `watch.jsonl` file disappears (or is renamed via rotation) and
  the bridge has to re-tail.

Recommendation: the implementation phase explicitly **monitors
`system/start_time` in the bridge** and emits a synthetic
`kind="event"` with `kind="wskop_restart"` to all connected
clients when it observes the value advancing past a prior
snapshot. UI consumers can then refresh manifest + reset their
local rate caches.

Not blocking; a useful little addition.

## 12. Where this contract intersects with the config-ingest design

[dpi_config_ingest_discovery 08 §11](../dpi_config_ingest_discovery/08_open_questions.md)
asked whether the runtime contract and the config-ingest report
should converge into a shared "config and observability surface"
view. **No** — the two surfaces serve different operators and
have different lifecycles:

- Config ingest is **operator-set, before wskop starts** (DB →
  JSON → CLI per
  [dpi_config_ingest 01 §6](../dpi_config_ingest_discovery/01_config_file_inventory.md)).
- Runtime contract is **wskop-produced, while running** (metrics
  tree → bridge → UI).

They share `working_dir` (`/mnt/tmp_ramfs/`) and `pci_id` strings,
but those are inputs to both, not outputs of either. Keep them
separate. The UI may eventually combine them into one display
pane; the contracts behind them stay distinct.

## 13. The most likely thing to surprise the implementation phase

In ranked order of how-likely-this-bites:

1. **The transmit lcore lacks a `metrics_handle_t` today.** The
   first stage of [01 §7](01_v1_producer_additions.md) needs
   transmit lcore to open one, register, and close it. This is
   a small new code block, but it lives inside a function that
   the discovery noted has no existing instrumentation — easy to
   miss in review and easy to misplace inside the for-loop.
2. **`pe_action_identify_prb_common` has multiple variants
   that need the same watched-stamp.** Per
   [discovery 05 §A.2](../dpi_runtime_data_contract_discovery/05_flow_watch_cost_analysis.md)
   there are at least three `pe_action_identify_prb*` paths. The
   watched stamp in [04 §3.3](04_watched_flag_and_propagation.md)
   needs to be in **all** of them; missing one means flows
   created on the missed path will not be watched. The
   policy-version-push refactor centralises this into the
   walker, but until that lands the implementation must cover
   each variant.
3. **DPDK's `rte_eth_stats_get` may not be safe to call from a
   non-main lcore without setup.** The discovery flagged that
   the function is not currently called periodically; I haven't
   verified that the watch lcore can call it safely. The
   implementation should test on the rig before committing to
   the watch-lcore placement; if it fails, move the per-port
   stats to the main lcore's startup-and-no-periodic shape and
   accept "stats from the boot moment only" until a better
   placement is found.

Each of these is solvable; flagged so the implementer knows to
budget time for them.

## 14. Final shape thought

This is a design where most of the value comes from declaring
boundaries — between v1 polled and v2 streaming, between
producer-side wskop changes and consumer-side bridge daemon,
between hot path and cold path. The boundaries do most of the
work; the code inside each box is small. The implementation
phase's main job is to **respect the boundaries** (no hot-path
write of values that v1 doesn't already publish on the cold tick;
no bridge involvement in writes to the metrics tree; no Redis
mirror creeping in). Everything else is pseudocode to keystroke
conversion.

---

# 08 — Open questions

For the implementation phase or the operator. Questions that had
a discoverable code answer were resolved into 01-06, not left
here.

Each question lists who I think can answer it. **Operator answers
2026-05-15 are inlined under each question** below; load-bearing
decisions have been folded back into the corresponding files
01-06.

## For the implementation phase

### Q1. Manifest generation — hand-written, code-generated, or hybrid?

Per [05 §6](05_counter_semantics_manifest.md): three options
(a) hand-written JSON, (b) code-generated from `metrics_register`
call sites, (c) hybrid. **The contract works with any of them**;
the question is which the implementation phase wants to maintain.

My weak recommendation: **(c) hybrid** — code-generated path list,
hand-written semantics. Catches "added a counter, forgot to
update the manifest" via a CI check; lets descriptions and
status be maintained in source review.

Stakes: low. Either (a) or (c) ships v1; (b) is awkward because
it can't infer `description` / `status` / `anomaly` linkage.

**A (operator, 2026-05-15):** **(c) hybrid.** Folded into
[05 §6](05_counter_semantics_manifest.md): the manifest
generator is code-driven for the path list, hand-written for
descriptions / `status` / anomaly linkage.

### Q2. Bridge daemon implementation language

C, Rust, Go, or other? The wskop tree is C. The bridge does not
link wskop; the bridge has no DPDK dependency; the bridge's
runtime cost is "epoll loop + JSON parser + file reads."

C keeps the toolchain consistent. Rust / Go cut the bug surface
on JSON parsing and error handling. Either works.

**Recommendation:** ship in C if there is an existing C-builder
in the implementation team; otherwise Rust or Go for the smaller
bug surface. Not a contract concern; satellite repo's `daemon/`
directory can host either.

**A (operator, 2026-05-15):** **C.** Same toolchain as wskop_v01.
Folded into [02 §1](02_bridge_daemon_design.md). Satellite repo's
`daemon/` directory will be a C source tree.

### Q3. `ports/` vs extended `system/<port_N>/` directory naming

Per [01 §1.2](01_v1_producer_additions.md): per-port DPDK stats
go under either `ports/port_<P>/` (new top-level role) or
`system/port_<P>/` (sub-namespace of system). The contract uses
`ports/` in this document; the implementation can pick either as
long as the manifest path strings match.

Stakes: low. `ports/` is slightly cleaner; either is consistent.

**A (operator, 2026-05-15):** **`ports/` is fine.** No change to
[01 §1.2](01_v1_producer_additions.md).

### Q4. `cmon_register_core` on shaper / transmit — wire now or defer?

Per [01 §4](01_v1_producer_additions.md) and the anomaly
[05 §4.5](05_counter_semantics_manifest.md). Adding the
registration would close the `shapers/lcore_*/heartbeat` and
`transmit/lcore_*/heartbeat` gap. The per-loop cost is roughly
worker-equivalent (3-5 cycles per loop iteration, not per
packet), measurable by the existing throughput A/B harness.

**Open** because (a) it is its own small workstream that doesn't
need to ride with this contract, and (b) it has its own per-loop
cost that needs measurement.

**Default:** ship v1 without; manifest entries for the missing
heartbeats carry `status="not_wired"` and the bridge returns
`value=null`. UI displays "n/a" for shaper / transmit heartbeats.

Owner: whoever owns `ssg_src/profiling/` per CLAUDE.md
"profiling broken, needs refactor."

**A (operator, 2026-05-15):** **ship v1 without.** Confirms the
default. Shaper/transmit heartbeat counters keep
`status="not_wired"` in the manifest until someone wires
`cmon_register_core` on those roles as a separate workstream.

### Q5. Stop-gap consumer for v2 if the broker slips

Per [06 §8](06_v2_streaming_layer.md). My recommendation in
that section is **do not ship v2 ahead of the broker**; v1's
1 Hz `watched_flow_count` covers the use case at lower
resolution.

**Open** because the broker timeline (operator A4: "broker is
coming soon") is not pinned to a specific date. If task (e)
slips by N months, the operator may want to revisit. The
question for the operator at that point: "is the watch lcore
acceptable as a stop-gap consumer of the per-worker
watch-emit rings, accepting some risk of pushing its loop
budget?"

Re-decision trigger: revisit when task (e) ships W1 of its own
or formally slips a quarter.

**A (operator, 2026-05-15):** **no shipping v2 before the
broker.** v2 producer-side hot-path enqueue (W5 in
[06 §7](06_v2_streaming_layer.md)) does not land ahead of the
broker. v1's `watched_flow_count` / `watched_subscriber_count`
at 1 Hz covers the UI need until the broker arrives. Re-decision
trigger above is also retired — the policy is now "wait."

### Q6. Wire protocol error code naming

[03 §4](03_wire_protocol.md) lists 9 error codes. The
implementation phase may want to consolidate or split (e.g.
distinguish `path_missing_in_manifest` from `path_missing_at_runtime`).
The contract's behavioural shape doesn't depend on the names;
the consumer-side error matching benefits from stable codes once
chosen.

**Stakes: low.** Lock in the names with the first golden-fixture
test ([07 §8](07_observations.md)) and bump
`schema_version`-style if any rename is needed later.

**A (operator, 2026-05-15):** **error code naming is fine
as-is.** Lock in with the first golden-fixture test.

### Q7. Whether the bridge should auto-detect wskop restarts

Per [07 §11](07_observations.md): bridge monitors
`system/start_time` and emits a synthetic `kind="event"` /
`type="wskop_restart"` when it observes the value advancing.

**Stakes: low.** Useful but not in v1. The UI can do the same
detection client-side from the `system/start_time` value.

**Open** if v1 wants to ship this feature or punt to v1.1. I lean
punt.

**A (operator, 2026-05-15):** **ship the auto-detect in v1.**
The bridge monitors `system/start_time` and emits a synthetic
`kind="event"` with `event.type="wskop_restart"` to all
connected clients on observing the value advance. UI consumers
refresh the manifest + reset local caches on receipt. Promoted
from observation
([07 §11](07_observations.md)) to spec in
[02 §5.4](02_bridge_daemon_design.md).

### Q8. Subscribe rate-cap and limit

[03 §3.4](03_wire_protocol.md) clamps `interval_seconds` to
`[1.0, 60.0]` and limits each client to 16 active subscribes.
Implementation may want these as config flags rather than
hardcoded.

**Stakes: low.** Reasonable defaults; tune as needed.

**A (operator, 2026-05-15):** **figures are fine.** `[1.0, 60.0]`
clamp on `interval_seconds` and 16-subscribe cap per client land
as the v1 defaults; revisit only if a workload exercises them.

### Q9. Bridge daemon's stance on the schema-version mismatch

If a UI client connects with a stale `schema_version` (the
client has v=`2026.05.15-1` and the bridge has v=`2026.06.01-2`),
the bridge accepts requests as long as the protocol `v` matches;
the client is responsible for refetching the manifest if it
cares.

**Open** whether the bridge should bounce stale clients or be
lenient. **Recommendation: lenient.** Force the UI to be the
source of truth on freshness.

**A (operator, 2026-05-15):** **lenient.** Bridge accepts
requests as long as wire protocol `v` matches; manifest
`schema_version` mismatch is the UI's concern. The UI refreshes
the manifest after a `kind="wskop_restart"` event (Q7) or on
schedule.

## For the operator

### Q10. UI consumer language / runtime

[Discovery 08 Q5 follow-up](../dpi_runtime_data_contract_discovery/08_open_questions.md#q5)
said the UI is a "container-wrapped OS service." Knowing the
runtime (Node.js? Python? Go?) sharpens the wire-protocol's
choice between `uint64` JSON numbers and `uint64_string`
strings ([03 §6](03_wire_protocol.md)) — a Node UI must use
strings for everything above `2^53`, while a Python or Go UI
parses both seamlessly.

**Doesn't block.** The protocol already declares both forms
explicitly. Knowing the UI runtime helps tune the default for
new counters (which type to assign in the manifest).

**A (operator, 2026-05-15):** **Python.**

**Technical implications — none restrictive.** Python's `int` is
arbitrary-precision and Python's stdlib `json` parses both JSON
numbers and JSON strings without precision loss, so the
contract's `type: "uint64"` (JSON number) and `type: "uint64_string"`
(JSON string of decimal) both decode cleanly in Python. Concretely:

- `json.loads('{"v": 9007199255000000}')['v']` → `9007199255000000`
  (Python `int`, exact). No `float` cast happens unless the UI
  code does it explicitly.
- `json.loads('{"v": "9007199255000000"}')['v']` → `"9007199255000000"`;
  the UI calls `int(...)` to widen.

The asymmetry between Node (where any number > 2^53 silently
loses precision unless serialised as a string) does not apply.
Recommendation for the manifest in
[05 §3.5](05_counter_semantics_manifest.md): **continue to use
`uint64_string` for byte-cumulative counters** (the safer
default), but the manifest is free to use plain `uint64` for
anything bounded under 2^53 (most counters in practice). The UI
handles both.

If the UI later swaps to a Node frontend, this choice protects
it; if the UI stays Python, neither form is wrong.

### Q11. Bind-mount path conventions in the UI container

The bridge socket is at `/run/wskop-wsi/<instance>/bridge.sock`
on the host ([02 §3](02_bridge_daemon_design.md)). The UI
container bind-mounts it — but at what path inside the container?

`/run/wskop-bridge.sock`? `/var/run/wskop/<instance>.sock`?
This is a deployment convention, not a contract concern, but
worth pinning before the UI starts looking for it.

**A (operator, 2026-05-15):** **yes — pin before the UI starts
looking.** Specific path is a deployment-phase decision; the
contract doesn't constrain it. Implementation phase records the
chosen path in the `wskop_bridge_v01` install script + the UI
deployment doc together; both sides agree before either codes
against it.

### Q12. Manifest distribution to the UI

The bridge serves the manifest on connection via
`kind="manifest"` ([03 §3.2](03_wire_protocol.md)). The UI then
caches it. But the **build-time** manifest also exists as a file
at `/etc/wskop-wsi/<instance>.bridge_manifest.json`. Should the
UI consume the build-time file directly (mounted into the
container) as a back-up if the bridge is unreachable?

I would not. The bridge is the runtime source of truth; if it is
unreachable the UI shouldn't pretend to know what the data shape
is. **Confirm.**

**A (operator, 2026-05-15):** **confirmed.** The UI does not
fall back to the build-time `/etc/wskop-wsi/<instance>.bridge_manifest.json`
on bridge unreachability. Bridge is the single source of truth;
when it is unreachable the UI shows a "bridge unavailable" state
rather than serving stale shape metadata.

### Q13. Watched-flow / watched-subscriber retention semantics

When the operator marks a flow as watched, does the bit persist
across:

- (a) wskop restart — likely **no**, because per-flow state
  resets. But the subscriber-side `watched` flag on
  `subsc_w_prb_ref` persists if it is rebuilt from Redis on
  startup (the standard subscriber-cache restore path).
- (b) flow aging — **yes**, until the flow is reaped. A new
  flow with the same 5-tuple after aging gets the bit only if
  the subscriber is watched (§3.3 path).
- (c) subscriber unmark — flow-level bit persists on existing
  flows until explicit `unset_watched` walk; new flows of that
  subscriber don't get the bit. **My read: this is the right
  behaviour** — explicit per-flow marks should outlive the
  subscriber-level mark. **Confirm.**

The contract behaves correctly under all three readings of §3.2;
the operator should choose the semantics consciously.

**A (operator, 2026-05-15):**

- **(a) wskop restart: no.** Per-flow watched bits do not survive
  wskop restart; per-flow state is rebuilt fresh on the new
  process. The subscriber-side `watched` flag on
  `subsc_w_prb_ref` survives because subscriber state is restored
  from Redis at startup, so new flows of a watched subscriber
  inherit the bit at flow creation — same shape as (c) below.
- **(b) flow aging: yes.** The bit persists with the flow until
  the flow is reaped. After aging, a new flow with the same
  5-tuple gets the bit only if the subscriber is watched (the
  §3.3 path in [04](04_watched_flag_and_propagation.md)).
- **(c) subscriber unmark: subscriber-unmark behaviour as
  described** — flow-level bits on existing flows persist until
  an explicit `unset_watched` walk; new flows of that subscriber
  do not get the bit. Explicit per-flow marks outlive the
  subscriber-level mark.

Folded into [04 §8 (Unwatch)](04_watched_flag_and_propagation.md)
as an explicit retention/persistence section so the implementation
phase has the semantics in one place.

### Q14. Does the contract need a "drain" / "snapshot" operation for the UI?

The UI may want to capture a full state snapshot for an incident
report. v1's `kind="get"` returns one or many paths; the UI can
build a snapshot by enumerating the manifest and issuing one big
`get`. That works for v1.

For v2's streaming events, "snapshot" is murkier — a UI saying
"give me all watched flows now" needs a way to ask the broker for
a one-shot "emit one record per currently-watched flow" pulse.
Not in v2's scope as designed; **flag for v2.1 if the UI wants
it.**

**A (operator, 2026-05-15):** **defer to v2.1; not urgent.**
v1's enumerate-the-manifest + bulk `get` covers the polled
snapshot need. The streaming "one-shot drain" is a future
enhancement.

## Hygiene / refactor questions (lower priority)

### Q15. `WSKOP_BUILD_VERSION` macro at link

Per [discovery 08 Q13](../dpi_runtime_data_contract_discovery/08_open_questions.md#q13)
and [05 §9](05_counter_semantics_manifest.md). Trivial Makefile
change; not blocking. The contract works with the macro unset
(reads `unknown`); the UI handles that case.

### Q16. `simple_cli` revival

Per [07 §10](07_observations.md). If the operator wants a
write-back control channel for "watch flow N" gestures separate
from Redis, `simple_cli` is the place. **Not the bridge.** This
is a separate design phase.

**A (operator, 2026-05-15):** **no revival. `simple_cli` will be
removed.** The runtime contract bridge stays read-only;
write-back gestures continue to flow through the existing Redis
command channel. Removal of `simple_cli` from wskop is its own
cleanup task, out of scope here.

### Q17. The `flow_info_d` SERVICE_KEY overrun

Per [discovery 08 Q3](../dpi_runtime_data_contract_discovery/08_open_questions.md#q3).
Operator answer: hygiene-only, not blocking. The runtime contract
does not need this resolved to claim bit 38. **Flag for a
separate inspection PR** if anyone is touching `bit_operations.c`.
At HEAD the implementation is missing entirely (declarations only
in the header), which makes it even safer than the discovery
suggested.

**A (operator, 2026-05-15):** **acknowledged (informational).**
Hygiene-only; doesn't block bit-38 claim or any other part of
the contract.

### Q18. The IPDR cadence bug

Operator A2 in discovery 08: "assume the bug is fixed." The
contract sizes against the intended seconds-cadence. **No action
from this design; coordinate with whoever fixes
`check_time_interval_precise_ts`** before the implementation
phase ships.

The contract's manifest carries the `ipdr_cadence_1000x` anomaly
entry until the fix lands; on fix, the anomaly is removed and
`schema_version` bumps.

**A (operator, 2026-05-15):** **acknowledged (informational).**
Same posture as Q17 — the contract carries the anomaly entry in
its manifest until the upstream fix lands.

## Summary — what is left open

After the 2026-05-15 operator answers, all 18 questions are
resolved. Load-bearing decisions are folded into 01-06. **Open
items remaining for the implementation phase are operational
choices** that don't change the contract's shape:

- Concrete bind-mount path for the UI container side (Q11) —
  agree at deployment time.
- Specific manifest generator script (Q1 / [05 §6](05_counter_semantics_manifest.md))
  — hybrid generator code lives in `wskop_bridge_v01/scripts/`.

Nothing else is awaiting an answer to start implementation.

## Out of scope (called out explicitly)

The brief is explicit on what is out:

- **Editing `.c` / `.h` files in `wskop_v01`** — pseudocode only.
- **Redesigning `ssg_src/metrics/`** — extending it via new
  counters and (small) new role directories.
- **Packet capture / cloning** — counters only.
- **Redis mirror** — UNIX socket only.

This file does not raise questions in those zones.
