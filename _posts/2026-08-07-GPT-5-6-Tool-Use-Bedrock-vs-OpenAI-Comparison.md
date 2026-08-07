--- 
title: "GPT-5.6 Tool Use: Bedrock Server-Side vs. OpenAI Server-Side vs. Client-Side"
social:
  name: Dr Richard Kang
  links:
    - https://www.linkedin.com/in/kangks/
---

## Real implementation details and measured comparisons, for anyone evaluating how to wire up tool-using GPT-5.6 agents

This post compares three tool-execution patterns relevant to giving a GPT-5.6 agent tools today. It isn't an exhaustive list of every tool-integration option either provider ships (OpenAI also has built-in hosted tools and Programmatic Tool Calling; Bedrock's AgentCore Gateway also routes to API Gateway and OpenAPI targets, not just Lambda) — these three are the patterns that determine request count, latency, credential exposure, and how much infrastructure you build:

1. **Client-side function calling** — available on both OpenAI and Bedrock. The calling application parses the model's tool-call output, executes the tool(s) itself, and resubmits the result(s). One round trip per *sequential round* of tool calls — a single round can contain multiple parallel calls, resolved together in one resubmission (see the correction below).
2. **Bedrock server-side tool execution, in this demo** — the model calls a Lambda function behind an AgentCore Gateway, authorized with AWS IAM. No publicly reachable or always-on MCP server for the customer to operate — but the customer still builds and operates the Lambda code, Gateway targets, IAM policy, and monitoring.
3. **OpenAI server-side tool execution** — for a custom internal tool, the customer typically builds and operates the remote MCP endpoint (or a private one via OpenAI's Secure MCP Tunnel). OpenAI-maintained connectors and third-party-hosted MCP servers have different hosting responsibilities — see below.

**Both providers support both a client-side and a server-side pattern.** This is worth stating plainly, because it's easy to describe this space in a way that implies OpenAI lacks a server-side option — it doesn't. The real, accurate distinction between the two providers' server-side mechanisms is **where the tool is hosted and how it's authorized**, not whether server-side execution exists at all.

This post presents live code and measurements for Bedrock's server-side and client-side mechanisms, run back-to-back against a real deployed Gateway, plus a documentation-based description of OpenAI's remote-MCP mechanism, which was not live-tested here. That distinction is called out explicitly throughout rather than blended into one number.

The Bedrock side of this demo started from a reference implementation AWS GenAI Specialist Solutions Architect Vincent Wang pointed us to: [`aws-samples/sample-bedrock-server-side-tool-call`](https://github.com/aws-samples/sample-bedrock-server-side-tool-call) ("ShopAssist," a generic e-commerce sample). We adapted it to a villa-booking domain and, in the process, found and fixed the Responses API path bug described below — a real bug in that upstream sample, not specific to this adaptation.

## Architecture

| Mechanism | Where tool execution happens | Credential responsibility |
|---|---|---|
| Client-side function calling (`type: "function"`) — either provider | On the calling application's own infrastructure | The calling application holds and manages downstream credentials directly |
| OpenAI server-side (`type: "mcp"`, `server_url`/`connector_id`) | On a remote MCP server the customer typically builds and operates (connectors and provider-hosted MCP servers differ — see below) | Caller may supply an MCP `authorization` token per request (OpenAI does not store it — resend it on every call that reuses the tool); the MCP server holds any downstream credentials |
| Bedrock server-side (`type: "mcp"`, AgentCore Gateway `connector_id`) | Inside AWS, via AgentCore Gateway → Lambda (or other supported Gateway target) | Caller uses AWS credentials to call Bedrock; the Lambda's own execution role holds downstream AWS permissions — the caller never sees them |

![Sequence diagram of client-side function calling: client sends a request, parses the model's tool_call response(s), executes the tool(s) itself against a backend it holds credentials for, then resubmits the result(s) in one batched request — repeated once per sequential round of tool calls in the turn. This pattern is the same shape on both OpenAI and Bedrock.](/assets/images/2026-08-07-GPT-5-6-Tool-Use-Bedrock-vs-OpenAI-Comparison/sequence-client-side-function-calling.png)

![Sequence diagram of Bedrock server-side tool execution: client sends one signed request; Bedrock Mantle discovers tools via the AgentCore Gateway's MCP tools/list, calls tools/call against an IAM-authorized Lambda as needed, and returns a single final answer over the same connection](/assets/images/2026-08-07-GPT-5-6-Tool-Use-Bedrock-vs-OpenAI-Comparison/sequence-bedrock-server-side-tool-execution.png)

![Sequence diagram of OpenAI server-side tool execution: client sends one request; OpenAI's own Responses API backend discovers tools via a remote MCP server's tools/list, calls tools/call as needed (relayed through a Secure MCP Tunnel if the server is private), and returns a single final answer over the same connection — the tool's business logic executes on the customer's own MCP server, not on OpenAI's infrastructure](/assets/images/2026-08-07-GPT-5-6-Tool-Use-Bedrock-vs-OpenAI-Comparison/sequence-openai-server-side-mcp.png)

With client-side function calling, a tool-using turn costs **1 initial request + 1 request per sequential round of tool calls** — not one request per individual tool call. A model can return several `function_call` items in a single response (parallel calls in one round); the client executes all of them and batches every result into one resubmission, per [OpenAI's function-calling guide](https://developers.openai.com/api/docs/guides/function-calling) ("model responses can include zero, one, or multiple calls... after appending the results to your input, you can send them back to the model"). Three parallel calls in one round still costs 2 requests total; three *sequentially dependent* calls (each one only knowable after the previous result comes back) cost 4. With either provider's server-side mechanism, the same turn costs exactly 1 request **only when no approval step or other client intervention interrupts execution** — tool discovery, selection, execution, and final-answer generation then all happen inside that one call. On Bedrock's AgentCore Gateway integration, that currently means `require_approval` has to be `"never"` — it's the only accepted value. On OpenAI, `require_approval` defaults to pausing before each tool call (returning an `mcp_approval_request` the caller must approve and resubmit) unless explicitly set to `"never"`. Both conditions are detailed below.

## Implementing Bedrock server-side tool execution

One implementation detail worth knowing before writing this code: the GPT-5.6 family (Sol, Terra, Luna) is served on a **different** Bedrock Mantle Responses API path than most other models. AWS's own model card for GPT-5.6 Sol states this directly:

> "This model is available on the `openai/v1/responses` path on the `bedrock-mantle` endpoint. This is different from the `v1/responses` path used by other models on the responses endpoint."
> — [GPT-5.6 Sol model card — Amazon Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/model-card-openai-gpt-56-sol.html)

This isn't mentioned in AWS's general Responses API guide or its server-side tool-use guide — only on the specific model's own card. Models like `openai.gpt-oss-120b` use the plain `/v1/responses` path those general guides document. Get this wrong and you get a real HTTP 400 (`"The model 'openai.gpt-5.6-sol' does not support the '/v1/responses' API"`). A resolver function handles it:

```python
OPENAI_PATH_MODEL_PREFIXES = ("openai.gpt-5.",)

def _responses_url(region: str, model_id: str) -> str:
    if model_id.startswith(OPENAI_PATH_MODEL_PREFIXES):
        return f"https://bedrock-mantle.{region}.api.aws/openai/v1/responses"
    return f"https://bedrock-mantle.{region}.api.aws/v1/responses"
```

We tested `openai.gpt-5.6-sol` and `openai.gpt-5.6-luna` at the corrected path — both returned real, successful SSE tool-call events. `openai.gpt-oss-120b` was tested at the plain path — confirming both paths are real and model-family-specific. The `"openai.gpt-5."` prefix as written is broader than what we verified: it also silently matches `gpt-5.4` and `gpt-5.5` — which AWS's [Web Search documentation](https://docs.aws.amazon.com/bedrock/latest/userguide/web-search.html) confirms use the same `bedrock-mantle` Responses API, implying the same path, though we didn't exercise those two ourselves — and it would just as silently match any future `gpt-5.x` model AWS ships, whose path behavior is unknown until tested. Prefer an explicit allowlist of model IDs you've actually verified over this prefix if you're adapting this code.

The request body that drives server-side execution — the `mcp` tool block tells Bedrock Mantle to discover and execute tools against the Gateway itself, rather than returning a tool-call payload for the client to execute:

```python
payload = {
    "model": model_id,
    "stream": True,
    "background": False,
    "store": False,
    "instructions": system_prompt,
    "tools": [
        {
            "type": "mcp",
            "server_label": "luxconcierge_tools",
            "connector_id": gateway_arn,
            "server_description": "AgentCore Gateway providing villa/itinerary/booking tools",
            "require_approval": "never",
        }
    ],
    "input": [
        {
            "type": "message",
            "role": "user",
            "content": [{"type": "input_text", "text": user_message}],
        }
    ],
}
```

The request is signed with SigV4 using AWS credentials from the standard boto3 credential chain — no separate Bedrock API key needed, matching what [Inference using Responses API](https://docs.aws.amazon.com/bedrock/latest/userguide/bedrock-mantle.html) documents as a supported authentication method.

The mechanism, deployed for real (11 tools across a villa-booking domain — search, itinerary, and booking Lambdas behind one AgentCore Gateway — as a stand-in for any customer's own tool set), against a real query:

```
You: Find villas in Bali for 6 guests under $2000 a night.

POST https://bedrock-mantle.us-east-1.api.aws/openai/v1/responses
  model=openai.gpt-5.6-sol
  [Gateway: tool discovery complete]
  [MCP tool call: ({"destination":"Bali","guests":6,"max_price":2000})]
  [MCP tool call completed]
I found 2 Bali villas for 6 guests under $2,000/night:

- Villa Cascade (BALI-001) — $1,450/night
  - Sleeps 8, 4 bedrooms, Rating 4.9
  - Cliffside infinity pool, ocean view, spa, private chef included

- Sawah Retreat (BALI-002) — $890/night
  - Sleeps 6, 3 bedrooms, Rating 4.7
  - Ubud rice-field views, pool, rooftop yoga deck

[1 MCP tool call(s), 0 native web_search call(s) executed]
```

And a tool call with a real side effect, resolved the same way — one request, no client-side resubmission loop:

```
You: Confirm my booking with a credit card.

  [MCP tool call: ({"customer_id":"CUST-001","payment_method":"credit_card"})]
  [MCP tool call completed]
Booking confirmed.

- Booking ID: BK-20260807-24203C
- Payment: Credit card
- Status: Deposit charged
- Confirmation: Itinerary emailed

[1 MCP tool call(s), 0 native web_search call(s) executed]
```

**This ran with no human approval step.** AWS's own [server-side tool-use documentation](https://docs.aws.amazon.com/bedrock/latest/userguide/tool-use-server-side.html) currently requires `require_approval` to be set to `"never"` for the AgentCore Gateway integration — it's the only accepted value, not a choice made for this demo. That means native per-tool approval isn't available through this parameter today: a side-effecting tool like `confirm_booking` above executes as soon as the model decides to call it. If you need a human (or an external policy check) in the loop before a side-effecting tool fires, that has to come from an external approval workflow, a client-side execution path instead, or some other compensating control — not from this parameter.

`BK-20260807-24203C` is a real booking ID generated by the Lambda during this run, not a hardcoded example. CloudWatch tracing on this Gateway (enabled via `put-delivery-source`/`put-delivery-destination`/`create-delivery`, on top of the account's existing CloudWatch Transaction Search setup) shows the real trajectory a tool call takes once it reaches the Gateway: `AgentCore.Gateway.InvokeTool` → the specific tool's span → the Lambda service span → the Lambda execution-environment span, nested as one connected trace with real millisecond durations — not just inferred from the SSE event log.

![CloudWatch GenAI Observability trace waterfall for a real Bedrock AgentCore Gateway tool call, showing the nested spans: AgentCore.Gateway.InvokeTool, the Gateway's tool-invocation span, the Lambda service span, and the Lambda execution-environment span, with real millisecond durations.](/assets/images/2026-08-07-GPT-5-6-Tool-Use-Bedrock-vs-OpenAI-Comparison/agentcore-trace-server-side-timeline.png)

## Implementing Bedrock client-side function calling

The client-side counterpart uses the same model and the same Gateway's tools, but the calling process does the discover → select → execute → resubmit loop itself, using Bedrock's `type: "function"` tools instead of `type: "mcp"`:

1. Call the Gateway's MCP `tools/list` once at startup (SigV4-signed for the `bedrock-agentcore` service) and convert the MCP tool schemas into `{"type": "function"}` Responses API tools, per [AWS's client-side tooling guide](https://docs.aws.amazon.com/bedrock/latest/userguide/tool-use-client-side.html).
2. POST to Bedrock Mantle. If the model returns a `function_call` output item, sign and send a `tools/call` request straight to the Gateway (bypassing the server-side mechanism entirely for that hop), then POST the result back to Bedrock as a `function_call_output` input item, and repeat until the model stops requesting tools.

That loop is exactly the "parse tool-call output, execute the tool, serialize the result, send it back" pattern both [AWS's](https://docs.aws.amazon.com/bedrock/latest/userguide/tool-use-client-side.html) and [OpenAI's](https://platform.openai.com/docs/guides/function-calling) client-side tooling guides describe — the same shape on either provider, just pointed at a different endpoint.

Real output from the same query, run client-side instead:

```
You: Find villas in Bali for 6 guests under $2000 a night.

  [Bedrock request #1] POST https://bedrock-mantle.us-east-1.api.aws/openai/v1/responses
  [Client executes tool: VillaTools___search_villas({"destination": "Bali", "guests": 6, "max_price": 2000})]
  [Gateway tools/call completed]
  [Bedrock request #2] POST https://bedrock-mantle.us-east-1.api.aws/openai/v1/responses

LuxConcierge: Two Bali villas fit 6 guests under $2,000/night:
[... same two villas as the server-side run ...]

[1 client-executed tool call(s), 2 Bedrock request(s), 5.95s wall-clock]
```

Two requests instead of one, for the identical result. That's the mechanism difference in one transcript.

## Implementing OpenAI's server-side tool execution (from documentation, not live-tested)

OpenAI's Responses API supports a `type: "mcp"` tool that connects to a remote MCP server. Per [OpenAI's documentation](https://developers.openai.com/api/docs/guides/tools-connectors-mcp), the request shape is:

```json
{
  "model": "gpt-5.6-sol",
  "tools": [
    {
      "type": "mcp",
      "server_label": "my-tools",
      "server_url": "https://my-mcp-server.example.com/mcp",
      "require_approval": "never"
    }
  ],
  "input": "..."
}
```

OpenAI's own backend — not the client SDK — contacts that `server_url` to discover its tools (`mcp_list_tools`) and to invoke them (`mcp_call`), placing the result directly into the model's context. The client is not involved in either step; it sees one request and one final answer, the same request-count profile as Bedrock's server-side mechanism — **but only if `require_approval` is set to `"never"`, as in the example above.** Left at its default, OpenAI pauses execution before each tool call and returns an `mcp_approval_request` item for the caller to approve and resubmit, which costs an extra round trip per approval. That default is the opposite of Bedrock's current AgentCore Gateway integration, which requires `"never"` and offers no per-tool approval pause at all (see the callout above). Two more things differ from Bedrock:

- **Connectors** (`connector_id` instead of `server_url`) are OpenAI-maintained MCP wrappers for popular third-party services (Google Workspace, Dropbox, etc.) — not applicable to a custom internal tool.
- **Secure MCP Tunnel**, for a private/on-prem MCP server: a downloadable tunnel-client binary run on the customer's own network connects that server to OpenAI's products without exposing it to the public internet.

For the custom internal MCP pattern shown here, the business logic runs on **a customer-operated MCP server** — one the customer must build, deploy, and operate, not OpenAI's infrastructure. OpenAI-maintained connectors and third-party-hosted MCP servers have different hosting responsibilities than that — the customer isn't necessarily the host in those cases. OpenAI is always the orchestrator; who hosts the tool code depends on which of the three options above is in play. Client-side function calling on OpenAI (`type: "function"`) is the same shape as Bedrock's, described above — one request per sequential round of tool calls, no MCP server required.

## Combining a custom server-side tool with a provider's native tool, in one request

Bedrock ships a built-in `"type": "web_search"` tool on the same Responses API that needs no Gateway, Lambda, or `connector_id`. Per AWS's [Web Search documentation](https://docs.aws.amazon.com/bedrock/latest/userguide/web-search.html), it's "hosted and built by AWS," served by default from "the Amazon Bedrock web index and cache" so "your data stays within the AWS boundary" — a genuinely separate mechanism from the AgentCore-Gateway-backed MCP tools above, currently supported on `openai.gpt-5.4`, `openai.gpt-5.5`, and `openai.gpt-5.6` (luna, terra, sol). Both tool types can be declared in the same request:

```python
tools = [
    {"type": "web_search", "external_web_access": False},
    {
        "type": "mcp",
        "server_label": "luxconcierge_tools",
        "connector_id": gateway_arn,
        "server_description": "AgentCore Gateway providing villa/itinerary/booking tools",
        "require_approval": "never",
    },
]
```

Per that same documentation, `external_web_access` defaults to `true` (matching the OpenAI Responses API), but reaching the external web at `true` also requires an IAM identity holding the separate `bedrock-websearch:ExternalWebAccess` permission — the common `AmazonBedrockFullAccess` policy doesn't grant it, so a default-`true` request from an identity without it gets a `403` on that specific check and falls back to AWS's own indexed cache. This demo sets `external_web_access` to `False` explicitly, keeping retrieval inside the AWS boundary without needing that extra permission at all. Leave it at `true` (with `ExternalWebAccess` granted) if you need live external fetches; `false` if you don't.

Tested live, with a query designed to need both in one turn:

```
You: Search the web for any current travel advisories for Bali. Then
separately use your villa tools to find villas there for 6 guests under
$2000/night. Do both.

  [Native web_search: searching]
  [Native web_search: completed]
  [MCP tool call: ({"destination":"Bali","guests":6,"max_price":2000})]
  [MCP tool call completed]
  [Native web_search: searching]
  [Native web_search: completed]
  [Native web_search: searching]
  [Native web_search: completed]

## Current Bali travel advisories — August 7, 2026
- U.S. advisory: Exercise increased caution across Indonesia due to
  terrorism, civil unrest, and natural disasters. Bali is not within the
  Level 4 "Do Not Travel" areas, which are in Central and Highland Papua...
- UK advisory: No specific warning against travel to Bali. Exclusion zones
  currently apply around several volcanoes elsewhere in Indonesia...

## Villas for 6 guests under $2,000/night
| Villa | Nightly rate | Rating |
|---|---:|---:|
| Villa Cascade (BALI-001) | $1,450 | 4.9 |
| Sawah Retreat (BALI-002) | $890 | 4.7 |

[1 MCP tool call(s), 3 native web_search call(s), 1 Bedrock request, 18.97s wall-clock]
```

The full response text summarizes what `web_search` returned rather than quoting its source pages verbatim; the abbreviated excerpt above doesn't itself show source URLs, so treat it as a summary of grounded content rather than a citation trail — check AWS's `web_search` documentation for how (and whether) it surfaces source links in the raw tool output.

One signed request, four total tool calls across two independent mechanisms, still exactly one request from the client. The client-side demo used for comparison in this post did not include a web-search integration, so this specific query wasn't run through it — this is a gap in what this particular demo tested, not a claim that client-side function calling can't call a web-search tool at all (it can, on either provider, wired up like any other tool).

## Measured comparison: request count and wall-clock

To move the request-count claim from architecture diagram to number, we ran Bedrock's server-side and client-side scripts back-to-back, same model, same Gateway, same tools, same queries. **This measures Bedrock against itself** — client-side function calling as a stand-in for the generic client-side pattern — not Bedrock against OpenAI's actual API, which we did not live-test.

Five matched, single-tool-call queries:

| | Server-side | Client-side |
|---|---:|---:|
| Requests per query | 1 (always) | 2 (1 initial + 1 resubmission, since every query in this run needed exactly one sequential round of tool calls) |
| Total requests, 5 queries | 5 | 10 |
| Total wall-clock, 5 queries | 30.48s | 53.84s |
| Average wall-clock per query | 6.10s | 10.77s |
| Orchestration code the caller must write | No runtime tool loop — one `mcp` tool block, though the Gateway/Lambda setup behind it still has to be built and operated | Tool discovery, schema conversion, response parsing, Gateway calling, result resubmission |

Two things generalize differently from this table:

- **The request-count ratio is structural**, not a property of this run: server-side execution is architecturally 1 request regardless of tool-call count — provided no approval step or other client intervention interrupts it (see the `require_approval` note above); client-side is architecturally `1 request + 1 request per sequential round of tool calls` — parallel tool calls within a round are batched into a single resubmission, so this only grows with genuinely dependent rounds, not with raw tool-call count. This holds on OpenAI's client-vs-server choice too, by the same logic — both providers implement the same `type: "function"` shape.
- **The wall-clock seconds are this run's**, not a benchmark: one sandbox account, one region, five queries, one point in time, not repeated across trials. Treat 30.48s vs. 53.84s as directionally consistent with the request-count story, not as a number to plug into your own capacity planning.

The turn that needed four tool calls across two mechanisms (the combined web-search-plus-villa-search query above) was still one client-initiated server-side request. We did not run the equivalent client-side case, so its request count can't be inferred from the four server-side calls — the transcript shows the model calling web_search, then the MCP tool, then web_search twice more, and there's no way to tell from a server-side trace alone how many of those the model would batch into one round versus spread across sequential rounds if it were driving a client-side loop. It would be 2 client-side requests if the model emitted all of them in one round; more if later calls depended on earlier results. Treat the request-count formula as the generalizable claim here, not this specific query's request count.

## Evaluation checklist

Before treating any of the above as relevant to your own workload:

- **Tool calls per turn.** The request-count ratio widens with more *sequential rounds* of tool calls per turn, not with raw tool-call count — parallel calls in one round still cost one resubmission. If your typical turn needs 3–4 dependent rounds (each only knowable after the previous result), that's 4–5 client-side requests against server-side's still-1; if those same tool calls can all be issued in parallel in one round, it's still just 2.
- **Existing tool-calling code.** Moving from client-side function calling to server-side execution means replacing that orchestration code — on Bedrock, with an AgentCore Gateway plus one or more Lambda functions (or other supported Gateway target — this demo used 3 Lambdas for 11 tools, not one per tool); on OpenAI, with a remote MCP server you build and operate (or tunnel from on-prem). Neither is a drop-in config change.
- **Portability.** Client-side function-calling code is the more portable option across providers, since both implement the same shape. Bedrock's server-side pattern is AWS-specific infrastructure (AgentCore Gateway); OpenAI's server-side pattern is a self-hosted MCP server — portable in principle across any MCP-compatible provider, but yours to build and run.
- **Latency sensitivity.** The round-trip tax client-side calling adds may not matter for batch/async work; it's more likely to matter for a synchronous, user-facing flow. Verify against your own network path and query shapes.

## References

- [Server-side tool use — Amazon Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/tool-use-server-side.html)
- [Client-side tool use — Amazon Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/tool-use-client-side.html)
- [Remote MCP servers and connectors — OpenAI](https://developers.openai.com/api/docs/guides/tools-connectors-mcp)
- [Secure MCP Tunnels — OpenAI](https://developers.openai.com/api/docs/guides/secure-mcp-tunnels)
- [Function calling — OpenAI](https://platform.openai.com/docs/guides/function-calling)
- [Web search — Amazon Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/web-search.html)
- [Inference using Responses API — Amazon Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/bedrock-mantle.html)
- [Use an AgentCore gateway (MCP tools/list, tools/call) — Amazon Bedrock AgentCore](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway-using.html)
- [GPT-5.6 Sol model card — Amazon Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/model-card-openai-gpt-56-sol.html)
- [AWS sample: sample-bedrock-server-side-tool-call (ShopAssist)](https://github.com/aws-samples/sample-bedrock-server-side-tool-call)
