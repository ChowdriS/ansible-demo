# Lineage, Explained Simply

*A plain-English companion to `lineage-platform-plan.md`. Same idea, none of the engineering detail — just the problem, the fix, the pitch, and what it does.*

---

## The Problem

Companies are starting to let AI "manager" agents hand off small pieces of work to other, smaller AI "worker" agents. A manager gets a request, breaks it into steps, and asks a few workers to each handle one step — sometimes those workers even ask *other* workers for help, and sometimes that chain of AI helpers crosses from one company's cloud (say, Microsoft's) into another's (say, Amazon's).

Here's the problem: right now, when the manager hands a task to a worker, that worker usually just gets **the manager's entire set of keys** — not just the one key it actually needs. Nobody bothered to cut it a smaller key, because doing that by hand for every single worker, every single time, takes too much engineering effort. So in practice, an AI whose only job is "draft a follow-up email" often *also* has the ability to issue a refund, delete a record, or read private customer data — because it inherited everything its manager could do, instead of just what its own small job required.

And it gets worse the more AI helpers are involved: nothing today actually checks that a worker's permissions only ever get *smaller* the further down the chain you go. A worker three steps removed from the original request can, in practice, end up with *more* access than it should ever have had — and there's no alarm that goes off when that happens.

**This isn't a "someday" risk. It's already the most common cause of real AI security breaches:**

- **78%** of AI agents involved in real breaches had far more access than their actual job needed.
- Breaches involving AI agents jumped **340%** in a single year.
- **88%** of companies say they've already had a suspected or confirmed AI agent security incident in the last year.
- Yet only **44%** of companies have *any* rule in place to manage this — even though **92%** agree it's critical.
- Over **a third** of companies admit they couldn't even shut off a rogue AI agent today if they had to.
- Two real, named attacks in 2025 already exploited exactly this gap — one where an AI worker's access quietly *grew* instead of shrank as a request passed through several agents, and another where one AI's identity got smuggled further down a chain than it was ever supposed to travel.

Nobody has fixed this yet — not Microsoft, not Amazon, not any of the well-funded security startups racing into this space right now (and there are a lot of them — two of the biggest names in this exact space were bought out in the same week this past summer, which tells you how urgent and valuable solving this is considered to be).

---

## The Solution

Think of Lineage as a **smart, temporary visitor-badge system for AI agents.**

When a human asks an AI manager to do something, and that manager asks smaller AI workers to help, every worker gets its own badge — but that badge is:

1. **Never more powerful than the human who asked.** No matter how many AI helpers get involved, nobody can ever end up with more access than the actual person who started the request was allowed to have.
2. **Cut specifically for the one job it was given.** Not "everything the manager can do" — just the exact doors needed for this one task, automatically figured out, with no engineer hand-writing a rule for every single worker.
3. **Stamped so it can't be faked.** Every badge carries tamper-proof proof that it's a smaller version of the badge before it in the chain. If any worker's badge ever shows up claiming *more* access than it started with — which is exactly what those two real 2025 attacks did — the whole chain is stopped immediately, automatically, no matter how many steps deep it happened or whether it crossed from one company's cloud into another's.
4. **Expires the moment the job is done.** No badge sits around active after its task is finished, waiting to be misused later.

And if something does go seriously wrong, there's **one big switch** that instantly shuts down every single AI worker in the entire chain at once — not just the one that misbehaved, all of them, everywhere, immediately.

---

## The Marketing Pitch

> **"Every AI agent's power traces back to a real person — and it only ever gets smaller from there."**

Right now, letting a team of AI agents work together on your behalf means trusting that nobody made a mistake handing out permissions. Lineage removes that trust requirement entirely: permissions are computed automatically for exactly the task at hand, proven mathematically to only shrink as they pass between agents, and killed the moment they're no longer needed — even when your AI agents span more than one cloud provider.

**Who buys this:** any company letting AI agents take real actions — approve refunds, touch customer data, manage infrastructure — especially ones already required to prove, to a regulator or an auditor, exactly who authorized what and why. That's most large companies in healthcare, finance, and any business now subject to the EU's new AI regulations.

**Why an IT/security consulting company like Presidio is the right one to sell it:** this isn't a new category of trust to build with a client — it's the exact same identity-and-access-management relationship they already have, just extended to cover AI agents instead of only human employees and traditional software. Same buyer, same conversation, new and urgent reason to have it now.

---

## What It Actually Does (Features)

| What we call it | What it does, in one sentence |
|---|---|
| **The Badge Maker** | Automatically figures out the smallest set of permissions a worker AI needs for its specific task — nobody writes this by hand. |
| **The Tamper-Proof Stamp** | Every badge carries mathematical proof it's smaller than the one before it — impossible to fake or quietly upgrade. |
| **The Doorman Who Can't Be Bribed** | The thing that actually checks every badge lives *outside* the AI agents themselves — so even if an agent gets tricked or hijacked, it can't just grant itself more access, because it was never the one holding the keys to begin with. |
| **The Break-In Alarm** | The instant any badge shows up with more access than it should have, the whole chain stops, right there, and logs exactly why. |
| **Badges That Expire** | Access dies automatically the moment the task is finished — nothing is left lying around active and unused. |
| **The Cloud Translator** | Lets a badge issued on one company's cloud (say, Microsoft's) keep working correctly when the chain crosses into another's (say, Amazon's), instead of breaking or silently granting too much access at the seam. |
| **The Stranger Handshake** *(stretch feature)* | Lets your AI agent and a completely different company's AI agent trust each other and verify badges, even with no shared login system between the two companies at all. |
| **The Decoy Room** | A fake, never-used piece of data planted as bait — if any AI agent ever touches it, that's instant, unmistakable proof something's gone wrong. |
| **The Big Red Button** | Instantly shuts down every AI agent in the entire chain at once, everywhere, the moment something serious is detected. |
| **The Instant Paperwork** | Turns the system's own activity record into a ready-to-hand-over compliance report for regulators, automatically, instead of someone assembling it by hand under deadline pressure. |
| **The Plain-English Explanation** | A simple, non-technical screen that tells an auditor or compliance officer exactly what was blocked and why — no engineering knowledge required to read it. |

**One honest note:** this isn't reinventing everything from scratch. Some of the "doorman" plumbing above can run on top of tools that already exist and are already trusted in production — Lineage is the brain that decides what should and shouldn't be allowed; it doesn't need to replace the pipes that already carry the traffic.
