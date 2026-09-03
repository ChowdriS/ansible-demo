# Demo Plan: OAuth Passthrough + Per-User Restriction in Azure AI Foundry (and to an External Endpoint)

*Refined 2026-09-03 — see "Refinement" note in §6 for what changed and why.*

## 1. The use case / why this matters

This is the original aim of the whole A2A project: not just "can a sub-agent
find out who's really asking" (OAuth identity passthrough — already proven),
but **"can we actually restrict what a sub-agent and its MCP tools will do,
per real signed-in user."** Identity passthrough is the plumbing; per-user
restriction is the point.

We are preparing a **live demo session** to show both halves working
together, in two parallel scenarios, so the pattern reads as general — not a
Foundry-only trick or a trick specific to one cloud.

## 2. The two demo cases

- **Case 1 — all-Foundry**: orchestrator (Foundry Prompt Agent) + sub-agents
  + MCP tools all live inside Azure AI Foundry.
- **Case 2 — Foundry to an external endpoint**: orchestrator still lives in
  Foundry, but the sub-agent + its MCP tools live outside Foundry entirely,
  reached via Foundry's generic **"Connect via endpoint" + Custom OAuth**
  capability. AWS Bedrock AgentCore is the example external target used here
  (with Amazon Cognito as the identity provider, since that's already set up
  and proven) — but the point of this case is to showcase that Foundry can
  connect to *any* OAuth-secured external agent this way, not to showcase
  AWS/AgentCore-specific features. See §6 for what this changed.

**Hard constraint, both cases**: the orchestrator must be a Foundry **Prompt
Agent**, never a Hosted Agent — the OAuth consent flow is a Prompt Agent
tool-invocation mechanism only, confirmed by direct testing, not assumed.

## 3. What's already proven (before this restriction work)

- **Case 1 identity passthrough**: confirmed working — the real caller's
  Entra Object ID arrives at a Foundry Hosted Agent sub-agent via the
  `x-client-user-id` header (confirm this exact header name live once more
  before wiring real logic to it — see Open Items).
- **Case 2 identity passthrough**: confirmed working end-to-end — Foundry
  orchestrator → Cognito consent → AWS Bedrock AgentCore-hosted GitLab agent,
  real response returned, cross-cloud.
- Full step-by-step setup for both is documented in
  `a2a/a2a_withrbac/native_obo_connection_poc/A2A_OAUTH_PASSTHROUGH_SUCCESS.md`.

## 4. The actual gap this plan closes

Identity passthrough ≠ restriction. Two tiers, both currently unbuilt:

- **Tier 1** — can this user reach this specific sub-agent at all (e.g. User
  A can reach Subagent1 but not Subagent2).
- **Tier 2** — once reachable, which specific tools/MCP capabilities can this
  user use inside that sub-agent (e.g. read vs. write).

## 5. Case 1 — Foundry-to-Foundry (unchanged)

**Reuse, no new agents needed:**
- `a2aSubagent` / `subagent` (Prompt Agent) → **Subagent1**, Tier 1 demo only
  (Prompt Agents can't run custom filtering code).
- `identityProbeAgent` / `ProbesubagentAgent` (Hosted Agent, our own
  container) → **Subagent2**, Tier 1 *and* Tier 2 (we control its code).

### Tier 1 — native agent-scope RBAC
Microsoft Foundry supports RBAC role assignment scoped to a **single agent**,
not just the whole project — enforced by Azure RBAC itself, before the
sub-agent ever runs, no custom code needed:
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
(Portal UI doesn't support agent-scope assignment yet — CLI/REST only.
Caveat: agent-scope only governs endpoint invocation, not broader
control-plane/management permissions.) This deliberate cross-matrix (A→1
only, B→2 only) proves it's genuinely per-(user, agent) — B *can* reach
Subagent2, ruling out "maybe Subagent2 is just broken for everyone."

Live demo moment: grant User A access to Subagent2 too, on stage, to
transition into the Tier 2 segment:
```bash
az role assignment create --assignee-object-id "$USER_A_OID" --assignee-principal-type User \
  --role "Foundry Agent Consumer" --scope "$SUBAGENT2_SCOPE"
```
Also grant both users `Foundry User` on the project (per `REFERENCE.md`), to
avoid the known "consent completes but tool calls fail" false negative.

### Tier 2 — tool filtering inside `identityProbeAgent`
Port the proven `specialist2/main.py` pattern into
`a2a/a2a_withrbac/native_obo_connection_poc/probe_subagent/main.py`:
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
In the response handler: `user_id = context.client_headers.get("x-client-user-id")`
→ look up role → build a fresh `Agent(...)` per request with only the
allowed tool subset. Keep a `checkrole` debug branch (mirroring the existing
`checkheaders`/`checkrawheaders`/`checkbaggage`/`checkmetadata` commands
already in this file) for rehearsal verification.

**Why this is the strong version of anti-hallucination**: the unauthorized
tool is never in the model's tool list at all — nothing for the model to
launder into a hallucinated answer.

## 6. Case 2 — Foundry to an external endpoint — REFINED

**What changed and why**: the first pass of this design leaned on
AgentCore-native authorization features (a `customClaims` rule on
AgentCore's inbound JWT authorizer, matched against Cognito Groups). That
was the wrong emphasis. **The point of Case 2 is to showcase Foundry's
generic "Connect via endpoint" OAuth passthrough capability — AgentCore is
just an example of *some* external agent, not something to configure
deeply.** So restriction logic should not depend on anything
AgentCore-specific, and should work the same way against any other external
OAuth-secured agent Foundry might connect to. Restriction now happens
entirely by reading a **standard claim already present in the passed-through
OAuth token**, inside the external agent's own code — the exact same
approach Case 1's Tier 2 already uses, just applied on the AWS side.

**Design principle**: the AgentCore side requires nothing beyond what's
already needed for OAuth passthrough to function at all (its existing
`customJWTAuthorizer` with `discoveryUrl` + `allowedClients` — already
configured, already proven). No `customClaims` authorizer rule, no Cognito
Groups, no `update-agent-runtime` changes. All restriction logic lives in
the external agent's own runtime code.

**Reuse, no new AgentCore runtime needed:** `gitlab_pipeline_subagent-X2jgkC9HQl`
(connected to Foundry as `agentcoreGitlabPipeline`/`GitlabSubagent`).

### Identity source
Decode the bearer token already arriving on every request (the same token
AgentCore's authorizer validated for authenticity) and read a stable,
always-present claim — `sub` or `username` (Cognito's Access Token, which is
what Foundry actually sends per the earlier token-claim-split finding,
carries `username`/`sub` but not `email`/`aud`). **Confirm which of
`sub`/`username` is actually present via one live decode before wiring the
real lookup** (mirrors Case 1's open item on header naming).

### Tier 1 AND Tier 2 — same mechanism, one hardcoded map
Inside `simplestrands/subagent_a2a/agent.py`:
```python
# Decoded once per request via lightweight middleware (mirrors the
# already-proven RawHeaderCaptureMiddleware/_raw_headers_var pattern from
# probe_subagent/main.py, and the BeforeInvocationEvent hook pattern from
# hack/agents-for-demo/subagent_a2a/agentrelay_session.py's AgentRelayScopedTools).

_IDENTITY_TO_ROLE = {
    "testuser": "write",   # User A
    "testuser2": "read",   # User B
    # anything else / no claim at all -> "none" (fail closed)
}

def get_role_for_caller(identity: str | None) -> str:
    return _IDENTITY_TO_ROLE.get(identity or "", "none")
```
In the `BeforeInvocationEvent` hook:
- `role = get_role_for_caller(claims.get("username"))`
- `role == "none"` → `event.cancel = "ACCESS_DENIED: ..."`, no tools
  attached, no real MCP call ever made (**Tier 1**).
- otherwise → attach read-only tools always, write tools only if
  `role == "write"` (**Tier 2**), via
  `event.agent.tool_registry.register_dynamic_tool(...)`.

### No new AWS-side configuration
Beyond what's already deployed: no `update-agent-runtime` call, no Cognito
Groups, no group-add/remove steps. Only a code change to the existing
runtime + a redeploy, plus one new Cognito test user (`testuser2`) so there
are two distinct identities to demo against — Cognito is used here purely as
the already-working identity provider issuing tokens, nothing more.

### Trade-off, stated explicitly
This Tier 1 check runs *inside* the agent's own code, after the container
has already started processing the request — AgentCore's authorizer only
validates that the token is legitimately issued and from the right client,
it does not itself gate per-user. This is a weaker platform-level boundary
than Case 1's native Azure RBAC (which rejects before the sub-agent runs at
all). Accepted deliberately: the point of Case 2 is showing the generic "any
external OAuth-secured endpoint + passthrough token + the endpoint's own
code decides" pattern, not leaning on cloud-specific IAM plumbing.

### Demo-flow implication
Since the role map is a static hardcoded table (not a live-editable AWS
resource like a role assignment or group membership), there's no cheap
"grant access live on stage" moment for Case 2 the way Case 1 has.
**Recommended**: pre-stage both users with their final roles before the demo
starts (User A = write, User B = read-only) and demonstrate the *contrast*
directly, rather than a live permission change. (A live-editable version —
env var + hot-reload, or a debug endpoint to flip a role at runtime — is
possible as an optional enhancement, not required.)

### What's kept only as reference, not built
AgentCore's `customJWTAuthorizer` *does* support a `customClaims` field that
could match against `cognito:groups` before any agent code runs — real,
documented AWS capability, worth one line in the guide as "AgentCore can
also do this natively, if you want a stronger platform-level Tier 1
boundary" — but it is **not** what this demo builds, per the refinement
above.

## 7. Anti-hallucination guardrail (both cases)

The security boundary is the enforcement in §5–6 (unauthorized tools are
never in the model's tool list; denied identities never reach real tool
logic) — not prompt wording. Prompt wording is added only as UX
defense-in-depth, appended to `orchestratorAgent`'s instructions:
```
If a tool call fails, times out, or returns a result indicating a permission
or authorization problem — including but not limited to text starting with
"ACCESS_DENIED", or any mention of 403, Forbidden, Unauthorized, or
AuthorizationFailed — do NOT answer that part of the question using your own
knowledge. Tell the user plainly that they do not have access to that
capability and to contact their administrator. Still answer any other parts
of the question you DO have a successful tool result for.
```
Finalize the exact wording only after seeing what error text actually
reaches the model on a real denial in each cloud (Open Items #3).

## 8. One orchestrator fronts both cases

Same `orchestratorAgent`, same denial convention (`ACCESS_DENIED:` string
prefix) across both — lets one live session show "same orchestrator, same
identity-passthrough mechanism, two different external targets, same
enforcement pattern," which is the actual thesis of the demo.

## 9. Demo script

**Case 1** (unchanged): cross-matrix Tier 1 (User A only reaches Subagent1,
User B only reaches Subagent2), each denial shown clean/non-hallucinated via
Traces; then a **live** `az role assignment create` moment granting User A
access to Subagent2; then a Tier 2 contrast (User A can write, User B —
despite now also reaching Subagent2 — cannot).

**Case 2** (refined, no live permission change):
1. Sign in as User A (`testuser`, role=write). Ask a read-style GitLab
   question — succeeds. Ask a write-style question — succeeds.
2. Sign in as User B (`testuser2`, role=read). Ask a read-style question —
   succeeds. Ask a write-style question — orchestrator reports it can't do
   that for this user (tool never offered).
3. Optional: an identity not in the map at all, to show outright Tier 1
   denial — narrate that this is the same mechanism as Case 1's Tier 1
   denial, just enforced in-code instead of by Azure RBAC.
4. Optionally show AgentCore CloudWatch logs for the role-scoping hook's log
   line as code-level proof, mirroring the Traces-tab proof on the Foundry
   side.

## 10. Documentation

New "Part 3: Per-User Restriction (Tier 1 + Tier 2)" appended to
`a2a/a2a_withrbac/native_obo_connection_poc/A2A_OAUTH_PASSTHROUGH_SUCCESS.md`:
- 3.1 Tier 1 — Foundry agent-scope RBAC (commands + scope table)
- 3.2 Tier 2 — Foundry in-agent tool filtering (code + role table)
- 3.3 Tier 1+2 — external endpoint, in-code universal-claim approach (code +
  role table), with a short callout noting AgentCore's native `customClaims`
  option exists but isn't what this guide builds, and why
- 3.4 Denial convention (the `ACCESS_DENIED:` string, why no A2A error code)
- 3.5 Orchestrator instructions (the exact clause from §7)
- "Verifying Tier 1 / Tier 2" section, reusing `checkrole`-style debug
  commands, same style as the existing "Verifying It Works" section.

## 11. A separate discovery, deliberately not part of this plan

While verifying an earlier draft against actual code, found a separate, more
mature system already built under `hack/` — an "AgentRelay" gateway
(`hack/mcp-session-gateway/`, `hack/backend/app/`, `hack/agents-for-demo/`)
with real proxy-level MCP tool enforcement (stronger than an in-agent
allowlist), but authenticated via admin-issued API keys with tiers/scopes,
not the real OAuth-passed user identity this demo is about. **This is left
alone** — noted here only so it isn't mistaken for something this plan
depends on, in case it comes up later.

## 12. Open items to verify live before the demo

1. Exact identity header name on the Foundry side (`x-client-user-id` vs.
   `x-agent-user-id`) — one `checkrawheaders` call.
2. Exact claim name present on the Cognito Access Token actually sent by
   Foundry (`sub` vs. `username`) — one live token decode.
3. Exact error shape a Prompt Agent orchestrator receives on a native Tier 1
   denial, both clouds — needed to finalize orchestrator instruction wording
   precisely.
4. AgentCore runtime's current authorizer state — confirm with
   `aws bedrock-agentcore-control get-agent-runtime` that the existing
   `customJWTAuthorizer` (discoveryUrl + allowedClients) is really active,
   since a local `.bedrock_agentcore.yaml` snapshot showed
   `authorizer_configuration: null` (likely just stale).
5. Real GitLab MCP tool names for the Tier 2 read/write split — confirm via
   one `list_tools_sync()` call rather than illustrative names.
6. Contextvar propagation across Strands' async event loop for the Case 2
   hook — needs one live end-to-end check; fails closed if wrong, but
   shouldn't be assumed.
7. Possible day-to-day instability flagged by an older test-results file
   (`LIVE_TEST_RESULTS.md`) that once concluded Prompt-Agent-to-Prompt-Agent
   A2A was broken — worth a 2-minute sanity re-check of
   `orchestratorAgent → a2aSubagent` the morning of the demo.

## 13. Resource inventory

| Resource | Status |
|---|---|
| `orchestratorAgent` (Foundry Prompt Agent, `A2aProj`) | Existing — fronts both cases |
| `a2aSubagent` / `subagent` (Prompt Agent) | Existing — Case 1 / Subagent1 |
| `identityProbeAgent` / `ProbesubagentAgent` (Hosted Agent) | Existing — Case 1 / Subagent2 |
| AgentCore runtime `gitlab_pipeline_subagent-X2jgkC9HQl` | Existing — Case 2 target, **no authorizer config changes needed** |
| Cognito pool `us-east-1_Fp7jFI1dU`, client `23vj8ta52cffe5bbctaehpnjs1`, user `testuser` | Existing |
| Cognito user `testuser2` | New — just a second identity, no group assignment |
| 2 Entra test users w/ known Object IDs (User A / User B) | New (or reuse from earlier POC work) |
| Agent-scope role assignments (×3, see §5) | New |
| Code change: `probe_subagent/main.py` (Tier 2 role map) | New |
| Code change: `simplestrands/subagent_a2a/agent.py` (Tier 1+2 role map, claim-decode middleware, scoping hook) | New |
| Redeploy of the AgentCore runtime with updated code | New |
| Orchestrator instructions update (new agent version) | New |

No Cognito Groups, no `update-agent-runtime` call, no new AgentCore runtime,
no new Foundry agents, no new ACR images beyond re-pushing `identityprobe`
and redeploying the existing AgentCore runtime — deliberately, to keep
new-setup risk off the demo-prep critical path.
