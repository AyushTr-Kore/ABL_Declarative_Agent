# Scenario families

## Core families

| Family                          | Examples                                                                           |
| ------------------------------- | ---------------------------------------------------------------------------------- |
| Happy path per top intent       | Each intent from the experience contract, end to end                               |
| Partial accept or deny          | "Yes to the replacement card, no to expedited shipping"                            |
| Objections and pushback         | "That fee is ridiculous", "Why do you need my PIN?", "I already told you that"     |
| Corrections and changes of mind | Correcting a gathered value two turns later                                        |
| Digressions and side questions  | A side question in the middle of a gather                                          |
| Multi-intent                    | Two requests in one message; dependent and contradictory requests                  |
| Handoff and return              | Specialist hand-back; out-of-scope question inside a specialist; nested agents     |
| Failure paths                   | Tool timeout, empty result, error, authentication failure, model error, escalation |
| Adversarial and policy          | Prompt injection, PII over-sharing, off-topic, abusive language                    |
| Voice                           | Noise, accents, languages, backchannel, long identifiers, barge-in, silence        |
| Efficiency                      | The same scenarios with `maxToolCalls`, `maxTurns` and latency budgets             |

## Multi-intent families

Multi-intent is where untested projects break most often. Cover each of these:

| Family                       | Examples                                                                                        |
| ---------------------------- | ----------------------------------------------------------------------------------------------- |
| Offer replies                | Plain yes, plain no, "later", a new request instead of an answer, a yes after the offer expired |
| Objections during a plan     | Complaining mid-plan, asking why information is needed                                          |
| Questions and blocking       | A specialist asks a question; the caller answers before anything else starts                    |
| Request shapes               | Two independent requests, dependent requests, contradictory requests, duplicates                |
| Agent topology               | Nested child and grandchild agents and their return paths                                       |
| Failures                     | The first request's tool fails, times out or is refused                                         |
| Safety and integrity         | A sensitive value typed for one agent must not reach another                                    |
| Channel, language and config | Each channel, a language switch mid-plan, each configuration in use                             |

## Behaviors to decide before testing

Agree with the customer what should happen, then write the scenario:

- a partial denial plus a new request;
- complaining in the middle of a multi-step plan;
- cancelling everything;
- changing one's mind after accepting;
- a request that depends on another's result ("check my balance, then transfer if there's enough");
- contradictory requests ("block my card, and use it to pay my bill").
