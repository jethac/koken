# Build guide for agents

You are picking up kōken at the design stage. **There is no code yet** — this repo is
`README.md`, `DESIGN.md`, and this file. Read all three before writing anything.

Your job is to build the thing described in `DESIGN.md`. This document exists to stop
you re-deriving what has already been measured, and to stop you walking into traps that
have already cost someone a day.

Environment-specific details (hostnames, IPs, paths, credentials) are deliberately **not
in this public repo** — see `ENVIRONMENT.local.md` if the operator has provided one, or
ask them for: the inference host, the robot host, and how to reach both.

---

## 1. What this is, in one paragraph

A realtime speech-to-speech model (PersonaPlex-7B, Moshi architecture) holds the
conversation. It knows almost nothing. A large MoE is resident on the same box to
answer real questions. They **cannot run at the same time** (§3). So when the persona
needs facts, kōken suspends the speech session, plays hold music to the human, runs the
MoE with the whole machine, feeds the answer to the model as audio the human never
hears, and resumes. See `DESIGN.md` for the full flow.

kōken does **not** own the robot or the audio path. [kuroko](https://github.com/jethac/kuroko)
does, and it works today. kōken drives it through a four-call surface (§4).

## 2. Ground truth — measured, do not re-litigate

These were established empirically on the target hardware (NVIDIA GB10 / DGX Spark
class, 128 GB unified LPDDR5X, ~273 GB/s, sm_121). Treat them as facts.

| Fact | Value | Why it matters |
|---|---|---|
| Frame budget | 80 ms (one Mimi frame) | Miss it and the model is *incoherent*, not slow |
| Speech model frame time | 46.5 ms pinned, ~53.5 ms in container | ~27 ms headroom |
| Speech model bandwidth | ~6.5 GB/frame ≈ 140 GB/s | Over half the machine's peak |
| Speech model weights (w8a16) | ~8 GB resident, 16 GiB on disk | Leaves ~100 GB |
| Cold start | ~135 s | Anything loaded per-query is a non-starter |
| Suspension cost | **zero** | Serve loop only advances on frame arrival |

**The suspension property is the load-bearing claim of this entire project.** The
PersonaPlex serve loop reads PCM and, if none has arrived, continues without stepping
the model. So a connected-but-unfed session freezes: it does not idle, poll, or age its
context. Verified twice — a warm session holds `rx_bytes` flat indefinitely. And because
frame arrival *is* the model's clock, a suspended session experiences **zero elapsed
time**: it resumes with full context and does not know it was gone.

Everything in `DESIGN.md` depends on that. **Step 1 of your build is to prove it holds
for a 15 s freeze mid-conversation.** If it doesn't, stop and redesign before doing
anything else.

## 3. Why the two models must never overlap

The speech model is weight-streaming-bound, not compute-bound. `lm_step` is ~49 of its
53.5 ms, and it scales close to linearly with available memory bandwidth. An MoE taking
even a third of the bandwidth pushes a frame past 70 ms.

Budget misses do not degrade gracefully. In a Moshi-architecture model, frame arrival is
the clock, so starving it desynchronises the model's sense of time: it monologues, talks
over the user, and never yields the floor — while every log line and byte counter looks
perfectly healthy. This exact failure was diagnosed the hard way in kuroko.

**Therefore:** enforce mutual exclusion structurally (a lock the MoE must hold, which
cannot be acquired while the speech session is unsuspended), not by convention.

## 4. The kuroko control surface

`kuroko/control.py`. This is all you get, and all you should need:

```python
bridge.suspend(reason="")     # freeze the model; full context preserved
bridge.resume(reason="")      # unfreeze; it never knew it stopped
bridge.is_suspended() -> bool
bridge.quiet_for() -> float   # seconds since the model last produced output
bridge.inject(pcm24k)         # audio the MODEL hears, the human does not
bridge.play(pcm, sample_rate) # audio the HUMAN hears, the model does not
bridge.on_text(callback)      # model's text stream, piece by piece
```

Two properties to internalise:

- **The audio paths are independent.** `play()` reaches only the speaker; `inject()`
  reaches only the model's input. This is what makes a hold possible — music for the
  human, an answer for the model, neither aware of the other.
- **`on_text` fires before the audio plays.** The text stream leads the speaker by the
  playout buffer, so it is the cheapest possible place to detect intent.

If you need something not in this list, add it to kuroko deliberately as an API change —
do not reach into `VoiceBridge` internals from kōken.

## 5. Build order, with acceptance criteria

Ordered so the cheap risky things are tested before the expensive ones. **Do not skip
ahead to the MoE.**

### Step 1 — suspend/resume for real
Drive a live kuroko session: wait for the model to finish an utterance
(`quiet_for() > 1.0`), `suspend()`, wait 15 s, `resume()`.

**Accept when:** the model resumes coherently and continues the conversation without
restating, glitching, or losing context. **Reject and redesign if:** it resumes
mid-word, repeats itself, or behaves as though the conversation restarted.

### Step 2 — hold audio
Loop an audio file through `play()` while suspended. Fade in/out.

**Accept when:** the human hears clean looping audio, and on resume the model shows no
awareness of it (it should not comment on music). Note the robot's hardware AEC already
scrubs speaker output from the mic path, so this should hold — verify, don't assume.

*Use royalty-free or generated audio. The obvious think-music cue is under copyright.*

### Step 3 — injection
TTS a canned sentence ("The capital of Australia is Canberra") and `inject()` it while
suspended, then resume.

**Accept when:** the model paraphrases the content naturally in its own voice. **Watch
for:** parroting it verbatim, ignoring it, or reacting to it as an interruption ("who
said that?"). If the model is confused, frame it in the persona's system prompt — it has
a research assistant who tells it things.

### Step 4 — the trigger
Prompt the persona to announce lookups with a fixed phrase and restate the question.
Watch `on_text` for the phrase; capture the restatement as the query.

**Accept when:** normal conversation does not trigger, and factual questions do, with a
usable query string. **Expect to tune:** over-triggering makes every exchange a hold.
Rate-limit it.

### Step 5 — the MoE
Only now. Resident, never loaded per query. Behind the exclusion lock from §3.

Sizing: ~120B-class at 4-bit is 60–70 GB, leaving 30–40 GB for KV cache. Choose a serving
stack that works on aarch64 + sm_121 — **verify this early**, it is a real constraint and
several popular stacks do not.

**Accept when:** a full cycle works end to end and the speech model's frame time is
unaffected (check kuroko's `tx_fps` stays at 12.5 and the server's `total=` stays under
80 ms during and after a hold).

### Step 6 — polish
Thinking posture during hold, a "still looking" beat at ~8 s, and streaming the MoE's
first sentence early so the robot can start talking before the full answer lands (the
cheapest step toward KAME-like behaviour without retraining).

## 6. Traps that have already cost time

Inherited from building kuroko. All of these were real.

- **Probe a speech DSP with speech, never a tone.** The robot's mic array suppresses
  non-speech by design. A sine-wave test reports a dead microphone that is perfectly
  healthy. Two wrong diagnoses came from this.
- **The robot's audio hardware is excellent — do not "fix" it.** Hardware AEC strips the
  robot's own voice ~28 dB while leaving the user's intact (barge-in +3.1 dB). Adding
  echo cancellation makes things worse.
- **`sphn` must be `<0.2`.** 0.2.x removed `read_bytes`/`read_pcm`. It also publishes no
  linux-aarch64 wheels, so it builds from source (needs rust + cmake, and
  `CMAKE_POLICY_VERSION_MINIMUM=3.5` to get past audiopus_sys).
- **Opus wants exact frame sizes.** 1920 samples at 24 kHz. Feed it anything else and it
  raises.
- **Keep blocking calls off the event loop.** `sphn` codec calls and the Reachy SDK's
  `goto_sleep`/`wake_up` (~5 s each!) are synchronous. Inline they stall everything.
- **Do not reassign `asyncio.Event` objects** that long-lived tasks hold references to —
  clear them instead, or those tasks end up watching a dead event forever.
- **The PersonaPlex server discards everything before its `\x00` handshake**, including
  the opus stream header. Wait for it.
- **Grammar-constrained recognizers silently ignore out-of-vocabulary words**, so a
  phrase containing an unknown word never matches and never errors. `reachy` is not in
  the small English model. Check with kuroko's `probe/vocab.py`.

## 7. Conventions

- **Measure, don't assume.** Every number in §2 came from a probe. kuroko's `probe/`
  directory is the model for this — small, single-purpose, documents its own reasoning.
  Add probes to kōken as you go.
- **Public content carries no AI attribution.** The operator owns authorship of anything
  published. Commit trailers are fine; READMEs, articles, and PRs are not.
- **Draft, don't post.** Do not open PRs, publish, or push to third-party repos without
  explicit approval.
- Commit messages: what changed and *why it was not obvious*, not a diff summary.

## 8. Decisions left open

Raise these with the operator rather than guessing:

- **Which MoE**, and which serving stack works on aarch64 + sm_121.
- **Where TTS comes from** for injection, and whether its voice matters (the model
  paraphrases, so probably not — worth confirming in step 3).
- **How the query is captured** if the persona's restatement proves unreliable — real
  ASR over kuroko's ring buffer is the fallback, at extra cost.
- **Whether kōken owns the session lifecycle** long-term, with kuroko demoted to pure
  embodiment. Cleaner layering, but a real refactor; not needed for v1.
