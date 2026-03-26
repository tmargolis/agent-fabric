# agent-fabric — Technical Specification

**Status:** Investigation / Pre-build
**Last updated:** 2026-03-26

---

## Purpose of This Document

This spec defines the intended architecture of agent-fabric, the open questions that must be answered before each layer can be built, and the external standards and artifacts that inform each decision. It is organized by architectural layer. Each section includes: what the layer does, what is known, what must be investigated, and what decisions need to be made.

---

## Background and Positioning

### What Agent-Fabric Is Not

- Not an orchestration framework (LangGraph, CrewAI, OpenAI Agents SDK already exist)
- Not an LLM provider abstraction
- Not an agent runtime
- Not a business-logic layer

### What Agent-Fabric Is

A **vendor-neutral control plane** that sits above runtimes and provides:
- Fleet inventory and capability discovery across vendors
- Identity, trust, and delegated authority across agent boundaries
- Policy enforcement at the tool/action layer
- Durable orchestration and harness management for long-running tasks
- Unified observability and outcome assurance

The distinction that matters: orchestration answers "what should happen next?" — agent-fabric answers "is what's happening safe, authorized, correct, and attributable?"

---

## Layer Specifications

---

### Layer 1: Inventory and Registry

**What it does:**
Maintains a live registry of known agents — first-party and third-party — including their capabilities, health, and current task state.

**What is known:**
- A2A defines `Agent Cards` as the standard capability-discovery document (JSON, published at a well-known URI)
- Agent Cards can carry identity tags, capability declarations, and compliance metadata
- A2A supports optional digital signatures (JWS) and canonicalization (JCS/RFC 8785) for Card authenticity
- AGNTCY (Linux Foundation) is building a directory layer explicitly for A2A agent and MCP server discoverability

**Open questions:**
- [ ] Should agent-fabric implement its own registry or federate with AGNTCY-style directories?
- [ ] How do we handle ephemeral agents (task-scoped, not long-lived) in the registry?
- [ ] What is the TTL/staleness model for Agent Card data?
- [ ] Should health status (is this agent currently reachable / performing?) be part of the registry or a separate liveness layer?
- [ ] How are third-party agents that don't publish A2A Agent Cards discovered and cataloged?

**Design decisions needed:**
- Registry backend: embedded (sqlite/KV), external (etcd/Consul), or federated with AGNTCY directory?
- Push vs. pull model for Agent Card freshness
- Schema for internal agent metadata beyond what A2A Agent Cards provide

**References:**
- [A2A Specification — Agent Cards](https://github.com/a2aproject/A2A/blob/main/docs/specification.md)
- [AGNTCY docs](https://docs.agntcy.org/)

---

### Layer 2: Identity and Trust

**What it does:**
Establishes who an agent is, who it is acting on behalf of, and what authority it carries — in a way that is cryptographically verifiable and revocable.

**What is known:**
- A2A supports Agent Card signing (JWS) for authenticity but does not mandate it
- A2A explicitly warns against in-band credential propagation across chained agents; recommends secure out-of-band handling for in-task authorization
- W3C DIDs (Decentralized Identifiers) define a standard for verifiable agent identity not tied to a vendor
- W3C Verifiable Credentials provide a tamper-evident claims model (issuer/holder/verifier) that can represent delegated authority, compliance status, or scoped permissions
- The gap: identity standards exist but are not operationally integrated into A2A/MCP protocol flows

**Open questions:**
- [ ] What is the right identity primitive for an agent? (DID? OAuth client credential? Vendor-issued keypair?)
- [ ] How is delegated authority represented when User → Agent A → Agent B? What does Agent B receive as proof of delegation scope?
- [ ] How is authority revoked mid-task? (Agent B is acting when User revokes Agent A's delegation)
- [ ] How do we handle the "blast radius" problem: chained agents that inherit too much scope?
- [ ] Should identity be per-agent-instance or per-agent-type?
- [ ] Can Verifiable Credentials be practically embedded into A2A task requests without breaking existing A2A clients?

**Design decisions needed:**
- Identity anchor: DID-based, OAuth 2.0 client credentials, or hybrid?
- Delegation chain representation format (JWT, VC, custom?)
- Revocation mechanism: immediate (token revocation list) vs. short-lived tokens with no long-lived state
- How identity integrates with the Policy Engine (Layer 3)

**References:**
- [A2A Spec — security and authentication](https://github.com/a2aproject/A2A/blob/main/docs/specification.md)
- [W3C DIDs v1.0](https://www.w3.org/TR/did-core/)
- [W3C Verifiable Credentials v2.0](https://www.w3.org/TR/vc-data-model-2.0/)

---

### Layer 3: Policy and Authorization Engine

**What it does:**
Encodes and enforces least-privilege rules, approval thresholds, and "safe tool boundaries" for every agent action — before execution, not after.

**What is known:**
- MCP's specification centers user consent/control and tool safety as non-negotiable — but leaves enforcement to the host/client layer (not the protocol)
- OpenAI's security guidance explicitly requires: least privilege, explicit consent for write access, assuming prompt injection will occur, and full audit logs
- OWASP Top 10 for LLM elevates "excessive agency" as a primary risk — agents acquiring more permissions or scope than required for the task
- Google's Antigravity codelabs include explicit browser allowlist security guidance, showing that policy must be built into the agent harness, not bolted on
- Enterprise control planes (AI Control Tower, PwC agent OS) implement RBAC + governance workflows but with proprietary policy languages

**Open questions:**
- [ ] What is the right policy language for agent actions? OPA/Rego? Cedar? Custom DSL?
- [ ] How are policies attached to agents vs. tools vs. tasks? (Who "owns" a policy: the agent, the tool server, the control plane?)
- [ ] How do we handle prompt injection as an orchestration-level risk, not just a model-level risk?
- [ ] What is the approval gate model for irreversible actions (financial, identity-affecting, legal)? Synchronous human approval? Async with timeout fallback?
- [ ] How do policies compose when multiple agents with different policies operate on the same task?
- [ ] Can policy violations be surfaced to the Observability layer (Layer 7) as structured events?

**Design decisions needed:**
- Policy language and evaluation engine
- Policy attachment model (per-agent, per-tool, per-task-type, per-scope)
- Approval gate mechanics (sync/async, escalation, timeout behavior)
- How policy integrates with the Identity layer (delegated scope constraints)

**References:**
- [MCP Specification — security principles](https://modelcontextprotocol.io/specification/2025-11-25)
- [OpenAI Security & Privacy guidance](https://developers.openai.com/apps-sdk/guides/security-privacy/)
- [OWASP Top 10 for LLM — LLM08: Excessive Agency](https://owasp.org/www-project-top-10-for-large-language-model-applications/)
- [OWASP Top 10 — LLM01: Prompt Injection](https://genai.owasp.org/llmrisk/llm01-prompt-injection/)

---

### Layer 4: Orchestration and Execution

**What it does:**
Routes tasks to agents, manages state transitions, handles handoffs between agents, and decomposes complex goals into subtasks.

**What is known:**
- A2A formalizes task lifecycle: creation → working → artifact → completion, with structured `Task` objects and `Artifact` outputs
- LangGraph (graph-based routing), LlamaIndex Workflows (event-driven, pause/resume), Microsoft Agent Framework, OpenAI Agents SDK (handoffs + streaming), and CrewAI (role-based crews) are the dominant orchestration primitives
- Anthropic's harness research identifies "initializer agents," "handoff artifacts," and "context compaction" as the primary patterns for bridging long-running sessions
- Context resets (clean slate with explicit handoff artifacts) are sometimes more reliable than compaction alone
- Evaluator/generator separation is a core harness lever: separate agents for producing and grading output reduces self-evaluation bias

**Open questions:**
- [ ] Should agent-fabric implement its own orchestration runtime or act as a governance wrapper around existing runtimes (LangGraph, etc.)?
- [ ] How do handoff artifacts get structured so they are runtime-agnostic? (Can a LangGraph task hand off to a CrewAI agent via agent-fabric?)
- [ ] What is the context reset protocol? Who decides when a reset is needed, and what must the handoff artifact contain?
- [ ] How does the evaluator/generator separation get surfaced as a first-class primitive in agent-fabric rather than a pattern each team reimplements?
- [ ] How does task state persist during human approval gates (paused tasks)?

**Design decisions needed:**
- Orchestration model: native runtime vs. façade/adapter over existing runtimes
- Handoff artifact schema
- Context management policy: compaction thresholds, reset triggers
- Evaluator agent attachment model

**References:**
- [Anthropic: Effective harnesses for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)
- [Anthropic: Harness design for long-running application development](https://www.anthropic.com/engineering/harness-design-long-running-apps)
- [A2A Specification — Task lifecycle](https://github.com/a2aproject/A2A/blob/main/docs/specification.md)
- [LangGraph](https://www.langchain.com/langgraph)
- [LlamaIndex Workflows](https://www.llamaindex.ai/workflows)
- [OpenAI Agents SDK](https://developers.openai.com/api/docs/guides/agents-sdk/)

---

### Layer 5: Durable Execution

**What it does:**
Ensures long-running tasks survive failures, retries, and intermittent tool/API unavailability without requiring agent logic to implement its own retry/recovery.

**What is known:**
- Temporal explicitly positions itself as a durable workflow layer for AI applications, focusing on state persistence, retries, and failure modes when tools are flaky or long-running
- Long-running agent tasks can span hours to days; the execution layer must survive process crashes, network partitions, and human review delays
- The "Ralph Wiggum loop" pattern (continuous retry-until-done) shows how durable loops replace constant human prompting — but requires strong verifiers to avoid infinite loops or premature success declarations

**Open questions:**
- [ ] Should agent-fabric use Temporal as its durable execution backend, or implement a lighter-weight alternative?
- [ ] How does durable execution interact with context resets? (If a task resumes from a checkpoint, does the agent get a fresh context window or a compacted one?)
- [ ] What is the failure taxonomy? (Tool failure vs. agent failure vs. policy violation vs. timeout vs. human rejection — each has different recovery semantics)
- [ ] How are durable task checkpoints represented and stored? What data must be in a checkpoint vs. can be recomputed?
- [ ] What is the maximum task duration, and how does that interact with model context limits?

**Design decisions needed:**
- Durable execution backend (Temporal, custom event sourcing, or embedded)
- Checkpoint schema and storage
- Failure taxonomy and recovery strategy per failure type
- Interaction contract between durable execution and the Harness layer

**References:**
- [Temporal for AI](https://temporal.io/solutions/ai)
- [Ralph Loop Agent](https://github.com/vercel-labs/ralph-loop-agent)
- [Anthropic: Effective harnesses for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)

---

### Layer 6: Tool and Data Connectivity (Harness Boundary)

**What it does:**
Standardizes how agents connect to tools and data sources, and enforces that every tool call passes through a harness that validates scope, captures provenance, and applies context management.

**What is known:**
- MCP is the standard: JSON-RPC 2.0 between hosts (LLM apps), clients (connectors), and servers (tool/data providers)
- MCP is now multi-vendor: OpenAI's Responses API supports MCP tool connectors; Google supports MCP in Antigravity IDE
- MCP defines tools, resources, and prompts on the server side; sampling, roots, and elicitation on the client side
- Tool calls frequently cross trust boundaries — a remote MCP server may have its own data retention and compliance posture
- MCP security model: user consent and tool safety are application-layer responsibilities, not protocol-level enforcement

**Open questions:**
- [ ] How does agent-fabric proxy MCP calls to add policy enforcement, consent verification, and audit logging without becoming a bottleneck?
- [ ] How is data provenance tracked across tool calls? (Agent A reads from Tool X; that data flows into Agent B's context — who authorized that?)
- [ ] What is the posture toward third-party MCP servers that agent-fabric doesn't control? (Trust levels, sandboxing, allowlists)
- [ ] How do we handle "MCP servers as supply chain risk"? (A malicious or compromised tool server injecting context)
- [ ] Can tool-level consent be encoded in the Policy Engine (Layer 3) and enforced here automatically?

**Design decisions needed:**
- MCP proxy architecture (intercepting proxy vs. SDK wrapping vs. sidecar)
- Data provenance model and storage
- Trust tier model for MCP servers (first-party / verified third-party / untrusted)
- Context contamination mitigation for untrusted tool responses

**References:**
- [MCP Specification](https://modelcontextprotocol.io/specification/2025-11-25)
- [OpenAI: MCP and Connectors](https://developers.openai.com/api/docs/guides/tools-connectors-mcp/)
- [OpenAI: Data controls](https://developers.openai.com/api/docs/guides/your-data/)
- [OWASP LLM03: Supply Chain](https://owasp.org/www-project-top-10-for-large-language-model-applications/)

---

### Layer 7: Observability

**What it does:**
Emits and aggregates structured telemetry across all agents, tool calls, and task transitions to provide a unified audit trail and debugging surface.

**What is known:**
- OpenTelemetry has defined GenAI semantic conventions (spans, events, attributes) for standardizing telemetry across GenAI model calls and agentic operations
- Without standard telemetry, a multi-vendor agent fleet is un-debuggable — every tool invents its own trace format
- LangSmith positions itself as an "agent black box recorder": end-to-end traces including tool calls, decision points, cost/latency, and evaluation loops
- Google's Antigravity Manager Surface shows what "mission control" UX looks like for observing multiple agents asynchronously

**Open questions:**
- [ ] Should agent-fabric emit OTel traces natively, or wrap/translate existing runtime-specific trace formats?
- [ ] What is the minimum viable telemetry schema that enables cross-vendor audit trails?
- [ ] How do we correlate telemetry across chained A2A calls where each agent may be on a different vendor's infrastructure?
- [ ] What is the retention and access-control model for telemetry data? (Traces contain sensitive tool inputs/outputs)
- [ ] How do we surface policy violations and identity events as structured observability events (not just logs)?

**Design decisions needed:**
- OTel integration approach (native vs. wrapper)
- Telemetry schema extensions beyond OTel GenAI conventions
- Trace correlation model for cross-vendor agent chains
- Storage and query backend for traces

**References:**
- [OpenTelemetry GenAI Semantic Conventions](https://opentelemetry.io/docs/specs/semconv/gen-ai/)
- [LangSmith observability](https://www.langchain.com/langsmith/observability)
- [Google Antigravity Manager Surface](https://developers.googleblog.com/build-with-google-antigravity-our-new-agentic-development-platform/)

---

### Layer 8: Evaluation and Outcome Assurance

**What it does:**
Continuously verifies that agent outputs are actually correct, not just self-declared done — using structured evaluators separate from the agents doing the work.

**What is known:**
- Anthropic identifies self-evaluation bias as a primary long-running agent failure mode: agents give mediocre outputs high marks when evaluating their own work
- Separating evaluator and generator agents (using distinct agent instances with a skeptical evaluator stance) is the most concrete lever identified for improving outcome quality
- Anthropic's parallel Claudes C compiler experiment establishes that test harness quality is the primary determinant of autonomous agent performance — without strong verifiers, continuous loops become chaotic
- The "Ralph Wiggum loop" is powerful but incomplete: a retry loop without a strong stopping condition becomes an automation liability
- Outcome assurance is still more art than science — no standardized component exists

**Open questions:**
- [ ] How do evaluator agents get attached to tasks in agent-fabric? (Always-on? Triggered by confidence thresholds? Task-type-specific?)
- [ ] What is the interface contract between a generator agent and its evaluator? (What does the evaluator receive, and what format do its verdicts take?)
- [ ] How do we handle subjective tasks where "correct" is not binary? (Draft quality, negotiation outcomes, creative work)
- [ ] What is the escalation path when an evaluator repeatedly rejects an agent's output? (Human-in-loop? Alternative agent? Task abort?)
- [ ] Can evaluator verdicts feed back into the Registry (Layer 1) to update agent capability and quality metadata over time?
- [ ] How do we prevent evaluator agents from being compromised by the same prompt injection risks as generator agents?

**Design decisions needed:**
- Evaluator attachment model (per-task-type registry, dynamic, always-on)
- Verdict schema (structured pass/fail/partial with reasoning)
- Escalation policy for repeated evaluation failures
- Feedback loop from evaluator verdicts to agent registry

**References:**
- [Anthropic: Effective harnesses for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)
- [Anthropic: Harness design for long-running application development](https://www.anthropic.com/engineering/harness-design-long-running-apps)
- [Anthropic: Building a C compiler with parallel Claudes](https://www.anthropic.com/engineering/building-c-compiler)
- [Ralph Loop Agent](https://github.com/vercel-labs/ralph-loop-agent)

---

## Cross-Cutting Concerns

### Governance and Compliance

Agent-fabric must be able to map its enforcement surface to external governance frameworks — not as a compliance checkbox, but because these frameworks define the requirements backlog for what a control plane must measure.

- **NIST AI RMF 1.0** — risk management across the AI lifecycle: roles, approvals, audit, incident response, continuous improvement
- **ISO/IEC 42001** — AI management system standard: transparency, accountability, risk management

**Open questions:**
- [ ] What is the minimum set of NIST AI RMF controls that agent-fabric provides out of the box?
- [ ] How do we represent governance posture to downstream enterprise consumers without hardcoding compliance assumptions?

**References:**
- [NIST AI RMF 1.0](https://nvlpubs.nist.gov/nistpubs/ai/nist.ai.100-1.pdf)
- [ISO/IEC 42001](https://www.iso.org/standard/42001)

---

### Personal vs. Enterprise Control Plane Unification

Existing control planes are either enterprise-centric (ServiceNow, Dataiku, PwC) or developer-centric (Antigravity). Neither spans consumer identity + enterprise identity + multi-organization interoperability.

The missing category: a **personal agent control plane** that can:
- Act on behalf of an individual user with consumer identities (OAuth, retail accounts, messaging platforms)
- Interoperate with enterprise systems and third-party agents via A2A
- Maintain the same governance and observability model across both contexts

**Open questions:**
- [ ] Is this a separate product mode (personal vs. enterprise), or can one control plane serve both with different policy configurations?
- [ ] What does user-level delegation look like when agents act across personal and work identity boundaries?
- [ ] How do consent models differ between consumer (explicit, granular, revocable) and enterprise (delegated to IT admin) contexts?

---

## Key Standards Summary

| Standard | Layer(s) | Status | Notes |
|---|---|---|---|
| A2A (Agent2Agent) | Registry, Identity, Orchestration | Live, Linux Foundation | Agent Cards, task lifecycle, signing |
| MCP (Model Context Protocol) | Tool Bus | Live, multi-vendor | JSON-RPC 2.0, tool/resource/prompt primitives |
| OpenTelemetry GenAI | Observability | Live, evolving | Semantic conventions for model calls + agentic ops |
| W3C DIDs | Identity | Stable W3C Rec | Decentralized agent identity anchor |
| W3C Verifiable Credentials | Identity, Policy | Stable W3C Rec | Delegated authority, compliance claims |
| OWASP LLM Top 10 | Policy, Harness | Stable, updated regularly | Threat model for agentic systems |
| NIST AI RMF | Governance | Stable | Risk management framework |
| ISO/IEC 42001 | Governance | Published 2023 | AI management system standard |

---

## Investigation Priorities

Ordered by which unknowns will block architecture decisions most immediately:

1. **Identity and delegation model** — blocks Policy, Registry, and Observability design
2. **Policy language selection** — blocks tool call enforcement and approval gate design
3. **Orchestration strategy** (native runtime vs. facade) — blocks Harness and Durable Execution design
4. **MCP proxy architecture** — blocks Tool Bus and data provenance design
5. **Evaluator attachment model** — blocks Evaluation layer and feedback loop design
6. **Registry backend and federation approach** — can proceed in parallel once Identity is clearer
7. **OTel integration approach** — largely decoupled; can progress independently

---

## Competitive Landscape Summary

| Product | Type | Strengths | Gaps |
|---|---|---|---|
| ServiceNow AI Control Tower + Agent Fabric | Enterprise | Governance, RBAC, third-party agent visibility | Platform-locked, enterprise-only |
| Dataiku Agent Hub | Enterprise | Lifecycle management, design + deploy | Dataiku platform dependency |
| WSO2 Agent Manager | Enterprise | Multi-framework, zero-trust, OWASP alignment | Integration complexity |
| PwC agent OS | Enterprise | MCP integration, governance, RBAC | Consulting-delivered, not self-serve |
| Google Antigravity | Developer | Mission control UX, MCP store, artifact review | Dev-only scope, Google-centric |
| LangSmith | Observability | Deep LangChain integration, traces, eval loops | LangChain ecosystem dependency |

None span consumer identity, are vendor-neutral end-to-end, and expose governance as a portable, policy-as-code layer.
