# STATES-OF/wmux-observation-layer

Models layers 2 and 3 (per `states-of`'s README) for the specific
observation channel used throughout this session: wmux's MCP tools
(`terminal_read`, `pane_list`'s `agentStatus`) as a window onto some
other process's real state (most often a Claude Code session, modeled
separately in `claude-code-session-lifecycle`).

**This repo is not about what the observed agent is doing. It's about
how much to trust what wmux says the observed agent is doing**, and what
observation protocol closes that gap when it matters (e.g. before acting
on what looks like a stuck state).

## States of the channel itself

| State | Meaning |
|---|---|
| `LIVE_SYNCED` | Rendered buffer (layer 2) matches real PTY output with no meaningful lag |
| `BATCH_LAGGED` | wmux's `terminalOutputScheduler` has received bytes but not yet flushed them into xterm's parsed state (documented: up to ~1 rAF frame, longer under scheduler pressure) |
| `STALE_FRAME` | The *painted* screen (not just the parsed buffer model) still shows content from before some transition — confirmed real this session: switching Claude Code's `/tui` mode reveals the primary screen buffer, unpainted since before alt-screen was entered, for roughly 1–1.5s before the classic renderer catches up |
| `CONTEXT_EVICTED_BLANK` | wmux issue class #197/#198: WebGL context churn (split/remount) can exceed Chromium's live-context ceiling, evicting a *live* pane's GL context and showing solid black for ~3s recovery grace. Distinguishable from `STALE_FRAME` in principle (nothing painted at all vs. old content painted) but not always distinguishable by a human glance — confirmed directly this session: a report of "completely black, not just hard to read" was initially mischaracterized by the observing agent as the modal-visibility case before being corrected. |
| `METADATA_DISAGREES` | wmux's own derived `agentStatus` field (idle/running/waiting) disagrees with what the pane's actual content shows — documented in wmux issue #935 as a real, independent bug class from the rendering ones above |

## The observation function (how this connects to layer-1 models)

A `terminal_read` result is evidence for a `claude-code-session-lifecycle`
state, not proof of it, with confidence depending on which channel-state
above is active at read time:

- `LIVE_SYNCED` read → treat the text as a direct, trustworthy report
- `BATCH_LAGGED` / `STALE_FRAME` → a single read is unreliable; re-read
  after >1.5s and require the SAME content before treating it as current
- `CONTEXT_EVICTED_BLANK` → a single all-black read means nothing about
  layer-1 state at all; wait out the recovery grace and re-read
- `METADATA_DISAGREES` → never trust `agentStatus` alone; corroborate
  against an actual `terminal_read` of content

**Protocol used successfully this session, worth keeping as the default:**
when a layer-2 read looks anomalous (stuck, blank, contradictory), take a
*second* independent read separated by real wall-clock time (not just a
second tool call in the same turn) before concluding the real subject
is actually in that state. This is what distinguished AO_PFM's genuinely
stuck `CONTEXT_EXHAUSTED` (same failure recurring across two reads,
minutes apart, each following an independent scheduled-trigger wake)
from a transient `STALE_FRAME` artifact (which self-heals within ~1.5s
and would NOT reproduce identically on a later independent read).

## What this repo does not yet cover

- No real adapter exists in `Sensorium-Of/sensorium` that senses wmux pane
  state as a `source` — today this channel is only read ad hoc, by an
  agent calling the MCP tools directly mid-task. A `wmux-pane-state`
  Sensorium-Of source adapter, informed by the channel states above,
  would be the natural next piece of real infrastructure (not built yet
  — flagging the gap, not closing it, per this turn's "map, don't fix
  yet" instruction).
- `CONTEXT_EVICTED_BLANK` is a hypothesis for the reported all-black
  symptom, not a confirmed root cause for this specific session's case —
  issues #197/#198 are themselves marked closed/fixed upstream, so if
  this is still reproducing, it's either a regression or a related-but-
  distinct cause. Unlike the mouse-wheel-history-recall bug (which has a
  confirmed source-level mechanism — see the chat transcript / a future
  filed issue), this one is still open.
