# Findings: `zellij pipe` process pileup from the Claude Code hook

Handoff notes for a follow-up agent. Documents a production incident, the fix
that shipped, and the **unresolved root cause** that the fix only contains.

## Summary

`scripts/zellaude-hook.sh` bridges Claude Code hook events to the zellaude
Zellij plugin by running `zellij pipe --name zellaude -- <payload>` on every
hook event. `zellij pipe` is a **blocking** call: the CLI process stays alive
until the plugin unblocks the pipe. When a plugin instance stops draining its
pipe input, every subsequent `zellij pipe` to that session **hangs forever**.
Because the hook fires on every Claude event (tool calls, prompts, stops,
notifications…), the hung processes accumulate without bound.

Observed in the field: **600+ orphaned `zellij pipe --name zellaude` processes**,
the oldest ~38 minutes old, enough to stall the machine's Zellij IPC.

## What already shipped (the containment)

`scripts/zellaude-hook.sh` now wraps the send in a self-contained watchdog
(commits `d35432a`, `adf27bd`, `8b12d6c`):

```sh
_ZELLAUDE_PIPE_TIMEOUT=5
zellij pipe --name "zellaude" -- "$PAYLOAD" &
_zellaude_pipe_pid=$!
( sleep "$_ZELLAUDE_PIPE_TIMEOUT"; kill -KILL "$_zellaude_pipe_pid" 2>/dev/null ) &
_zellaude_watchdog_pid=$!
wait "$_zellaude_pipe_pid" 2>/dev/null
kill "$_zellaude_watchdog_pid" 2>/dev/null
```

Design choices, all deliberate:

- **`sleep; kill -KILL`, not `timeout(1)`** — `timeout` is absent on stock macOS
  (no coreutils); the sleep/kill pair is portable.
- **SIGKILL, not SIGTERM** — a deadlocked Zellij server ignores TERM.
- **`wait` then cancel the watchdog** — the normal case (pipe returns in ms)
  cancels the pending kill, and this also avoids SIGKILLing an unrelated process
  should the OS recycle the pipe's PID within the 5s window (a busy session
  churns through many short-lived shells).
- **Async assumption** — the hook is registered `async: true` in
  `~/.claude/settings.json`, but Claude Code does **not** enforce that timeout on
  async hooks; it fires and forgets, so a hung pipe is never reaped by the
  harness. The hook must bound itself.

This caps the damage: a wedged instance can now leak at most one short-lived
process per event instead of a permanent one. It does **not** un-wedge a stuck
instance.

## The unresolved root cause (for the next agent)

The watchdog treats the symptom. The real bug is upstream of it:
**why does a plugin instance stop draining its pipe?**

Diagnostic evidence gathered during the incident:

1. **It is per-instance, not global.** A fresh `zellij pipe --name zellaude` to a
   *zero-backlog* live session (plugin loaded) returns in **0s**. The same call
   to the wedged session times out at the full watchdog window. So healthy
   instances drain instantly; one specific instance had stopped consuming.
2. **It is transient / self-healing.** Left alone, the wedged instance
   eventually recovered on its own, at which point all of its blocked pipes
   drained/exited at once (the process count collapsed from 600+ back to the
   baseline without any manual kill).
3. **Load-correlated.** The overwhelming majority of the stuck pipes came from a
   single, very busy Claude session emitting events faster than the plugin
   consumed them — consistent with the plugin's pipe-input handler falling
   behind or briefly deadlocking under burst load, then catching up.

### What the code confirms (architecture)

Reading the plugin (`src/`) against the vendored `zellij-tile 0.43.1` crate
(`~/.cargo/registry/.../zellij-tile-0.43.1`) settles the mechanism:

- **The plugin is single-threaded and strictly serialized.** `register_plugin!`
  stores plugin state in a `thread_local! RefCell` (crate `lib.rs`); `load`,
  `update`, `pipe`, and `render` all `borrow_mut()` that same cell. Zellij's
  server calls them one at a time — they cannot overlap. So a hung pipe means
  the plugin's single execution context isn't *getting to* the next `pipe()`
  call: it is blocked or backlogged behind other work.
- **The CLI pipe unblocks implicitly when `pipe()` returns.** The plugin never
  calls `block_cli_pipe_input`, yet healthy pipes drain in 0s. The blocking
  `zellij pipe` CLI process therefore stays alive exactly until the server has
  run this plugin's `pipe()` for its message. (Confirmed observation #1 above.)
- **Returning `true` forces a render.** Per the `ZellijPlugin` trait docs,
  returning `true` from `update`/`pipe` makes Zellij call `render()`.
  `render_status_bar` (`src/render.rs`) ends in `print!` + `stdout().flush()` —
  a WASM→host boundary call on every render.
- **Not the inter-instance sync path.** `pipe()` also serves
  `zellaude:request`/`:sync`/`:settings` for multi-instance state sharing, and
  those call `pipe_message_to_plugin`. This is *not* the culprit: the sync
  handlers are terminal (they mutate state and return — never re-broadcast), and
  broadcasts only fire from startup (`PermissionRequestResult`) and a manual
  settings toggle (`save_config`) — **never from a hook event**. Under a
  hook-event burst, zero broadcasts happen. `pipe_message_to_plugin` is also
  fire-and-forget (a queued host command), so it can't block the loop anyway.

### The in-repo trigger (fixable here) vs. the server-side block (upstream)

Two distinct things, and it matters not to conflate them:

- **The block itself is server-side and self-heals** — consistent with evidence
  #2 (blocked, then *all* pending pipes drain at once, not slowly catching up).
  A plain O(tabs×sessions) render is only single-digit milliseconds of cheap
  string work; that produces *lag*, not a 38-minute / 600-process wedge. "Drains
  all at once" is the signature of the loop being **blocked and then released**,
  not slow-but-progressing. The most plausible mechanism is host-boundary
  backpressure — under a high render rate the `stdout().flush()` in `render`
  fills a pipe the server drains slowly, stalling the plugin thread so it can't
  service the next `pipe()` — **but this is a hypothesis; it lives in the Zellij
  server and is not confirmable from this repo.** Do not go spelunking the
  server source to prove it: it does not change the fix below.
- **The plugin *does* own the trigger — it over-requests rendering.** `pipe()`
  returns `true` **unconditionally** for the `"zellaude"` branch
  (`src/main.rs`), so *every* hook event forces a full render + `stdout` flush —
  even a `Notification` event, which `handle_hook_event` explicitly treats as
  "refresh the timestamp, keep current activity" (nothing visible changes).
  `update()` is similarly liberal. Under a burst this multiplies the render/flush
  rate far beyond what the visible state actually needs.

### Recommended fix (correct regardless of the exact server mechanism)

Cut the render rate at its source in the plugin:

- **Return `true` only when visible state changed.** Thread a
  `visible_changed: bool` out of `handle_hook_event` and return it from `pipe()`
  instead of the hard-coded `true`. The `Notification` no-op path (and any event
  that doesn't alter what's drawn) then returns `false` and skips the render +
  flush entirely. Audit `update()` the same way (e.g. `PaneUpdate` that produces
  an identical pane map need not re-render).
- This reduces render/`stdout`-flush frequency under burst whether the wedge is
  queue saturation, stdout backpressure, or a server-side lock — so it's the
  right move without confirming the upstream mechanism.

Complementary, optional:

- **Coalesce/debounce at the hook source** (`scripts/zellaude-hook.sh`) — e.g.
  collapse rapid `PreToolUse`/`PostToolUse` pairs — to cut input rate before it
  reaches the plugin.
- **Instrument the render path** (a cheap enter/exit counter written to a debug
  file) if you want to *measure* the render rate before/after the fix, rather
  than trying to catch the server-side block directly.

## Reproduction sketch

1. In a Zellij session with the zellaude plugin loaded, run a Claude Code session
   that emits many tool calls in a burst.
2. Watch `ps -eo etime,command | grep 'zellij pipe --name zellaude'` — under the
   pre-fix hook, count climbs monotonically and processes never exit.
3. With the watchdog, each stuck process self-terminates after 5s, but the
   underlying instance can still be wedged (fresh pipes to that session hang for
   the watchdog window) until it self-heals.
