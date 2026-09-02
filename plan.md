# Demo Plan: OAuth Passthrough + Per-User Restriction in Azure AI Foundry (and Across to AWS)

## 1. The use case / why this matters

This is the original aim of the whole A2A project: not just "can a sub-agent
find out who's really asking" (OAuth identity passthrough — already proven),
but **"can we actually restrict what a sub-agent and its MCP tools will do,
per real signed-in user."** Identity passthrough is the plumbing; per-user
restriction is the point.

We are preparing a **live demo session** to show both halves working
together, in two parallel cloud scenarios, so the pattern reads as general
(not a Foundry-only trick or an AWS-only trick).

## 2. The two demo cases

- **Case 1 — all-Foundry**: orchestrator (Foundry Prompt Agent) + sub-agents
  + MCP tools all live inside Azure AI Foundry.
- **Case 2 — cross-cloud**: orchestrator still lives in Foundry, but the
  sub-agent + its MCP tools live in AWS Bedrock AgentCore, with Amazon
  Cognito as the shared identity provider.

**Hard constraint, both cases**: the orchestrator must be a Foundry **Prompt
Agent**, never a Hosted Agent. The OAuth consent flow (the "Open consent" /
sign-in-then-retry dance) is a Prompt Agent tool-invocation mechanism only —
there is no equivalent for a Hosted-Agent-as-orchestrator. This was hit
directly in earlier testing, not assumed.

## 3. What's already proven (before this restriction work)

- **Case 1 identity passthrough**: confirmed working — real caller's Entra
  Object ID arrives at a Foundry Hosted Agent sub-agent via the
  `x-client-user-id` header (see §8 note — this is the header actually
  observed in code, not `x-agent-user-id` as earlier docs assumed; needs one
  live reconfirmation, see Open Decisions).
- **Case 2 identity passthrough**: confirmed working end-to-end — Foundry
  orchestrator → Cognito consent → AWS Bedrock AgentCore-hosted GitLab agent,
  real response returned, cross-cloud.
- Full step-by-step setup for both is documented in
  `a2a/a2a_withrbac/native_obo_connection_poc/A2A_OAUTH_PASSTHROUGH_SUCCESS.md`
  (Part 0: which connection tab/auth type to use; Part 1: Foundry-to-Foundry
  setup; Part 2: Foundry-to-AgentCore setup; a "Verifying It Works" section;
  a references table).

## 4. The actual gap this plan closes

Identity passthrough ≠ restriction. Everything proven so far shows the real
user's identity *arrives*. Nothing yet *uses* that identity to allow or deny
anything. Two tiers of restriction, both currently unbuilt:

- **Tier 1** — can this user reach this specific sub-agent at all (e.g. User
  A can reach Subagent1 but not Subagent2).
- **Tier 2** — once reachable, which specific tools/MCP capabilities can this
  user use inside that sub-agent (e.g. read vs. write).

## 5. Key research findings that shape the design

### Azure AI Foundry — Tier 1 is natively supported at agent granularity
Microsoft Foundry supports RBAC role assignment scoped to a **single agent**,
not just the whole project — a real, documented "agent scope":
```
/subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.CognitiveServices/accounts/<account>/projects/<project>/agents/<agentName>
```
Granting `Foundry Agent Consumer` at this scope means a user can invoke
*only* that one agent's endpoint — genuinely per-(user, agent), enforced by
Azure RBAC itself, no custom code needed. (Portal UI doesn't support this yet
— CLI/REST only. Caveat: agent-scope only governs endpoint invocation, not
broader control-plane/management permissions.)
Source: Microsoft Learn `rbac-foundry.md` ("Agent-scope role assignments")
and `enable-agent-to-agent-endpoint.md`.

There is **no** built-in Foundry mechanism for per-user tool-level (MCP tool)
restriction inside one agent — `allowed_tools` on an MCP connection is
static and agent-wide, same list for every caller. Tier 2 must be our own
code, reading the passed-through identity.

### AWS AgentCore — Tier 1 is natively supported via JWT claims
`customJWTAuthorizer` on an AgentCore runtime supports a `customClaims`
field that can require a specific claim value (e.g. Cognito's
`cognito:groups`, a string array) to match, **before the agent's own code
ever runs** — a coarse, all-or-nothing gate enforced at the authorizer layer.
Example:
```json
{
  "inboundTokenClaimName": "cognito:groups",
  "inboundTokenClaimValueType": "STRING_ARRAY",
  "claimMatchOperator": "CONTAINS",
  "claimMatchValue": "GitlabPipelineUsers"
}
```
Source: AWS docs `inbound-jwt-authorizer.html`, `API_CustomClaimValidationType`.

AWS's own samples (`aws-samples/sample-a2a-gateway`, AWS Security Blog
"Propagate user authorization context...") push fine-grained (Tier 2)
authorization to a **Gateway interceptor** or IAM session tags, explicitly
saying "the agent is an orchestrator, not a gatekeeper" — they do **not**
recommend trusting in-agent code for this. We decided anyway (see §6) to do
Tier 2 as in-runtime code for demo speed, accepting this is not AWS's
recommended production pattern — flagged explicitly, not hidden.

### No standard "access denied" signal in A2A
The A2A protocol has no dedicated JSON-RPC error code for authorization
failures (named codes only cover -32001 through -32009 for other things).
Decision: keep using the plain string convention already proven in the old
Tier1/Tier2 code — a tool/task result whose text starts with
`"ACCESS_DENIED: ..."` — as the uniform cross-cloud denial signal. Simple,
already works, doesn't depend on transport-level differences between clouds.

### Anti-hallucination guardrail — must be real enforcement, not just wording
Research (arXiv "Causality Laundering: Denial-Feedback Leakage in
Tool-Calling LLM Agents"; OSO "Why Prompt-Based Safety Is Not Enough";
Microsoft Security Blog "Least Privilege for AI Agents") is consistent:
**prompt text alone is not reliable protection** against an LLM ignoring a
denial and answering from its own parametric knowledge. The only real
defense is making sure the model genuinely never receives real data when
denied — there must be nothing to "launder." Prompt-level instructions are
still valuable as defense-in-depth for *how* the denial is communicated to
the user, but are not the security boundary.

This is exactly what the existing (already proven) pattern does right:
- Tier 1 denial → the disallowed tool is never even in the model's tool
  list, or the call never returns real data — nothing to leak.
- Tier 2 denial → same principle: an unauthorized tool is never in the
  per-request tool list at all, so the model can't call it or see it exists.

## 6. Design decisions already made with you

- **Case 2 Tier 2 = in-runtime code branching** (reading `cognito:groups`
  from the validated token inside the AgentCore runtime's own handler),
  **not** a new AgentCore Gateway + interceptor. Chosen for demo speed; the
  trade-off against AWS's own recommended pattern is explicitly documented,
  not glossed over.
- Default scope (not explicitly confirmed, flagged as an assumption to
  correct if wrong): **build full Tier 1 + Tier 2 in both cases**, not just
  one case as a stretch.
- **One orchestrator, not two**: `orchestratorAgent` should front both
  Case 1's sub-agents and the Case 2 AgentCore connection at once. Same
  denial convention works across both, so one set of instructions covers
  both; a single session can show "same orchestrator, same user, two
  different back ends, same enforcement pattern," which is a stronger proof
  of the general thesis than two separate orchestrators would be.

## 7. Concrete build — Case 1 (Foundry-to-Foundry)

**Reuse, no new agents needed:**
- `a2aSubagent` / `subagent` (Prompt Agent) → **Subagent1**, Tier 1 demo only
  (Prompt Agents can't run custom filtering code).
- `identityProbeAgent` / `ProbesubagentAgent` (Hosted Agent, our own
  container) → **Subagent2**, Tier 1 *and* Tier 2 demo (we control its code).

### Tier 1 — agent-scope RBAC
```bash
SUB_ID=$(az account show --query id -o tsv)
ACCOUNT="a2aPocChowdri"; RG="a2aPocChowdriRg"; PROJECT="A2aProj"
SUBAGENT1="subagent"; SUBAGENT2="ProbesubagentAgent"
USER_A_OID="<UserA-object-id>"; USER_B_OID="<UserB-object-id>"

SUBAGENT1_SCOPE="/subscriptions/$SUB_ID/resourceGroups/$RG/providers/Microsoft.CognitiveServices/accounts/$ACCOUNT/projects/$PROJECT/agents/$SUBAGENT1"
SUBAGENT2_SCOPE="/subscriptions/$SUB_ID/resourceGroups/$RG/providers/Microsoft.CognitiveServices/accounts/$ACCOUNT/projects/$PROJECT/agents/$SUBAGENT2"

# User A: Subagent1 only
az role assignment create --assignee-object-id "$USER_A_OID" --assignee-principal-type User \
  --role "Foundry Agent Consumer" --scope "$SUBAGENT1_SCOPE"

# User B: Subagent2 only
az role assignment create --assignee-object-id "$USER_B_OID" --assignee-principal-type User \
  --role "Foundry Agent Consumer" --scope "$SUBAGENT2_SCOPE"
```
This deliberate cross-matrix (A→1 only, B→2 only) proves it's genuinely
per-(user, agent) — B *can* reach Subagent2, ruling out "maybe Subagent2 is
just broken for everyone."

Later, as a live demo step, grant User A access to Subagent2 too (see demo
script §9) to transition into the Tier 2 segment:
```bash
az role assignment create --assignee-object-id "$USER_A_OID" --assignee-principal-type User \
  --role "Foundry Agent Consumer" --scope "$SUBAGENT2_SCOPE"
```
Also grant both users `Foundry User` on the project (per `REFERENCE.md`),
to avoid the known "consent completes but tool calls fail" false negative.

### Tier 2 — tool filtering inside `identityProbeAgent`
Port the proven `specialist2/main.py` pattern
(`a2a/specialist2/main.py`) into
`a2a/a2a_withrbac/native_obo_connection_poc/probe_subagent/main.py`,
replacing HMAC role-claim verification with a lookup keyed on the real
OAuth-passed identity:

```python
_OBJECT_ID_TO_ROLE = {
    "<UserA-object-id>": "write",
    "<UserB-object-id>": "read",
}

def get_role_for_caller(user_id: str | None) -> str:
    return _OBJECT_ID_TO_ROLE.get(user_id or "", "none")

async def read_status(item: str) -> str:
    return f"[DUMMY] status for {item!r}"

async def update_status(item: str, note: str) -> str:
    return f"[DUMMY] updated {item!r} with note {note!r}"

_ALL_TOOLS = {"read_status": read_status, "update_status": update_status}

def get_allowed_tool_names(role: str) -> list[str]:
    if role == "write":
        return ["read_status", "update_status"]
    if role == "read":
        return ["read_status"]
    return []  # unknown/absent identity -> fail closed, zero tools
```
In the response handler:
```python
user_id = context.client_headers.get("x-client-user-id")  # confirm exact header name live first
role = get_role_for_caller(user_id)
tools = [_ALL_TOOLS[name] for name in get_allowed_tool_names(role)]
agent = Agent(client=client, name="OboProbeAgent", instructions=INSTRUCTIONS, tools=tools)  # fresh per request
```
Keep a `checkrole` debug branch (mirroring the existing
`checkheaders`/`checkrawheaders`/`checkbaggage`/`checkmetadata` debug
commands already in this file) reporting `user_id`/`role`/`allowed_tools`,
for rehearsal verification.

**Why this is the strong version of the anti-hallucination defense**: the
unauthorized tool is never in the model's tool list at all — the model
can't call it or discover it exists, so there's no real content it could
"launder" into a hallucinated answer.

## 8. Concrete build — Case 2 (Foundry-to-AgentCore)

**Reuse, no new AgentCore runtime needed:** `gitlab_pipeline_subagent-X2jgkC9HQl`
(connected to Foundry as `agentcoreGitlabPipeline`/`GitlabSubagent`).

### Tier 1 — Cognito groups + AgentCore `customClaims`
```bash
POOL_ID="us-east-1_Fp7jFI1dU"

aws cognito-idp create-group --user-pool-id "$POOL_ID" --group-name GitlabPipelineUsers \
  --description "Tier 1 gate: can reach the GitLab pipeline sub-agent at all"
aws cognito-idp create-group --user-pool-id "$POOL_ID" --group-name GitlabPipelineWriters \
  --description "Tier 2: can use write-style GitLab pipeline tools"

# User A = testuser (existing) -> full access
aws cognito-idp admin-add-user-to-group --user-pool-id "$POOL_ID" --username testuser --group-name GitlabPipelineUsers
aws cognito-idp admin-add-user-to-group --user-pool-id "$POOL_ID" --username testuser --group-name GitlabPipelineWriters

# User B = testuser2 (new), deliberately left OUT of GitlabPipelineUsers at first
aws cognito-idp admin-create-user --user-pool-id "$POOL_ID" \
  --username testuser2 --user-attributes Name=email,Value=testuser2@example.com \
  --temporary-password 'TempPass123!' --message-action SUPPRESS
aws cognito-idp admin-set-user-password --user-pool-id "$POOL_ID" \
  --username testuser2 --password 'YourRealPassword123!' --permanent
```

Update the existing runtime's authorizer (do not recreate the runtime):
```bash
aws bedrock-agentcore-control update-agent-runtime \
  --agent-runtime-id gitlab_pipeline_subagent-X2jgkC9HQl --region us-east-1 \
  --authorizer-configuration '{
    "customJWTAuthorizer": {
      "discoveryUrl": "https://cognito-idp.us-east-1.amazonaws.com/us-east-1_Fp7jFI1dU/.well-known/openid-configuration",
      "allowedClients": ["23vj8ta52cffe5bbctaehpnjs1"],
      "customClaims": [{
        "inboundTokenClaimName": "cognito:groups",
        "inboundTokenClaimValueType": "STRING_ARRAY",
        "claimMatchOperator": "CONTAINS",
        "claimMatchValue": "GitlabPipelineUsers"
      }]
    }
  }'
```
Confirm the exact JSON shape against the live CLI/API schema before running
(`aws bedrock-agentcore-control update-agent-runtime help`) — the field
names are verified, the enclosing structure should be dry-run checked.

User B, not in the group, is rejected by the authorizer itself before the
container runs at all — no second AgentCore runtime is needed to
demonstrate "can this user reach this agent."

### Tier 2 — in-runtime code branching on `cognito:groups`
Extend the runtime's own handler code
(`simplestrands/subagent_a2a/agent.py`) with:
1. Middleware that decodes the incoming bearer JWT's claims into a
   contextvar (mirrors the already-proven `RawHeaderCaptureMiddleware` /
   `_raw_headers_var` pattern from the Foundry side, in
   `probe_subagent/main.py`).
2. A Strands `HookProvider` on `BeforeInvocationEvent` (mirrors the
   already-existing `AgentRelayScopedTools` pattern in
   `hack/agents-for-demo/subagent_a2a/agentrelay_session.py`) that reads
   `cognito:groups` from that contextvar and calls
   `event.agent.tool_registry.register_dynamic_tool(tool)` only for the
   tools the user's groups permit — read-only tools always; write tools
   only if `GitlabPipelineWriters` is present.
3. Build the `Agent` with `tools=[]` statically; the hook attaches the
   real subset per task.

Two things to call out explicitly in the write-up:
- This is a deliberate deviation from AWS's own recommended
  Gateway-interceptor pattern (see §5) — chosen for demo speed, not a
  production recommendation.
- The contextvar-across-async-boundary trick needs a live end-to-end check
  on the Strands/AgentCore stack (it's proven on the Foundry ASGI side, but
  new here) — if it doesn't propagate correctly, groups reads empty and the
  fail-closed behavior (deny/empty tools) is the safe direction, but this
  should be confirmed, not assumed.

## 9. Demo script — how we prove it, step by step

**Setup state before starting**: User A has Consumer role on Subagent1 only;
User B has Consumer role on Subagent2 only; Cognito `testuser` is in both
groups; `testuser2` is in neither.

**Case 1 — Foundry-to-Foundry**
1. Sign in as User A. Ask a question routed to Subagent1 — succeeds; open
   Traces to show a real outbound A2A call happened.
2. Same session, ask a question routed to Subagent2 — orchestrator responds
   with a plain "you do not have access" message, no hallucinated content;
   Traces show the tool call itself failed, not that the model chose
   silence.
3. Sign in as User B. Repeat both — mirror result (Subagent2 succeeds,
   Subagent1 denied). Proves it's per-(user, agent), not "Subagent2 is
   broken."
4. **Live Tier 2 transition**: run the `az role assignment create` grant
   giving User A access to Subagent2 too. Narrate: "User A can now reach
   Subagent2 — but reaching it isn't the same as full access inside it."
5. As User A, ask Subagent2 to do a write-style action — succeeds.
6. As User B (had Subagent2 access from the start), ask the same
   write-style action — orchestrator reports it simply can't do that for
   this user (the tool was never offered to the model for User B's role).
   Contrast with step 2/3's outright denial to make Tier 1 vs Tier 2
   concrete on stage.

**Case 2 — Foundry-to-AgentCore**
1. Sign in as User A (`testuser`). Ask a GitLab pipeline question — succeeds.
2. Sign in as User B (`testuser2`, not yet in `GitlabPipelineUsers`). Ask the
   same — clean denial; note out loud this happened at the AgentCore
   authorizer, before the container ever ran (no data was ever generated to
   leak).
3. **Live Tier 2 transition**: add `testuser2` to `GitlabPipelineUsers` (not
   `GitlabPipelineWriters`). Force a fresh sign-in for User B.
4. Ask a read-style question as User B — succeeds now (Tier 1 passed).
5. Ask a write-style question as User B — orchestrator reports it can't do
   that for this user; contrast with User A performing the same action
   successfully.
6. Optionally show AgentCore CloudWatch logs for the tool-scoping hook's log
   line as code-level proof, mirroring the Traces-tab proof on the Foundry
   side.

## 10. Orchestrator instructions update

Append to `orchestratorAgent`'s existing instructions (same
`create_version(...)` mechanism as `create_prompt_agent.py` uses):
```
If a tool call fails, times out, or returns a result indicating a permission
or authorization problem — including but not limited to text starting with
"ACCESS_DENIED", or any mention of 403, Forbidden, Unauthorized, or
AuthorizationFailed — do NOT answer that part of the question using your own
knowledge. Tell the user plainly that they do not have access to that
capability and to contact their administrator. Still answer any other parts
of the question you DO have a successful tool result for.
```
This is defense-in-depth for wording/UX only — the real security boundary is
the RBAC/claims/tool-list enforcement in §7–8, not this text. Finalize the
exact wording only after seeing what error text actually reaches the model
on a real denial in each cloud (Open Decisions #2).

## 11. Where this gets documented

New "Part 3: Per-User Restriction (Tier 1 + Tier 2)" appended to
`a2a/a2a_withrbac/native_obo_connection_poc/A2A_OAUTH_PASSTHROUGH_SUCCESS.md`,
matching its existing practical how-to tone (numbered steps, tables, no
narrative/proof-of-work framing):
- 3.1 Tier 1 — Foundry agent-scope RBAC (commands + scope table)
- 3.2 Tier 2 — Foundry in-agent tool filtering (code + role table)
- 3.3 Tier 1 — AgentCore `customClaims` + Cognito groups (commands + table)
- 3.4 Tier 2 — AgentCore in-runtime hook (code + explicit AWS-best-practice
  deviation note)
- 3.5 Denial convention (the `ACCESS_DENIED:` string, why no A2A error code)
- 3.6 Orchestrator instructions (the exact clause from §10)
- "Verifying Tier 1 / Tier 2" section, same style as the existing
  "Verifying It Works" section, reusing `checkrole`-style debug commands.

## 12. A separate discovery worth resolving — the "AgentRelay" system

While verifying the plan against actual code, found a **separate, more
mature system already built** under `hack/` that was not part of the
context this plan was designed from:
- `hack/mcp-session-gateway/` — a working MCP gateway that mints
  short-lived, signed tokens scoping a session to a specific tool subset,
  with **real enforcement at a proxy layer** (denied tool calls never reach
  the real backend at all — stronger than an in-agent allowlist, which the
  gateway's own code explicitly calls out: "an allowlist inside the agent
  would only ever be a request to the model, not a restriction on it").
- `hack/backend/app/` — a FastAPI backend ("AgentRelay") with API-key-based
  auth (`auth.py`: `require_agent_scope`, `require_mcp_scope`,
  `allowed_agent_names`, `allowed_mcp_names`), i.e. Tier 1 (which agents a
  key can call) and Tier 2 (which MCPs/tools) already implemented — but
  keyed on **admin-issued API keys with tiers/scopes**, not on Entra/Cognito
  OAuth user identity.
- `hack/agents-for-demo/` — a full three-runtime GitLab demo (orchestrator +
  two sub-agents on AWS AgentCore) that already routes every sub-agent call
  through AgentRelay's governed endpoint instead of calling
  `InvokeAgentRuntime` directly.

**This is not yet reconciled with the plan above.** It's a real, working,
arguably stronger enforcement mechanism, but it authenticates callers via
API keys, not via the real signed-in user's OAuth token — a different trust
model than the one this whole demo is about proving ("the real user's
identity flows through and is enforced against"). Two ways this could go,
not yet decided:
- **Leave it alone**: treat it as unrelated prior work (possibly a different
  product surface or an earlier exploration), and build Tier 1/Tier 2 as
  designed in §7–8, independent of AgentRelay.
- **Integrate it**: swap AgentRelay's API-key identity for the real
  OAuth-passed user identity (Entra Object ID / Cognito `sub`+`groups`), and
  use its already-working, already-tested proxy-level enforcement instead of
  writing the in-agent Tier 2 code in §7–8 fresh. This would be more robust
  (real enforcement boundary, not just "don't put the tool in the list") but
  is a bigger scope change and needs its own investigation of how much
  rework that swap would take.

**Decision needed before Tier 2 code is actually written** — flagging here
so it isn't silently skipped.

## 13. Open items to verify before the live demo

1. **Exact identity header name on the Foundry side** — code currently
   checks `x-client-user-id`; earlier docs assumed `x-agent-user-id`.
   Confirm live via `checkrawheaders`/`checkheaders` before wiring the Tier
   2 role lookup.
2. **Exact error shape a Prompt Agent orchestrator receives on a native
   Tier 1 denial**, both clouds — needed to finalize the orchestrator
   instruction wording precisely rather than the broad catch-all in §10.
3. **AgentCore runtime's current authorizer state** — a local
   `.bedrock_agentcore.yaml` snapshot shows `authorizer_configuration:
   null`, which may just be stale relative to the live control-plane state.
   Confirm with `aws bedrock-agentcore-control get-agent-runtime` before
   assuming JWT auth is already active and before layering `customClaims`
   on top.
4. **Exact `update-agent-runtime` request shape** for
   `authorizerConfiguration`/`customClaims` — dry-run check against the live
   CLI/API schema.
5. **Real GitLab MCP tool names** for the Tier 2 read/write split — confirm
   via one `list_tools_sync()` call rather than the illustrative names used
   above.
6. **Contextvar propagation across Strands' async event loop** in the
   AgentCore Tier 2 hook — needs one live end-to-end check; fails closed if
   wrong, but shouldn't be assumed.
7. **The AgentRelay question in §12** — needs an explicit decision before
   Case 2's Tier 2 code is written for real.
8. Possible day-to-day instability flagged by an older test-results file
   (`LIVE_TEST_RESULTS.md`, one day older than the success doc) that once
   concluded Prompt-Agent-to-Prompt-Agent A2A was broken — worth a 2-minute
   sanity re-check of `orchestratorAgent → a2aSubagent` the morning of the
   demo, since this specific integration has shown instability before.

## 14. Resource inventory

| Resource | Status |
|---|---|
| `orchestratorAgent` (Foundry Prompt Agent, `A2aProj`) | Existing — fronts both cases |
| `a2aSubagent` / `subagent` (Prompt Agent) | Existing — Case 1 / Subagent1 |
| `identityProbeAgent` / `ProbesubagentAgent` (Hosted Agent) | Existing — Case 1 / Subagent2 |
| AgentCore runtime `gitlab_pipeline_subagent-X2jgkC9HQl` | Existing — Case 2 target |
| Cognito pool `us-east-1_Fp7jFI1dU`, client `23vj8ta52cffe5bbctaehpnjs1`, user `testuser` | Existing |
| 2 Entra test users with known Object IDs (User A / User B) | New (or reuse from earlier POC work) |
| Cognito user `testuser2` | New |
| Cognito groups `GitlabPipelineUsers`, `GitlabPipelineWriters` | New |
| Agent-scope role assignments (×3, see §7) | New |
| `update-agent-runtime` call adding `customClaims` | New |
| Code changes: `probe_subagent/main.py`, `simplestrands/subagent_a2a/agent.py` | New |
| Redeploy of the AgentCore runtime with updated code | New |
| Orchestrator instructions update (new agent version) | New |

No new Foundry agents, no new AgentCore runtimes, no new ACR images beyond
re-pushing `identityprobe` and redeploying the existing AgentCore runtime —
deliberately, to keep new-setup risk off the demo-prep critical path.
