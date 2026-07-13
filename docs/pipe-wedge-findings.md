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

Hypotheses worth investigating in `src/` (the plugin, Rust/wasm):

- **Pipe-input handler backpressure/deadlock under burst.** Does the plugin
  process `PipeMessage` synchronously on the same path as render? A slow render
  or a lock held across an `.await` could stall the pipe-drain loop while events
  queue, then recover when the contended resource frees.
- **Unbounded event queue vs. bounded pipe.** If the plugin buffers events and
  the buffer/lock interacts badly with Zellij's pipe backpressure, a burst could
  wedge the CLI side until the plugin drains.
- **Multiple Claude clients, one plugin instance.** A single Zellij session can
  host several Claude panes, all piping into that session's *one* zellaude
  plugin instance. Concurrent bursts from N clients multiply the input rate.
  Reproduce by driving 2–3 busy Claude sessions in one Zellij session and
  watching for the wedge.

Suggested next steps:

- Add instrumentation to the plugin's pipe handler (enter/exit timestamps, queue
  depth) to catch the wedge in the act.
- Try making the pipe-input path non-blocking / decoupled from render.
- Consider whether the hook should also *coalesce* rapid events (e.g. debounce
  PreToolUse/PostToolUse pairs) to cut input rate at the source — complementary
  to the plugin-side fix.

## Reproduction sketch

1. In a Zellij session with the zellaude plugin loaded, run a Claude Code session
   that emits many tool calls in a burst.
2. Watch `ps -eo etime,command | grep 'zellij pipe --name zellaude'` — under the
   pre-fix hook, count climbs monotonically and processes never exit.
3. With the watchdog, each stuck process self-terminates after 5s, but the
   underlying instance can still be wedged (fresh pipes to that session hang for
   the watchdog window) until it self-heals.
