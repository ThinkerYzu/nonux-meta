# AI-RULES.md — Rubric for Registry Rules R1–R8

**Status:** Authoritative rubric for AI-driven authoring and review of nonux components.
**Last Updated:** 2026-04-21 (Session 13; reorganised from DESIGN.md §AI Verification into a standalone rubric after the realisation that most rules are AI-verified, not machine-checkable.)

---

## Purpose

nonux is an AI-driven kernel: components are written and reviewed by AI agents, not hand-auditted by humans. The registry rules R1–R8 in [DESIGN.md §AI Verification](DESIGN.md#ai-verification) are the contract every component must satisfy. Two layers enforce them:

1. **Machine checks** — `tools/verify-registry.py` runs before `make` and `make test`. Today it covers **R2 and R4** (regex-decidable rules). Violations fail the build.
2. **AI verification** — The remaining rules (**R1, R3, R5, R6, R7, R8**) require reasoning beyond regex: slot-pointer dataflow, interface-schema awareness, held-refs tracking, borrowed-cap lifetime, generator identity, and ISR/kthread call-graph. This file is what the authoring AI reads *while writing* a component, and what the reviewing AI walks *while reviewing* one.

"AI-verified" is not a synonym for "deferred." A rule is AI-verified when its enforcement mechanism is an agent reading code against the rubric below, not a static tool. The agent is expected to be as deterministic as a tool: walk the rubric, apply the checks, report findings, never "judge" whether a violation is acceptable.

## How to use this file

**As an authoring agent** — Before emitting a new component's source, open this file and scan each rule's *Violation shapes* list. Your output must not match any of them.

**As a reviewing agent** — For every component in the changeset, walk R1 → R8 in order. For each AI-verified rule, work through the *AI verification procedure* step by step and record `pass` or `fail: <one-line reason referencing file:line>`. Emit a rubric report before approving the component.

**When in doubt** — Stop and ask the human operator. Rules R1, R5, R6 especially can present edge cases (framework helpers that return slot pointers, cross-component factories, opaque-handle idioms). The rubric covers the common cases; when you see a shape it doesn't describe, surface it rather than guess.

---

## R1 — No fabricated slot pointers

**Rule.** Every `struct nx_slot *` value inside a component must originate from a framework-approved source. The component may never synthesise one from an integer, another pointer type, or a non-framework function return.

**Why.** Slot pointers are the kernel's hot-swap hinge. If a component reads `slot->active` through a pointer the registry doesn't know about, a swap can free the impl while the component still holds a path to the freed memory. Confining slot-pointer provenance to registry-minted sources makes the swap provably safe.

**Machine check.** None today. A regex pass flags `(struct nx_slot *)` casts with high false-positive rate (framework internals legitimately cast), so it isn't run. R2 catches one adjacent shape — struct-field declarations — but not assignments.

**Approved sources (exhaustive).**
1. Dependency injection: the framework fills `self->deps.<name>` at `init()` time from the manifest.
2. Registry lookup: `nx_slot_lookup(name)` (returns a `struct nx_slot *` owned by the registry).
3. Cap retention: `nx_slot_ref_retain(self, cap)` on a cap delivered in an inbound message's `msg->caps[]`.

**AI verification procedure.**
1. Grep the component's `.c` files for every assignment whose LHS is `struct nx_slot *` (struct fields or local variables).
2. For each RHS, trace it to one of the three approved sources above. A function-return RHS is approved only if that function is `nx_slot_lookup`, `nx_slot_ref_retain`, or a framework helper documented as returning a minted slot pointer.
3. Any RHS that is a cast, an integer arithmetic result, a `void*` of uncertain origin, or a return from a non-framework function → **fail**.

**Compliant example.**
```c
int my_init(struct nx_component *self) {
    struct my_state *s = self->state;
    s->timer = self->deps.timer;                    /* dep injection */
    s->stats = nx_slot_lookup("stats_counter");     /* registry */
    return 0;
}

int my_on_msg(struct nx_component *self, struct nx_message *m) {
    struct my_state *s = self->state;
    s->log = nx_slot_ref_retain(self, &m->caps[0]); /* retained cap */
    ...
}
```

**Violation shapes.**
```c
s->timer = (struct nx_slot *)0xffff0000;            /* fabricated */
s->timer = (struct nx_slot *)some_void_pointer;     /* laundered */
s->timer = foo_get_slot();                          /* non-framework return */
s->timer = &global_slot_table[3];                   /* hand-indexed */
```

---

## R2 — Slot fields match manifest

**Rule.** Every `struct nx_slot *` field in a component's state struct corresponds to a `requires` or `optional` entry in its `manifest.json`. Manifest entries without fields also fail.

**Why.** The manifest is the author-facing declaration of what the component depends on. Silent drift between code and manifest breaks dependency injection, the change log, swap notifications, and the R2 build gate itself.

**Machine check.** **Enforced by `verify-registry.py`.** Regex matches `struct nx_slot *<ident>;` at indent ≥ 1 (column-0 hits are treated as file-scope forward declarations and skipped); both drift directions flagged. AI review is still expected to catch edge cases the regex doesn't see.

**AI verification procedure.** Usually unnecessary — the machine check is authoritative. Do verify manually when:
- Multi-word type aliases hide a slot pointer (`typedef struct nx_slot *my_slot_t; my_slot_t foo;`) — regex won't match.
- A slot pointer is embedded in a nested struct (`struct { struct nx_slot *x; } inner;`) — regex matches but the author's mental model of "state struct" may not.
- A `#ifdef` hides a field from one build — regex sees only the preprocessed path the scanner chose.

For any of these, walk the state struct's full type tree and cross-check the manifest manually.

---

## R3 — Cap-only slot transfer

**Rule.** Slot references travel between components exclusively through `ipc_message.caps[]`. No interface message schema declares a slot ref (or a raw `slot_id`, slot pointer, or any aliased form) as a payload field.

**Why.** The router scans `caps[]` generically to validate ownership and apply borrow/transfer semantics. A slot ref buried in payload bytes is invisible to the router — the receiver can stash it without the retain discipline the `caps[]` path enforces.

**Machine check.** None today. When interface schemas land (slice 3.8+), a JSON-schema validator on `manifest.json`'s `interfaces[].messages[].payload` will reject `slot_ref`-shaped fields. Until then this rule is AI-only.

**AI verification procedure.**
1. For every interface the component declares or consumes, inspect every message's payload struct.
2. **Fail** if any payload field has a type that is, resolves to, or aliases `struct nx_slot *`, `nx_slot_ref_t`, `nx_cap_t`, a `uint32_t` named `slot_id`/`slot_handle`/equivalent, or a `void *` documented to carry a slot.
3. **Fail** if handler code reads a slot-typed value out of `msg->payload` rather than `msg->caps[]`.
4. Check send sites symmetrically: a `nx_ipc_send()` whose `msg->payload` contains a slot reference fails.

**Compliant example (manifest).**
```json
{
  "interfaces": [{
    "name": "vfs_ops",
    "messages": [{
      "name": "open",
      "payload": {"path": "string", "flags": "u32"},
      "caps": [{"name": "vfs", "type": "slot_ref", "mode": "borrow"}]
    }]
  }]
}
```

**Violation shapes.**
```c
/* Payload smuggling */
struct open_msg { const char *path; struct nx_slot *vfs; };

/* ID smuggling */
struct open_msg { const char *path; uint32_t vfs_slot_id; };

/* Cast-through-void */
struct open_msg { const char *path; void *target; };  /* docs say "VFS slot" */
```

---

## R4 — Retain/release pairing

**Rule.** Every `nx_slot_ref_retain(` call in a component has a matching `nx_slot_ref_release(` call, and every release is reachable from `disable()` or `destroy()` (or from an unwind on a failed `init()`/`enable()`).

**Why.** Unreleased retains keep a slot ref alive past component destruction — the registry will refuse the destroy, and the component leaks. Releases without matching retains corrupt the refcount and destabilise hot-swap.

**Machine check.** **Partially enforced by `verify-registry.py`.** Per-component call counts of `nx_slot_ref_retain(` and `nx_slot_ref_release(` must match; mismatches emit one finding per excess call. The count match does **not** prove reachability from `disable`/`destroy` — that's the AI verification procedure below, pending a pycparser-backed control-flow extension.

**AI verification procedure.**
1. Locate every `nx_slot_ref_retain(` call. For each, follow the stored slot ref: does a `nx_slot_ref_release` call consume it on every path out of the component?
2. Paths that must end in a release (on success):
   - `enable() → ... → disable()` path.
   - `init() → ... → destroy()` path.
   - Any error path in `init()` or `enable()` that unwinds a successful retain before returning a non-zero code.
3. **Fail** if a retain can persist past `destroy()` on any path. **Fail** if a release exists on a path that did not retain (double-free risk).
4. Don't trust "it's matched in count" — the machine check already asserted that. Your job is the reachability half.

**Compliant example.**
```c
int my_on_msg(struct nx_component *self, struct nx_message *m) {
    struct my_state *s = self->state;
    struct nx_slot *stored = nx_slot_ref_retain(self, &m->caps[0]);
    if (!stored) return -ENOMEM;
    s->stored_log = stored;
    return 0;
}

int my_disable(struct nx_component *self) {
    struct my_state *s = self->state;
    if (s->stored_log) {
        nx_slot_ref_release(self, s->stored_log);
        s->stored_log = NULL;
    }
    return 0;
}
```

**Violation shapes.**
- Retain in `enable()`, release only on the error path of `enable()` — matched count, unreachable on success.
- Retain in a message handler, release only on another handler's code path that never executes for the retained cap.
- Release in `init()` on a `NULL` without a preceding retain — count may still balance if other releases match other retains, but the semantics are wrong.

---

## R5 — Sender owns what it passes

**Rule.** When a component calls `nx_ipc_send()` with caps in the outbound message, every cap must reference a slot the sender **currently holds** — either via dependency injection (`self->deps.*`), via a `nx_slot_lookup` result the component owns, or via a prior `nx_slot_ref_retain`.

**Why.** The kernel's cap model treats the sender as having to *own* a slot to transfer or lend it. A sender that forwards a slot it only borrowed from an earlier inbound message (without retaining it) smuggles a reference the framework doesn't account for — the R3 payload loophole closed by the caps array would re-open via a short-lived borrow chain.

**Machine check.** None. Enforcement requires symbolic tracking of the sender's held-refs set at every send site.

**AI verification procedure.**
1. For each `nx_ipc_send()` (or equivalent) call, enumerate the caps it attaches.
2. For each cap, answer: *which slot does it reference, and how does the sender hold it at this moment?*
   - If the slot is `self->deps.<name>` → **ok** (dependency).
   - If the slot was stored in `self->state` via `nx_slot_lookup()` earlier → **ok** (registry-minted and owned).
   - If the slot was stored in `self->state` via `nx_slot_ref_retain()` on an inbound cap → **ok** (retained).
   - If the slot was read from `msg->caps[k]` on an *inbound* message and *not yet retained* → **fail** (borrowed; forwarding requires retaining first, which also makes the lifetime owner-visible).
   - If the slot came from any other source → trace until it matches one of the above; if it doesn't, **fail**.
3. A cap with `mode = transfer` requires the sender to *give up* its hold (release after send); verify the release happens on every post-send path.

**Compliant example.**
```c
int my_forward(struct nx_component *self, struct nx_message *in) {
    struct my_state *s = self->state;

    /* Retain the borrowed cap so we own it before forwarding. */
    struct nx_slot *target = nx_slot_ref_retain(self, &in->caps[0]);
    if (!target) return -ENOMEM;

    struct nx_message out = { /* ... */ };
    out.caps[0] = (struct nx_cap){.slot = target, .mode = NX_CAP_BORROW};
    int rc = nx_ipc_send(self->deps.peer, &out);

    nx_slot_ref_release(self, target);
    return rc;
}
```

**Violation shape.**
```c
int my_forward(struct nx_component *self, struct nx_message *in) {
    struct nx_message out = { /* ... */ };
    out.caps[0] = in->caps[0];   /* borrowed, not retained — not owned */
    return nx_ipc_send(self->deps.peer, &out);
}
```

---

## R6 — Handler does not stash borrowed caps

**Rule.** Inside a message handler, a borrowed cap's slot (a cap the handler did not retain) may not be assigned into component-visible state (`self->state`, component-owned allocations, or the `caps[]` of a message enqueued for later delivery).

**Why.** Borrowed caps are valid only for the duration of the handler that received them. The sender may release its hold the instant the handler returns; the slot can be reclaimed on the next swap. Stashing a borrowed ref gives the component a dangling pointer on the next message.

**Machine check.** None. Enforcement requires dataflow from the "borrowed cap" marker on inbound caps to every assignment sink.

**AI verification procedure.**
1. For each message handler, classify every cap the handler touches:
   - *Borrowed* — arrived as `msg->caps[k]` and the handler did not call `nx_slot_ref_retain` on it. This is the default.
   - *Retained* — arrived as `msg->caps[k]` and the handler called `nx_slot_ref_retain` on it (the retained return value, not the original, is owned).
2. For every borrowed cap, walk every use. If the slot ref (or anything derived from it, including a `slot_resolve()` result captured as a pointer) is assigned into:
   - any field of `self` or `self->state`,
   - any component-owned container (lists, maps, caches),
   - the `caps[]` or `payload` of a later-delivered message,
   - a global variable,
   — **fail**.
3. Pure read-and-forward-within-handler use is **ok**: calling `slot_resolve()` on a borrowed slot *during the handler's execution* and invoking an op through the result is allowed, because the preempt-disable + bounded-handler discipline keeps the result alive for the call.

**Violation shapes.**
```c
int my_on_msg(struct nx_component *self, struct nx_message *m) {
    struct my_state *s = self->state;
    s->last_sender = m->caps[0].slot;           /* stash — fail */
    cache_insert(s->cache, m->caps[0].slot);    /* stash — fail */

    struct nx_message later = { .caps[0] = m->caps[0] };   /* forwarded — fail */
    nx_ipc_enqueue_delayed(self->deps.timer, &later, 1000);

    return 0;
}
```

---

## R7 — Manifest is source of truth

**Rule.** The generated `gen/<component>_deps.h` is byte-identical to a fresh `gen-config.py` run from the current manifest. Authors never hand-edit generated files; the manifest is always the source.

**Why.** The framework's dep-injection machinery (R7's ultimate customer) relies on the generated header encoding the manifest's `requires`/`optional` declarations exactly. A hand-edit that drifts the generated table from the manifest silently breaks the `offsetof()` calculation `resolve_deps()` depends on — the framework injects wrong pointers, with no build-time signal.

**Machine check.** The original DESIGN says "regenerate and diff against the checked-in copy." Today `.gitignore` excludes `gen/`, so there's no on-disk artefact to diff against. The *intent* (generator determinism + no hand-edits) is covered by `tools/tests/test_gen_config.py::*determinism*`, which asserts the same manifest produces byte-identical output across runs. R7 flips to machine-checkable when the workflow commits `gen/<name>_deps.h` alongside the source.

**AI verification procedure.**
1. Confirm `gen/` is still in `.gitignore`. If yes, R7 rests on generator determinism — no author-visible step is needed beyond "don't hand-edit generated files."
2. Grep the changeset for any modifications to files under `gen/`. **Fail** on any hit (authors shouldn't be editing generator output).
3. If a manifest changed but the component's `gen/<name>_deps.h` is stale (older `mtime` than the manifest, or the build hasn't re-run), trigger `gen-config.py` before verification.
4. If the workflow ever commits generated headers: regenerate and byte-diff against the committed copy. Any diff = **fail**. Promote R7 to a machine check in `verify-registry.py` at that point.

---

## R8 — Slot-resolve locality

**Rule.** Reading `slot->active` (directly, via `nx_slot_resolve()`, or via `slot->active->ops->...`) is permitted only on a framework-owned dispatcher thread: a per-CPU dispatcher, or a `dedicated` component's private dispatcher thread. ISRs and arbitrary kernel threads may only **enqueue** messages; they may never dereference slots.

**Why.** Hot-swap's pause protocol drains dispatcher queues and waits for in-flight handlers to finish, then destroys the old impl. If an arbitrary thread had read `slot->active` and been descheduled before calling through the pointer, the drain step can't see it — the thread resumes holding a pointer to freed memory. Confining slot-resolve to dispatcher threads makes the drain step *complete*.

**Hook handlers are in scope.** `NX_HOOK_IPC_SEND` / `_IPC_RECV` / `_COMPONENT_{ENABLE,DISABLE,PAUSE,RESUME}` / `_SLOT_SWAPPED` (slice 3.8) run on the dispatcher — same slot-resolve-locality domain as message handlers. A hook that stashes a resolved `impl *` into a worker closure violates this rule exactly as a regular handler would. The procedure below applies unchanged.

**Machine check.** None today; needs a call-graph built from explicitly-tagged ISR/kthread entry points. An ISR-tagging convention is on the slice 3.9 roadmap. The test harness asserts dispatcher-context at every runtime slot-resolve — that backstop catches violations but only when a test exercises the code path.

**AI verification procedure.**
1. Enumerate every function in the component that can run outside a dispatcher:
   - ISR handlers registered via `nx_irq_register` (or equivalent).
   - Kthread entries passed to `nx_kthread_spawn` / `nx_component_spawn_worker`.
   - Softirq / tasklet callbacks.
2. Walk each function's call graph (intra-component first, then through any framework calls it makes).
3. **Fail** if any reachable path reads `slot->active`, calls `nx_slot_resolve()`, or dereferences a stashed resolved `impl *`.
4. Cross-thread subtlety — the one the reviewer agent is the authority on: a dispatcher handler that passes a resolved `impl *` into a worker closure (via arguments, captured state, or a shared struct) violates R8 even though the resolve itself was on the right thread. The worker then holds a pre-resolved pointer across a void-boundary the framework can't track. **Fail** on any such handoff; workers must carry slot *references* and re-enqueue messages, never pre-resolved pointers.
5. `dedicated` components: their private thread *is* a dispatcher; slot-resolve inside that thread is fine. Confirm the component declared `concurrency: dedicated` in its manifest if its workers dereference slots.

**Compliant example.**
```c
/* ISR: enqueue only, no resolve. */
void my_irq(void *ctx) {
    struct my_state *s = ctx;
    nx_ipc_enqueue_from_irq(s->deps.handler, &(struct nx_message){...});
}

/* Dispatcher handler: resolve freely. */
int my_on_tick(struct nx_component *self, struct nx_message *m) {
    struct my_state *s = self->state;
    struct nx_impl *timer = nx_slot_resolve(s->deps.timer);
    return timer->ops->tick(timer);
}
```

**Violation shapes.**
```c
/* ISR deref — fail */
void my_irq(void *ctx) {
    struct my_state *s = ctx;
    struct nx_impl *log = nx_slot_resolve(s->deps.log);
    log->ops->record(log, "irq fired");
}

/* Cross-thread pre-resolve handoff — fail */
int my_on_msg(struct nx_component *self, struct nx_message *m) {
    struct my_state *s = self->state;
    struct nx_impl *vfs = nx_slot_resolve(s->deps.vfs);    /* resolved here */
    nx_component_spawn_worker(self, worker_fn, vfs);       /* handed off */
    return 0;
}
```

---

## Review report shape

When a reviewing agent walks this rubric over a changeset, it emits a report in this shape:

```
component: sched_rr
  R1 pass
  R2 pass (verify-registry.py confirmed)
  R3 pass
  R4 pass (verify-registry.py confirmed; disable()-reachability checked manually)
  R5 pass
  R6 fail: components/sched_rr/irq.c:47 stashes m->caps[0].slot into s->last_caller
  R7 pass (gen/ gitignored; generator determinism covered by test_gen_config.py)
  R8 pass
```

A component with any `fail` line is not complete. The authoring agent addresses each fail and resubmits.

---

## Evolution

When the framework grows a new registry rule, add a section here with the same shape: Rule / Why / Machine check / AI verification procedure / Compliant example / Violation shapes. Update the R1–R8 table in [DESIGN.md §AI Verification](DESIGN.md#ai-verification) to keep the one-line summaries in sync with the rubric.

When a machine check becomes feasible (pycparser dataflow for R1, interface schemas for R3, committed `gen/` for R7, ISR/kthread tagging for R8), promote that rule's "Machine check" section in place and shrink the AI procedure to the fallback cases the tool can't see. Never remove a rule's AI procedure entirely — the rubric serves both as enforcement and as design documentation for human readers catching up on the framework.
