# Attenuate — Every Hop Narrows, Never Escalates

**One-line pitch:** When a multi-agent system delegates a task across several agents — and especially when that delegation crosses from Azure to AWS or back — nothing today cryptographically guarantees that the calling human's original permissions only get *narrower* at each hop. Attenuate closes that gap: it verifies every hop in a delegation chain, blocks the moment a token tries to come back with more access than it left with, and stitches the whole chain into one legible audit trail — even when it crosses clouds.

**Themes:** Security & Governance at Scale (primary).

**Built on:** Azure AI Foundry / Entra Agent ID and AWS Bedrock AgentCore Identity, together — this is a genuinely cross-cloud pitch, which directly uses the hackathon's "any combination" language.

---

## 1. The problem, precisely

When a human calls an AI orchestrator, and that orchestrator delegates work to a sub-agent, the sub-agent needs to know: is it acting with the *orchestrator's* broad service-level access, or with *this specific human's* actual, narrower permissions? Get this wrong, and every sub-agent in the system effectively has the same broad access no matter who asked for it — a modern version of the decades-old "confused deputy problem," now happening inside AI agent chains.

**What's already solved (as of 2026) — don't rebuild this:**
- AWS Bedrock AgentCore Identity added native On-Behalf-Of (OBO) token exchange in April 2026 (RFC 8693) — the AgentCore Gateway transparently swaps an inbound user token for a scoped, audience-bound downstream token.
- Azure Entra Agent ID has equivalent native OBO support — an agent identity is a service principal that impersonates the signed-in user via the standard OAuth OBO flow.

One hop, one cloud, is a solved problem today. Attenuate is not about rebuilding that.

**What's confirmed still broken — this is the actual build:**
- RFC 8693 itself states that its "prior-actor" claims (the record of who delegated to whom) are **informational only**. Nothing in the standard cryptographically enforces that a token's permissions can only shrink as it passes through a second or third agent, never grow. Two real, named, dated attacks already exploit exactly this gap: **Cross-Agent Privilege Escalation** (September 2025, disclosed against GitHub Copilot agents) and **Agent Session Smuggling** (November 2025).
- Cross-cloud makes it worse, concretely: AgentCore's default runtime auth is IAM/SigV4, which an Azure Foundry container has no way to sign or verify. AWS has no formal A2A specification of its own yet. When a delegation chain crosses from one cloud's authorization server to another's, the audit trail fragments across systems with completely different logging conventions — nobody can currently answer "which human ultimately authorized this" once a chain crosses that boundary.
- This is being actively discussed as an urgent, unaddressed problem right now, not a stale one: SANS Institute published a piece titled "Your AI Agent Is an Easily Confused Deputy: Why Cloud Security Needs a Credential Broker," and a widely-shared security write-up is titled "The Confused Deputy Problem Just Hit AI Agents — And Nobody's Scanning for It." Academic proposals exist (Invocation-Bound Capability Tokens, cryptographic chain verification) but nothing has shipped as an actual product.

---

## 2. The core mechanic

Three things, layered on top of the OBO exchange both clouds already do natively — not replacing it:

1. **Chain Integrity Enforcement.** Every time a token is exchanged for the next hop, Attenuate cryptographically signs a claim proving the new token's permission set is a *subset* of the previous hop's. If a hop ever comes back claiming *more* access than it started with — the exact shape of the Cross-Agent Privilege Escalation and Agent Session Smuggling attacks — it's rejected immediately, chain-wide, not just at that one hop.
2. **Cross-Cloud Identity Bridge.** At the exact point a delegation chain crosses from Azure (Entra OBO tokens) to AWS (AgentCore IAM/SigV4) or back, Attenuate translates between the two token formats while preserving the attenuation guarantee from #1. If the translation can't preserve that guarantee, the call fails **safely and legibly** — a clear "this hop couldn't prove it wasn't escalating" message — instead of either an opaque 401 or, worse, silently falling back to the orchestrator's own broad service credentials (the actual dangerous failure mode today).
3. **Unified Cross-Vendor Audit Trail.** Every hop's attenuation proof is stitched into one timeline, regardless of which cloud or authorization server handled it — so "which human ultimately authorized this specific tool call, and through how many agents and clouds did it travel to get here" is a single, legible answer instead of three separate logs in three different formats.

---

## 3. Full feature set

| Feature | What it does |
|---|---|
| **Attenuation Proof** | A cryptographic claim attached at every hop, proving this token's permission set is a strict subset of the one before it. This is the core novel piece — everything else is enforcement and visibility around it. |
| **Escalation Trap** | The moment a hop's proof fails to verify (permissions grew instead of shrank), the entire chain is halted and the triggering hop is flagged — this is what catches Cross-Agent Privilege Escalation and Agent Session Smuggling live. |
| **Cross-Cloud Bridge** | Translates Entra Agent ID OBO tokens ↔ AgentCore IAM/SigV4 tokens at a cloud boundary, preserving the attenuation chain across the translation instead of silently dropping it. |
| **Fail-Safe, Not Fail-Open** | If the bridge or a hop can't prove attenuation, the call is denied by default — never falls back to the orchestrator's broad service identity, which is the actual failure mode that makes today's confused-deputy risk dangerous rather than theoretical. |
| **Unified Chain Audit** | One timeline per request, spanning every hop and every cloud, naming the original human and every agent that touched the request on the way. |

Build only the Attenuation Proof + Escalation Trap + one cross-cloud demo hop for the hackathon. That alone is a complete, winnable pitch — the audit trail is a strong bonus if time allows, not a requirement.

---

## 4. Architecture

```
  Human user                                                     
  (real RBAC role,                                               
   e.g. "Region A only")                                         
       │  OAuth token                                            
       ▼                                                          
┌─────────────────────┐        native OBO           ┌─────────────────────┐
│   Orchestrator        │ ───────────────────────▶  │  Entra Agent ID /    │
│   (Azure AI Foundry)  │        (already solved)     │  AgentCore Identity  │
└──────────┬───────────┘                              └─────────────────────┘
           │  A2A call, attenuated token
           │  + Attenuation Proof #1
           ▼
┌─────────────────────┐
│   Attenuate Bridge    │  ← verifies Proof #1, translates Entra OBO
│   (the actual build)  │     token → AgentCore-compatible token,
└──────────┬───────────┘     re-signs Attenuation Proof #2
           │  A2A call, attenuated token
           │  + Attenuation Proof #2
           ▼
┌─────────────────────┐
│   Sub-agent            │  ← if this agent (or an injected instruction
│   (AWS AgentCore)      │     reaching it) tries to act with MORE access
└──────────┬───────────┘     than Proof #2 allows, Attenuate blocks it
           │
           ▼
   Unified Chain Audit: "Region A user → Orchestrator → Sub-agent,
   permissions narrowed at every hop, blocked attempt at hop 3"
```

- **Foundation (real, already exists, don't rebuild):** AWS AgentCore Identity's native OBO (April 2026) and Azure Entra Agent ID's native OBO.
- **The actual build:** the Attenuation Proof format, the verification/signing service at each hop (an interceptor, same shape as the `ext_authz` pattern used elsewhere in this ecosystem), and the cross-cloud token bridge.
- **For a 22-hour build:** an in-memory or lightweight-store proof chain is enough — durable storage is a post-hackathon concern.

---

## 5. The 22-hour build plan

| Hours | Focus | Deliverable |
|---|---|---|
| 0–2 | Setup | Confirm working credentials/API access on both AgentCore Identity and Entra Agent ID; get a single, native OBO call working end-to-end on each cloud separately first, so you know the baseline works before adding anything. |
| 2–7 | Attenuation Proof + single-cloud chain | Define the proof format (a signed claim: previous scope, new scope, proof new ⊆ previous). Get a 2-hop chain working *within one cloud* first (orchestrator → sub-agent → sub-sub-agent, all on Azure Foundry, say) with real verification at each hop. |
| 7–11 | Escalation Trap | Deliberately break the chain (a hop asks for more than it was given) and confirm it's caught and blocked, with a clear reason logged. This is the single most important thing to have working. |
| 11–15 | Cross-Cloud Bridge | Add the AWS AgentCore hop — translate the token at the boundary, re-verify the attenuation proof survives translation. This is the hardest and most novel part; protect this time block above all else. |
| 15–18 | The attack moment | Script and rehearse the live demo attack (§6) — a sub-agent past the cloud boundary tries to escalate beyond what the original human's role allowed, and gets caught in real time. |
| 18–20 | Unified audit view | A simple, readable timeline view stitching both clouds' hop records into one chain, shown live after the blocked attempt. |
| 20–21.5 | Rehearse | Run the full demo start to finish at least three times, timed. |
| 21.5–22 | Buffer | Reserved for anything that broke during rehearsal. |

**If behind schedule, cut in this order:** the unified audit view (fall back to reading each cloud's log side by side) → the single-cloud multi-hop chain from hours 2–7 (collapse to a 1-hop demo per cloud) → never cut the Cross-Cloud Bridge or the Escalation Trap — those two together are the entire pitch. If genuinely out of time, a working 2-hop, cross-cloud chain with one live-blocked escalation attempt is still a complete, winnable demo on its own.

---

## 6. Demo script

**Setup (30 seconds):** "This user's real role is 'Region A viewer only.' Watch what their permission looks like as it travels through three agents across two different clouds."

1. Human logs in with a real, limited Entra role — Region A data only, nothing else.
2. They ask the orchestrator (Azure Foundry) a question that requires delegating to a sub-agent hosted on AWS AgentCore, which itself delegates to a third agent.
3. Live on screen: the Unified Chain Audit view grows in real time, showing the token's scope at each hop — narrower or equal at every step, never wider, visibly crossing from Azure's token format to AWS's at hop 2.
4. **The attack moment:** at the third hop, simulate the real, named Cross-Agent Privilege Escalation pattern — instruct that agent (as if manipulated by a bad prompt or a compromised tool) to request Region B data, which this human was never authorized to see, but which the orchestrator's own broad service identity normally *could* reach.
5. Live: Attenuate's Escalation Trap blocks it — the attempted token fails its attenuation proof against hop 2's scope, chain halted, logged with a plain-English reason: *"This request would have exceeded what the original user's session ever had access to — blocked before it reached AWS."*
6. Close: "Today, in most systems, this exact request either silently succeeds using the orchestrator's own credentials, or fails with an opaque error nobody can diagnose. Here it fails safely, and you can see exactly why."

Target runtime: under 4 minutes.

---

## 7. Why it wins

- **Engineering Excellence & Vision** — this isn't rebuilding OBO (which both clouds already shipped natively in 2026); it's fixing the specific, admitted gap in the standard itself (RFC 8693's own "informational only" language), addressed so far only by unshipped IETF drafts. Building on top of real infrastructure, not reinventing it, while closing a documented protocol-level hole.
- **Creativity & Innovation** — directly defends against two named, dated, real attack classes (Cross-Agent Privilege Escalation, Agent Session Smuggling) that a body as established as SANS Institute is actively flagging as unaddressed. Nobody's shipped this as a product.
- **Viability & Execution Plan** — the foundation (single-hop OBO on each cloud) already works today; Attenuate is a focused verification-and-bridging layer on top, not new ground-up infrastructure. Scoped tightly for 22 hours per §5, with an explicit fallback plan if behind.
- **GTM Strategy** — "confused deputy" is a formal, decades-old term enterprise security teams already recognize and take seriously, which makes this the easiest of any idea in this project to explain to a CISO in one sentence. Maps directly onto identity and access management (IAM) — one of the oldest, most established, highest-trust service lines any IT integrator already sells — as "we already manage your IAM; now we extend it to your AI agents." Sellable immediately into any client running multi-agent systems across more than one cloud, which is exactly the audience the hackathon's own "any combination" language is written for.

---

## 8. Team split (if two builders)

- **Builder A — Chain mechanics:** Attenuation Proof format, signing/verification, the single-cloud multi-hop chain, the Escalation Trap.
- **Builder B — Cross-cloud + demo:** the Azure↔AWS token bridge, the Unified Chain Audit view, the attack script, rehearsal, and pitch narration. Owns cutting scope per §5 if time runs short.

---

## 9. What to say if a judge asks "doesn't OAuth already handle this?"

OAuth's On-Behalf-Of flow handles one hop correctly — it's designed for a middle-tier service calling one downstream service on a user's behalf, and both AWS and Azure ship that natively today. What it was never designed for is a *chain* of several agents delegating to each other, especially across two different companies' clouds. The standard itself (RFC 8693) says outright that its record of who-delegated-to-whom is informational only — nothing in OAuth cryptographically stops permissions from growing instead of shrinking as a request passes through a second or third agent. That's not a configuration mistake teams are making; it's a gap in the standard itself, which is exactly why real, named attacks already exploit it and why security researchers are calling this out publicly right now.
