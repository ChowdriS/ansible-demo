---
title: "Per-User RBAC in an Azure AI Foundry Multi-Agent System: What Actually Works"
author: Chowdri S
date: 2026-08-10
tags: [azure-ai-foundry, a2a-protocol, rbac, multi-agent, agent-framework]
summary: >
  We set out to restrict which users can reach which specialist agents — and
  which tools inside them — in an orchestrator + sub-agent system built on
  Azure AI Foundry's A2A protocol. Along the way we hit two dead ends that
  are silently by-design platform limitations, not bugs in our code, and
  landed on an architecture that actually enforces per-user access control
  end to end. This is the full story, with the code and the receipts.
---

# Per-User RBAC in an Azure AI Foundry Multi-Agent System: What Actually Works

| # | Question | Answer |
|---|---|---|
| 1 | Can an orchestrator restrict which *sub-agent* a user can reach? | **Yes** — agent-level RBAC, working in production |
| 2 | Can a sub-agent restrict which *tools* a caller can use inside it? | **Yes** — tool-level RBAC, working in production |
| 3 | Does the caller's identity survive a real A2A protocol call to the sub-agent? | **No** — confirmed platform limitation, not a bug we introduced |
| 4 | So how does tool-level RBAC work at all? | The orchestrator resolves the caller's role itself (Azure RBAC is the source of truth), then hands the sub-agent a signed, short-lived claim over a **direct, non-A2A** call |

---

## 1. What We Set Out to Build

An orchestrator agent that takes a user's question and routes it to the
right specialist sub-agent over the A2A (Agent-to-Agent) protocol — plus
**per-user access control** on top: not every user should reach every
sub-agent, and not every user who *can* reach one should get every
capability inside it. We wanted that restriction driven by the user's
**actual Azure RBAC role**, not a hand-maintained permission list living
in application code.

Test system: a travel assistant with two specialists —
`tamilDestinations` (temples, beaches, hill stations — open to everyone)
and `tamilFoodCulture` (cuisine, festivals, culture — treated as
restricted/premium). Goal: users without access to the food/culture
project shouldn't get answers from it, even indirectly through the
orchestrator — and among users who *can* reach it, only some should be
allowed to *write* new content, not just read.

---

## 2. Tech Stack

| Layer | What we used |
|---|---|
| Agent hosting | **Azure AI Foundry Agent Service** — hosted agents (Docker containers behind a Foundry-managed endpoint) |
| Agent SDK | **Microsoft Agent Framework** (`agent_framework`) — `Agent`, `FoundryChatClient`, tools/skills |
| Server runtime | **`azure-ai-agentserver-responses` / `-core`** — `ResponsesAgentServerHost`, `ResponseContext`, `get_request_context()` |
| Agent-to-agent | **A2A protocol** — `a2a-sdk`, `agent_framework.a2a.A2AAgent`, `A2ACardResolver` |
| Access control | **Azure RBAC / ARM `roleAssignments` REST API** — the single source of truth for every decision |
| Build & deploy | Docker + Azure Container Registry, `az` CLI + Foundry Agents REST API (`POST /agents`, `PATCH /agents/{name}`) |
| Language | Python 3.11 |

**Reference:** [Deploy a Hosted Agent — Microsoft Learn](https://learn.microsoft.com/en-us/azure/foundry/agents/how-to/deploy-hosted-agent)

---

## 3. A Quick Primer: Agent Framework + A2A

`agent_framework`'s core idea: an `Agent` wraps a chat client
(`FoundryChatClient`, pointed at a Foundry model deployment) plus a list of
Python functions as "tools." Separately, `agent_framework.a2a` lets you
wrap *another agent's* A2A endpoint so it looks like just another tool —
`A2AAgent(...).as_tool(...)` — so an orchestrator's LLM calls a sub-agent
exactly like it calls a local function.

Foundry hosts each agent as its own container behind a managed endpoint
that can expose several protocols side by side: **`responses`** (plain
request/response, every hosted agent supports it) and, optionally,
**`a2a`** (discovery/invocation via a standard Agent Card — served at
Foundry's own `/agentCard/v1.0` path rather than the A2A spec's usual
`/.well-known/agent.json`).

**References:**
[A2A Tool Guide — Microsoft Learn](https://learn.microsoft.com/en-us/azure/foundry/agents/how-to/tools/agent-to-agent) ·
[Enable A2A Endpoint — Microsoft Learn](https://learn.microsoft.com/en-us/azure/foundry/agents/how-to/enable-agent-to-agent-endpoint) ·
[A2A Protocol Specification](https://a2a-protocol.org/latest/) ·
[agent-framework repo](https://github.com/microsoft/agent-framework) ·
[a2a-sdk repo](https://github.com/google-a2a/a2a-python)

---

## 4. First Attempt: All Hosted Agents, Sub-agents Registered as A2A Tools

The initial architecture: deploy all three agents (orchestrator + 2
specialists) as Foundry Hosted Agents, have the orchestrator discover each
specialist's Agent Card, and register it as a callable tool invoked over
the real A2A endpoint.

The two documented ways to do this both turned out to be broken:

```python
# Method 1 - broken
from azure.ai.projects.models import A2APreviewTool
tool = A2APreviewTool(project_connection_id=connection.id)

# Method 2 - broken, same underlying bug
from agent_framework.foundry import FoundryChatClient
tool = FoundryChatClient.get_a2a_tool(project_connection_id=conn_id)
```

Both fail with:
```json
{
  "message": "Agent card path must target the same host, project, and agent as the server URL.",
  "type": "invalid_request_error",
  "code": "tool_user_error"
}
```

**Reference:** [GitHub Azure/azure-sdk-for-python#47419](https://github.com/Azure/azure-sdk-for-python/issues/47419) — open, labeled `needs-team-attention`, unresolved as of this writing.

**What actually worked** — the lower-level `A2AAgent` class with a manual
auth interceptor. From `a2a/orchestrator/main.py:283-314`
(`build_a2a_agent()`):

```python
async def build_a2a_agent(name, url, description, http_client):
    resolver = A2ACardResolver(
        httpx_client=http_client,
        base_url=url,
        agent_card_path="agentCard/v1.0",   # Foundry-specific, not .well-known
    )
    agent_card = await resolver.get_agent_card()
    a2a_agent = A2AAgent(
        name=agent_card.name,
        description=agent_card.description or description,
        agent_card=agent_card,
        url=url,
        http_client=http_client,   # pre-authenticated with a bearer token
    )
    await a2a_agent.__aenter__()
    return a2a_agent
```

This got real A2A communication working end to end — orchestrator to both
specialists, LLM correctly routing by topic.

---

## 5. What We Found: Identity Wasn't Flowing Through At All

Getting A2A *connectivity* working just exposed the actual problem. The
`http_client` above is authenticated with the **orchestrator's own managed
identity token**, never the real end user's. So:

- Call the restricted specialist **directly** with a user's own token →
  Azure RBAC applies correctly, unauthorized users get `403`.
- Call the *same* specialist **through the orchestrator** → succeeds
  regardless of the real user's access, because only the orchestrator's
  own (broadly-privileged) identity is ever checked.

We proved this with a VM whose managed identity had access to the public
specialist's project but deliberately **not** the restricted one:

| Test | Result |
|---|---|
| Direct call to `tamilDestinations` (public) | ✅ PASS |
| Direct call to `tamilFoodCulture` (restricted) | ❌ 403 DENIED — correct |
| Same question, through the orchestrator | ✅ succeeded anyway |

We also inspected `ResponseContext` directly for anything resembling the
caller's identity:

```
context_attrs: ['client_headers', 'conversation_id', 'created_at',
                'get_history', 'get_input_items', 'get_input_text',
                'is_shutdown_requested', 'mode_flags', 'platform_context',
                'query_parameters', 'request', 'response_id']
context.isolation           -> does not exist
context.isolation.user_key  -> does not exist
context.request.headers     -> Authorization already stripped
```

**References:**
[GitHub #45797 — "Hosted Agent removes Authorization Header disabling OBO"](https://github.com/Azure/azure-sdk-for-python/issues/45797) (open) ·
[Microsoft Q&A: User Identity Passthrough for Hosted Agents](https://learn.microsoft.com/en-us/answers/questions/5872669/user-identity-passthrough-for-hosted-agents-callin) — official confirmation: *"For code-first Hosted Agents, OAuth flows are handled externally by the service layer... user tokens are not exposed within container runtime."*

---

## 6. The Breakthrough: Agent-Level RBAC via a Prehook

While chasing the above, we found Foundry **does** inject one genuinely
trustworthy piece of per-request identity: an `x-agent-user-id` header,
surfaced in code as `get_request_context().user_id`. Verified empirically
to be the caller's real Entra Object ID — available on a direct call to
the orchestrator's own Responses endpoint (the first hop: user →
orchestrator).

That gave us a design for **agent-level** RBAC — can this caller reach a
given sub-agent at all — as a "prehook" the orchestrator runs *before* it
builds the LLM's tool list. From `a2a/orchestrator/main.py:104-142`:

```python
async def check_access(credential, principal_id: str | None, project_name: str) -> bool:
    if not principal_id:
        return False  # fail closed - no identity, no access
    token = credential.get_token("https://management.azure.com/.default").token
    headers = {"Authorization": f"Bearer {token}"}
    async with httpx.AsyncClient(timeout=httpx.Timeout(30.0)) as client:
        for scope in _scope_chain(project_name):  # project -> account -> subscription
            resp = await client.get(
                f"{ARM_BASE}{scope}/providers/Microsoft.Authorization/roleAssignments",
                headers=headers,
                params={"api-version": "2022-04-01", "$filter": f"principalId eq '{principal_id}'"},
            )
            resp.raise_for_status()
            for a in resp.json().get("value", []):
                if a["properties"]["scope"].lower() == scope.lower():
                    return True
    return False
```

**Reference:** [ARM Role Assignments — List REST API](https://learn.microsoft.com/en-us/rest/api/authorization/role-assignments/list-for-scope)

Wired into `build_agent_for_caller()` (`a2a/orchestrator/main.py:321-376`):
allowed callers get a real tool that fetches the agent card and makes the
real A2A call; everyone else gets a **dummy tool** — same name and
description — that returns `ACCESS_DENIED: ...` immediately, with **no
network call at all** (`create_dummy_tool()`, `main.py:207-217`). The
system prompt instructs the LLM to relay that denial plainly and never
fabricate an answer for it.

> **A real bug we hit and fixed:** Azure's role-assignment *list* API,
> queried at scope `S`, returns matches **at and below** `S` by default —
> not only exact matches at `S`. Walking *up* the ancestor chain
> (project → account → subscription) to catch broader inherited grants
> means a naive "any result = allowed" check can be fooled by a grant that
> only exists on a **sibling** project under the same account. Fix: only
> count a match whose own `properties.scope` **exactly equals** the scope
> you queried — visible in the exact-match comparison above.

This alone gives working, verified agent-level RBAC: a caller either can
or cannot reach a given sub-agent, enforced before any sub-agent code runs.

---

## 7. The Next Ambition: Tool-Level RBAC, and the Wall We Hit

Agent-level access is binary. We wanted finer granularity: *within* one
sub-agent, some callers get read-only tools, others get read+write. That
means getting the caller's identity — or a resolved permission — **into
the sub-agent's own container code**: a second network hop
(orchestrator → sub-agent), a fundamentally different problem than the
first hop that `x-agent-user-id` already solved for free.

We tested it directly: called the orchestrator as ourselves, had it call a
specialist over real A2A using the orchestrator's own separate managed
identity, and inspected what the specialist's container actually saw.
**Result: only ever the orchestrator's own identity arrived as the caller
— never the real end user's.** Debug tooling used to confirm this, still
present in the code today (`a2a/specialist1/main.py:69-106`):

```python
if user_input.strip().lower() == "checkheaders":
    return TextResponse(context, request,
        text=f"client_headers={context.client_headers!r} "
             f"x-client-user-id={context.client_headers.get('x-client-user-id')!r}")

if user_input.strip().lower() == "checkbaggage":
    return TextResponse(context, request, text=f"baggage={dict(otel_baggage.get_all())!r}")

if user_input.strip().lower() == "checkrawheaders":
    return TextResponse(context, request, text=f"raw_headers={_raw_headers_var.get()!r}")
```

Same conclusion via the orchestrator's own `checkbaggage` debug command
(`a2a/orchestrator/main.py:401-422`), which calls a specialist over A2A and
reports both the real caller's `user_id` and what the specialist says it
received — they never matched.

---

## 8. Every Channel We Tried to Get Identity Across the A2A Hop

All tested empirically against live deployed Foundry hosted agents, not
simulated:

| Channel tried | Result | Detail & reference |
|---|---|---|
| Custom `x-client-*` HTTP header | ❌ Arrives empty | Client SDK confirmed to put it on the wire correctly (source-read of `agent_framework_a2a`/`a2a-sdk`) — the loss is server-side |
| OpenTelemetry `baggage` header | ⚠️ Survives, but gets replaced | Our own `caller-user-id` value never arrived; only Foundry's own keys did — see [W3C Baggage spec](https://www.w3.org/TR/baggage/) for how the mechanism is supposed to work |
| A2A `Message.metadata` | ❌ Overwritten entirely | Sent `{"caller_user_id": ...}`, got back Foundry's own `{'protocol': 'a2a', 'a2a_context_id': ..., ...}` |
| A2A structured `Part.data` | ❌ Rejected outright | `ContentTypeNotSupportedError` — agent card only declares a `text` input mode. The A2A spec's own repo tracks this exact gap: [a2aproject/A2A#19 — "Delegated User Authorization for Agent2Agent Servers"](https://github.com/a2aproject/A2A/issues/19), open, unresolved |
| `context.isolation` / `.user_key` | ❌ Doesn't exist | Confirmed via direct attribute inspection |
| Plain text message content | ✅ The only thing that reliably survives | Everything that works in this system routes through this |

**Root cause, confirmed by capturing raw ASGI headers** (a temporary debug
middleware dumping every header the container process actually receives —
`a2a/specialist1/main.py:22-38`, `RawHeaderCaptureMiddleware`): there are
**two separate strip points**, not one.

1. An **ingress reverse proxy** allowlists a fixed set of Foundry's own
   headers plus anything prefixed `x-client-`. Anything else is dropped
   before the container sees it — true for direct calls and A2A calls
   alike.
2. An **A2A-specific internal step**: even `x-client-*` headers, which
   survive step 1 on both paths, still arrive empty specifically on the
   A2A path. Foundry's A2A-to-Responses bridge reconstructs a brand-new
   internal request to actually invoke the sub-agent's handler, and that
   reconstruction doesn't carry custom headers forward.

We checked whether a newer SDK version changes this (installed the newest
available alpha at the time, `agent-framework-foundry-hosting`
1.0.0a260630, and read its source) — built on the identical underlying
primitives, no change.

**Corroborating references:**
[Agent Identity Concepts — Microsoft Learn](https://learn.microsoft.com/en-us/azure/foundry/agents/concepts/agent-identity) (defines Attended/OBO vs. Unattended modes — hosted agents run Unattended) ·
[Securing a Multi-Agent AI Solution: the Complexities of OBO — Microsoft Tech Community](https://techcommunity.microsoft.com/blog/azurearchitectureblog/securing-a-multi-agent-ai-solution-focused-on-user-context--the-complexities-of-/4493308) ·
["Authorization Propagation in Multi-Agent AI Systems," arXiv:2605.05440](https://arxiv.org/pdf/2605.05440) — *"RBAC cannot express 'this agent may access this dataset only when acting on behalf of a specific user within a specific workflow.'"*

---

## 9. The Working Solution: Skip A2A for This Hop

A2A's Agent Card is still fine for *discovery* — name, description,
skills. But for the actual tool invocation on the restricted specialist,
we call its **Responses-protocol endpoint directly** instead. A custom
header sent straight to that endpoint arrives intact — no bridge, no
reconstruction.

Concretely: `tamilFoodCulture` skips A2A discovery and invocation
entirely — its project name, agent name, and Responses URL are hardcoded
Python constants in the orchestrator, since we already know them
statically (`tamilDestinations` still goes through real A2A discovery,
since it doesn't need tool-level RBAC).

### 9.1 Resolving *which* role the caller has

`a2a/orchestrator/main.py:163-204`, `get_effective_role()`:

```python
_BUILTIN_ROLE_GUIDS = {
    "acdd72a7-3385-48ef-bd42-f606fba81ae7": "Reader",
    "b24988ac-6180-42a0-ab88-20f7382dd24c": "Contributor",
}
ROLE_PRECEDENCE = ["Contributor", "Reader"]
```

**Reference:** [Azure built-in roles — Reader](https://learn.microsoft.com/en-us/azure/role-based-access-control/built-in-roles/general#reader) · [Contributor](https://learn.microsoft.com/en-us/azure/role-based-access-control/built-in-roles/general#contributor) — GUIDs are fixed and identical in every Azure tenant.

> **A bug we hit here too:** we originally resolved the human-readable role
> name via a live `GET .../roleDefinitions/{guid}` call. It silently
> failed and always returned `None` — role *definitions* are a
> subscription-level resource type, not nested under the Cognitive
> Services account, so the orchestrator's account-scoped `Reader` grant
> (enough for reading role *assignments*) doesn't cover reading role
> *definitions*. Fix: skip the live lookup and use the hardcoded GUID map.

### 9.2 Delivering the role to the sub-agent, securely

Not as a plain header — anyone reaching the sub-agent's Responses endpoint
directly could forge that and grant themselves write access — but as an
**HMAC-signed, short-lived claim** (`a2a/orchestrator/role_token.py:31-47`,
`sign_role_claim()`):

```python
def sign_role_claim(role, principal_id, ttl_seconds=60) -> str:
    if not _SECRET:
        return ""  # fail closed - unsigned claims are never trusted
    payload = {"role": role or "none", "principal_id": principal_id or "", "exp": int(time.time()) + ttl_seconds}
    payload_b64 = _b64url_encode(json.dumps(payload, separators=(",", ":")).encode("utf-8"))
    sig = hmac.new(_SECRET.encode("utf-8"), payload_b64.encode("ascii"), hashlib.sha256).hexdigest()
    return f"{payload_b64}.{sig}"
```

**Reference:** [RFC 2104 — HMAC: Keyed-Hashing for Message Authentication](https://www.rfc-editor.org/rfc/rfc2104)

Sent as `x-client-caller-role-token` on a direct POST to the sub-agent's
Responses URL (`a2a/orchestrator/main.py:244-260`,
`call_food_culture_direct()`).

### 9.3 Sub-agent side: verify, then build a scoped tool list

`a2a/specialist2/main.py`:

```python
def get_allowed_tool_names(role: str) -> list[str]:
    if role == "Contributor":
        return ["read_food_info", "write_food_review"]
    return ["read_food_info"]          # unknown/missing role fails closed

@app.response_handler
async def handler(request, context, _cancellation_signal):
    role_token = context.client_headers.get("x-client-caller-role-token")
    role, signed_principal_id = verify_role_claim(role_token)   # role_token.py
    ...
    tools = [_ALL_TOOLS[name] for name in get_allowed_tool_names(role)]
    agent = Agent(client=client, name="TamilFoodCultureSpecialist", instructions=INSTRUCTIONS, tools=tools)
```

For a Reader (or forged/missing-token) caller, `write_food_review` is
**genuinely absent** from the LLM's tool list — not exposed and then
blocked internally.

**Verified end to end:** granted a test principal `Contributor` on the
restricted project; the orchestrator's `checkrole` debug command confirmed
`role='Contributor'`; a real request ("tell me about jigarthanda, and
submit a review saying it's delicious") correctly used both
`read_food_info` and `write_food_review` in the same turn.

---

## 10. Inference From This POC

- **Agent-level RBAC** is fully solved and running: real caller identity
  via `x-agent-user-id` on the first hop, checked against live Azure RBAC
  role assignments, enforced by registering dummy vs. real tools before
  the LLM ever runs.
- **Tool-level RBAC** is also solved, but required abandoning A2A as the
  transport for that specific hop — identity/permission data doesn't
  survive Foundry's A2A bridge through any channel we could find, and this
  is a platform-wide, by-design limitation confirmed via multiple official
  Microsoft Q&A threads, not something specific to our setup.
- **What actually carries the permission decision**: the orchestrator
  resolves it once, up front, using Azure RBAC as the single source of
  truth, then hands it to the sub-agent as a signed, short-TTL claim over
  a direct (non-A2A) call. The sub-agent never has to know anything about
  RBAC itself — it only verifies a signature and builds its tool list.
- **This is self-enforced authorization, not true delegated
  authorization** — the orchestrator still does the real work using its
  own identity; it just refuses to act unless it has separately verified
  the real caller is allowed to ask. Good enough to gate access correctly;
  not the same security model as a sub-agent independently validating a
  real per-user token.

---

## 11. Known Limitations

- **Role scope conflation** — `get_effective_role()` resolves against the
  restricted project's own scope, so a `Contributor` grant given purely
  for infra-management reasons on that project would *also* unlock the
  write tool. Cleaner fix (not yet implemented): resolve against a
  separate, otherwise-unused "marker" resource instead.
- **Direct role assignments only** — a caller whose access comes solely
  from an Entra group membership isn't detected by either tier's check.
- **No revocation propagation** — the per-caller tool set is cached for
  the life of the container process; a mid-session access revocation
  won't be reflected until a new process picks up the request.
- **A caller who reaches the sub-agent's Responses endpoint directly**,
  bypassing the orchestrator, with no valid signed role token, fails
  closed to read-only.
- **A two-static-instance alternative was considered and validated as
  legitimate, but not adopted**: deploy two copies of the same image with
  a different `AGENT_ROLE` environment variable per deployment, orchestrator
  picks which URL to call. Officially supported Foundry pattern; sidesteps
  the whole "does data survive the hop" problem, at the cost of doubling
  compute per tier.

---

## Appendix A — Required Azure RBAC Grants

**Orchestrator's own managed identity:**
```bash
# Data-plane access to every project it may proxy to
az role assignment create --role "Cognitive Services User" \
  --assignee-object-id "<orchestrator-principal-id>" --assignee-principal-type ServicePrincipal \
  --scope "/subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.CognitiveServices/accounts/<account>/projects/<ProjectName>"

# Read access to list OTHER users' role assignments (needed for the prehook)
az role assignment create --role "Reader" \
  --assignee-object-id "<orchestrator-principal-id>" --assignee-principal-type ServicePrincipal \
  --scope "/subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.CognitiveServices/accounts/<account>"
```

**Each real end user**, on the restricted sub-agent's project:
- Tier 1 (can reach it at all): any role assignment there.
- Tier 2 (which tools): specifically `Reader` or `Contributor`.

```bash
az role assignment create --role "Reader" \
  --assignee-object-id "<user-object-id>" --assignee-principal-type User \
  --scope "/subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.CognitiveServices/accounts/<account>/projects/<SubAgentProject>"
```

---

## Appendix B — Verification Commands

```bash
TOKEN=$(az account get-access-token --resource https://ai.azure.com --query accessToken -o tsv)
BASE="https://<account>.services.ai.azure.com/api/projects/<AgentFrameworkProject>"

# 1. Confirm the orchestrator sees the real caller identity
curl -s -X POST "$BASE/agents/travelOrchestrator/endpoint/protocols/openai/responses?api-version=v1" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"input": "whoami", "store": false}'

# 2. Tier 1 - agent-level reachability
curl -s -X POST "$BASE/agents/travelOrchestrator/endpoint/protocols/openai/responses?api-version=v1" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"input": "checkaccess", "store": false}'

# 3. Tier 2 - which role, bypassing the LLM entirely
curl -s -X POST "$BASE/agents/travelOrchestrator/endpoint/protocols/openai/responses?api-version=v1" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"input": "checkrole", "store": false}'

# 4. Does identity survive A2A to a sub-agent's own container?
curl -s -X POST "$BASE/agents/travelOrchestrator/endpoint/protocols/openai/responses?api-version=v1" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"input": "checkbaggage", "store": false}'
```

---

## References

**Microsoft Learn**
1. [Deploy a Hosted Agent](https://learn.microsoft.com/en-us/azure/foundry/agents/how-to/deploy-hosted-agent)
2. [Enable A2A Endpoint](https://learn.microsoft.com/en-us/azure/foundry/agents/how-to/enable-agent-to-agent-endpoint)
3. [A2A Tool Guide](https://learn.microsoft.com/en-us/azure/foundry/agents/how-to/tools/agent-to-agent)
4. [Agent Identity Concepts](https://learn.microsoft.com/en-us/azure/foundry/agents/concepts/agent-identity)
5. [MCP Authentication](https://learn.microsoft.com/en-us/azure/foundry/agents/how-to/mcp-authentication)
6. [ARM Role Assignments — List REST API](https://learn.microsoft.com/en-us/rest/api/authorization/role-assignments/list-for-scope)
7. [Azure built-in roles reference](https://learn.microsoft.com/en-us/azure/role-based-access-control/built-in-roles/general)

**Microsoft Q&A**
8. [User Identity Passthrough for Hosted Agents Calling a Custom MCP Server](https://learn.microsoft.com/en-us/answers/questions/5872669/user-identity-passthrough-for-hosted-agents-callin)

**GitHub Issues**
9. [Azure/azure-sdk-for-python#47419 — A2APreviewTool "Agent card path" error](https://github.com/Azure/azure-sdk-for-python/issues/47419) (open)
10. [Azure/azure-sdk-for-python#45797 — Hosted Agent removes Authorization Header disabling OBO](https://github.com/Azure/azure-sdk-for-python/issues/45797) (open)
11. [Azure/azure-sdk-for-python#47474 — Error messages masked as "internal server error"](https://github.com/Azure/azure-sdk-for-python/issues/47474)
12. [a2aproject/A2A#19 — Delegated User Authorization for Agent2Agent Servers](https://github.com/a2aproject/A2A/issues/19) (open, protocol-level gap)

**SDK / Protocol**
13. [agent-framework repository](https://github.com/microsoft/agent-framework)
14. [a2a-sdk (Python) repository](https://github.com/google-a2a/a2a-python)
15. [A2A Protocol Specification](https://a2a-protocol.org/latest/)
16. [W3C Baggage specification](https://www.w3.org/TR/baggage/)
17. [RFC 2104 — HMAC](https://www.rfc-editor.org/rfc/rfc2104)

**Industry / Academic**
18. ["Authorization Propagation in Multi-Agent AI Systems," arXiv:2605.05440](https://arxiv.org/pdf/2605.05440)
19. [Securing a Multi-Agent AI Solution: the Complexities of OBO — Microsoft Tech Community](https://techcommunity.microsoft.com/blog/azurearchitectureblog/securing-a-multi-agent-ai-solution-focused-on-user-context--the-complexities-of-/4493308)

**Internal (this repo, code cited throughout)**
20. `a2a/orchestrator/main.py`, `a2a/orchestrator/role_token.py`
21. `a2a/specialist1/main.py`, `a2a/specialist2/main.py`, `a2a/specialist2/role_token.py`

> Internal source code isn't yet pushed to a Git remote — citations above
> are local paths; swap for GitLab links once pushed.
