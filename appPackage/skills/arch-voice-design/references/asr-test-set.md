# ASR test set

Build it before the flows. Score it on every change to the ASR provider, timeouts, latency profile
or gather fields.

| Dimension              | What to include                                                                       | What to measure                                  |
| ---------------------- | ------------------------------------------------------------------------------------- | ------------------------------------------------ |
| Recognition per intent | 10–20 utterances per top intent, natural phrasing                                     | Word accuracy; intent still recognized           |
| Language               | Every supported language, plus mid-call switches                                      | Detected language and confidence; time to switch |
| Accents                | Several accents per language                                                          | Accuracy by accent                               |
| Entity values          | Names, IDs, account and phone numbers, dates, amounts, addresses, spelled-out letters | Exact match of the gathered value, per language  |
| Long identifiers       | Numbers read with pauses and in groups                                                | Whether the turn ends mid-value                  |
| Backchannel            | "uh-huh", "okay", "yes yes", "hmm" while the agent speaks                             | False barge-ins; turns wrongly ended             |
| Noise                  | Street, call center, car, TV, music                                                   | Accuracy and false turn starts per noise profile |
| Silence and hesitation | Long pauses, "umm", restarts                                                          | Premature end of turn; no-input handling         |

Write results as a failure table: language | entity type | noise profile | expected | heard |
gathered value | cause (ASR, endpointing, extraction).
