---
name: arch-voice-design
description: Design and configure voice agents on the Kore.ai Agent Platform, for pipeline voice (speech-to-text, model, text-to-speech) and realtime speech-to-speech. Use for design and configuration questions about ASR quality and test sets, language switching, end-of-speech and silence timeouts, barge-in and greetings, fillers and dead air, voice latency profiles, pipeline vs realtime, and realtime orchestration modes (runtime_guarded vs runtime_owned). For one bad call, start with arch-explain-session.
---

# Voice design

Voice is its own architecture. The most expensive voice mistakes are decisions deferred until after
the functional build: ASR quality, timeouts and realtime ownership. Decide them in week one.

## Gather the evidence

- Read the project runtime and voice latency profile settings with `platform_config`
  (`get_runtime_config`), and the agents' `VOICE` and `EXECUTION.voice` blocks with
  `platform_package_model` (it reads an exported package: pass its `path` or `files`).
- Sample real or test calls with `debug_session_history` (filter `channel`) and read voice events
  with `debug_traces`: `voice_dead_air`, `voice_filler_playback` and `voice_playback_watchdog`. For a
  call that is not live, pass `projectId` and `source: "historical"`.
- Measure silence from the end of the caller's speech, not from the start of the turn.
- Runtime traces can miss provider and gateway events (connection closes, provider settings such as
  temperature). Ask for gateway logs when the trace does not explain a silence.

## Week-one decisions

| Decision             | Do this                                                                                                                                                                                                                                                                                  |
| -------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| ASR quality          | Build an ASR test set before the flows: recognition per intent, language confidence, entity values (names, IDs, dates, amounts, addresses) in every language and accent, backchannel ("uh-huh", "okay"), background noise. See [references/asr-test-set.md](references/asr-test-set.md). |
| Timeouts             | Set provider and per-turn listening settings. Yes/no turns end quickly; identifiers need a pause allowance; open answers need more time.                                                                                                                                                 |
| Pipeline vs realtime | Decide from the experience contract and an eval comparison, not a slide.                                                                                                                                                                                                                 |
| Realtime ownership   | `runtime_guarded` for freeform agents: the model converses and proposes, Runtime validates order, context writes, required values and tool access. `runtime_owned` for payments, regulated changes and irreversible operations.                                                          |
| Latency profile      | Start from `customer_service_voice_low_latency_v1` and override only the keys your use case needs.                                                                                                                                                                                       |

## Per-turn endpointing (pipeline voice)

Set the expected answer shape on a `RESPOND`, a FLOW step's `VOICE` block, a behavior profile, or
`EXECUTION.voice` as the agent default:

```yaml
RESPOND: 'Please confirm the appointment.'
VOICE:
  end_of_speech_turn_type: simple_confirmation
```

```yaml
RESPOND: 'Please describe what happened.'
VOICE:
  end_of_speech_timeout_ms: 1500
```

- Turn types: `simple_confirmation`, `structured_value` (account numbers, IDs, dates, codes),
  `open_ended_question`, `complex_explanation`, `anything_else`.
- `end_of_speech_timeout_ms` accepts 100–5000 ms and overrides the turn type's recommended value;
  a value outside that range is ignored and the turn type's value applies.
- The most specific setting wins: response, then FLOW step, then behavior profile, then
  `EXECUTION.voice`, then the conversation baseline.
- This applies to pipeline recognition only. Native realtime (speech-to-speech) providers keep their
  own turn detection, tuned through the channel's VAD settings.

## Latency profile and fillers

- The low-latency profile sets fast model tiers, reasoning off, inline gather, structured tool
  results and tight compaction for voice sessions only. Give the workspace a fast-tier model; without one the turn falls back to the project's
  default model and a warning is recorded, losing the speed gain.
- Precedence differs by setting. Fillers: project filler settings override the profile, and the
  profile's own `filler` overrides win over both. Compaction and model routing: project settings
  win over the profile. Authored agent settings (inline gather, reasoning) always win.
- Fillers hide waiting; they do not remove it. If generated fillers never play and the static line
  always does, check whether the filler model can respond within the filler voice delay.
- Re-check runtime voice settings (fillers, VAD) after every deploy.

## Preview features for low-latency voice

The platform team offers preview (beta) capabilities that are made available per environment and
then switched on selectively, for example per project and per decision surface. They are not on by
default; confirm availability and status for the customer's
environment before designing around them, and prove each one with an eval comparison against the
same project with the feature off. Details: [references/low-latency-previews.md](references/low-latency-previews.md).

| Preview                                                         | What it does                                                                                                                                                                                                                      | Measure before keeping it                                                               |
| --------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------- |
| Small models for barge-in                                       | Decide whether caller speech during playback is a real interruption                                                                                                                                                               | False barge-ins (backchannel, noise), missed interruptions, time to stop playback       |
| Small models for turn detection                                 | Decide when the caller has finished speaking, beyond a fixed silence timeout                                                                                                                                                      | Cut-off answers, end-of-turn delay, identifier turns with pauses                        |
| Decision models (early development, not yet in any environment) | Small typed-judgment models answer specific decisions (routing, input safety, confirmation replies, gather interactions, flow escape) before the main model; uncertain, failed or slow answers fall back to the normal model path | Decision accuracy per surface, fallback rate, latency saved, any confident wrong answer |
| Speculative inference                                           | Starts work from partial transcripts so the reply is ready sooner                                                                                                                                                                 | Time to first audio, mismatches between partial and final transcripts, extra model cost |

## Realtime speech-to-speech

Speech-to-speech is not chat with audio. Read
[references/speech-to-speech.md](references/speech-to-speech.md) before building or debugging a
realtime agent. The short version:

- Choose the orchestration mode explicitly and verify the effective mode in traces. New realtime
  channel connections default to `runtime_owned`; connections created before the setting existed
  keep legacy behavior. Moving a live agent from legacy to a guarded or owned
  mode changes which tool calls are accepted, so run the eval set first.
- Put ordering, field ownership, validation and confirmation in FLOW. The prompt is not an
  authorization boundary, and it must never contradict the FLOW.
- Declare `effect` on every tool, use `provides` and `on_result` SETs, `process.before.require`, and
  `confirm` for writes. Make backend writes idempotent.
- Never assume the provider speaks a tool result on its own. Test every tool exit path.

## Local voice test harness

Build one on a laptop with your coding agent:

1. Generate caller audio per scenario with text-to-speech in every target language and several
   accents.
2. Mix in noise profiles (street, call center, car, TV) and add backchannel and hesitation clips.
3. Play each clip through the voice channel against a local or dev project.
4. Capture the recognized text, detected language, gathered values and per-turn timings.
5. Score against expected values and write a failure table by language, entity type and noise.

Run it whenever the ASR provider, timeouts, latency profile or gather fields change.

## Output contract

1. **Recommendation** in two plain sentences (pipeline or realtime, and the orchestration mode).
2. **FLOW step table:** step | what the caller does | structural or model-driven | why.
3. **Listening table:** turn | expected answer shape | end-of-speech setting | ASR risks.
4. **Latency profile** and every override, each with a reason.
5. **Eval comparison:** scenarios, configurations, and metrics (task success, first-audio latency,
   tool-order violations, dead-air intervals).
6. **Glossary:** endpointing, barge-in, filler, VAD, orchestration mode and any other voice term,
   each with an example from this project.
7. **Confidence and gaps.**
