## Table of Contents

1. [How to Read This Document](#how-to-read-this-document)
2. [Old Model vs. New Model: Read This First](#old-model-vs-new-model-read-this-first)
3. [Core Resource Architecture](#core-resource-architecture)
4. [The Two Networking Axes: Egress and Inbound Access](#the-two-networking-axes-egress-and-inbound-access)
5. [Agent-Specific Networking: Why It's Different](#agent-specific-networking-why-its-different)
6. [The One-Way-Door Constraint](#the-one-way-door-constraint)
7. [Subnet Sizing and IP Planning](#subnet-sizing-and-ip-planning)
8. [Private Endpoints: Setup and DNS](#private-endpoints-setup-and-dns)
9. [Cross-Region Private Endpoints (and the Agent Service Exception)](#cross-region-private-endpoints-and-the-agent-service-exception)
10. [Firewall Rules vs. VNet Service Endpoints vs. Private Endpoints](#firewall-rules-vs-vnet-service-endpoints-vs-private-endpoints)
11. [Managed VNet vs. BYO VNet](#managed-vnet-vs-byo-vnet)
12. [Controlling Agent Egress](#controlling-agent-egress)
13. [Trusted Service Bypass](#trusted-service-bypass)
14. [Putting APIM in Front of Foundry](#putting-apim-in-front-of-foundry)
15. [Legacy Pattern: The Hub/Project Security Model](#legacy-pattern-the-hubproject-security-model)
16. [Common Failure Modes](#common-failure-modes)
17. [Recommended Architectures by Scenario](#recommended-architectures-by-scenario)
18. [Microsoft's Own Reference Architectures](#microsofts-own-reference-architectures)
19. [2026 Changelog Timeline](#2026-changelog-timeline)
20. [Open Questions and Gaps in This Research](#open-questions-and-gaps-in-this-research)
21. [References](#references)

---

## How to Read This Document

Each section is written to stand alone, so you can jump straight to
whichever part is relevant rather than reading top to bottom. Where a
claim comes from an official Microsoft Learn doc, it's marked as
confirmed. Where it comes from a community blog, a Microsoft Q&A thread,
or a Tech Community post, that's noted too, since the accuracy and
freshness varies. This space has moved fast through 2026, so anything not
explicitly dated should be treated with a bit of caution.

---

## Old Model vs. New Model: Read This First

Before anything else, one disambiguation that affects every other section
in this document.

Azure AI Foundry has **two generations of project model**:

- **The old Hub-based model** — a Hub resource (under
  `Microsoft.MachineLearningServices`) containing one or more Projects,
  each with its own networking and RBAC surface. Some older blog posts
  and even some still-live documentation describe this model.
- **The new Foundry-native model** — a single Foundry resource (under
  `Microsoft.CognitiveServices`, kind `AIServices`) containing Projects as
  a direct subresource. Networking, security, and connections are
  configured once, at the resource level, not per-project.

**This matters a lot for networking specifically**, because the two
models have genuinely different architecture, not just different
terminology. Per Microsoft's own community content, new generative AI and
model-centric features, including ongoing agent networking work, are
landing **only** in the new Foundry-native model. Hub-based projects are
not receiving new agent or networking features, and there's no stated
forward roadmap for Hub.

**If you're starting fresh, build against the Foundry-native model.**
Anywhere this document cites a source describing the Hub model, it's
explicitly labeled as legacy context, not a current recommendation.

---

## Core Resource Architecture

*Confirmed, from the official Foundry architecture documentation.*

The current model is a three-layer hierarchy:

1. **Foundry resource** — the top-level resource
   (`Microsoft.CognitiveServices/accounts`, kind `AIServices`). This is
   where model deployments, security settings, and connections to other
   resources live. It shares the same `Microsoft.CognitiveServices`
   provider namespace as Azure OpenAI, Speech, Vision, and Language,
   meaning it uses the same management APIs, RBAC action patterns,
   networking configuration shape, and Azure Policy aliases as those
   services.
2. **Project** (`Microsoft.CognitiveServices/accounts/projects`) — an
   explicit subresource of the Foundry resource. This is where agents,
   evaluations, and files actually live.
3. **Project assets** — the agents, evaluation runs, and files
   themselves.

**Connected resources** — Storage, Key Vault, and Azure AI Search — are
explicitly called out as independent Azure resources with their own
governance boundaries. You network, secure, and manage compliance for
them separately from the Foundry resource itself. The Foundry resource
only holds a *connection* to them, it doesn't absorb their network
configuration.

RBAC is split into control-plane actions (creating deployments and
projects) and data-plane actions (building agents, running evaluations,
uploading files), and each can be scoped independently at either the
resource or the project level.

---

## The Two Networking Axes: Egress and Inbound Access

*Confirmed, from the official networking-options documentation.*

Foundry networking is really two separate, related decisions, not one
setting.

### Egress (outbound) models — exactly three

| Model | What it means |
|---|---|
| **Public egress** | The resource and its agents can reach the internet freely, the default, simplest option. |
| **BYO virtual network** (customer-managed VNet) | You provide the VNet. The platform injects a delegated subnet into it so agents and resources communicate over your own network. |
| **Managed virtual network** | Azure creates and manages the VNet for you, automating most of the isolation work. |

### Inbound access — two options, dependent on your egress choice

- **Public** (optionally restricted to specific IP ranges)
- **Private endpoint only**

Your egress choice gates which inbound options are actually available to
you, they aren't fully independent. Broadly: the more isolated your
egress model, the more likely you are to also want (and be pushed
towards) private-endpoint-only inbound access.

---

## Agent-Specific Networking: Why It's Different

*Confirmed, from the agents-networking-deep-dive documentation, with one
caveat noted at the end of this section about incomplete retrieval.*

This is the part that's genuinely different from "just network the AI
Services resource," and it's easy to miss if you're coming from general
Azure networking experience.

- **Hosted Agents** run in a dedicated Micro VM, each with its own
  network interface and IP address. But — and this is the detail that
  matters — **any tool invocation is still routed through a shared data
  proxy, not the agent's own IP.** The agent having its own NIC doesn't
  mean its tool calls go out that way.
- **Prompt Agents** don't get a dedicated IP at all. All of their
  outbound traffic, including every tool call, goes through the same
  shared proxy.
- **The single-tenant data proxy** is the actual convergence point: a
  platform-managed networking component dedicated to your specific
  Foundry project, handling outbound connectivity for all of your agents,
  hosted or prompt. Both paths meet here before reaching your private
  resources through private endpoints in your private-endpoint subnet.
- **Not every tool supports network isolation.** Some tools bypass the
  private path entirely and reach the public internet regardless of how
  locked-down your project is. This needs to be checked tool by tool
  before you commit to an isolated design, don't assume isolation is
  total just because you've configured private networking at the project
  level.

**Gap in this research:** the full topology diagram and detailed
IP-allocation math in the source deep-dive document weren't fully
retrievable in this pass (the rendered view was truncated). If you need
that level of detail, worth a direct fetch of the raw markdown source
rather than the rendered GitHub page.

---

## The One-Way-Door Constraint

*Confirmed, from the official architecture documentation. This is
important enough to have its own section rather than being buried in a
bullet point.*

**Your network configuration is set once, at account creation, and takes
effect on the creation of your first hosted agent. It cannot be changed
afterward.** If you need to switch networking models later, deleting and
recreating the configuration isn't an option, you have to stand up a new
project or account entirely and migrate.

This means the networking decision needs to happen genuinely early in
planning, before any agents are built, not as something to iterate on
once you see how things are going.

---

## Subnet Sizing and IP Planning

*Confirmed, from the official architecture documentation.*

- **Production with hosted agents**: a **/24 subnet** is the recommended
  size, giving headroom for scaling, concurrent sessions, and the
  temporary IP churn that happens during in-place upgrades.
- **/27 is the supported minimum**, but it's only realistically viable
  for prompt-agent-only setups or small hosted agent deployments.
- **Target no more than 80% subnet utilization**, reserving headroom for
  platform maintenance spikes rather than sizing right up to the edge.
- **IP consumption is asymmetric between agent types:**
  - Hosted agents consume one IP per Micro VM, which scales with the
    number of projects, agents, and concurrent sessions, and can
    temporarily double during a revision rollout (old and new versions
    briefly coexisting).
  - Prompt agents use a small, roughly static pool of about 10 IPs per
    project, regardless of how many agents or revisions exist.
- **No automatic cross-region failover.** If you need multi-region
  resilience, that means separate Foundry resources per region with your
  own application-layer sync and routing, Foundry doesn't do this for
  you.

---

## Private Endpoints: Setup and DNS

*Confirmed from multiple sources: Microsoft's Cognitive Services VNet
documentation and a detailed community blog post specifically covering
Foundry private endpoints.*

### The three DNS zones you need

```
privatelink.cognitiveservices.azure.com
privatelink.openai.azure.com
privatelink.services.ai.azure.com
```

All three need to exist, and each needs to be linked to every VNet that
should be able to resolve the private endpoint. **Use DNS zone groups**
rather than manually creating records in each zone, a DNS zone group
keeps the A records across all three zones automatically in sync with the
private endpoint as it changes; manual management risks the zones
drifting out of sync with each other over time.

### The silent failure mode to know about

If a client outside the VNet where the private endpoint lives tries to
resolve the resource's URL, it doesn't fail loudly, **it silently
resolves to the public endpoint instead.** This means a DNS
misconfiguration doesn't necessarily look like an error, it can look like
things are working while your traffic quietly bypasses the private path
entirely. Worth explicitly testing DNS resolution from inside the correct
VNet (`nslookup` or equivalent) rather than assuming it's correct because
requests are succeeding.

### The custom subdomain requirement

Requests to a private endpoint must specify the resource's **custom
subdomain** in the base URL, not a private IP address and not a generic
hostname. Using the wrong hostname format is a specific, documented
failure mode, not a hypothetical edge case.

### Custom DNS servers

If you're using your own DNS server rather than Azure-provided DNS, it
must forward queries for the privatelink subdomains to Azure DNS
(`168.63.129.16`), otherwise resolution fails for anyone relying on that
custom server.

---

## Cross-Region Private Endpoints (and the Agent Service Exception)

*Confirmed from a detailed community blog post with a real worked example,
this is one of the most important findings in this whole document.*

**The general case:** Azure Private Link does support connecting a
VNet-local private endpoint to a Foundry resource in a *different* region
than the VNet itself. The private endpoint has to live in the same region
as the VNet it's placed in, but the target resource it points at can be
in any Azure region. A documented real example connects a VNet in New
Zealand North to a Foundry account in Australia East, with no VNet
peering required at all, Azure's own backbone network handles the routing
internally.

- **Latency cost measured**: roughly 30 to 60 milliseconds of added
  latency for that cross-region hop. Called acceptable for typical AI/LLM
  workloads, but explicitly flagged as **problematic for real-time voice
  applications** specifically.
- **Multi-VNet setups** need either every VNet linked directly to all
  three DNS zones, or DNS forwarding configured between a hub VNet and
  its spokes so resolution reaches everywhere it needs to.

**The exception, and the one to remember:** if you're using **Foundry
Agent Service specifically**, not just model inference, cross-region
private endpoints will not work. All Foundry workspace resources must be
deployed in the same region as the VNet for Agent Service. This is stated
as a flat limitation, not a configuration nuance you can work around.
Given that the general cross-region story sounds permissive, this is
exactly the kind of thing worth double-checking before assuming it
applies uniformly across everything Foundry does.

---

## Firewall Rules vs. VNet Service Endpoints vs. Private Endpoints

*Confirmed, from Microsoft's Cognitive Services virtual networking
documentation (the content is shared across Azure global and Azure China
docs for this topic).*

| Approach | How it works | Trade-offs |
|---|---|---|
| **Selected Networks / firewall rules (IP allowlisting)** | Default-deny once enabled. Supports up to 100 IP rules and 100 VNet rules per resource, combinable. | `/31` and `/32` prefixes aren't supported. Private address ranges (10.x, 172.16-31.x, 192.168.x) aren't allowed as IP rule entries, only as VNet rules. Portal/CLI/PowerShell management access keeps working even with rules active. |
| **VNet service endpoints** | Enabled per subnet. The subnet's identity travels with each request rather than routing through a dedicated private IP. | Not every Cognitive Service is covered by this route. Cross-tenant VNet rules (granting access to a subnet in a different Entra tenant) are only supported via PowerShell/CLI/REST, not the portal. |
| **Private endpoints** | Gives the resource a private IP inside your VNet. Lets you fully block the public endpoint. Works over VPN/ExpressRoute from on-premises. Prevents data exfiltration paths out of the VNet. | Requires a connection approval workflow, not instant. Speech service specifically needs its own separate private endpoint configuration apart from the main resource. |

**The one universal trap across all of these**: if you don't explicitly
set the default network action to **Deny**, none of your IP or VNet
allowlisting rules have any effect at all. This is called out repeatedly
as the single most common, easy-to-miss configuration mistake across
every network-restriction method.

---

## Managed VNet vs. BYO VNet

*Confirmed, from official Microsoft Learn documentation and a Microsoft
Tech Community post specifically on this trade-off.*

### Managed VNet

- Azure creates and manages the VNet for you. Private endpoints to your
  dependencies get provisioned automatically, without you building or
  maintaining a VNet yourself.
- Managed private endpoints in this model don't even create visible
  network interfaces, they're fully abstracted away.
- **Reached General Availability in May 2026.**
- **A real cost catch worth knowing upfront**: if you add an FQDN-based
  outbound rule while running in "Allow Only Approved Outbound" mode,
  Foundry automatically provisions a managed Azure Firewall on your
  behalf, and that firewall carries its own real, ongoing cost. The
  default SKU is Standard; Basic SKU is selectable if you don't need the
  advanced features and want to reduce cost.

### BYO VNet

- Materially more complex to set up and operate. Requires a subnet
  delegated to Azure Container Apps (`Microsoft.App/environments`),
  correct "capability host" provisioning, private (non-public,
  non-CGNAT) IP ranges, and a minimum **/27 subnet** specifically for
  agent delegation.
- The delegated subnet cannot be shared across multiple Foundry
  resources.
- Community consensus, reflected in Microsoft's own guidance, is that
  Managed VNet meaningfully reduces the operational burden compared to
  self-managing a BYO VNet, unless you have a specific reason you need
  full control over the VNet yourself (existing enterprise network
  topology, specific compliance requirements, integration with existing
  hub-and-spoke architecture).

---

## Controlling Agent Egress

*Confirmed, from official networking-options and hosted-agent-guardrails
documentation. This is specifically about controlling what an agent
calls out to, distinct from controlling inbound access to the Foundry
resource itself.*

**Network egress controls (currently preview)** exist specifically for
Hosted Agents:

- You set a **default action** (Allow or Deny) for outbound requests from
  the agent.
- You then add per-host rules, each with a **Mode** (Audit or Enforce)
  and an explicit host match plus an action.
- Audit mode lets you observe what an agent is actually trying to reach
  before you commit to blocking it, useful for understanding real
  behavior before locking things down.

For full isolation beyond what the built-in egress controls offer, BYO
VNet or Managed VNet paired with your own firewall, NSGs, and user-defined
routes gives you complete control, with Azure Firewall able to inspect
and control all outbound VNet traffic.

If agents genuinely need to call external, non-Azure APIs, either allow
outbound HTTPS explicitly through your firewall, or route that traffic
through a **NAT Gateway** for controlled, auditable egress rather than
leaving it unrestricted.

As of March 2026, tool connectivity (MCP servers, Azure AI Search indexes,
Fabric data agents) all operate over private network paths under the
"Standard Setup," and managed VNet logging was added specifically to give
visibility into firewall, NSG, and flow-log activity inside these
otherwise fairly opaque isolated environments.

---

## Trusted Service Bypass

*Confirmed, from official documentation.*

By default, the Standard Setup with private networking has **no public
egress at all**. There is a trusted service bypass mechanism, but it's
narrow, not a general-purpose allowlist:

- **Foundry Tools, Azure AI Search, and Azure Machine Learning** are the
  named trusted services that can reach a private Foundry resource, and
  only if their managed identities hold the correct role assignment,
  granted through an explicit network rule exception.
- This is a short, specific list, not something you can extend to
  arbitrary services just by asking for a bypass.

Architecturally, PaaS dependencies (storage, Key Vault, container
registry, monitoring) are isolated via Private Link, while the Agent
Service itself uses VNet injection into your subnet, giving it
private-endpoint-based outbound reach into your Azure PaaS resources.

---

## Putting APIM in Front of Foundry

*Confirmed, from a detailed community blog post with real policy XML and
NSG rules, cross-referenced against a Bicep sample in Microsoft's own
`foundry-samples` repository.*

Putting Azure API Management in front of Foundry is a real, documented
pattern, primarily used for network-level access control (making Foundry
reachable only from inside a VNet, via VPN, Bastion, or ExpressRoute),
though it's also commonly used more broadly for rate limiting and
centralized cost tracking across multiple backend deployments.

**The confirmed architecture:**

- APIM deployed in **virtual network internal mode**, so it's only
  reachable from inside the VNet.
- Dedicated subnets: one for Foundry (with all its services behind
  private endpoints), one for APIM.
- A test VM inside the VNet, useful for validating connectivity without
  needing VPN/Bastion set up yet.
- APIM imports the Foundry model endpoint as an API, for example:
  ```
  https://<your-ai-foundry>.cognitiveservices.azure.com/openai/deployments/gpt-4o
  ```
- APIM authenticates to Foundry using **its own system-assigned managed
  identity**, granted the **Cognitive Services OpenAI User** role, no API
  keys involved anywhere in this flow.

**Confirmed policy XML:**

```xml
<authentication-managed-identity resource="https://ai.azure.com" output-token-variable-name="msi-access-token" ignore-error="false" />
<set-header name="Authorization" exists-action="override">
  <value>@("Bearer " + (string)context.Variables["msi-access-token"])</value>
</set-header>
```

**Required NSG rules on the APIM subnet:**

- Allow the `ApiManagement` service tag inbound to the VNet on port 3443.
- Allow the `AzureLoadBalancer` service tag inbound to the VNet on port
  6390.

**A working, more complete Bicep implementation of this exact pattern**
exists in Microsoft's own `azure-ai-foundry/foundry-samples` repository,
under the sample named `15-private-network-standard-agent-setup`, worth
pulling directly if you want a runnable starting point rather than
building the policy and network rules from scratch.

---

## Legacy Pattern: The Hub/Project Security Model

> **This section describes the older Hub-based project model. It is
> explicitly superseded by the Foundry-native model described throughout
> the rest of this document. Included here only as historical context in
> case you encounter it in older blog posts, existing deployments, or
> older Microsoft documentation that hasn't been updated yet.**

The Hub model used a three-layer network model of its own:

1. **Customer VNet** — private endpoints for the Hub resource itself and
   for customer-owned PaaS resources like AI Search and storage.
2. **Managed VNet** — automatically created by the platform, holding
   compute, with system-generated private endpoints (prefixed `_Sys`)
   for the registry, Key Vault, and blob storage.
3. **On-premises access** — required an Application Gateway placed inside
   the Hub's VNet, since the managed VNet in this model couldn't be
   directly peered with anything else.

The core outbound rule in this model was to set the workspace to "Allow
only approved outbound," then manually disable public access on every
attached service (Container Registry, Key Vault, Blob Storage
individually), each with "Allow trusted Microsoft services" enabled
separately.

If you're maintaining an existing Hub-based deployment, this pattern
still applies to it. If you're planning anything new, use the
Foundry-native model described in the rest of this document instead.

---

## Common Failure Modes

*A mix of confirmed documentation and community-reported issues (Microsoft
Q&A threads), labeled accordingly.*

| Symptom | Likely cause | Source |
|---|---|---|
| Requests seem to succeed but bypass private networking entirely | DNS resolving to the public endpoint because the client isn't inside the correct VNet, or the private DNS zone isn't linked there | Documented, silent failure mode |
| Connection fails with no obvious reason despite a private endpoint existing | Wrong hostname format, not using the resource's custom subdomain in the request URL | Documented requirement |
| All IP/VNet allowlist rules appear to do nothing | Default network action left as Allow instead of explicitly set to Deny | Documented, called out as the most common mistake |
| DNS resolution fails intermittently with a custom DNS server | Custom DNS server not forwarding privatelink-subdomain queries to Azure DNS (`168.63.129.16`) | Community-reported, Microsoft Q&A |
| Cross-region private endpoint setup works for inference but not for Agent Service | Agent Service specifically requires same-region deployment, a flat limitation, not a config issue | Documented |
| 403 "Access denied due to Virtual Network/Firewall rules" on the Agents API in the new portal | Live, reported issue as of this research | Community-reported, Microsoft Q&A |
| 403 UnauthorizedUserAction on evaluation runs with Selected Networks restriction | Evaluation runs execute on Microsoft-managed backend services that don't originate from your VNet, so a private endpoint alone doesn't guarantee reachability | Community-reported, Microsoft Q&A |
| Mysterious Foundry portal errors under restricted egress | The portal itself calls "nextgen" APIs under `ai.azure.com`, allow that FQDN explicitly if egress is locked down | Community-reported |
| Hard to tell if an issue is network-related or an auth/code problem | Network reachability is validated **before** authentication, so a request from outside an allowed VNet fails even with fully valid credentials | General diagnostic pattern, community-reported |

**A useful general diagnostic approach reported repeatedly**: temporarily
relax network restrictions to confirm whether an issue is genuinely
network-policy-related before spending time chasing what looks like an
authentication or code problem.

---

## Recommended Architectures by Scenario

*Synthesized from the architecture documentation and the sizing/cost
sections above, not a direct quote from a single source.*

| Scenario | Suggested approach |
|---|---|
| Solo exploration, no agents needed | A standalone Azure OpenAI resource, no Foundry project overhead needed at all |
| Solo exploration, using agents | One Foundry resource, one project, public egress, default networking |
| Small production deployment, prompt agents only | Public or IP-restricted inbound is often sufficient; if isolation is required, a /27 minimum subnet is viable specifically because prompt agents' IP consumption is small and largely static |
| Production with hosted agents | Managed VNet preferred over BYO VNet unless you have a specific reason for full control; /24 subnet; private endpoints for all connected resources (Storage, Key Vault, AI Search) |
| Enterprise landing zone, full isolation, no public egress | BYO VNet, Standard Setup with private networking, per-host egress rules in Enforce mode, Azure Firewall or NAT Gateway for any necessary external calls, APIM in internal mode as the sole entry point if human/system access from outside the VNet is needed |
| Multi-region resilience | Separate Foundry resources per region, your own application-layer sync/routing, do not assume Agent Service will work across a cross-region private endpoint |

---

## Microsoft's Own Reference Architectures

*Confirmed to exist, from the broad best-practices research pass. Worth
reading directly rather than relying on a summary here, since these are
living, maintained architecture guides.*

- [Baseline Microsoft Foundry Chat Reference Architecture](https://learn.microsoft.com/en-us/azure/architecture/ai-ml/architecture/baseline-microsoft-foundry-chat) — the standard starting point for a secure chat-style Foundry deployment.
- [Baseline Microsoft Foundry Landing Zone](https://learn.microsoft.com/en-us/azure/architecture/ai-ml/architecture/baseline-microsoft-foundry-landing-zone) — the same baseline extended into an enterprise landing zone context.
- [Configure secure networking for Azure AI platform services (Cloud Adoption Framework)](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ai/platform/networking) — broader platform-level guidance, not Foundry-specific alone.

---

## 2026 Changelog Timeline

*Confirmed, from Microsoft's own monthly "What's new in Microsoft
Foundry" devblog series.*

| When | What changed |
|---|---|
| March 2026 | Standard Setup with private networking (BYO VNet, no public egress) shipped for Foundry Agent Service. Tool connectivity (MCP servers, AI Search, Fabric data agents) began operating over private network paths. Managed VNet logging added for firewall/NSG/flow-log visibility. |
| May 2026 | Managed VNet reached General Availability. |

Given the pace of change through 2026, it's worth checking the current
devblog series directly for anything more recent than this research pass
before finalizing a design.

---

## Open Questions and Gaps in This Research

Being explicit about what this document does **not** yet answer, since
none of it has been hands-on tested:

1. **The full agent networking topology diagram and IP-allocation math**
   weren't fully retrievable from the deep-dive documentation in this
   pass, the rendered view was truncated. Worth a direct fetch of the raw
   markdown source if that level of detail matters for planning.
2. **The actual Bicep/IaC content in the `roie9876/Azure-AI-Foundry-Networking`
   GitHub repo** wasn't retrievable through a standard web fetch, only
   the folder structure was visible (`bicep/`, `deployment/`, `docs/`,
   `agent-tool/`, `eval/`). If real, runnable templates are needed, this
   repo likely has them, but it needs a proper `git clone` or direct
   raw-file fetch, not just a page fetch.
3. **Whether the APIM-in-front pattern has been tested against a Foundry
   Agent Service endpoint specifically** (as opposed to a plain model
   deployment endpoint) isn't confirmed, the documented example uses a
   model deployment URL, not an agent endpoint.
4. **No hands-on confirmation yet** of any DNS zone setup, private
   endpoint provisioning, or subnet sizing described here, this is all
   from documentation and community reports, not our own testing.
5. **The exact behavior of "not all tools support network isolation"**
   isn't itemized anywhere found in this research, there's no list of
   which specific tools bypass the private path. This would need to be
   checked directly, tool by tool, against whatever tools are actually
   planned for use.

---

## References

| # | Source | What it covers |
|---|---|---|
| 1 | [Azure AI Foundry Architecture — Microsoft Learn](https://learn.microsoft.com/en-us/azure/foundry/concepts/architecture) | Core resource hierarchy, connected resources, RBAC model |
| 2 | [Networking options for Foundry Agent Service — Microsoft Learn](https://learn.microsoft.com/en-us/azure/foundry/agents/concepts/networking-options) | Egress models, inbound access, agent egress controls |
| 3 | [Agents networking deep dive — MicrosoftDocs/azure-ai-docs (GitHub)](https://github.com/MicrosoftDocs/azure-ai-docs/blob/main/articles/foundry/agents/concepts/agents-networking-deep-dive.md) | Hosted vs. Prompt agent traffic paths, the data proxy |
| 4 | [Set up private networking for Foundry Agent Service — Microsoft Learn](https://learn.microsoft.com/en-us/azure/foundry/agents/how-to/virtual-networks) | Canonical private networking how-to |
| 5 | [Configure managed virtual network — Microsoft Learn](https://learn.microsoft.com/en-us/azure/foundry/how-to/managed-virtual-network) | Managed VNet setup, the auto-firewall cost catch |
| 6 | [Add guardrails to a hosted agent — Microsoft Learn](https://learn.microsoft.com/en-us/azure/foundry/agents/how-to/add-hosted-agent-guardrails) | Per-host egress rules, Audit vs. Enforce mode |
| 7 | [Cognitive Services virtual networks — Azure China docs](https://docs.azure.cn/en-us/ai-services/cognitive-services-virtual-networks?tabs=portal) | Firewall rules, VNet service endpoints, private endpoints trade-offs |
| 8 | [Microsoft Foundry cross-region with private endpoints, Part 1 — clouddev.blog](https://clouddev.blog/Azure/AI/Networking/microsoft-foundry-cross-region-with-private-endpoints-part-1/) | Cross-region private endpoint setup, the Agent Service same-region exception |
| 9 | [4.3 Private network setup — DeepWiki, azure-ai-foundry/foundry-samples](https://deepwiki.com/azure-ai-foundry/foundry-samples/4.3-private-network-setup) | Bicep structure patterns, network isolation flags |
| 10 | [roie9876/Azure-AI-Foundry-Networking — GitHub](https://github.com/roie9876/Azure-AI-Foundry-Networking) | Reference repo, content not fully retrievable via web fetch, needs direct clone |
| 11 | [APIM + Foundry private networking pattern — liupeirong.github.io](https://liupeirong.github.io/apimAIFoundry/) | APIM internal mode, managed identity auth policy, NSG rules |
| 12 | [Security architecture for an AI Hub project — azuredoctor.com](https://www.azuredoctor.com/posts/security-ai-hub-project/) | Legacy Hub/Project model, explicitly superseded |
| 13 | [Azure AI Foundry overview — myengineeringpath.dev](https://myengineeringpath.dev/tools/azure-ai-foundry/) | General overview, thin networking content |
| 14 | [Design the network before you deploy — Microsoft Community Hub](https://techcommunity.microsoft.com/blog/azure-ai-foundry-blog/design-the-network-before-you-deploy-best-practices-for-microsoft-foundry-standa/4537860) | BYO VNet best practices for Standard Agents |
| 15 | [Private networking and inference in Microsoft Foundry — Microsoft Community Hub](https://techcommunity.microsoft.com/blog/azure-ai-foundry-blog/private-networking-and-inference-in-microsoft-foundry-architecture-impact-on-ent/4513822) | Enterprise architecture framing |
| 16 | [The Great Foundry Shift: New vs. Classic — Microsoft Community Hub](https://techcommunity.microsoft.com/blog/healthcareandlifesciencesblog/%F0%9F%9A%80-the-great-foundry-shift-microsoft-foundry-new-vs-classic-explained/4499574) | Hub-to-Foundry-native migration context |
| 17 | [Baseline Microsoft Foundry Chat Reference Architecture — Azure Architecture Center](https://learn.microsoft.com/en-us/azure/architecture/ai-ml/architecture/baseline-microsoft-foundry-chat) | Reference architecture |
| 18 | [Baseline Microsoft Foundry Landing Zone — Azure Architecture Center](https://learn.microsoft.com/en-us/azure/architecture/ai-ml/architecture/baseline-microsoft-foundry-landing-zone) | Enterprise landing zone variant |
| 19 | [Configure secure networking for Azure AI platform services — Cloud Adoption Framework](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ai/platform/networking) | Broader platform networking guidance |
| 20 | [Microsoft Q&A: Endpoint host record doesn't resolve](https://learn.microsoft.com/en-in/answers/questions/5551488/endpoint-host-record-provided-by-azure-ai-services) | DNS resolution troubleshooting |
| 21 | [Microsoft Q&A: Private DNS policies with Azure AI Services](https://learn.microsoft.com/en-us/answers/questions/2279428/azure-private-dns-policies-with-azure-ai-services) | DNS forwarding requirements |
| 22 | [Microsoft Q&A: Foundry Agents API blocked by Azure network/firewall rules](https://learn.microsoft.com/en-us/answers/questions/5810010/foundry-(new-portal)-agents-api-blocked-by-azure-a) | Reported 403 issue on the new portal |
| 23 | [Microsoft Q&A: Evaluation run fails with 403 UnauthorizedUserAction](https://learn.microsoft.com/en-au/answers/questions/5911430/azure-ai-foundry-evaluation-run-fails-with-403-una) | Evaluation-specific network reachability issue |
| 24 | What's new in Microsoft Foundry (devblogs.microsoft.com/foundry) | Monthly changelog, March and May 2026 editions cited |
