# Session 8: Registry events, change log, snapshot, JSON (slice 3.2)

**Date:** 2026-04-21
**Phase:** 3 — Component framework
**Branch:** master

---

## Goals

- Layer the rest of slice 3.2 onto the registry skeleton: change events, bounded change log, refcounted snapshot, JSON serialisation. These are the pieces the hooking framework, the recomposer, the test harness's invariant checker, and the future `/kstat/graph` dump all consume.

## What Was Done

### Change events

- 9 event types (`NX_EV_SLOT_CREATED / DESTROYED / SWAPPED`, `NX_EV_COMPONENT_REGISTERED / UNREGISTERED / STATE`, `NX_EV_CONNECTION_ADDED / REMOVED / RETUNED`) covering every mutation.
- `struct nx_graph_event` carries `type`, `generation`, `timestamp_ns`, and a tagged union of payloads. All payload strings are `const char *` referencing caller-owned static storage on the original `nx_slot` / `nx_component` (manifest_id / instance_id / slot name) — no pointers to caller structs themselves, eliminating the UAF risk later when events sit in the change log past component lifetime.
- `nx_graph_subscribe(cb, ctx)` / `nx_graph_unsubscribe(cb, ctx)` — fixed 16-slot subscriber array, dedup'd by `(cb, ctx)` pair (`NX_EEXIST` on duplicate, `NX_ENOMEM` on capacity exceeded). `unsubscribe` of an unknown pair is a no-op.
- Internal `emit_event()` is called from each mutation point AFTER `bump_gen()`. It stamps the event with the current generation and timestamp, appends to the change log, then iterates subscribers synchronously. Per DESIGN.md, callbacks must be short and non-blocking — we don't enforce that, just document it.

### Change log (registrator)

- 256-entry ring buffer (`NX_CHANGE_LOG_CAPACITY`).
- `_Atomic uint64_t g_log_total` is the head + global event counter; relaxed-order `atomic_fetch_add` produces the slot index. C11 atomics from day 1 per the project preemption-atomics rule (memory: "Preemption requires atomics") — cheap on x86, correct on ARM64 with multiple dispatcher threads later.
- Public API: `nx_change_log_total()` (events ever appended), `nx_change_log_size()` (currently retained = `min(total, capacity)`), `nx_change_log_read(since_gen, out, max)` (filters by generation, oldest-first), `nx_change_log_reset()` (test-only).
- On overflow, oldest entries are silently overwritten — `total - held` gives the logical index of the oldest still-readable event.

### Snapshot

- Opaque `struct nx_graph_snapshot`; refcounted (`_take` returns refcount=1, `_retain` bumps, `_put` releases on zero).
- `_take()` walks the live registry under no concurrency for now (host-only, single-threaded), copies entity fields into private arrays — strings remain pointers to caller storage, but the array itself is independent of the live lists. Subsequent registry mutation does not affect a held snapshot.
- Per-entity views: `struct nx_snapshot_slot` / `_component` / `_connection` are deliberately separate from the live `struct nx_slot` etc. — they're frozen copies, not pointers into the live graph. This is what makes mutation-stability cheap.
- Accessors: `_generation`, `_timestamp_ns`, `_slot_count` / `_component_count` / `_connection_count`, indexed `_slot(s, i)` / `_component(s, i)` / `_connection(s, i)`.
- DESIGN.md mentioned reference counting but no public retain — added `_retain()` because tests (and real code) need a way to bump explicitly when a snapshot is shared between two consumers.

### JSON serialisation

- `nx_graph_snapshot_to_json(s, buf, buflen)` and `nx_change_log_to_json(buf, buflen)`. Both NUL-terminate, return bytes-written or `NX_ENOMEM` on truncation (with the buffer still NUL-terminated and containing as much as fit).
- Internal `struct json_buf` appender tracks `pos` and `truncated` flag; `jb_putc` / `jb_puts` / `jb_puts_escaped` / `jb_putu64` are the only emit primitives.
- Escaping: `\"`, `\\`, `\n`, `\r`, `\t`, `\u00xx` for other controls. Other inputs trusted (registry doesn't accept arbitrary user strings — the gen-config to be written in slice 3.5 will validate identifiers upstream).
- Six enum-to-string helpers (`mutability_str`, `concurrency_str`, `mode_str`, `policy_str`, `lifecycle_str`, `event_type_str`) keep the JSON stable and machine-parseable.
- JSON shape:
  ```
  snapshot: {generation, timestamp_ns, slots:[{name,iface,mutability,concurrency,active:{manifest,instance}|null}], components:[{manifest,instance,state}], connections:[{from,to,mode,stateful,policy,installed_gen}]}
  log:      {total, held, events:[{type, generation, timestamp_ns, ...payload}]}
  ```

### Tests added (26 new, total now 62 host + 6 kernel = 68)

- **Events:** `subscribe_then_register_slot_fires_slot_created_event`, `subscribe_observes_full_event_taxonomy` (one big script that emits one of every event type and verifies the union payload each time), `unsubscribe_stops_delivery`, `subscribe_duplicate_returns_eexist`, `subscribe_distinct_ctx_is_not_a_duplicate`, `subscribe_null_callback_is_einval`, `unsubscribe_unknown_pair_is_a_noop`.
- **Change log:** `change_log_starts_empty_after_reset`, `change_log_records_each_mutation`, `change_log_read_filters_by_since_gen`, `change_log_overflow_keeps_most_recent` (300 register/unregister cycles → 600 events → ring holds last 256, oldest generation = 345), `change_log_read_max_caps_output`, `change_log_read_with_null_or_zero_returns_zero`, `graph_reset_clears_subscribers_and_log`.
- **Snapshot:** `snapshot_of_empty_registry_is_empty`, `snapshot_captures_current_state`, `snapshot_is_stable_against_subsequent_mutation`, `snapshot_refcount_keeps_alive_until_last_put`, `snapshot_put_null_is_a_noop`, `snapshot_index_out_of_range_returns_null`.
- **JSON:** `snapshot_to_json_empty_shape`, `snapshot_to_json_full_shape`, `snapshot_to_json_truncated_returns_enomem`, `snapshot_to_json_escapes_special_chars`, `change_log_to_json_emits_each_event`, `change_log_to_json_handles_null_optional_strings` (boot edge `from_slot=NULL` → `"from":null`).

## Key Findings

- **Event count off-by-one in the taxonomy test.** First run of `subscribe_observes_full_event_taxonomy` failed asserting 11 events when 12 fired. I'd undercounted in the comment — `nx_slot_unregister(&vfs)` was missing. Fixed the assertion (and added a comment enumerating the 12) rather than the code.
- **`stdio.h` needed for `snprintf`.** GCC 15 with `-Werror` rejects implicit declaration. Added the include alongside `<stdatomic.h>`, `<time.h>`, `<stdlib.h>`, `<string.h>`. Host build uses `clock_gettime(CLOCK_MONOTONIC)` for timestamps; kernel build will swap `now_ns()` for a tick reader (single point to swap, declared as `static`).

## Decisions Made

- **Strings in events, not struct pointers.** DESIGN.md sketches use `struct component *comp` in the event union. I went with `manifest_id` / `instance_id` `const char *` instead. Reason: the change log holds events past their mutation point; if it stores pointers to caller structs, those structs may be gone by the time someone reads the log — UAF in the dump path. Strings are pointers to (in production) gen-config-emitted literals that live for the kernel's lifetime; UAF is impossible by construction.
- **`emit_event` writes to the log first, then fans out.** Built-in sink, not a regular subscriber. Saves one indirection and means the log is always present even before any test subscribes.
- **`_retain()` added to the public API.** DESIGN.md's snapshot sketch implies refcounting but only exposes `_take` / `_put`. The opaque struct prevents callers from bumping refcount themselves; explicit `_retain()` is the smallest API addition that lets shared ownership work without breaking opacity.
- **C11 `_Atomic` for the change log head, even on host single-threaded.** Per the project preemption-atomics rule, shared counters use atomics from day 1, not deferred to "SMP future". Host runs single-threaded today; the same code is correct when the kernel adds per-CPU dispatchers.
- **JSON escape only the things JSON requires.** No HTML / URL escaping, no Unicode normalisation. The strings are validated identifiers in production; the escape exists to handle test inputs and defensive cases, not arbitrary user data.

## Status at End of Session

- `make test` → **68/68 (62 host + 6 kernel), 0 leaks, 0 errors, exit 0**.
- `framework/registry.{h,c}` is now ~750 LOC. Public API surface: 22 new functions / 4 new types / 9 event types beyond slice 3.1's baseline.
- Production `kernel.bin` still unchanged this session — registry is host-side only.
- Next-slice-ready: lifecycle state machine (3.3) can plug into `nx_component_state_set` + emit `NX_EV_COMPONENT_STATE` (already wired); the recomposer (3.4+) can consume snapshots and the change log.

## Next Steps

- **Slice 3.3 — Lifecycle state machine** (`framework/component.c`): formalise the init / enable / pause / resume / disable / destroy transitions, enforce legal transition matrix, hook each transition to `nx_component_state_set` (which already emits `COMPONENT_STATE`).

---

**Files Changed:**
- `sources/nonux/framework/registry.h` — added: change-event taxonomy, subscribe/unsubscribe, change log API, snapshot opaque type + accessors + `_retain`, JSON serialisation (`nx_graph_snapshot_to_json`, `nx_change_log_to_json`).
- `sources/nonux/framework/registry.c` — added: subscriber array + dispatch, ring-buffer change log with C11 atomics, snapshot take/retain/put + entity copies, JSON appender + enum-to-string helpers; wired `emit_event()` into every mutation point; `nx_graph_reset()` now clears subscribers and the log.
- `sources/nonux/test/host/registry_test.c` — 18 new tests covering the new surface.
