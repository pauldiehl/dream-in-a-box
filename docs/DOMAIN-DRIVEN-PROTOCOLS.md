# Domain-Driven Protocols (DDP)

> "The domain model IS the protocol. A founder can read it. An agent can execute it. A developer can implement infrastructure around it."

## What This Document Covers

Domain-Driven Protocols are the business logic layer of a Dream in a Box node. They define how a dream serves its customers — not through code, but through human-readable protocol files that agents execute directly.

This document defines:
- How DDD concepts map to DIAB's protocol-first architecture
- The DDP Toolkit: how a founder creates their driving protocols
- Journey Mappings (JM) and User Flows (UF) as first-class protocol structures
- The No-Gatekeeping principle and how it shapes every customer interaction
- Agent experiences derived from DDPs
- How readonly artifacts replace traditional UX
- How admin and customer portals dissolve into conversations

---

## DDD → DDP: The Mapping

Eric Evans' Domain-Driven Design was always about protocols. The industry just implemented them wrong — burying domain logic in Java class hierarchies that only developers could read. DDP corrects this by making the domain model a file, not a codebase.

| DDD Concept | DDP Equivalent | Why It's Better |
|---|---|---|
| **Bounded Context** | Dream | Each dream has its own domain language. Plain Fun talks about "lands" and "players." Coach Kid talks about "sessions" and "milestones." They share node infrastructure but never domain vocabulary. |
| **Aggregate Root** | Journey Mapping (JM) | The consistency boundary. A JM holds the entire customer experience together. Nothing inside a JM makes sense outside of it. |
| **Entity** | User Flow (UF) | Has identity and lifecycle within a JM. "Payment flow" and "intake flow" are distinct UFs within the same journey. JM:UF is 1:many. |
| **Value Object** | Artifact | Immutable once created. A receipt, an agreement, a nutrition plan. Defined by content, not identity. Readonly. |
| **Domain Event** | Stage Transition | "Customer completed intake." "Payment confirmed." "Meeting scheduled." Protocol-triggered, not code-triggered. These drive the entire system forward. |
| **Repository** | Artifact/Data Storage | Where artifacts and stage data live. One schema per JM/UF stage. |
| **Application Service** | Agent Experience Pattern | The 7+ common patterns (fulfillment, human handoff, business intel, etc.) that DDPs activate contextually. |
| **Ubiquitous Language** | Protocol Vocabulary | Not aspirational — literally the document everyone (founder, agent, customer, developer) works from. |

---

## The DDP Toolkit: Onboarding a Dreamer

When a founder first touches DIAB, they don't read documentation. They have a conversation.

The DDP Toolkit is a Forkless agent experience that walks a founder through creating their driving protocols. It IS the onboarding — both to the technical platform and to their own business model.

### The Conversation Flow

**Phase 1 — Domain Discovery (the "Context Mapping" conversation)**

The agent asks the founder to describe their business in plain language. No forms. No templates. Just talk.

- Who is your customer? (Segment)
- What problem are you solving? (Scope)
- Walk me through what happens from first contact to delivery.
- What could go wrong at each step?
- What does "done" look like for your customer?

**Output:** `domain.md` — the bounded context definition. Customer segments, problem scope, solution system.

**Phase 2 — Journey Mapping**

For each offering/product the founder describes, the agent maps the journey:

- What are the stages? (e.g., Discovery → Intake → Recommendation → Agreement → Fulfillment → Follow-up)
- What triggers each transition?
- What artifacts get produced at each stage?
- Where might a customer stall? What happens then?

**Output:** `journeys/{journey-slug}.md` — the full JM with stages, transitions, checkpoints.

**Phase 3 — User Flow Definition**

For each stage within a journey, the agent digs into the specifics:

- What does the customer DO at this stage?
- What does the system DO at this stage?
- What are the expected flow of events? (UML-style: actor → action → response)
- What are the key logic paths? (Happy path, edge cases, failures)
- What checkpoints exist? (consent required? payment required? human review?)

**Output:** `journeys/{journey-slug}/{stage}/flow.md` — the UF definition.

**Phase 4 — Schema Generation**

Each JM/UF stage gets its own data schema. This isn't an afterthought — it's generated directly from the protocol.

- What data do we need to track at this stage?
- What indicates progress or stall?
- What do admins/support need to see?

**Output:** `journeys/{journey-slug}/{stage}/schema.md` — maps to a DB table. Tracks customer progress per stage. Enables admin visibility, stall detection, and agent-driven progression.

### What the Toolkit Produces

After the conversation, the founder has:

```
protocols/domain/{dream-slug}/
├── domain.md                    # Bounded context: customer + problem + solution
├── journeys/
│   ├── {journey-slug}.md        # JM: stages, transitions, domain events
│   └── {journey-slug}/
│       ├── discovery/
│       │   ├── flow.md          # UF: expected events, logic paths, checkpoints
│       │   └── schema.md        # DB schema for tracking this stage
│       ├── intake/
│       │   ├── flow.md
│       │   └── schema.md
│       ├── recommendation/
│       │   ├── flow.md
│       │   └── schema.md
│       ├── agreement/
│       │   ├── flow.md
│       │   └── schema.md
│       ├── fulfillment/
│       │   ├── flow.md
│       │   └── schema.md
│       └── followup/
│           ├── flow.md
│           └── schema.md
└── agents/
    ├── founder.md               # Agent config for builder experience
    ├── operations.md            # Agent config for business user operations
    └── support.md               # Agent config for customer-facing support
```

This is both documentation AND execution configuration. The agent reads `flow.md` to know what to do. The `schema.md` maps to real DB tables. The agent configs define personas and permissions.

---

## The No-Gatekeeping Principle

> "SHOW ME WHAT YOU GOT."

This is a foundational design principle, not a feature. It changes everything about how dreams serve customers.

### The Old Way (Every App Ever Built)

Customer arrives → Complex intake form → Onboarding flow → Meet with a person → Wait → Maybe get a recommendation → More forms → Eventually see something useful.

Gatekeeping. The customer has to prove they're worthy of the product before they even see what it does.

### The DIAB Way

Customer arrives → Clicks "Start Now" → Gets a plan. Immediately. Right now.

"I want a custom nutrition plan and I don't want to answer any questions first."

We give them one. A plain-vanilla, one-size-fits setup based on whatever we know (which might be nothing). It won't be perfect. It doesn't need to be. It's a first look. It's proof that this system has something to offer. The customer can see what we've got before committing anything.

### How It Works Technically

The plan is a **readonly artifact** — an HTML file (or markdown) driven by a JSON config.

```
artifacts/
├── plans/
│   ├── {customer-id}.json      # The data (updated by agent)
│   └── {customer-id}.html      # The view (readonly, regenerated from JSON)
```

No interactive components. No form fields. No state mutations from the UI. The JSON is the source of truth. The HTML is a projection of that truth. When the plan needs to change, the AGENT updates the JSON and regenerates the view.

### Customization Through Conversation, Not Configuration

At the bottom of every plan artifact, minimal CTAs route back to the conversation:

```
┌─────────────────────────────────────────────┐
│           YOUR NUTRITION PLAN               │
│                                             │
│  [... readonly plan content ...]            │
│                                             │
│─────────────────────────────────────────────│
│                                             │
│  🔄 I WANT A DIFFERENT PLAN                │
│  🍽️ I WANT DIFFERENT FOODS                 │
│  📋 I WANT GREATER MEAL VARIETY            │
│  ⏰ I WANT TO CHANGE MY MEAL TIMES         │
│                                             │
└─────────────────────────────────────────────┘
```

Each CTA is a link: `/chat?action=change_plan`, `/chat?action=change_foods`, etc. It redirects to the conversation with the action pre-loaded so the agent knows what topic to drive. The agent updates the JSON. The plan regenerates. The customer sees the change.

No configuration screens. No settings pages. No UX components to build, test, debug, and babysit.

### Fulfillment CTAs

When a journey stage requires concrete action, the plan includes fulfillment CTAs:

- **"SIGN MY PERSONAL AGREEMENT"** → Opens a sign-and-print page (readonly document + signature capture, minimal)
- **"WORK WITH HEALTH COACH"** → Entry point into a new JM (human-assisted journey)
- **"SIGN ME UP"** → Opens chat where agent finalizes details (phone number, notification times) and completes enrollment

These are the ONLY interactive moments. Everything else is readonly content or conversation.

---

## The End of Portals

### Admin Portal → Conversation + Queue

The admin portal doesn't exist as a traditional interface. Business users interact through:

**The Queue (Inbox):** A minimal list of items requiring attention. Think email inbox, not dashboard. Each item links to a conversation context.

**The Conversation:** Business users talk to the agent about their customers and operations. The agent has full context because it executed the protocols that got the customer to this point.

### Customer Portal → Contextual Transaction Layer

The customer portal doesn't exist either. Customers have:

**Scheduled engagement:** The system reaches out when something is needed (heartbeat pattern — check, notify, wait).

**On-demand conversation:** Customer initiates when they want something.

**Readonly artifacts:** Plans, receipts, agreements, summaries — accessible anytime, never interactive.

There is no dashboard. There is no settings page. There are no toggles. If a customer needs to change something, they talk to the agent. If they need to see something, it's a readonly artifact.

---

## Agent Experience Patterns

DDPs define WHEN and HOW these patterns activate. The patterns themselves are common across all dreams — the protocol determines context.

### Pattern 1: Customer → Agent Fulfillment

Customer converses with agent to fulfill journey stages. As concrete items are completed (agreement signed, consent given, payment processed, meeting scheduled), the agent produces **transaction notes** directly in conversation: a log entry plus a link to a readonly PDF/HTML artifact (receipt, confirmation, agreement).

**Triggered by:** Any UF stage with `fulfillment: agent` in flow.md.

### Pattern 2: Customer → Human Handoff

Customer needs direct human interaction. Agent notifies the designated business user. Business user communicates directly with customer until satisfied. Business user signals completion and passes the baton back to the agent, who resumes protocol-driven execution.

**Triggered by:** Any UF stage with `fulfillment: human` or when agent detects customer frustration/complexity beyond protocol scope.

### Pattern 3: Business User → Customer Observation

Business user views conversations the customer is having with the agent. Read-only. No intervention. Used for quality assurance, training, and understanding customer needs.

**Triggered by:** Business user request via operations agent.

### Pattern 4: Business User → Customer Direct

Business user communicates directly with customer, bypassing the agent. Used for relationship building, complex situations, or personal touch.

**Triggered by:** Business user request via operations agent or queue item.

### Pattern 5: Business User → Agent → Customer (Indirect)

Business user instructs the agent to guide a customer in a specific direction. "Get more intake on XYZ and guide them toward this offer." The agent executes the instruction naturally within the ongoing conversation — the customer experiences a seamless interaction, not a handoff.

**Triggered by:** Business user instruction via operations agent. This pattern doesn't exist in traditional software. There's no UX for "tell the agent to do this." It only works in agent-first architecture.

### Pattern 6: Business User → Agent (Encounter Planning)

Business user talks to agent about a specific customer for planning purposes. Meeting prep. Post-meeting notes. Action items. JM status updates. The agent has full context from protocol execution and can synthesize customer history into actionable briefs.

**Triggered by:** Business user request via operations agent, often before/after scheduled interactions.

### Pattern 7: Business User → Agent (Business Intelligence)

Business user asks agent for aggregate metrics, trends, and insights across customers. "How many customers are stalled at the agreement stage?" "What's our average time from intake to fulfillment?" The agent queries across JM/UF schemas to produce insights.

**Triggered by:** Business user request via operations agent.

### DDP Controls Pattern Activation

The protocol doesn't just define journeys — it defines which agent patterns are available at each stage. A UF's `flow.md` might specify:

```
## Stage: Intake
fulfillment: agent
observation: enabled
human_handoff: on_escalation
indirect_guidance: enabled

## Stage: Agreement
fulfillment: agent + human_review
observation: enabled
human_handoff: always_available
```

The agent reads this configuration and knows exactly what's possible at each moment. No hardcoded logic. No feature flags. Protocol-driven.

---

## Done Is the Era of Big UX

The tech sector's model has been: complex UX → constant developer babysitting → any small change has big impacts → framework churn → more developers → more complexity. This cycle is over.

### The Three Surfaces

In a DIAB dream, there are exactly three UI surfaces:

**1. The Conversation (Agent Layer / Forkless)**
This is where everything happens. Customer intake, recommendations, fulfillment, support, business operations. The conversation IS the application.

**2. Readonly Artifacts (Transaction Layer)**
Plans, receipts, agreements, blog posts, summaries. HTML or markdown files driven by JSON configs. No interactivity beyond minimal CTAs that route back to conversation. Readonly means: no state mutations from the UI, no bugs from the UI, no developer babysitting the UI.

**3. Minimal Fulfillment Moments**
The shopping-cart-to-confirmation flow for customers not yet ready to trust the agent with payments. Sign-and-print for agreements. These are the concession to current reality — customers are learning to trust agents but aren't there on everything yet. These surfaces are intentionally small, predictable, template-based.

### Why Readonly Is the Key

Readonly artifacts have:
- No form state to manage
- No validation logic to break
- No event handlers to debug
- No framework dependencies to update
- No accessibility edge cases in interactive components
- No security surface area from user input

Less real estate means less code. Less code means less review. Less review means fewer tokens to find problems. The JSON config is the single source of truth. The agent is the single writer. The artifact is a projection.

### What About Knowledge Content?

Blog posts, guides, FAQs, documentation — these are also readonly artifacts. Not gushed into conversation (nobody wants a 2000-word knowledge dump in chat). Instead: the agent references them, links to them, summarizes relevant parts conversationally. The full content lives as a clean, minimal, predictable HTML template.

---

## DDP to Database: Schema Per Stage

Each JM/UF stage maps to its own schema. This isn't a monolithic `users` table with 47 nullable columns. It's stage-specific tracking that mirrors the protocol structure.

### Why Per-Stage Schemas

- Each stage tracks different data (intake tracks questions asked; agreement tracks signature status; fulfillment tracks delivery milestones)
- Admins can query any stage independently ("show me everyone stalled at agreement")
- The agent can progress customers forward by updating stage-specific records
- Stall detection is trivial: check timestamps per stage, flag anything past threshold

### How Stalls Get Resolved

When a customer stalls (no progress in a stage for N days, defined in the protocol), the system doesn't send a generic reminder email. The agent — through the heartbeat pattern — detects the stall, checks the protocol for the appropriate response, and acts. Maybe it reaches out to the customer. Maybe it notifies the business user. Maybe it adjusts the plan.

This happens through the agent, not through configuration screens or admin buttons. No UX fail points. No clunky interfaces. Agentic execution.

---

## DIAB Delivery Modes

DDPs get delivered to real customers through one of two paths:

### Mode 1: Gift Delivery (Founder-Initiated)

The dreamer (founder) identifies someone who could benefit from a DIAB-powered business. The founder:

1. Runs the DDP Toolkit themselves, walking through the recipient's domain
2. Sets up a DIAB node as a child AWS account under the founder's parent AWS organization
3. Proves it out — sells product, validates the journey, demonstrates viability
4. Hand-delivers the result: a working business with real customers

The reaction is the point. "Here's the business you always dreamed of building. It's already running. It already has customers. Here are the keys."

This is the radical generosity play. Not every recipient will run with it — but the ones who do spread the movement. Not advocating a brand or a platform. Advocating a way of building. Web 4.0 as a gift, not a sales pitch.

**Technical setup:** AWS Organizations child account, DIAB node deployed via infrastructure agent (Layer 0), protocols pre-configured by founder, Forkless + services running, dream active with real data.

### Mode 2: Self-Serve (Clone and Converse)

A founder discovers DIAB and wants to build their own dream:

1. Clones the repo locally
2. Runs `npm start` (or equivalent) — boots a local Forkless agent
3. The agent IS the toolkit — walks the founder through domain discovery, journey mapping, user flow definition
4. Produces working protocol files and schema definitions
5. Founder iterates locally until satisfied
6. When ready to go live: the agent walks them through deploying to a cloud provider of their choice (AWS, DigitalOcean, etc.) — or they run it on their own hardware (Mac Mini sovereign node)

No CLI wizard. No documentation-first approach. The onboarding IS a conversation with the same agent that will eventually serve their customers. They learn the system by using the system.

---

## Bootstrap: What Every Node Gets

Before DDPs come into play, every DIAB node starts with a common skeleton:

**Skeleton:**
- Landing page + featured product/offering
- Basic branding (name, colors, logo placeholder)

**Dev Server:**
- Express + SQLite + live reload
- Vanilla HTML/CSS/JS (no frameworks — the AI is the framework)

**Forkless Widget:**
- Chat engine (Agent Layer)
- Claude API integration
- WebSocket for real-time updates

This is cookie-cutter. Every node gets it. The DDP Toolkit then builds on top of this skeleton, generating the protocols and schemas that make the node specific to a dream.

---

## What Comes Next

This document defines the framework. Implementation priorities:

1. **DDP Toolkit Agent** — The conversation that produces protocols. This is the product.
2. **JM/UF Protocol Format** — Finalize the markdown/JSON structure for journey and flow definitions.
3. **Schema Generator** — Given a flow.md, produce the corresponding DB schema.
4. **Agent Pattern Engine** — Read flow.md configs to activate the right pattern at each stage.
5. **Readonly Artifact Renderer** — JSON config → HTML/markdown view with CTA routing.
6. **Stall Detection + Heartbeat Integration** — Background monitoring of customer progress per stage.

The DDP Toolkit is the entry point for every dreamer. Build it first, and everything else follows from the protocols it generates.
