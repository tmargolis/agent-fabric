# agent-fabric

A vendor-neutral control plane for heterogeneous, cross-vendor autonomous agent ecosystems.

---

## Vision

We are moving toward a world where agents mediate most of our routine interactions — not just with companies and services, but with each other. Your agents talk to your colleagues' agents. They negotiate with your suppliers' agents. They track your purchases, manage your schedule, find the best deal, and handle the back-and-forth that currently requires your attention and often requires another person's attention on the other side.

This isn't far off. The question isn't whether it will happen — it's whether the infrastructure beneath it gets built thoughtfully.

**The future state this project is working toward:**

You have a set of agents. They handle your Amazon purchases — finding the right product, comparing prices, tracking shipping. They coordinate with your work colleagues' agents to schedule meetings, share context, and move projects forward. They communicate with your friends' agents to make plans, share recommendations, surface things you'd each care about. You don't manage each of these interactions. Instead, you have a single layer that shows you what your agents are doing, ensures they're using the right data, flags decisions that need your judgment, and tells you when outcomes aren't tracking right.

Your friends and colleagues use different AI vendors — Anthropic, OpenAI, Google, others. That shouldn't matter. Their agents and yours interoperate, and the quality of the interaction doesn't depend on which platform anyone chose.

This is what agent-fabric is for.

---

## The Risk Worth Taking Seriously

This future has a real shadow. COVID gave us a preview.

During lockdowns, human-to-human interaction dropped sharply — and the effects were not uniform. Productivity adapted faster than wellbeing. Adults with established social networks found workarounds. But younger people, whose social and developmental needs depend on unmediated physical interaction, faced measurable harm: elevated anxiety and depression, disrupted identity formation, weakened peer relationships, a collapse of the casual spontaneous contact that doesn't survive being scheduled on a calendar.

The risk with a heavily agent-mediated world isn't that people stop interacting — it's that the texture of interaction changes in ways we don't fully account for. If agents handle the friction and coordination of social life, do people lose the practice of navigating that friction themselves? If your agent can always find you a better deal faster, do you lose the serendipity of browsing — the unplanned discovery, the conversation it starts? If coordination becomes frictionless, does it also become shallow?

These aren't arguments against building this. They're arguments for building it with intention.

**Agent-fabric's design commitment:** the control plane layer should make human contact easier to choose, not harder to maintain. Agents should handle the transactional so that human attention can concentrate on what is actually relational. The goal is not to replace human interaction — it is to reduce the overhead that competes with it.

**Concrete design implications:**
- Agents should surface "a human may want to handle this" as a first-class outcome, not a failure mode
- The control plane should make it easy to see which interactions have been delegated and easy to step back in
- Agents communicating with other humans' agents should not obscure the human behind the agent — identity and representation matter
- The system should be auditable enough that you can always understand what was done on your behalf and why

**Open questions to investigate:**
- What interaction types should agents never fully own? (Emotional conversations, conflict resolution, relationship-defining decisions?)
- How do we design agent behavior that preserves the social byproducts of "friction" — the serendipity, the unplanned contact, the developmental value of working something out yourself?
- What do we know from COVID research about which populations are most at risk from reduced direct human contact, and how does that inform design constraints?
- How does IRL interaction change when routine coordination is handled by agents? Does it concentrate or diminish?
- What signals would tell us the system is working well for human flourishing, not just task throughput?

---

## What This Is

Agent-fabric is an infrastructure layer that sits *above* individual agent runtimes and *below* business applications. It does not replace orchestration frameworks (LangGraph, CrewAI, OpenAI Agents SDK) — it makes a fleet of agents built on those frameworks **governable, observable, interoperable, and auditable** across vendor boundaries.

The category has a name: **agent control plane**. Forrester formally defines it as "an enterprise control plane that inventories, governs, orchestrates, and assures heterogeneous agents across vendors and domains." Market products exist (ServiceNow AI Control Tower, Dataiku Agent Hub, WSO2 Agent Manager, PwC agent OS), but they are either platform-locked, enterprise-only, or developer-tooling-only. None address the full surface area.

Agent-fabric is a bet that this layer should be:
- **Vendor-neutral** — not owned by a cloud provider or LLM vendor
- **Protocol-first** — built on A2A + MCP + OpenTelemetry as the interoperability substrate
- **Personally and organizationally applicable** — spanning consumer identity, enterprise identity, and multi-organization agent coordination

---

## The Problem in One Paragraph

When you run a fleet of agents — some yours, some from other vendors, operating on long-running tasks — you immediately hit a set of problems that orchestration frameworks don't solve: How do you *discover* what agents exist and what they can do? How do you *trust* that an agent acting on behalf of a user actually has the right delegation? How do you *enforce* that a tool call has the right scope? How do you *know* when an outcome is actually correct and not just self-declared done? And how do you do all of this **without** requiring a human to coordinate between every agent boundary?

---

## Why Now

Three converging signals make this the right time to build:

1. **Standards are real.** A2A (Agent-to-Agent, now under Linux Foundation governance) and MCP (Model Context Protocol, adopted by OpenAI, Anthropic, Google) are live and gaining adoption. They define agent discovery, task lifecycle, and tool connectivity. They do *not* define governance, policy enforcement, or outcome assurance.

2. **Harness engineering is the differentiator.** Anthropic's engineering research establishes that long-running agent performance is determined by the "outer loop": context management, evaluator/generator separation, verifiers, and reset strategies — not the model or the orchestration graph. This is buildable infrastructure, not research.

3. **Control-plane products are fragmented.** Every major enterprise platform is building a control tower. All are bundling proprietary assumptions about identity, policy, and evaluation. The interoperability gap is structural, not incidental.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                   Applications & Consumers                  │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│                     agent-fabric                            │
│                                                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │  Inventory  │  │  Identity   │  │   Policy / Authz    │  │
│  │  & Registry │  │  & Trust    │  │   Engine            │  │
│  └─────────────┘  └─────────────┘  └─────────────────────┘  │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐    │
│  │           Orchestration & Execution Layer           │    │
│  │   Workflow Runtime │ Durable Execution │ Harness    │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                             │
│  ┌──────────────────┐  ┌──────────────────────────────┐     │
│  │  Tool / Data Bus │  │  Observability & Evaluation  │     │
│  │  (MCP)           │  │  (OTel + Evaluators)         │     │
│  └──────────────────┘  └──────────────────────────────┘     │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│          Agent Runtimes (LangGraph, CrewAI, Agents SDK…)    │
│          MCP Servers (tools, data sources)                  │
│          External / Third-party Agents (via A2A)            │
└─────────────────────────────────────────────────────────────┘
```

---

## Layers at a Glance

| Layer | What It Does | Key Standards/Tech |
|---|---|---|
| Inventory & Registry | Discover and catalog agents, capabilities, health | A2A Agent Cards, AGNTCY directory |
| Identity & Trust | Sign, verify, and delegate agent authority | A2A JWS signing, W3C DIDs, Verifiable Credentials |
| Policy / Authz Engine | Enforce least-privilege, approval gates, scope | MCP consent model, RBAC, OWASP LLM Top 10 |
| Orchestration & Execution | Route tasks, manage state, handle handoffs | LangGraph / LlamaIndex Workflows / Temporal |
| Durable Execution | Persist long-running tasks across failures | Temporal, event-driven workflow backends |
| Harness Boundary | Context management, tool safety, verifiers | Anthropic harness patterns |
| Tool / Data Bus | Standardized tool and data connectivity | MCP (JSON-RPC 2.0) |
| Observability | Unified telemetry across vendors | OpenTelemetry GenAI semantic conventions |
| Evaluation | Continuous outcome assurance, evaluator separation | Evaluator/generator pattern, regression suites |

---

## What Is Still Unsolved (The Investigation Surface)

The following areas have no settled answer and are the core research targets for this project:

1. **Agent identity and delegated authority** — How does an agent prove who it's acting on behalf of, and how is that authority scoped, chained, and revoked across vendor boundaries?
2. **Outcome assurance** — How do you define and enforce a "done" condition that cannot be gamed by self-evaluation?
3. **Cross-vendor trust propagation** — When Agent A (yours) calls Agent B (third-party), what trust surface gets inherited? How is blast radius bounded?
4. **Personal vs. enterprise control plane unification** — Consumer identity (OAuth, retail, scheduling) and enterprise identity (SAML, SCIM, RBAC) are separate worlds. Can one control plane span both?
5. **Policy-as-code for agent actions** — What does a portable, enforceable policy language for agent tool calls and task transitions look like?

These are expanded into investigation sections in [SPEC.md](SPEC.md).

---

## Reference Reading

- [Forrester: Announcing Our Evaluation of the Agent Control Plane Market](https://www.forrester.com/blogs/announcing-our-evaluation-of-the-agent-control-plane-market/)
- [Google: Announcing the Agent2Agent Protocol (A2A)](https://developers.googleblog.com/en/a2a-a-new-era-of-agent-interoperability/)
- [Anthropic: Effective harnesses for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)
- [Anthropic: Harness design for long-running application development](https://www.anthropic.com/engineering/harness-design-long-running-apps)
- [MCP Specification](https://modelcontextprotocol.io/specification/2025-11-25)
- [A2A Specification](https://github.com/a2aproject/A2A/blob/main/docs/specification.md)
- [AGNTCY — Agent directory and messaging infrastructure](https://docs.agntcy.org/)
- [OpenTelemetry GenAI Semantic Conventions](https://opentelemetry.io/docs/specs/semconv/gen-ai/)
- [OWASP Top 10 for LLM Applications](https://owasp.org/www-project-top-10-for-large-language-model-applications/)
- [Anthropic: Building a C compiler with parallel Claudes](https://www.anthropic.com/engineering/building-c-compiler)
- [Ralph Loop Agent — continuous autonomy pattern](https://github.com/vercel-labs/ralph-loop-agent)

---

## Status

Early-stage. Spec and investigation in progress. See [SPEC.md](SPEC.md) for open questions and design decisions.

---

## License

### Creative Commons Attribution-NonCommercial-NoDerivatives 4.0 International (CC BY-ND 4.0)

#### What is Permitted:
- Viewing the work
- Forking the work for discussion
- Commenting on the work

#### What is Prohibited:
- Modifications of the work
- Commercial use without permission

#### Attribution Requirements:
- You must give appropriate credit, provide a link to the license, and indicate if changes were made. You may do so in any reasonable manner, but not in any way that suggests the licensor endorses you or your use.

#### Contact Information for Licensing Inquiries:
- For questions regarding licensing, please contact [info@toddmargolis.net].

