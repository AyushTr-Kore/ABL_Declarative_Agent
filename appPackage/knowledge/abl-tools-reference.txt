# ABL tools reference

> Curated from the Agent Platform ABL reference. The full source is maintained
> in `apps/docs-internal/content/abl-reference/tools.mdx`.

The `TOOLS:` section declares the typed capabilities an agent may call. A tool
has a name, typed parameters, a return type, a description, and an optional
execution binding.

## Declaration syntax

```abl
TOOLS:
  search_products(query: string, max_results: number = 5) -> {products: object[], total: number}
    description: "Search the product catalog by keyword"
    type: http
    endpoint: "https://catalog.example.com/api/search"
    method: POST
```

Use lowercase `snake_case` tool names. Parameters without a default are
required; parameters with a default are optional. Return types should describe
the stable result shape the agent will receive.

## Descriptions and metadata

Tool descriptions guide model selection. State what the tool does, when it
should be used, important inputs, and meaningful limitations. Avoid vague
descriptions such as `Does something`.

```abl
  get_order(order_id: string) -> {status: string, items: object[]}
    description: "Retrieve the current status and line items for one order. Use after collecting an exact order ID."
```

Use metadata to make side effects and confirmation requirements explicit. A
tool's `confirmation` setting supports user approval flow; it does not replace
server-side authorization.

## HTTP tools

HTTP bindings support endpoint, method, path parameters, headers, result
transformation, timeout, and authentication configuration. Credentials must be
resolved from the platform credential store or an auth profile; never embed a
real key, token, or secret in an ABL file.

```abl
TOOLS:
  update_profile(name: string) -> {updated: boolean}
    description: "Update the authenticated user's display name after confirmation"
    type: http
    endpoint: "https://profiles.example.com/api/profile"
    method: PUT
    auth: oauth2_user
```

Use `{{config.NAME}}` or the platform's auth-profile binding for secret values.
Keep outbound domains allow-listed and validate external responses before using
them in later reasoning.

## MCP tools

MCP bindings connect a declared ABL tool to a configured MCP server. Dynamic
discovery can expose the server's available tool schemas, but the agent should
still declare clear local contracts and descriptions.

```abl
TOOLS:
  search_incidents(query: string) -> {incidents: object[]}
    description: "Search authorized incident records by symptom or identifier"
    type: mcp
    server: operations_mcp
    tool: search_incidents
```

The server remains responsible for authentication, authorization, target
scope, confirmation, and result bounds.

## Code tools

Code tools execute in a sandboxed JavaScript or Python environment. Use them
for bounded transformations and calculations, not as a way to bypass tool
authorization or access undeclared credentials. Set timeouts and avoid
unbounded loops or large outputs.

## Result and state handling

Use `on_result.set` when selected result fields should become explicit session
context. Do not expose runtime-private handles as model-visible memory.

```abl
TOOLS:
  lookup_case(case_id: string) -> {status: string, owner: string}
    description: "Retrieve one case and store its status for the next step"
    on_result:
      set:
        case_status: "status"
        case_owner: "owner"
```

Use `store_result: false` when retaining the raw result blob is unnecessary or
would expose more data than the agent needs.

## Tool file imports

Reusable tools can live in `.tools.abl` files and be imported into an agent.
Keep shared definitions stable and avoid silently changing a tool's parameter
or result contract for consumers.

## Tool debugging checklist

1. Confirm the tool is declared and bound to the agent.
2. Check the name, parameter types, required fields, and result shape.
3. Improve the description if the model chooses the wrong tool.
4. Test the tool independently when possible.
5. Inspect the trace for actual inputs, outputs, latency, and errors.
6. For `401`, OAuth, or credential errors, use Auth Profiles and never request
   raw credentials in chat.
