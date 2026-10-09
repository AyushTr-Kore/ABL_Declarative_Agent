# Worked example (illustrative)

This shows the expected shape. The session is invented.

**Story.** A caller gave a 10-digit phone number with a pause after the first three digits. The
turn ended at the pause, the agent saved "012" as the phone number, treated the remaining digits as
the answer to the next question, and created the order with a 3-digit number.

| Turn | Caller said → ASR heard                                       | Owner                          | Calls                                     | State change                                             | Latency                                         |
| ---- | ------------------------------------------------------------- | ------------------------------ | ----------------------------------------- | -------------------------------------------------------- | ----------------------------------------------- |
| 3    | "zero one two … three four five six seven eight nine" → "012" | Booking agent · `gather_phone` | 1 extraction                              | `phone: unset → "012"` (gather extraction)               | Pause 0.9 s outlasted the end-of-speech timeout |
| 4    | "three four five six seven eight nine"                        | Booking agent · `confirm_slot` | 1 reply                                   | `slot_time: unset → "09:00"` (model guessed from digits) | —                                               |
| 5    | "yes"                                                         | Booking agent · `create_order` | 1 tool: `create_guest_order(phone="012")` | `order_id: unset → "A-1031"` (tool result)               | Tool 1.8 s                                      |

**First wrong event.** Turn 3, ASR final "012".

**Layer.** Endpointing. The caller had not finished; extraction faithfully copied what it was given.

**Fix.** A longer per-turn end-of-speech setting for identifier turns, plus
`process.before.require` on phone length so a 3-digit number can never reach the tool.

**Would not fix.** "Always collect the full phone number" in the prompt.

**Eval scenario.** Phone number with a mid-number pause, in each supported language.

**Glossary.**

- _Endpointing_: deciding when the caller has finished speaking. Here a 0.9 s pause ended turn 3.
- _Gather extraction_: the step that copies a value from the caller's words into a field. Here it
  set `phone` to "012".
- _`process.before.require`_: a check that runs before a tool call and rejects bad inputs. Here it
  would reject a 3-digit phone number.

**Confidence and gaps.** Checked ASR finals, extraction events and the tool call. Did not have the
audio, so the pause length is taken from event timestamps.
