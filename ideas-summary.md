# Presidio Hackathon — All Ideas Discussed So Far

Themes: **Future of AI UX** and **Security & Governance at Scale**. Every idea below is grounded in a real, evidenced pain point (incident, CVE, survey, or documented complaint) and built on Azure AI Foundry or AWS Bedrock/AgentCore, unless noted otherwise.

---

## Round 1 — Two Verified Problems

1. **Chain of Custody** (Security) — AI agents that read untrusted content (an email, a pod log) have no way to stop that content from later leaking data through a tool call. Confirmed via three real incidents (EchoLeak, GitHub MCP exploit, Supabase leak) plus a live Kubernetes exploit (CVE-2026-47250).
2. **Vigil** (UX) — Humans approving agent actions rubber-stamp within weeks as volume outpaces attention, so "human oversight" quietly stops working. Backed by a 2026 academic paper and real practitioner complaints; nobody currently measures reviewer attention itself.

---

## Round 2 — The Wedge List (developer-tool platform angle)

3. **Glass Box** — MCP tool builders have no real dev/test loop; the official Inspector shows protocol frames, not their own server's logs. A local proxy that lets you test against the real runtime instead of a simulation.
4. **Flight Recorder** — Multi-agent failures produce unreadable, giant logs. A checkpoint-based, rewindable timeline for stepping through and replaying what went wrong.
5. **Agent CI** — A real agent silently dropped from 93% to 71% accuracy after an update with no one noticing. A CI gate that replays production traffic and blocks a release if quality drops.
6. **Tool Diet** — Loading MCP tool schemas can burn 72% of a context window before a query even runs. A cross-server proxy that only serves the tools actually relevant to the task.
7. **Cost Governor** — Agent retry loops silently blow budgets ($500 to $4,200 in two weeks, in one real case). Real-time, predictive spend enforcement that pauses a session before it overspends.
8. **Circuit Breaker** — Two agents built on different frameworks can get stuck in an infinite retry loop with each other. A framework-agnostic supervisor that detects and breaks the cycle.
9. **Rehearsal** — Enterprises don't trust agents enough to let them act unsupervised (86% deployed, only 34% trust). An explorable "flight simulator" replaying real traces so stakeholders can test behavior before go-live.
10. **Steady Hands** — Traditional RPA breaks constantly when a legacy screen's UI changes. A computer-use agent (H Company-style) that self-heals through UI changes instead of breaking.
11. **Ops Desk** — Ops teams juggle several different vendors' AI agents with no single place to dispatch or compare them. A Warp/Zed-style orchestrator panel ported from the IDE to the ops desk.
12. **Redline** — Non-coding agent output (a drafted contract, a CRM update) is reviewed as a wall of text with a blind accept/reject. Cursor-style inline diff review, ported to office work instead of code.

---

## Round 3 — Five Real Gaps (kagent / agentgateway / agentregistry angle — later set aside as not the actual target tool stack)

13. **Real Authority** — kagent (a real CNCF project) currently has zero authorization: anyone can control every agent in every namespace, confirmed in its own open bug tracker. Real RBAC plus a natural-language-to-policy AI layer on top.
14. **Second Reader** — Existing MCP gateway defenses against prompt injection are all regex/pattern-based; nobody's wired a real ML classifier natively into the gateway's own extension hook. A semantic classifier that catches paraphrased attacks regex misses.
15. **Trust Badge** — agentregistry (an AI-tool "app store") has no security or trust scoring, by its own admission. An AI scanner that flags suspicious permission requests before a tool is listed.
16. **Report Card** — kagent's own creators had to spin up a separate project because nothing tests whether an agent's behavior regressed after an update. An automatic regression check on every update.
17. **Memory Leak Check** — kagent just shipped long-term agent memory with no proof it can't leak data across tenants. A precautionary auditor for the new shared-memory store.

---

## Round 4 — Worth Watching (fresh, "mesmerizing," live-demo focused)

18. **The Honest Scribe** (UX) — Hospital AI scribes silently mis-transcribe clinical facts (documented wording flips, worse error rates for Black patients) and dodge FDA oversight by being classified as "administrative." A voice agent that visibly pauses and asks for confirmation instead of guessing.
19. **See It My Way** (UX) — AI is being made accessible to other AI crawlers while the National Federation of the Blind reports AI features actively breaking screen readers. A live, interruptible browsing agent that narrates and adapts mid-task for a blind or motor-impaired user.
20. **Caller Not Verified** (Security) — Deepfake voice/video fraud is real and already cost one company $25M; detection accuracy drops to ~45-50% in the real world. An approval agent that catches fakes via a live, unpredictable follow-up question instead of relying on acoustic detection.
21. **Read Before You Run** (Security) — A real 2026 incident saw 1,180+ malicious AI plugins hit 40,000+ live systems by hiding instructions inside normal-looking setup text. A scanner that reads a new plugin the way the AI would, catching hidden commands a human would miss.

---

## Round 5 — vLLM/LiteLLM-style platforms (narrow, installable, one thing done well)

22. **TrustPanel** (UX) — AWS shipped the AG-UI protocol into AgentCore in March 2026, but developers still hand-build every actual approval/attribution screen on top of it from scratch. An installable component library of ready-made UX patterns (approval prompts, "who's doing this" badges, progress views) wired natively to AgentCore's AG-UI stream.
23. **AccessAgent** (UX) — Over 95% of major websites fail basic accessibility checks, and the National Federation of the Blind has filed formal complaints that new AI features are actively breaking screen readers. A hosted service that deploys a live, speaking, interruptible browsing agent (AgentCore Browser Tool) as an accessible front door for any company's existing website.
24. **Universal A2A Version Bridge** (UX) — A2A v1.0 agents call methods (`SendMessage`) that older v0.3 agents don't expose (`message/send`), with no automatic version negotiation; today's only fix is manual, SDK-specific patches per language. A LiteLLM-style proxy that auto-translates between any two agents regardless of A2A version or vendor.
25. **Real-Time Voice Bridge for A2A** (UX) — The core A2A spec has no real-time transport (no WebRTC/WebTransport) — it's an open, unmerged proposal on the official A2A GitHub. AWS AgentCore already added its own bi-directional streaming for voice/vision, but only AWS-to-AWS. Build the missing cross-vendor bridge so two different companies' agents can hold a live, interruptible voice conversation with each other.

---

## Round 6 — Addendum

26. **Between the Lines** (Security) — A browsing or computer-use agent (like *Steady Hands* or *AccessAgent* above) reads whatever is on a webpage it scrapes or navigates — including text hidden from a human's eyes (invisible-color text, hidden `<div>`s, text stuffed into alt-tags) but perfectly visible to the agent's context. A malicious page can plant instructions there to hijack the agent — a well-documented "indirect prompt injection" attack that predates today's browser-use/computer-use agents (it broke early browsing plugins back in 2023) and is still unsolved for AgentCore Browser Tool / Foundry Computer-Using Agent today. This is the same *Chain of Custody* pattern (item 1) — untrusted content flowing into an agent's context — applied specifically to the web-browsing agents already in this list: a scanner that strips or flags hidden/invisible content on a page before it ever reaches the agent, so a page can't say one thing to the person and another to the AI reading it.

---

## Round 7 — Cross-checked against two other AIs

The same research prompt (see `research-prompt.md`) was given to two other AI assistants. Their ideas were checked for overlap with the list above and fact-checked before adding anything. Several were dropped as duplicates of ideas already on this list: *TokenBurn* (≈ Cost Governor/Tool Diet), *Agent Black Box* (≈ Flight Recorder/Report Card), *Agent Circuit Breaker* (literally the same idea as item 8), *Skill Passport* (≈ Read Before You Run/Trust Badge), *Chaos Monkey for Agents* (≈ Agent CI/continuous red-teaming), and *Cross-Cloud Notary* (the other AI flagged this as its own weakest pick; heavy overlap with Chain of Custody). What survived the check:

27. **Ghost Memory** (Security) — AI agents are gaining persistent memory across sessions, and Microsoft's own security documentation now treats that memory as a real attack surface: poisoned content read today can quietly influence a decision weeks later, a slower and harder-to-catch variant of *Chain of Custody*'s immediate-exfiltration pattern. Citation verified real and accurately represented. A governance layer with provenance, expiry, and a one-click rollback/quarantine per memory entry.
28. **Permission Lens** (Security) — A real Microsoft Research paper argues enterprise RAG systems can expose documents to users never authorized to see them, and that today's probabilistic defenses aren't enough — deterministic, explainable access control is needed at the moment of retrieval. Citation verified real. A layer that shows, for every retrieved document, why it was selected and which policy allowed it — swap the logged-in user and watch the retrieved set and explanation change live.
29. **Agent Lifecycle Passport** (Security) — Non-human identities now outnumber human ones 144-to-1 in cloud-native environments (confirmed, Entro Security/CSA), up 56% in two years, and GitGuardian's own 2026 research confirms 64% of secrets known-valid in 2022 still hadn't been revoked by January 2026 (also confirmed). Both clouds already give agents identities; neither auto-revokes one the instant its task ends. A credential scoped to a single task, killed automatically the moment the session finishes or goes orphaned.
30. **Agent Interlock** (Security) — A circuit breaker built specifically for the handoff between two different clouds' agents talking over A2A, not just two agents on different frameworks (item 8's version). Each cloud already caps its own agent fleet (AgentCore's iteration/timeout limits, Foundry's Control Plane guardrails), but neither has visibility into a loop that crosses the AWS↔Azure boundary — that seam is nobody's job today. Note: the specific "$47,000 over 264 hours" anecdote this idea originally cited does not check out — it traces to an unattributed, copy-pasted story with no real source across several blogs, likely itself AI-generated. Use our own already-verified evidence instead (Cost Governor's real $500→$4,200-in-two-weeks case).
31. **Focus Governor** (UX) — Distinct from *Vigil* (which asks "is the human still paying attention when they approve something") — this asks "should the human be interrupted at all right now." Grounded in HBR's "AI brain fry" framing of fatigue from overseeing multiple simultaneous agents (not independently verified in this pass, treat as plausible not confirmed). A layer that learns which agent outputs a specific person actually opens versus ignores, batching the rest into a digest and only breaking through immediately for what crosses a learned urgency threshold.

---

## Appendix — proposed but dropped as duplicates

These came from the two other AIs in Round 7 but were left out of the numbered list above because they overlap too closely with an idea already on it. Kept here so nothing discussed is lost.

- **TokenBurn** (Security) — A dashboard that flags AI agents burning tokens on idle polling/waiting instead of real work, citing a real GitHub issue where this ate ~20% of one agent's total usage. Dropped: same territory as *Cost Governor* (item 7) and *Tool Diet* (item 6).
- **Agent Black Box** (Security) — A one-click "generate a forensic incident report" tool for when an agent fails, packaging prompts, tool calls, and a timeline into a compliance-ready PDF. Dropped: same territory as *Flight Recorder* (item 4) and *Report Card* (item 16).
- **Agent Circuit Breaker** (Security) — Detects an agent stuck retrying a failing dependency and reroutes to a fallback instead of looping forever. Dropped: this is the same idea, same name, as *Circuit Breaker* (item 8) already on the list.
- **Skill Passport** (Security) — Cryptographic signing and provenance tracking for AI agent "skills," so an unsigned or unverified skill can't be installed. Dropped: same territory as *Read Before You Run* (item 21) and *Trust Badge* (item 15).
- **Chaos Monkey for Agents** (Security) — Deliberately injects failures (latency, revoked credentials, malformed responses) into an agent to test resilience before production. Dropped: same territory as *Agent CI* (item 5) and *Proving Ground* (the continuous red-teaming concept from Round 3).
- **Cross-Cloud Notary** (Security) — A tamper-evident audit trail reconstructing a decision chain that spans both AWS and Azure agents, timed against the EU AI Act's Article 12 deadline. The AI that proposed it flagged this as its own weakest idea (a small vendor category already does Article 12 logging). Dropped: heavy overlap with *Chain of Custody* (item 1); the cross-cloud angle wasn't sharp enough on its own to justify a separate entry.

---

## Current standing recommendation

- **Security & Governance:** *Ghost Memory* (freshest angle, verified citation, genuinely distinct from everything else on this list) now edges out *Caller Not Verified* and *Chain of Custody* as the strongest overall — but all three are solid.
- **Future of AI UX:** *The Honest Scribe* (most emotionally resonant, most novel "AI admits uncertainty" angle), or *Real-Time Voice Bridge for A2A* if leaning into your existing A2A deployment experience.
