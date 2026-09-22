# ABL examples by pattern

> Curated from the Agent Platform examples. The full source is maintained in
> `apps/docs-internal/content/examples/by-pattern.mdx`.

## 1. Open-ended reasoning agent

Use this pattern for advisory, search, Q&A, and exploratory conversations.
Without a `FLOW`, the model can choose when to gather information, call a
declared tool, and respond.

```abl
AGENT: Product_Advisor
VERSION: "1.0.0"
DESCRIPTION: "Recommends products based on customer needs"

GOAL: |
  Understand the customer's requirements and recommend suitable products.
  Explain trade-offs and never claim current availability without evidence.

PERSONA: "Knowledgeable, balanced product specialist"

TOOLS:
  search_products(query: string, max_results: number = 5) -> {products: object[]}
    description: "Search the authorized product catalog by keyword"

GATHER:
  use_case:
    prompt: "What are you looking for?"
    type: string
    required: true
```

## 2. Deterministic structured flow

Use this pattern for forms, regulated processes, and operations where order
matters. `REASONING: false` keeps a step deterministic; the flow still has all
declared agent capabilities and can use tools at its steps.

```abl
AGENT: Request_Intake
GOAL: "Collect and validate a service request"

TOOLS:
  create_request(summary: string, priority: string) -> {request_id: string}
    description: "Create a request after the user confirms the collected details"

FLOW:
  entry_point: collect_summary

  steps:
    - collect_summary
    - collect_priority
    - confirm
    - create

  collect_summary:
    REASONING: false
    GATHER:
      - summary: required
    THEN: collect_priority

  collect_priority:
    REASONING: false
    GATHER:
      - priority: required
    THEN: confirm

  confirm:
    REASONING: true
    RESPOND: "Please confirm the summary and priority before I create the request."
    THEN: create

  create:
    REASONING: false
    CALL: create_request(summary, priority)
    THEN: COMPLETE
```

## 3. HTTP-backed tool

The ABL file declares the contract. Configure the endpoint, authentication,
allowlist, and credentials in Project Tools or an auth profile. Never put a
real token in the agent file.

```abl
AGENT: Account_Lookup
GOAL: "Help an authenticated user check account status"

TOOLS:
  get_account(account_id: string) -> {status: string, balance: number}
    description: "Retrieve the current account status for an authorized account ID"
    type: http
    endpoint: "https://api.example.com/accounts"
    method: GET
    auth: oauth2_user
```

## 4. Supervisor and handoff

Use a supervisor when distinct domain agents own different capabilities. Keep
routing rules ordered, pass only the context the child needs, and define what
happens when control returns.

```abl
SUPERVISOR: Support_Supervisor
GOAL: "Route each request to the correct specialist"

INTENTS:
  account: "The user needs account information"
  technical: "The user needs technical support"

HANDOFF:
  - TO: Account_Specialist
    WHEN: intent.category == "account"
    CONTEXT:
      summary: "The user needs account information"
      history: auto
    EXPECT_RETURN: true

  - TO: Technical_Specialist
    WHEN: intent.category == "technical"
    CONTEXT:
      summary: "The user needs technical support"
      history: auto
    EXPECT_RETURN: true
```

## 5. Grounded knowledge agent

Use a knowledge-search tool for policy, SOP, FAQ, or product-document
questions. Instruct the agent to cite the retrieved source and distinguish
documented facts from interpretation.

```abl
AGENT: Policy_Advisor
GOAL: "Answer policy questions from authorized indexed documents"

TOOLS:
  search_policy(query: string, top_k: number = 5) -> {results: object[]}
    description: "Search authorized policy documents and return source metadata"

INSTRUCTIONS: |
  Search before answering policy questions.
  Answer from retrieved content only.
  Cite the document and section when available.
  Say when the indexed documents do not answer the question.
```

## Choosing a pattern

| Need | Starting pattern |
| --- | --- |
| Flexible conversation or research | Open-ended reasoning agent |
| Required order, retries, or confirmation | Structured flow |
| External system read/write | HTTP-backed tool |
| Multiple specialist domains | Supervisor and handoff |
| SOP, policy, or FAQ answers | Grounded knowledge agent |

Start with the smallest pattern that matches the use case. Add memory,
guardrails, delegation, or more tools only when the requirement is real and
the trace shows the capability is needed.
