# Low-latency preview features

These capabilities are previews: they are made available per environment and are off by
default, and their settings can change between releases. Treat everything below as a starting
point and confirm the current behavior for the customer's environment.

## How to adopt a preview safely

1. Write down what it should improve, as a number from the experience contract (first-audio
   latency, cut-off rate, false barge-ins).
2. Record a baseline eval run and a voice harness run with the feature off.
3. Enable it on one project (and one decision surface, for decision models) and re-run the same
   scenarios. Compare task success, latency and the failure modes in the table below.
4. Keep it only if success held and the target number moved. Check again after each platform
   release.

## Small models for barge-in and turn detection

There are no project settings to document yet; the nearest settings today are the realtime
channel's turn-detection options (for example semantic VAD on OpenAI realtime). Ask the platform
team for the current preview status.

- Barge-in models decide whether caller speech during playback is a real interruption.
  Turn-detection models decide when the caller has finished, beyond a fixed silence timeout.
- Test them with the local voice harness: backchannel ("uh-huh", "okay") during playback, noise,
  long identifiers read with pauses, hesitations, and every supported language.
- Compare against the per-turn endpointing settings you already use; a turn-detection model should
  reduce cut-offs on identifier turns without adding delay on yes/no turns.
- Keep protected messages (welcome, legal notices) protected with `barge_in: disallow` regardless.

## Decision models

Status: in early development and not yet available in any environment. Ask the platform team
before planning around it. When available it also needs a workspace feature flag set by a platform
admin.

Small typed-judgment models answer a fixed set of decisions before the main model. A confident
answer is used; a low-confidence, failed or slow answer falls through to the normal model path
within a time budget.

| Decision surface         | Options it decides                           | Notes                                                                                     |
| ------------------------ | -------------------------------------------- | ----------------------------------------------------------------------------------------- |
| Supervisor routing       | The supervisor's authored `INTENTS`          | Handoffs still happen only through authored `HANDOFF` rules                               |
| Input safety             | Safe or unsafe for a guardrail's model check | A confident "safe" clears the check without the model judge, so set a high confidence bar |
| Interaction confirmation | Accept, decline, defer                       | "Accept" stays with the main model by default; keep it that way for side effects          |
| Gather interaction       | Answer, pause, resume, cancel                |                                                                                           |
| Flow escape              | The step's authored escape intents           |                                                                                           |

- Workspace admins register providers; project admins switch on each surface. Configuration lives
  in project Runtime Config (routing in the reasoning pipeline settings,
  interaction surfaces in conversation handling) and on the Guardrails page for safety. Providers
  are registered per workspace. Nothing new is authored in ABL.
- Enable one surface at a time, with a per-surface confidence threshold, and compare evals.
- Check the provider's latency from the customer's region: a provider slower than the surface's
  budget always falls back, so it adds latency without deciding anything.
- Thresholds are meaningful only if the provider's confidence is calibrated; check before relying
  on them.
- Every decision appears as a trace event; watch the fallback rate and reasons per surface, and
  sample decisions for correctness.

## Speculative inference

Starts preparing the reply from partial transcripts. It is enabled by the platform operator in the Runtime environment configuration (not on an agent
deployment), applies only to
pipeline voice turns that do not call tools, and is off by default. Any new speech, barge-in or
transcript change cancels the speculative work, and cancelled provider work may still be billed.
Measure time to first audio, how often the final transcript differs from the partial one, and the
extra model cost.
