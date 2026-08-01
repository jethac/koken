# kōken 後見

**A knowledge back-end for realtime speech-to-speech, on one bandwidth-bound box.**

In kabuki, the *kuroko* moves the scenery. The **kōken** is the senior assistant who
attends the principal actor directly — hands them what they need mid-scene, and prompts
their lines. That is this repo's job: a fast conversational model does the performing,
and kōken feeds it what it doesn't know.

> **Status:** design + scaffold. See [DESIGN.md](DESIGN.md) for the full architecture.
> Nothing here is built yet; [kuroko](https://github.com/jethac/kuroko) works today.

## The problem

A full-duplex speech model like PersonaPlex-7B is a delightful conversationalist that
knows almost nothing — it's 7B of timing and prosody, not facts. The obvious fix is a
large model behind it. On a GB10 (128 GB unified, ~273 GB/s) there is room: the speech
model needs ~8 GB resident, leaving ~100 GB for a big MoE plus KV cache.

The obvious fix does not work, for a reason specific to this class of machine.

## Why you can't just run both

The speech model must emit one 80 ms audio frame every 80 ms. It is **weight-streaming
bound**, not compute bound: with 8-bit weights it streams ~6.5 GB per frame, roughly
140 GB/s — over half the machine's peak bandwidth. Its frame time is therefore almost
linear in available bandwidth.

Run a large MoE alongside it and the speech model misses its budget. And missing the
budget is not graceful degradation: in a Moshi-architecture model, **frame arrival is
the model's clock**, so starving it does not make it slow, it makes it *incoherent* —
it monologues, talks over you, and never yields the floor, while every log line and
byte counter looks perfectly healthy. (This was learned the hard way; see kuroko's
README.)

So the two models must not overlap. Ever.

## The approach: make the pause part of the act

[Sakana's KAME](https://github.com/SakanaAI/kame) solves this properly with a fourth
"oracle" stream that injects LLM knowledge *while* the speech model talks, letting it
revise mid-sentence. Two things put that out of reach here: the front-end must be
**trained** to consume that stream, and KAME's back-end was small and remote
(`gpt-4.1-nano`) — ours is large and local, which is exactly the contention above.

So instead of speaking while thinking, kōken **stops, thinks, and comes back**:

```
   conversing ──"let me look that up"──► HOLD ──answer──► conversing
                                          │
                          speech model FROZEN (0 GB/s)
                          MoE gets the whole machine
                          human hears hold music
```

This is cheap because of a property worth stating plainly: the speech model's serve
loop only advances when audio frames arrive. Stop sending and it does not idle or age —
it **freezes**. And because frame arrival is its clock, a suspended session experiences
**zero elapsed time**: on resume it needs no context reconstruction, no prompt reload,
no "as I was saying". It does not know it was gone.

## How the answer gets in

The driver owns two audio paths that are entirely independent:

| path | heard by |
|---|---|
| robot/client speaker | the human |
| model input stream | the model |

Nothing requires them to carry the same audio. So hold music covers the pause for the
human, while a TTS rendering of the MoE's answer is fed to the *model* — which the
human never hears. The model resumes and paraphrases the knowledge in its own voice.

A poor-man's oracle stream: no training required, at the cost of KAME's mid-sentence
refinement. Arguably a better interaction anyway — a person who says "let me look that
up" and comes back is normal; one who fluidly revises mid-sentence as facts arrive is
uncanny.

## Relationship to kuroko

[**kuroko**](https://github.com/jethac/kuroko) is the body: one Reachy Mini, one
PersonaPlex session, realtime audio discipline, embodiment. It is deliberately narrow
and works standalone — if you want a robot that talks, you want kuroko and not this.

kōken is the mind's scheduler. It **drives** kuroko rather than replacing it, through
the four-call surface in `kuroko/control.py`:

```python
bridge.suspend("looking something up")      # freeze; full context preserved
bridge.play(hold_music, sample_rate=16000)  # heard by the HUMAN only
bridge.inject(answer_pcm_24k)               # heard by the MODEL only
bridge.resume()
bridge.on_text(watch_for_trigger)           # model's text, before its audio plays
```

Nothing in that surface is robot-specific, so kōken should work against any front-end
that implements it — a robot, a kiosk, a phone, a web client.

## Plan

Ordered so the risky assumptions are tested before anything large is downloaded:

1. **Suspend/resume for real.** Confirm a session resumes coherently after a 15 s
   freeze. This is the whole premise — one afternoon to confirm or kill it.
2. **Hold audio.** Loop playback, ducking, and confirmation that the frozen model is
   undisturbed by it.
3. **Injection.** TTS a canned sentence into the model's input; confirm it paraphrases
   naturally rather than parroting or getting confused.
4. **Trigger.** Prompt the persona to announce lookups; parse it from the text stream
   (no ASR needed) and capture its restatement of the question as the query.
5. **The MoE.** Resident, behind a lock that makes overlap with an unfrozen speech
   model structurally impossible.
6. **Polish.** Thinking posture, a "still looking" beat, streaming the first sentence
   early so the robot can start talking before the full answer lands.

Steps 1–3 are testable against today's stack.

## License

MIT
