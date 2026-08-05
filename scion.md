# Scion — Permissions That Inherit, and Narrow, Automatically

**One-line pitch:** When an AI agent delegates a task to a sub-agent, Scion automatically computes a narrower, enforced permission scope for that sub-agent from the delegation itself — no developer hand-writes a separate IAM role per sub-agent, and no sub-agent can ever end up with more access than the task it was actually given.

**Themes:** Security & Governance at Scale (primary), with a genuine Future of AI UX angle in how the permission tree is visualized live.

**Built on:** AWS Bedrock AgentCore Identity (primary) and/or Azure AI Foundry / Entra Agent ID (secondary/stretch, for a cross-cloud demo moment).

---

## 1. The problem, precisely

Multi-agent systems work by delegation: an orchestrator agent breaks a task into pieces and hands each piece to a sub-agent. Today, every platform gives the orchestrator and its sub-agents identity — but permission scoping between them is entirely manual. A developer has to decide, in advance, exactly what IAM role or policy to attach to each sub-agent, by hand, before it ever runs.

This is confirmed, not assumed:

- A 2026 academic paper (arXiv:2605.05440) states plainly that classical access-control models — RBAC, ABAC, ReBAC — do not fully address how authorization should propagate through a chain of delegating AI agents. It proposes this needs to be treated as a property of the *workflow* (tracking delegation chains, permission aggregation, time-limited validity), not a static role assignment. This is an open research problem today, not solved practice.
- AWS Bedrock AgentCore Identity lets you scope an IAM role to each agent component — but it's per-component, hand-configured. Nothing computes a sub-agent's scope automatically from what it was actually delegated.
- Two well-funded 2026 entrants — Arthur AI's "Agentic Discovery and Governance" platform and Act Security ($60M raised for agent access-boundary enforcement) — are both racing toward comprehensive agent governance, but neither has shipped automatic hierarchical scope derivation specifically. They're enterprise dashboards (discovery, guardrails, audit logs), not this mechanism.
- The market gap backs the urgency: only 44% of organizations have implemented *any* policy to manage their AI agents, even though 92% agree it's critical.

The consequence of the gap: in practice, teams either (a) spend hours hand-writing a role per sub-agent and get it wrong, or — far more commonly — (b) give every sub-agent the same broad role as the orchestrator, because narrowing it by hand isn't worth the engineering time. That second failure mode is the real, everyday risk: a sub-agent whose only job is drafting an email can, today, usually also issue a refund, because nobody scoped it down.

---

## 2. The core mechanic

Three rules, enforced automatically, every time an agent delegates to another agent:

1. **Never exceed the parent.** A sub-agent's scope is always a subset of what its parent actually holds — mathematically impossible to escalate through delegation.
2. **Narrow at every level.** A sub-agent's scope is derived from *what it was actually asked to do* (the specific task description and tool manifest passed at delegation time), not the parent's full grant. A sub-sub-agent narrows further still.
3. **Expire with the task.** A derived scope is valid only for the lifetime of the delegated task. The moment the sub-agent reports done, times out, or the parent session ends, the scope is revoked — automatically, not on the next audit cycle. (This absorbs what would otherwise be a separate "auto-revoke credentials" feature — it's the same mechanism, not a bolt-on.)

The engineering core is a **Scope Compiler**: a function that takes (a) the parent's current scope, (b) the natural-language or structured task being delegated, and (c) the tool manifest available to the sub-agent, and outputs the minimal derived scope that still lets the sub-agent complete the task. This is the genuinely novel piece — everything else in this document is enforcement and visibility around that one function.

---

## 3. Full feature set

### 3.1 Core engine (build this first — this is the whole pitch)

| Feature | What it does |
|---|---|
| **Delegation Ledger** | Every time one agent spawns or delegates to another, it's recorded as a node in a live tree: who delegated to whom, what task, what tools, what scope was derived, when it expires. |
| **Scope Compiler** | Takes the parent scope + delegated task + tool manifest, outputs the derived child scope. Start simple (task → required tool subset, via keyword/embedding match against tool descriptions) and only add an LLM-as-judge pass if time allows for handling ambiguous delegations. |
| **Enforcement Interceptor** | Sits in front of every tool call a sub-agent makes (an `ext_authz`-style hook, same pattern used by agentgateway) — checks the call against that agent's current derived scope from the Ledger, allow or deny, no exceptions. |
| **Auto-Expiry** | A background sweep that revokes any derived scope whose parent task has completed, timed out, or whose parent session has ended. |

### 3.2 Modules (already-researched, already-evidenced — add in this order if time allows, each is a self-contained demo beat)

| Module | What it adds | Where it plugs in |
|---|---|---|
| **Escalation Trap** (from *Circuit Breaker* / *Agent Interlock*) | Detects a sub-agent retrying a denied call repeatedly (a live privilege-escalation attempt) and flags/kills the session instead of letting it hammer the interceptor. | Wraps the Enforcement Interceptor |
| **Tainted Delegation Check** (from *Chain of Custody*) | If the *task description* being delegated came from untrusted content (an email, a scraped page) rather than a verified source, the Scope Compiler treats it with extra suspicion — narrower default, flagged for review. | Feeds into the Scope Compiler |
| **Memory Guard** (from *Ghost Memory*) | If a sub-agent's derived scope was influenced by something in long-term memory, that memory entry is tagged, timestamped, and rollback-able — so a poisoned memory can't quietly widen future delegations. | Sits beside the Delegation Ledger |
| **Retrieval Lens** (from *Permission Lens*) | Extends scope enforcement to *retrieval*, not just tool calls — a sub-agent's derived scope also filters what documents/data it's allowed to retrieve, with a visible "why was this retrievable" explanation. | Extends the Enforcement Interceptor |
| **Regression Watch** (from *Report Card* / *Agent CI*) | Replays a saved set of past delegations through the current build of the Scope Compiler and flags if a derivation that used to correctly deny something now incorrectly allows it. | Runs offline, shown as a CI badge in the demo |

**Build only the core (3.1) plus one or two modules for the hackathon.** The core alone is a complete, winnable pitch. Modules exist to show the platform has legs beyond a single demo, not to all be built.

---

## 4. Architecture

```
                     ┌─────────────────────┐
   Orchestrator ───▶ │  Delegation Ledger   │◀─── records every
   Agent             │  (live tree, TTLs)   │     spawn/delegate event
                     └──────────┬───────────┘
                                │
                                ▼
                     ┌─────────────────────┐
   Task + tools ───▶ │   Scope Compiler     │───▶ derived scope
   being delegated   │ (parent ∩ task-need) │     for sub-agent
                     └──────────┬───────────┘
                                │
                                ▼
              ┌───────────────────────────────┐
   Sub-agent  │   Enforcement Interceptor      │
   tool call ▶│   (ext_authz-style gate)       │──▶ allow / deny
              └───────────────────────────────┘
                                │
                                ▼
                     ┌─────────────────────┐
                     │  AgentCore Identity   │  ← real IAM role scoped
                     │  or Entra Agent ID    │    per component, today
                     └─────────────────────┘
```

- **Identity layer (real, already exists):** AWS AgentCore Identity or Azure Entra Agent ID — each agent already gets its own identity and a scopable role. Scion sits *on top of* this, not replacing it — the derived scope maps onto that role's actual permission set, so the enforcement is real, not cosmetic.
- **Interceptor pattern:** Same shape as agentgateway's `ext_authz` hook (already confirmed as the sanctioned "bring your own logic" extension point) — a lightweight service every tool call passes through before it's allowed to execute.
- **Ledger storage:** For a 22-hour build, an in-memory tree with a simple event log is enough — durability/Postgres backing is a post-hackathon concern, not a demo requirement.
- **Cross-cloud stretch goal:** If time allows, run the orchestrator on one cloud and one sub-agent on the other (AgentCore ↔ Foundry via A2A), and show the Delegation Ledger tracking scope narrowing *across* the cloud boundary — genuinely nobody does this today, and it directly uses the "any combination" language in the hackathon rules.

---

## 5. The 22-hour build plan

| Hours | Focus | Deliverable |
|---|---|---|
| 0–2 | Setup | Repo, cloud accounts/credentials confirmed working, AgentCore Identity API calls tested end-to-end with a "hello world" scoped role. Pick the demo scenario now (see §6) and don't change it later. |
| 2–6 | Scope Compiler v1 | Task description + tool manifest → derived scope, using simple rule/keyword matching (task mentions "refund" → only refund-related tools included). No ML yet — get the mechanism working end-to-end first. |
| 6–10 | Delegation Ledger + Enforcement Interceptor | Wire the compiler's output into an actual `ext_authz`-style gate in front of real tool calls. This is the moment the pitch becomes real instead of a diagram — prioritize this over anything else. |
| 10–13 | Auto-Expiry + demo scenario agents | Build the three demo sub-agents (billing lookup, refund, email draft — see §6) and confirm each one's derived scope is visibly different and each one's access dies when its task ends. |
| 13–16 | The attack moment | Script and test the live privilege-escalation attempt (email agent tries to call the refund tool) and confirm it's blocked and logged, with a visible "why" in the Ledger. This is the single most important thing to have working — everything else is polish around it. |
| 16–19 | One module + visualization | Pick ONE module from §3.2 (Escalation Trap is the easiest and most demo-relevant — it directly extends the attack moment) and build a live tree view of the Delegation Ledger so judges can *see* the scope narrowing in real time, not just hear about it. |
| 19–21 | Rehearse | Run the full demo script (§6) start to finish at least three times. Time it. Cut anything that isn't load-bearing. |
| 21–22 | Buffer | Reserved for whatever broke during rehearsal. Do not schedule new features into this window. |

**If behind schedule, cut in this order:** cross-cloud stretch goal → the module from hours 16–19 → the tree visualization (fall back to reading the Ledger's log lines out loud) → auto-expiry (keep it in the pitch as "the next thing we'd build," don't fake it). **Never cut the attack moment** — it's the whole demo.

---

## 6. Demo script

**Setup (30 seconds):** "This is a customer-service orchestrator with broad account access. Watch what happens when it delegates."

1. Orchestrator receives a real customer request: *"I was double-charged, please fix it and let me know."*
2. Live on screen, the orchestrator spawns three sub-agents and the Delegation Ledger tree grows in real time:
   - **Billing Lookup** — scope: read billing records for this customer only. Nothing else.
   - **Refund Processor** — scope: issue a refund up to the disputed amount, for this customer, this transaction only.
   - **Email Drafter** — scope: draft (not send) a follow-up email. No billing or refund access at all.
3. Narrate: "Nobody wrote three IAM roles by hand. This was derived from the task each agent was actually given."
4. **The attack moment:** Manually instruct the Email Drafter — the least-privileged of the three — to try calling the refund tool directly, as if it had been compromised or manipulated by a bad prompt.
5. Live: the Enforcement Interceptor blocks it. The Delegation Ledger logs the denial with a visible reason: *"Refund tool not in derived scope for Email Drafter — task never required it."*
6. Close: "That's not a rule someone wrote for this specific attack. It's just what happens when permissions are computed from the task instead of assigned by hand."

Target runtime: under 4 minutes, leaving time for questions.

---

## 7. Why it wins (against the actual judging criteria)

- **Engineering Excellence & Vision** — the core mechanism answers a problem a 2026 academic paper explicitly says isn't solved by existing access-control models. This isn't a wrapper around an existing feature; it's new logic.
- **Creativity & Innovation** — confirmed absent from both a $60M-funded competitor (Act Security) and an enterprise incumbent (Arthur AI), and absent from AWS/Azure's own native tooling. Genuinely nobody's shipped this specific mechanism.
- **Viability & Execution Plan** — builds directly on a real, callable API (AgentCore Identity's per-component role scoping) rather than inventing new infrastructure; the interceptor pattern is a known, proven shape (same as agentgateway's `ext_authz`). Scoped tightly enough for 22 hours per §5.
- **GTM Strategy** — the 44%-have-a-policy vs. 92%-say-it's-critical gap *is* the sales pitch in one sentence. Sellable as a managed "Agent Governance-as-a-Service" line extending Presidio's existing IAM/security consulting relationship — the same motion used for MDR contracts, just pointed at agent delegation instead of network traffic.

---

## 8. Team split (if two builders)

- **Builder A — Core mechanism:** Scope Compiler, Delegation Ledger, Enforcement Interceptor, AgentCore Identity integration. This is the load-bearing half; protect this person's time above everything else.
- **Builder B — Demo surface:** the three demo agents, the attack script, the live tree visualization, rehearsal, and the pitch narration. Also owns cutting scope per §5 if things run behind.

---

## 9. What to say if a judge asks "isn't this just RBAC?"

No — classical RBAC assigns a role to an identity in advance, by a human, and it stays fixed until someone changes it. Scion computes the role *at the moment of delegation*, from the task actually being handed off, and it expires automatically when that task ends. The 2026 paper cited in §1 makes exactly this distinction: RBAC is static and identity-centric; what multi-agent delegation actually needs is workflow-aware, time-limited, and automatically derived. That's the whole difference, and it's the reason nobody's shipped it yet — it's a genuinely different mechanism, not a better dashboard for the old one.
