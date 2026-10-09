---
name: arch-agent-guide
description: Start here for any question about building, debugging, testing, optimizing or shipping an agent on the Kore.ai Agent Platform with the Arch MCP tools, such as "why did my agent do that", "which sessions failed", "which topology should I use", "why is it slow or expensive", "how do I test this", "is voice set up right" or "is it ready to ship". Sets up the MCP connection and project file, picks the right arch-* skill, and keeps answers in plain language.
---

# Arch Agent Guide

Use this skill first when a developer asks about an agent and you are not sure which Arch skill fits.
It sets up the connection, routes the question, and applies the working rules every other skill
assumes.

## 0. If the Arch MCP tools are not available

If no `platform_*` or `debug_*` tools are available, the MCP server is not connected. Offer to add it:

```bash
claude mcp add arch-agent-platform -- npx -y @koreai/arch-mcp-tools
```

```bash
codex mcp add arch-agent-platform -- npx -y @koreai/arch-mcp-tools
```

- Pin the environment by setting `AGENTS_URL` in the server's `env`: production
  `https://agents.kore.ai`, dev `https://agents-dev.kore.ai`, staging
  `https://agents-staging.kore.ai`, QA `https://agents-qa.kore.ai`.
- Pin the package version (`@koreai/arch-mcp-tools@<version>`) when a team must see the same tools.
- After restarting the client, confirm the server shows as connected (`/mcp` in Claude Code, the MCP
  list in Codex). If `platform_connect` fails, report its error as-is; do not fall back to REST
  calls or the UI.

## 1. Connect and scope before answering

1. Call `platform_connect`. If no environment is pinned, ask which one.
2. Get the project ID. If the developer does not know it, list projects with `platform_projects`.
3. For a debugging question, get the session ID, or find it with `debug_session_history`.
4. Read the project's `AGENTS.md` or `CLAUDE.md`. If there is none, offer to create one:

```markdown
# Agent project

- Platform environment: <production | dev | staging | QA>, project ID: <id>
- Channels and languages: <chat, voice pipeline, speech-to-speech; languages>
- Topology: <one paragraph: entry agent, specialists, how each is reached>
- Experience contract: docs/experience-contract.md (latency and model-call budgets per intent)
- Eval sets: <names>; baseline run: <run id>
- Always use the arch-agent-platform MCP tools to read sessions, traces, flow graphs and eval
  runs before answering or changing anything. Follow the arch-\* skills' output contracts.
```

An unscoped answer is a guess. Never answer from the UI's point of view when the evidence is one
tool call away.

## 2. Route the question

| The developer says something like                                                                | Use                                                                    | They get                                                                   |
| ------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------- | -------------------------------------------------------------------------- |
| "Why did it do that?", "the tool missed the input", "it handed off wrongly", "silent at turn 6"  | `arch-explain-session`                                                 | A turn-by-turn timeline, the first wrong event, its layer and the fix      |
| "Which sessions failed this week?", "is this happening to others?", "show me the worst sessions" | `arch-explain-session` (cross-session triage)                          | A ranked list of sessions with a pattern, then one session explained       |
| "A guardrail blocked a valid message", "the escalation fired wrongly"                            | `arch-explain-session`                                                 | The guardrail or decision event that went wrong, and the setting to change |
| "The knowledge-base answer is wrong", "set up a knowledge base", chunking, retrieval, citations  | `arch-explain-session`, then `arch-knowledge-search`                   | What was searched and retrieved, and the ingestion or setting change       |
| "Design this tool", "the tool was called twice", "the value was wrong or missing", confirmations | `arch-explain-session`, then `arch-tool-design`                        | Each tool's contract in a table, with the changes that make it safe        |
| "Call a workflow", "the workflow timed out", "the wrong version ran", a stuck approval step      | `arch-workflow-design`                                                 | The call contract, the live version and the execution timeline             |
| "What should we aim for?", "what does good look like?", starting a new project                   | `arch-experience-contract`                                             | Per-intent success definitions, latency targets and model-call budgets     |
| "Explain this project to me", "supervisor or one agent?", "add a specialist?", multi-intent      | `arch-topology`                                                        | The topology explained, call counts per turn and a recommendation          |
| "The model forgot", "prompt is huge", "tokens keep growing", "it repeated itself"                | `arch-explain-session`, then `arch-context-budget`                     | The turn where data was lost, then what grows and the lever that stops it  |
| "It's slow", "too many LLM calls", "cost is too high", "should we use a cheaper model?"          | `arch-latency-cost`                                                    | The most expensive turns, where the calls go, one lever each               |
| ASR, listening timeouts, barge-in, fillers, latency profile, pipeline vs realtime design         | `arch-voice-design`                                                    | Voice decisions per turn and per step, with an eval plan                   |
| "How do I test this?", personas, scenarios, evaluators, "did my change help?", repair loops      | `arch-eval-catalog`                                                    | A scenario catalog and an eval compare you can trust                       |
| "Is it ready?", "lint my ABL", before publish or deploy                                          | `arch-release-check`                                                   | Lint, validation, eval compare and a pre-ship checklist                    |
| "Improve my agent's instructions", "add a guardrail", "remember the user across sessions"        | `arch-context-budget` and `arch-explain-session`, then `arch-platform` | What the model sees and why, then the verified change                      |
| "Preview features", "decision models", "speculative inference", "turn-detection model"           | `arch-voice-design` (preview reference)                                | Status and a safe adoption plan                                            |
| "Create an agent", "create a tool or knowledge base", "deploy this", import or export            | `arch-platform`                                                        | A planned, verified platform operation                                     |

Tie-breaks:

- A symptom in one session or turn (including dead air on one call) goes to `arch-explain-session`
  first; a design or configuration question goes to the design skill.
- Find the cause before tuning: `arch-explain-session` before `arch-latency-cost` or
  `arch-context-budget`.
- `arch-eval-catalog` before `arch-release-check`.
- Designing or tuning a tool, workflow or knowledge base goes to its design skill; the create,
  update or deploy operation itself goes to `arch-platform`.

## 3. Where the developer is in the project

| Phase                | Skills, in order                                                                                      |
| -------------------- | ----------------------------------------------------------------------------------------------------- |
| Week one             | `arch-experience-contract` → `arch-topology` → `arch-voice-design` (voice only) → `arch-eval-catalog` |
| While building       | `arch-tool-design`, `arch-workflow-design`, `arch-knowledge-search` as each part is added             |
| Build and debug loop | `arch-explain-session` → `arch-context-budget` or `arch-latency-cost` → re-run evals                  |
| Before every release | `arch-release-check`                                                                                  |

Week-one checklist; offer to complete whatever is missing before deep debugging:

- [ ] Arch MCP connected, environment pinned, project ID known.
- [ ] Project folder with the exported package and an `AGENTS.md` or `CLAUDE.md`.
- [ ] Experience contract written and agreed with the customer.
- [ ] Topology chosen, with a written reason against the contract.
- [ ] Voice only: ASR test set, listening settings per turn type, orchestration mode, latency profile.
- [ ] First scenario catalog reviewed, and a baseline eval run recorded.

## 4. Working rules for every answer

- **Ask, don't click.** Read sessions, traces, flow graphs and eval runs with the MCP tools instead
  of sending the developer to the UI.
- **Evidence over feel.** Every claim cites a session, turn, event, file line or eval run.
- **The platform is not a black box.** Every decision leaves evidence in the traces, the model
  request, the compiled agent and the flow graph, and the operation catalog is derived from the
  code. Read the evidence before proposing any change.
- **First wrong event.** Find where behavior first departs from what the ABL says should happen;
  everything after it is consequence. Fix at that layer, not at the symptom.
- **No trial and error.** Never suggest trying a prompt tweak, temperature or model change "to see
  if it helps". If the developer has been doing that, stop, route to `arch-explain-session`, and
  find the first wrong event before anything else changes.
- **One change at a time.** Re-run the eval set after each change and compare with the last run.
- **Budget before build.** Latency, model calls per turn and tokens per conversation are design
  inputs, not afterthoughts.
- **Protect live agents.** A change to a deployed agent goes through an eval compare against the
  version in production before it ships.

## 5. Answer in plain language

Developers ignore answers they cannot follow and go back to the UI. Every arch-\* skill defines an
output contract; follow it exactly. In all answers:

- Start with the point in plain words: the story of what happened, the recommendation, or the
  verdict.
- Use tables for timelines, comparisons and findings.
- Quote evidence; do not paraphrase it.
- Explain each platform term once, in one line, with an example from this project.
- Do not summarize or shorten tables. If the answer is long, write it to a markdown file in the
  project folder and reply with the path plus the first paragraph.
- End with what you checked, what you could not check, and why.

When data is missing, say so and get it rather than guessing:

| Missing                         | Do this                                                                              |
| ------------------------------- | ------------------------------------------------------------------------------------ |
| Project ID                      | List projects with `platform_projects` and ask which one                             |
| Sessions to analyze             | Ask for a session ID, or run a scripted conversation with `debug_load_agent`         |
| The full model request (prompt) | Ask the developer to use "Export entire trace" on the session page and read the file |
| An experience contract          | Offer `arch-experience-contract` before judging "slow" or "wrong"                    |
| An eval set                     | Offer `arch-eval-catalog` before calling any change an improvement                   |

## 6. Installing the Arch skills

```bash
npx -y -p @koreai/arch-mcp-tools arch-mcp-guidance install --client claude --scope project
```

Use `--client codex` for Codex and `--scope user` to install for every project. Add
`--skill <name>` to install one skill, and run `arch-mcp-guidance list` to see them all. Re-running
`install` after a package upgrade updates the skills in place. Install the whole set so routing works,
and note that `install` refuses folders that were modified or not installed by Arch.
