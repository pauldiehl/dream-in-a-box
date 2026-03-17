# Decision Log

Record every architectural decision here. Especially decisions NOT to do something.

Format:
```
## [Date] — [Decision]
**Context:** Why this came up
**Decision:** What we decided
**Rationale:** Why
```

---

## 2026-03-14 — Standalone repo, not under sovereign-streams

**Context:** DIAB could live as a subdirectory of sovereign-streams (the Web 4.0 theory repo) or as its own project.
**Decision:** Standalone repo. Import what we need from sovereign-streams; don't nest inside it.
**Rationale:** Sovereign-streams is exhaustive — theory, POCs, demos, philosophical docs. DIAB is concrete implementation. It needs to stand alone so it can be cloned, forked, and run independently. Less baggage, clearer purpose.

## 2026-03-14 — Forkless and Greenspaces are services, not dreams

**Context:** Initially modeled Forkless as a standalone product ("dream") running on a node.
**Decision:** Forkless and Greenspaces are Layer 2 services — node-level tools available to every dream.
**Rationale:** A dream is a product/experience (Plain Fun, Coach Kid, Man vs Health). Forkless is the conversational commerce engine that powers the agent interface for ANY dream. Greenspaces is the community layer for ANY dream. They're infrastructure, not products. Every dream gets them the same way every OS gets a filesystem.

## 2026-03-14 — Protocol-first development

**Context:** Standard approach is code-first with documentation after.
**Decision:** Every significant capability starts as a protocol file (markdown/JSON) before any code is written.
**Rationale:** Protocols are the source of truth. They're human-readable, versionable, and portable between nodes. Code implements protocols. If you write code first, you end up with undocumented behavior that can't be shared, reviewed, or overridden. Protocols can exist before implementation — that's a feature.

## 2026-03-14 — Adopt four operational patterns from Paperclip analysis

**Context:** Analyzed Paperclip (paperclip.ing) — an open-source agent orchestration framework. Good infrastructure patterns, wrong product philosophy.
**Decision:** Adopt Heartbeat Pattern, Atomic Budget Enforcement, Task-to-Goal Traceability, and Multi-Company Data Isolation as core node agent patterns.
**Rationale:** These are infrastructure patterns, not product opinions. They solve real problems: background intelligence (heartbeat), cost control (budget), audit trails (traceability), and multi-dream isolation. Steal the patterns, don't adopt the framework. These are Layer 0 concerns — they belong in DIAB, not in individual services or dreams.

## 2026-03-16 — Domain-Driven Protocols as the business logic layer

**Context:** Needed a concrete framework for how dreams define their customer journeys and business logic. DDD (Eric Evans) provides proven patterns for domain modeling, but traditional implementations bury domain logic in code only developers can read.
**Decision:** Adopt Domain-Driven Protocols (DDP) — DDD concepts expressed as human-readable protocol files that agents execute directly. Journey Mappings (Aggregate Roots) contain User Flows (Entities) which produce Artifacts (Value Objects) driven by Stage Transitions (Domain Events). Each JM/UF stage gets its own DB schema.
**Rationale:** The domain model should be readable by founders, executable by agents, and implementable by developers. Protocol files achieve all three. Traditional DDD buries the ubiquitous language in class hierarchies. DDP makes it the literal document everyone works from. See docs/DOMAIN-DRIVEN-PROTOCOLS.md.

## 2026-03-16 — No gatekeeping: deliver value immediately

**Context:** Traditional web apps require complex intake, onboarding, and meetings before showing the customer anything useful. This is gatekeeping.
**Decision:** Every dream delivers a plain-vanilla plan/offering immediately on first interaction. No intake required. No questions first (unless customer engages). Show them what we've got. Customization happens through conversation after the first look.
**Rationale:** "SHOW ME WHAT YOU GOT" is the correct customer expectation. A first-look plan won't be perfect — it doesn't need to be. It proves the system has value. Customization CTAs route back to conversation (the agent updates the JSON driving the readonly view). This eliminates the biggest conversion killer in SaaS: making people work before showing them anything.

## 2026-03-16 — Readonly artifacts replace interactive UX

**Context:** Modern web apps are source-heavy, failure-ridden UX with constant hunger for developer babysitting. Every interactive component is a potential bug surface.
**Decision:** All customer-facing content is readonly — HTML/markdown files driven by JSON configs. No form fields, no state mutations from the UI, no interactive components beyond minimal CTAs that route to conversation. The agent is the only writer; the artifact is a projection.
**Rationale:** Readonly means: no validation logic, no event handlers, no framework dependencies, no accessibility edge cases in interactive components, no security surface from user input. Less real estate = less code = less review = fewer tokens to find problems. The three surfaces are: conversation (agent layer), readonly artifacts (transaction layer), and minimal fulfillment moments (payments, signatures — the concession to current reality).

## 2026-03-16 — Admin and customer portals are conversations, not dashboards

**Context:** Traditional SaaS builds admin dashboards and customer portals with dozens of screens, settings, and configuration options.
**Decision:** Admin "portal" is a conversation with the operations agent plus a queue/inbox of items needing attention. Customer "portal" is scheduled engagement + on-demand conversation + readonly artifacts. No dashboards. No settings pages. No toggles.
**Rationale:** Every dashboard screen is a maintenance burden and a failure point. Business users don't need 47 admin screens — they need to talk to an agent that has full context from protocol execution. Customers don't need a settings page — they need to tell the agent what they want changed. This is agentic execution: if the agent interface works, we get massive wins in development speed and business momentum.

## 2026-03-16 — JM as finite state machine, not workflow engine

**Context:** Customer journeys need state tracking. Options: status flags on user records (messy), workflow engines like Temporal/Step Functions (overkill), or a purpose-built state machine.
**Decision:** Journey Mappings are finite state machines backed by a single `journey_states` SQLite table with JSON `state_data`. Per-type transition maps define valid state changes. A `transitionState()` validator enforces consistency. Side effects fire automatically on transitions.
**Rationale:** Proven in MVH production. One table, ~200 lines of JS, handles the full lifecycle of any multi-step customer process. Temporal is a separate service to deploy and maintain with a steep learning curve — overkill for "track where a customer is and do something when they stall." The admin queue derives from the same table (SQL query with priority logic), eliminating the need for a separate queue infrastructure. See docs/MVH-CASE-STUDY-JM-AND-SCHEDULER.md.

## 2026-03-16 — Embedded scheduler over external orchestration

**Context:** Dreams need time-based actions: reminders, follow-ups, stall detection, recurring check-ins. Options: external orchestration (Temporal, Step Functions), cron jobs, or embedded polling.
**Decision:** Embedded scheduler — a `scheduler_events` SQLite table polled by `setInterval` every 5 minutes inside the Express process. JSON callbacks for conditional logic. Recursive event chaining (one event creates the next) instead of cron expressions.
**Rationale:** Proven in MVH production. No separate service to deploy. If Express is running, the scheduler is running. If it crashes, both restart together. Recursive chaining is simpler than cron: each customer has exactly ONE pending event at any time, cancellation = stop chaining, frequency changes are instant, and each event fires state-aware. The callback mechanism makes the scheduler smart (suppress, override, chain, escalate) without complex application logic. <500 lines of JS total for both JM engine and scheduler. See docs/MVH-CASE-STUDY-JM-AND-SCHEDULER.md.

## 2026-03-17 — Local vector DB (sqlite-vss) over external vector services

**Context:** The node needs memory and pattern recognition — the ability to recognize known requests and serve proven responses instead of computing everything from scratch via LLM. Options: cloud vector services (Pinecone, Weaviate, Qdrant), self-hosted vector DB (Milvus, ChromaDB), or embedded in-process (sqlite-vss).
**Decision:** Use sqlite-vss (SQLite Vector Similarity Search extension) with a local embedding model (all-MiniLM-L6-v2 via ONNX, 80MB). Meta vectors only — short structured embeddings, not full documents. Origin data in a flat key-value SQLite table. Everything runs in-process alongside journey_states and scheduler_events.
**Rationale:** Sovereign, local, cheap. No data leaves the node. No vendor dependency. No separate service to deploy. 1,000 vectors = 1.5 MB in memory. Cosine similarity search in milliseconds on CPU. The P2P scope (one dream's knowledge + one customer's context) means the vector space stays small — hundreds to low thousands of vectors, not millions. Cloud vector services are overkill, expensive, and violate sovereignty. This is the mechanical implementation of the Strangler Pattern's crystallization: agentic (computed) responses become deterministic (cached) over time. See docs/INTELLIGENCE-LAYER.md and docs/INTELLIGENCE-LAYER-HANDOFF.md.
