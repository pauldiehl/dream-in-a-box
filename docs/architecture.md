# DIAB Architecture

## The Four-Layer Model

### Layer 0 — Node Agent

The background process that makes a DIAB node alive rather than static.

**Responsibilities:**
- Boot and manage infrastructure (EC2, container, local machine — doesn't matter)
- Read and enforce protocols from `protocols/`
- Run the heartbeat loop (wake → check → act → sleep)
- Enforce budget limits across all dreams and services
- Maintain goal traceability (every action traces to a purpose)
- Isolate data between dreams and between tenants within dreams

**Key characteristic:** The node agent runs continuously in the background. It is the only process that understands the full state of the node. Dreams and services call into it; they never manage infrastructure directly.

**Implementation:** See `docs/OPERATIONAL-PATTERNS.md` for the four core patterns (Heartbeat, Budget, Traceability, Isolation).

---

### Layer 1 — Protocols

Human-readable files that define how the node behaves.

**Default protocols (ship with every node):**

| Protocol | Purpose | Format |
|----------|---------|--------|
| `identity.md` | Who owns this node, verification requirements | Markdown |
| `budget.md` | Spending limits per dream, per user, per conversation | Markdown + JSON |
| `syndication.md` | How this node discovers and connects to peers | Markdown |
| `dream-lifecycle.md` | How dreams boot, pause, archive, and terminate | Markdown |
| `verification.md` | Adult verification, age gating, parental controls | Markdown |
| `monetization.md` | P2P payment rules, sovereign revenue only | Markdown |

**Key characteristic:** Protocols are the source of truth. Code implements protocols; protocols don't describe code. When there's a conflict, the protocol wins. A protocol can exist before any code implements it — that's intentional.

**Custom protocols:** Each dream can add its own protocols (e.g., Plain Fun would have `land-rules.md` defining how kids create and govern game lands). Dream-level protocols extend node-level protocols; they cannot override them.

---

### Layer 2 — Services

Shared capabilities available to every dream on the node.

**Core services:**

| Service | What It Does | Status |
|---------|-------------|--------|
| **Forkless** | Conversational commerce engine. Agent Layer (chat UI) + Transaction Layer (artifact rendering). Every dream gets a conversational interface and artifact system. | Active development |
| **Greenspaces** | Community and environment layer. Shared spaces, collaboration, environmental context. | Early concept |

**Key characteristics:**
- Services are NOT standalone applications. They are node-level tools.
- Every dream has access to every service. A dream doesn't "install" Forkless — it's just there.
- Services share the node's protocols. Forkless respects the same budget limits and identity rules as everything else.
- New services can be added to a node. The architecture is open — not limited to Forkless and Greenspaces.

**Relationship to dreams:** A dream declares which service capabilities it uses. The node agent wires them together. The dream never talks to infrastructure directly — it talks to services, which talk to the node agent.

---

### Layer 3 — Dreams

The actual products, experiences, and businesses that give a node its purpose.

**What a dream is:**
- A self-contained product definition
- Declares its protocols, content, rules, and service requirements
- Does NOT know about infrastructure, servers, or deployment
- Has its own data space (isolated by the node agent)
- Can define custom protocols that extend (not override) node protocols

**Example dreams and what they stress-test:**

| Dream | Key Challenges | What It Proves |
|-------|---------------|----------------|
| **Man vs Health** | Health journeys, portal, 4 core experiences | Forkless as conversational commerce works |
| **Plain Fun** | Generative AI lands, real-time multiplayer, kid admins, adult verification, P2P payments | Full DIAB stack under pressure — verification, monetization, real-time state, parental controls |
| **Coach Kid** | Youth coaching, progress tracking | Service-based dream with Forkless + Greenspaces |
| **Good Vibes** | Wellness, community | Greenspaces-heavy dream |
| **Yomo** | (TBD) | (TBD) |
| **Torty Martial Arts** | Training, progression, community | Multi-service dream |

**Dream definition format:** TBD — likely extends the `project.json` pattern from Forkless. A dream manifest that declares name, protocols, service requirements, and content roots.

---

## Data Flow

```
User Request
    │
    ▼
┌─────────┐    ┌────────────┐    ┌─────────────┐
│  Dream   │───▶│  Service   │───▶│ Node Agent  │
│ (Plain   │    │ (Forkless) │    │ (Layer 0)   │
│  Fun)    │◀───│            │◀───│             │
└─────────┘    └────────────┘    └──────┬──────┘
                                        │
                                        ▼
                                 ┌─────────────┐
                                 │  Protocols   │
                                 │  (files)     │
                                 └─────────────┘
```

Every request flows down through the layers. The node agent checks protocols before allowing actions. Budget is enforced. Goals are traced. Data stays isolated.

---

## Plain Fun — Architecture Stress Test

Plain Fun deserves special attention because it exercises every layer simultaneously:

**Layer 0 (Node Agent):**
- Real-time state management for multiplayer
- Adult verification enforcement
- Per-game budget isolation
- Heartbeat monitoring game health

**Layer 1 (Protocols):**
- `verification.md` — mandatory adult verification for all accounts
- `land-rules.md` — custom protocol for how kids define game rules/constructs
- `monetization.md` — P2P payments, sovereign only, no middleman

**Layer 2 (Services):**
- Forkless — kids describe what they want, AI generates game elements conversationally
- Greenspaces — shared game worlds, collaboration between players

**Layer 3 (Dream):**
- Game definitions, land constructs, player permissions
- Kid Admin role (creates rules/constructs of their land)
- Kid Player role (adds resources, stylizes, limited actions under land rules)
- Adult role (full visibility, full control, message-level filtering)
- Real-time updates from all participants

If DIAB can run Plain Fun, it can run anything.

---

## What This Repo Is NOT

- **Not an app framework.** It doesn't generate CRUD apps or landing pages.
- **Not a PaaS.** It doesn't host other people's code on shared infrastructure.
- **Not all of Web 4.0.** Sovereign-streams covers the full theory. This is one concrete implementation of the cloud node concept.
- **Not centralized.** There is no "DIAB cloud." Each node is sovereign.
