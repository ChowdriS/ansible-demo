# Prompt — paste this into any AI to get more hackathon ideas

I'm competing in a Presidio-sponsored, 22-hour AI hackathon. I need help finding a strong project idea. Please read the full brief below, then propose 5-8 NEW ideas following the exact bar and format described at the end. Do your own web research/browsing if you're able to — don't just reason from memory.

## The hackathon

- 22-hour build sprint. Teams must build against a set of themes, with one non-negotiable rule: every idea needs a real path to market — it must solve a real problem for a real external customer, something a sales team could actually pitch to a paying client. Ideas scoped only to internal use at the sponsoring company do not qualify.
- I'm building for two themes specifically:
  1. **"Future of user experience in the world of AI"** — envision how people will naturally adopt and interact with AI in their everyday work.
  2. **"Security and Governance at Scale"** — as enterprises adopt AI, how do we secure and govern it at scale?
- Judging criteria: Engineering Excellence & Vision, Creativity & Innovation, Viability and Execution Plan, GTM (Go-To-Market) Strategy.
- Required tech: the idea must integrate with **AWS Bedrock / AWS Bedrock AgentCore** and/or **Microsoft Azure AI Foundry** — any combination is fine, doesn't have to use both.
- The sponsor is an IT/cloud/security solutions company serving mid-market and enterprise clients (healthcare, financial services, manufacturing, public sector, retail, etc.) — so an idea that maps onto something a systems integrator could realistically sell as an add-on, a managed service, or a marketplace listing scores extra well on GTM.

## The quality bar — read this carefully

I've already run several rounds of research and rejected a lot of ideas for failing one of these tests. Please hold every idea to the same bar:

1. **The problem must be real, not hypothetical.** Back it with something concrete — a real disclosed incident, a CVE, a GitHub issue thread, a survey with a specific number, a documented practitioner complaint, an academic paper. Not "enterprises struggle with AI trust" — something with a name, a date, and a source.
2. **Check whether it's already solved.** For each idea, explicitly check if a well-adopted product, open-source project, or protocol already does this well. If it's already solved (even partially), say so and either drop the idea or explain the specific narrower gap that's still open.
3. **State your confidence honestly.** If your evidence for something is thin (one blog post, no primary source), say so plainly instead of dressing it up. I'd rather hear "this is a weaker candidate, here's why" than a confident-sounding guess.
4. **"Mesmerizing" does not mean flashy.** I had a research pass on what actually wins hackathon judges over, and the finding was: judges are tired of flashy demos that fall apart under a real question. What lands is something happening live, watchably, autonomously, that keeps working even when interrupted or something goes wrong — imperfection reads as more credible than polish. Favor ideas with a genuine live "watch this happen" moment over a slide describing what could happen.
5. **Plain English.** Explain the problem and the fix the way you'd explain it to a smart friend outside tech — avoid unexplained acronyms and jargon.

## Ideas already covered — do not repeat these or lightly reskin them

**Security & Governance:**
- An AI agent that reads untrusted content (email, a webpage, a support ticket) with nothing tracking that content as "tainted" before it drives a data-leaking tool call — grounded in real incidents (a zero-click Microsoft 365 Copilot exploit, a GitHub MCP server exploit, a Supabase data leak, a Kubernetes token-exfiltration CVE).
- Humans approving AI agent actions rubber-stamping within weeks as volume outpaces attention, so "human oversight" quietly stops working.
- Deepfake voice/video fraud tricking an AI-mediated approval workflow (grounded in a real $25M deepfake video-call fraud incident).
- A malicious AI agent plugin/skill hiding instructions inside normal-looking setup text to poison a marketplace (grounded in a real 2026 incident: 1,180+ malicious skills hit 40,000+ live systems).
- A webpage a browsing/computer-use agent scrapes containing hidden instructions invisible to a human but readable by the agent (indirect prompt injection via web content).
- A version-mismatch bridge for the A2A (agent-to-agent) protocol — two agents on different protocol versions can't talk, and there's no automatic negotiation today.
- Giving a real-world open-source Kubernetes AI agent framework (kagent) actual permissions — it currently has none by default (confirmed in its own public issue tracker).

**Future of AI UX:**
- A live "trust cockpit" that shows which of several collaborating AI agents is doing what, with a confidence/provenance indicator per output, instead of a scrolling chat log.
- A hospital AI scribe that visibly pauses and asks for confirmation instead of silently mis-transcribing a clinical note (grounded in real, documented charting errors and a nurse-trust survey).
- An AI browsing agent that makes an inaccessible website usable for a blind or motor-impaired visitor in real time, interruptible mid-task (grounded in real accessibility-complaint filings).
- A component library of ready-made UX patterns (approval screens, "who's doing this" badges) for the AG-UI protocol, since the protocol plumbing is solved but the actual UI patterns on top of it aren't.
- A real-time voice bridge for the A2A protocol, since the core spec has no real-time/voice transport today, so two different companies' agents can't hold a live spoken exchange with each other.
- A checkpoint/rewind system for debugging multi-agent failures (Replit-style "time travel," applied to agent sessions).
- A CI/regression gate that catches an agent's quality silently degrading after an update (grounded in a real case: an agent's accuracy dropped from 93% to 71% unnoticed).
- Inline "before vs. after" review for non-coding AI agent output (contracts, CRM records), the way coding tools already do for code.
- A self-healing computer-use agent that replaces brittle RPA on legacy enterprise screens.
- A cross-vendor control panel that lets one person dispatch a task across several different companies' AI agents at once.

## What I want back

For each new idea, give me:
1. **Name** — short and memorable.
2. **The real problem** — plain English, with your source/evidence.
3. **Is it already solved?** — honest check against existing products/projects.
4. **The build** — what we'd actually build in 22 hours, and which cloud (AWS Bedrock/AgentCore or Azure AI Foundry) it plugs into.
5. **Why it's worth watching** — the live, watchable moment this would create in front of judges.
6. **Sell it as** — who would actually buy this and why.

Give me 5-8 ideas, ranked by how strong you think they are, and tell me plainly which one you'd build if you only had time for one.
