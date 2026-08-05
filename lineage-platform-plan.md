# Lineage — Every Agent's Authority Traces Back to a Human, and Only Ever Narrows

**One-line pitch:** Lineage combines two mechanisms into one platform — it automatically computes the narrowest permission scope a sub-agent needs for the task it was actually given, then cryptographically guarantees that scope can only shrink (never grow) as it passes through more agents, more hops, and even across cloud or company boundaries, tied back at every step to a real human's real permissions. Nobody — not AWS, not Azure, not any funded competitor — has shipped this combination yet.

**Themes:** Security & Governance at Scale (primary), with a genuine Future of AI UX thread in the live, explainable permission-tree visualization.

**Built on:** AWS Bedrock AgentCore Identity and Azure AI Foundry / Entra Agent ID, together — a real cross-cloud story, using the hackathon's "any combination" language directly.

---

## 1. The problem, precisely

Multi-agent systems work by delegation: a human asks an orchestrator for something, the orchestrator breaks it into pieces and hands pieces to sub-agents, which may hand pieces to further sub-agents, possibly on a different cloud or even a different company's infrastructure entirely. Two things go wrong today, and they compound each other:

**Nobody scopes sub-agents down automatically.** A developer has to hand-write a role for every sub-agent, in advance, or — far more common — every sub-agent just inherits the orchestrator's own broad access because narrowing it by hand isn't worth the engineering time. A 2026 academic paper (arXiv:2605.05440) states plainly that classical access-control models — RBAC, ABAC, ReBAC — don't fully address how authorization should propagate through a chain of delegating AI agents; it's an open research problem, not solved practice.

**Nobody verifies that permissions only shrink as they pass through more hops.** OAuth's On-Behalf-Of flow now works natively on both clouds for a single hop (AWS AgentCore Identity added it in April 2026; Azure Entra Agent ID already had it) — but the underlying standard, RFC 8693, says outright that its record of who-delegated-to-whom is "informational only." Nothing stops a token from coming back with *more* access after a second or third hop. Two real, named, dated attacks already exploit exactly this: **Cross-Agent Privilege Escalation** (Sept 2025) and **Agent Session Smuggling** (Nov 2025).

**This isn't a theoretical risk — it's the majority failure mode in real breaches, right now:**
- A 2026 breach post-mortem found **78% of agents involved in breaches had significantly broader permission scopes than their function required.**
- Agent-involved breaches grew **340% year-over-year** (2024→2025), and **88% of organizations** report a confirmed or suspected AI agent security incident in the past year.
- Only **44%** of organizations have implemented *any* policy to manage their AI agents, even though **92%** agree it's critical. Only **21%** report mature agent governance. Over **a third** admit they couldn't shut down a rogue agent today if they had to.

**The competitive landscape confirms the gap is still open, and the space is heating up fast.** Arthur AI's "Agentic Discovery and Governance" platform, Act Security ($60M raised), Thoughtworks' "Agent/works" (launched June 2026), and Microsoft's own "Project Perception" (launched July 27, 2026) are all racing toward comprehensive agent governance — none of them ship automatic sub-agent scope derivation or cross-hop delegation-chain integrity verification. The market itself is consolidating around this problem: in one 72-hour window in late July 2026, Okta acquired Permiso Security ($200M) and Cyera acquired Oasis Security ($1B) — real money confirming this is a real, urgent category, not a niche.

---

## 2. The core mechanic

One continuous chain, six steps, each closing a specific gap confirmed above:

1. **The human's real permissions are the ceiling.** When a person calls the orchestrator, their existing OAuth/RBAC identity — already natively supported by both clouds — sets the absolute upper bound. Nothing downstream can ever exceed it.
2. **Scope Compiler derives the narrowest scope for each sub-agent.** From the specific task and tool manifest being delegated — not the parent's full grant — Lineage computes the minimal permission set that still lets the sub-agent do its job, capped at whatever the original human's role permits.
3. **Semantic intent verification, not just keyword matching.** Following a very recent proposal (SentinelAgent, arXiv:2604.02767, proposing "intent-verified delegation chains"), the Scope Compiler checks that a sub-agent's *actual* requests semantically match the task it claims to be doing — catching a sub-agent that says it's drafting an email but is actually asking for refund-tool access.
4. **Attenuation Proof travels with every hop.** Each scope is cryptographically signed as a strict subset of the hop before it. If a hop ever comes back claiming more access than it left with — the exact shape of both named 2025 attacks — the chain halts immediately.
5. **Enforcement happens off-host.** Following another very recent proposal (aiAuthZ, arXiv:2607.05518, "off-host, identity-bound authorization"), the decision point that checks every tool call and retrieval lives *outside* the agent's own runtime — so a compromised agent can't simply talk itself past its own guardrails, because it never held the authority to grant itself access in the first place.
6. **Scope dies with the task, fails safe, bridges any boundary.** A derived scope expires automatically the moment its task ends. If a chain crosses a cloud boundary (Azure↔AWS) or an organizational boundary (your agent ↔ a vendor's agent, no shared identity provider), the bridge either preserves the full attenuation guarantee or fails closed with a clear reason — never silently falls back to broader access.

---

## 3. Full feature set

### 3.1 Core engine — build this first, this is the whole pitch

| Feature | What it does |
|---|---|
| **Delegation Ledger** | Every spawn/delegate event becomes a node in a live tree: who delegated to whom, what task, what scope was derived, when it expires. |
| **Scope Compiler** | Task + tool manifest + human's RBAC ceiling → minimal derived scope. Start with keyword/embedding matching; add the semantic intent check (step 3 above) if time allows. |
| **Attenuation Proof** | A signed claim at every hop proving the new scope is a strict subset of the one before it — the core novel piece defending against both named 2025 attack classes. |
| **Off-Host Enforcement Interceptor** | An `ext_authz`-style gate, deliberately hosted outside any agent's own runtime, checking every tool call/retrieval against the current proof. Allow or deny, no exceptions, un-tamperable by a compromised agent. |
| **Escalation Trap** | The moment a proof fails to verify, the whole chain halts and the triggering hop is flagged, logged, and explained. |
| **Auto-Expiry** | A derived scope dies automatically when its task completes, times out, or the parent session ends. |

### 3.2 Identity extensions — build if the core is solid with time to spare

| Feature | What it does |
|---|---|
| **Cross-Cloud Bridge** | Translates Entra Agent ID OBO tokens ↔ AgentCore IAM/SigV4 tokens at an Azure↔AWS boundary, preserving the attenuation proof across the translation. |
| **Cross-Organization Bridge** | Extends the same guarantee to a delegation chain reaching a *different company's* agent with no shared identity provider at all — using W3C Decentralized Identifiers and Verifiable Credentials (the same pattern behind TRAIL, a draft AI-agent identity spec, and MCP-I, donated to the Decentralized Identity Foundation in March 2026). This is the genuinely futuristic beat of the demo, if time allows: your agent and an unaffiliated vendor's agent establishing trust with zero shared infrastructure. |

### 3.3 Defense modules — each a self-contained demo beat, add in priority order

| Module | What it adds |
|---|---|
| **Tainted Delegation Check** | A task description sourced from untrusted content (an email, a scraped page) gets a narrower default scope and a review flag before the Scope Compiler even runs. |
| **Memory Guard** | If a derived scope was influenced by long-term agent memory, that memory entry is tagged, timestamped, and rollback-able — a poisoned memory can't quietly widen future delegations. |
| **Retrieval Lens** | Extends enforcement to retrieval, not just tool calls, with a visible "why was this retrievable" explanation. Extend further with a differential-privacy option: if a task only needs an aggregated view, the agent gets a noised/anonymized result instead of the raw record — a real, current requirement per the EU Data Protection Board's Q1 2026 guidance on automated agent processing. |
| **Honeytokens** | Unique, non-secret strings planted inside protected content that a legitimate agent would never touch. If one ever surfaces in an agent's output or a downstream call, it proves unauthorized access with zero false positives — lightweight to build, pairs directly with Memory Guard and Retrieval Lens. |
| **Fleet-Wide Kill Switch** | One control that instantly halts every agent across the whole delegation graph — not just one session — in response to a detected compromise. Genuinely unsolved today: no vendor has shipped sub-100ms, fail-closed, fleet-wide termination as of mid-2026, and there's a live legislative hook (the bipartisan "AI Kill Switch Act," introduced July 23, 2026, would require exactly this capability from major AI developers). |
| **Regression Watch** | Replays saved past delegations through the current build of the Scope Compiler and flags if something that used to be correctly denied now incorrectly passes. |

### 3.4 Compliance layer — the easiest "judge asks about GTM" answer of anything in this project

| Feature | What it does |
|---|---|
| **One-Click Article 12 Evidence Pack** | Turns the platform's own delegation/attenuation audit trail directly into a regulator-ready evidence pack for the EU AI Act's Article 12 record-keeping requirement — fully binding since August 2, 2026. A live, current pain point: point solutions exist for single-model inference logs, but nobody turns a *multi-agent delegation trail* specifically into this. |
| **Explain-This-Denial (compliance view)** | A separate, non-technical view of every blocked action — "what was requested, why it was denied, who was accountable" — aimed at an auditor or compliance officer, distinct from the developer-facing log. |
| **Unified Chain Audit** | One legible timeline per request, spanning every hop, cloud, and (if built) organization boundary — answering "which human ultimately authorized this" in one view instead of three fragmented per-system logs. |

**Illustrative example worth using in the pitch:** three separate agents, each independently approving a small budget allocation within its own authorized limit, whose *combined* effect exceeds what any single one of them was ever authorized to approve. Single-agent risk assessment (and single-agent RBAC) never catches this — it's a live, citable reason multi-agent governance has to be a first-class concept, not an extension of old tooling.

### 3.5 Roadmap — say these out loud, don't fake them live

Be explicit with judges that these are real, evidenced, and deliberately *not* claimed as working in the 22-hour build — this is a credibility move, not a weakness:

- **Hardware attestation (TEEs/confidential computing)** to prove an agent is running unmodified code before it's trusted — real and emerging (AWS Nitro Enclaves, Azure confidential computing), but too heavy to stand up live.
- **Behavioral fingerprinting** to catch an agent whose behavior pattern subtly changed even within its allowed scope — real technique, but needs 7–30 days of baseline data, so a live demo would have to fake it. Don't.
- **Zero-knowledge proof authorization** — proving "authorized for X" without revealing the underlying policy — real emerging research (an active Microsoft Research project called Vega, a new open protocol called x401), genuinely promising, but still proof-of-concept stage as of mid-2026.
- **Post-quantum-ready token format** — a design nod (build the Attenuation Proof format swappable to post-quantum algorithms later), not a working feature.

---

## 4. Architecture

```
  Human user (real OAuth/RBAC role — the ceiling for everything below)
       │
       ▼
┌─────────────────────┐   native OBO    ┌──────────────────────┐
│  Orchestrator          │ ─────────────▶ │  Entra Agent ID /      │
│  (Azure AI Foundry)    │  (already       │  AgentCore Identity     │
└──────────┬───────────┘   solved)        └──────────────────────┘
           │  task + tool manifest
           ▼
┌─────────────────────┐
│  Scope Compiler         │  task-derived scope, capped at human's ceiling,
│  (+ semantic intent      │  semantically checked against stated intent
│   check)                 │
└──────────┬───────────┘
           │  derived scope + Attenuation Proof #1
           ▼
┌─────────────────────┐
│  Cross-Cloud / Cross-   │  translates token format at a cloud or org
│  Org Bridge              │  boundary, preserves the attenuation guarantee
└──────────┬───────────┘   or fails closed
           │  Attenuation Proof #2
           ▼
┌─────────────────────┐
│  Off-Host Enforcement   │  lives OUTSIDE the sub-agent's own runtime —
│  Interceptor              │  a compromised agent can't grant itself access
└──────────┬───────────┘
           │  allow / deny
           ▼
   Sub-agent (AgentCore or Foundry, possibly a different company entirely)
           │
           ▼
   Delegation Ledger + Unified Chain Audit + One-Click Article 12 Pack
```

- **Foundation, don't rebuild:** native single-hop OBO on both clouds.
- **The actual build:** Scope Compiler, Attenuation Proof, off-host Enforcement Interceptor, the cross-boundary bridge.
- **For 22 hours:** in-memory Ledger and proof store is enough; durability is a post-hackathon concern.

---

## 5. The 22-hour build plan

| Hours | Focus | Deliverable |
|---|---|---|
| 0–2 | Setup | Working credentials on both clouds; one native OBO call confirmed on each, separately, before building anything on top. Lock the demo scenario (§6) now. |
| 2–6 | Scope Compiler v1 | Task + tool manifest → derived scope, keyword/rule-based to start. Get it working end-to-end within one cloud before adding anything else. |
| 6–10 | Attenuation Proof + off-host Enforcement Interceptor | The proof format, the signing/verification at each hop, and a real `ext_authz`-style gate hosted outside the agent process. This is the moment the pitch becomes real — prioritize above all else. |
| 10–13 | Escalation Trap + Auto-Expiry | Deliberately break a chain (a hop asks for more than it was given) and confirm it's caught, logged, and explained. Confirm scope dies when a task ends. |
| 13–16 | Cross-Cloud Bridge | Add the AWS↔Azure hop, confirm the proof survives translation. Hardest and most novel part — protect this time block. |
| 16–18 | Pick ONE defense module + ONE compliance feature | Recommended: Honeytokens (cheapest, most demo-friendly) + One-Click Article 12 Evidence Pack (strongest GTM beat) — build only these two, list the rest as roadmap. |
| 18–20 | The attack moment + visualization | Script and rehearse the live escalation attempt; build the live Delegation Ledger tree view so judges *see* it happen, not just hear about it. |
| 20–21.5 | Rehearse | Full run-through at least three times, timed. |
| 21.5–22 | Buffer | Reserved for whatever broke. Do not add new features here. |

**If behind schedule, cut in this order:** Cross-Organization Bridge (never build it live, mention as roadmap) → the compliance feature from hours 16–18 → the defense module from hours 16–18 → the tree visualization (fall back to narrating the Ledger's log lines). **Never cut the Cross-Cloud Bridge or the Escalation Trap — those two together are the entire pitch.** A working 2-hop, cross-cloud chain with one live-blocked escalation attempt is, on its own, a complete and winnable demo.

---

## 6. Demo script

**Setup (30 seconds):** "This user's real role is Region A viewer only. Watch what their permission looks like as it's delegated through three agents across two different clouds."

1. Human (real, limited Entra role) asks the orchestrator (Azure Foundry) a question requiring delegation to a sub-agent on AWS AgentCore, which delegates further to a third agent.
2. Live: the Delegation Ledger tree grows in real time, showing each hop's derived scope narrowing, and visibly crossing token formats at the cloud boundary.
3. **The attack moment:** instruct the third agent — as if manipulated by a bad prompt or compromised tool — to request Region B data, which this human was never authorized to see but which the orchestrator's own broad service identity normally could reach. This is the exact shape of the real, named Cross-Agent Privilege Escalation attack.
4. Live: the off-host Enforcement Interceptor blocks it — the escalation is caught chain-wide, with a plain-English reason: *"This request would have exceeded what the original user's session ever had — blocked before it reached AWS."*
5. **If time allows, a second beat:** hit the Fleet-Wide Kill Switch and show every live agent in the graph — orchestrator and all sub-agents, across both clouds — halt instantly, not just the one session.
6. Close: "Today, that escalation attempt either silently succeeds using the orchestrator's own credentials, or fails with an opaque error nobody can diagnose. Here, it fails safely, you can see exactly why, and if it needed to, we could have stopped the entire fleet in one click."

Target runtime: under 5 minutes including the optional kill-switch beat.

---

## 7. Why it wins

- **Engineering Excellence & Vision** — combines two real, distinct, currently-unsolved gaps (automatic scope derivation, and cross-hop chain integrity) into one coherent mechanism, grounded in four separate 2026 papers (arXiv:2605.05440, SentinelAgent, aiAuthZ, plus the decentralized-identity specs), not a wrapper around an existing feature.
- **Creativity & Innovation** — confirmed absent from every named competitor, old and brand-new: Arthur AI, Act Security, Thoughtworks' Agent/works (June 2026), and Microsoft's own Project Perception (July 27, 2026). Nobody's shipped this combination.
- **Viability & Execution Plan** — built on real, callable, already-shipped native OBO on both clouds; the interceptor pattern is a known, proven shape. Scoped with an explicit, defensible fallback plan if time runs short (§5).
- **GTM Strategy** — the numbers make the pitch for you: 78% of breached agents had excessive scope, 340% year-over-year breach growth, only 21% of organizations have mature governance, and the market is consolidating around this exact category (Okta/Permiso, Cyera/Oasis, both within the same week). Sellable immediately as an extension of Presidio's existing IAM and compliance consulting — "we already manage your identity and your audit trail; now we extend both to your AI agents."

---

## 8. Team split (if two builders)

- **Builder A — Chain mechanics:** Scope Compiler, Attenuation Proof, off-host Enforcement Interceptor, Escalation Trap, single-cloud multi-hop chain.
- **Builder B — Boundaries + demo:** Cross-Cloud Bridge, whichever module/compliance feature gets picked in hours 16–18, the Delegation Ledger visualization, the attack script, rehearsal, and pitch narration. Owns cutting scope per §5.

---

## 9. Anticipated questions

**"Isn't this just RBAC?"** No — classical RBAC assigns a fixed role to an identity in advance. Lineage computes a scope *at the moment of delegation*, from the actual task being handed off, cryptographically proves it can only narrow at every further hop, and expires it automatically. That's a workflow-aware, time-limited mechanism — the exact distinction a 2026 paper on this problem draws explicitly.

**"Doesn't OAuth already handle this?"** OAuth's On-Behalf-Of flow handles one hop correctly, and both clouds ship that natively today. It was never designed for a *chain* of several agents delegating to each other, especially across two companies' infrastructure — the standard itself admits its delegation record is informational only, which is exactly why two real, named attacks already exploit that gap.

**"Why does enforcement have to live off-host?"** Because if the decision point lives inside the same agent whose access it's supposed to be limiting, a compromised agent can eventually talk itself past its own guardrails. Keeping authorization external and identity-bound — a principle a paper published just weeks ago (aiAuthZ) argues for directly — means the blast radius of a single compromised agent stays bounded no matter what that agent is tricked into doing.

**"Is the cross-organization piece real, or a stretch?"** It's real and moving fast (a draft W3C identity spec for AI agents, and a related protocol donated to the Decentralized Identity Foundation earlier this year), but it's the one piece of this platform explicitly marked as a stretch goal, not a core claim — say so plainly if asked, rather than overselling it.

---

## 10. Evidence log

- arXiv:2605.05440 — authorization propagation gap in multi-agent systems
- SentinelAgent, arXiv:2604.02767 — intent-verified delegation chains
- aiAuthZ, arXiv:2607.05518 — off-host, identity-bound authorization
- RFC 8693 — "prior-actor" claims are informational only
- Cross-Agent Privilege Escalation (Sept 2025) and Agent Session Smuggling (Nov 2025) — named real attacks
- AWS Bedrock AgentCore Identity native OBO support (April 2026); Azure Entra Agent ID native OBO support
- SANS Institute — "Your AI Agent Is an Easily Confused Deputy: Why Cloud Security Needs a Credential Broker"
- 2026 breach post-mortem — 78% of breached agents had excessive permission scope; 340% YoY breach growth; 88% of orgs report an incident in the past year
- Market maturity stats — 44% have any policy vs. 92% say it's critical; only 21% mature governance; over a third couldn't shut down a rogue agent today
- "AI Kill Switch Act," introduced July 23, 2026 — legislative hook for the Fleet-Wide Kill Switch feature
- EU AI Act Article 12 — fully binding since August 2, 2026
- EU Data Protection Board Q1 2026 guidance — data minimization applies to automated agent processing
- ISO 42001 — fewer than 100 organizations certified worldwide as of January 2026
- TRAIL (draft W3C AI agent identity spec) and MCP-I (donated to the Decentralized Identity Foundation, March 2026) — cross-organization identity
- Competitive landscape: Arthur AI (Agentic Discovery and Governance), Act Security ($60M raised), Thoughtworks Agent/works (June 2026), Microsoft Project Perception (July 27, 2026) — none ship this combination
- Recent consolidation: Okta acquired Permiso Security ($200M) and Cyera acquired Oasis Security ($1B), both in the same week in late July 2026
