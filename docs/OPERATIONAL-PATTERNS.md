# Operational Patterns: Heartbeats, Budgets, Traceability, Isolation

**Patterns borrowed from agent orchestration systems, adapted for Forkless.**
**Author:** Paul Diehl (1,000 Hands) & Claude (Anthropic)
**Date:** March 14, 2026

---

> These four patterns come from studying Paperclip and similar agent orchestration platforms. They solve real problems: runaway costs, lost context, untraceable work, and multi-tenant isolation. Adapted here for the Forkless architecture — sovereign, conversation-first, Majestic Monolith.

---

## 1. The Heartbeat Pattern

### What It Is

Agents don't run continuously. They wake up in **heartbeats** — short execution windows where they check state, do work, and go back to sleep. This applies to background processes, not the user-facing Agent Layer (which is always responsive to conversation).

### Why It Matters

Without heartbeats, background agents either run 24/7 (expensive, wasteful) or don't run at all (things don't happen unless a human triggers them). Heartbeats give you a middle ground: scheduled intelligence that acts on your behalf without burning tokens continuously.

### Where It Applies in Forkless

**The Waystation proxy (Phase 3+):** Your always-on cloud proxy wakes on a heartbeat to check the message queue, process routine interactions, and forward complex ones to the monolith. Not running an LLM 24/7 — checking a DynamoDB queue and acting when needed.

**Dream Beacon processing:** A heartbeat reviews queued dream beacons periodically. Groups similar requests. Identifies patterns. Flags high-impact items for founder attention. Doesn't need to run continuously — once per hour or once per day is fine.

**Profig maintenance:** A heartbeat scans profigs for stale data, incomplete journeys, or engagement signals. "User hasn't logged in for 2 weeks — queue a re-engagement artifact." "This profig has sparse nutrition data — flag for next conversation."

**Protocol sync:** A heartbeat checks if external protocols (.well-known/) are in sync with internal protocol changes. If you update governance.md, the heartbeat regenerates governance.json.

**Crystallization jobs (from PROTOCOL-MATURITY.md):** A heartbeat reviews conversation logs, identifies high-frequency patterns, and generates deterministic handlers. This is the automatic Strangler Pattern in action.

### Implementation

```javascript
// lib/heartbeat.js

const HEARTBEAT_INTERVAL = process.env.HEARTBEAT_INTERVAL || 60 * 60 * 1000; // 1 hour default

const jobs = [];

function registerJob(name, fn, options = {}) {
  jobs.push({
    name,
    fn,
    interval: options.interval || HEARTBEAT_INTERVAL,
    lastRun: null,
    enabled: options.enabled !== false,
    budget: options.budget || null  // max tokens per run
  });
}

async function tick() {
  const now = Date.now();
  for (const job of jobs) {
    if (!job.enabled) continue;
    if (job.lastRun && (now - job.lastRun) < job.interval) continue;

    console.log(`[heartbeat] Running: ${job.name}`);
    try {
      await job.fn();
      job.lastRun = now;
      console.log(`[heartbeat] Done: ${job.name}`);
    } catch (err) {
      console.error(`[heartbeat] Failed: ${job.name}`, err.message);
    }
  }
}

function start() {
  // Check every minute if any job is due
  setInterval(tick, 60 * 1000);
  console.log(`[heartbeat] Started with ${jobs.length} jobs`);
}

module.exports = { registerJob, start, tick };
```

**Registration in core/index.js:**
```javascript
const heartbeat = require('../lib/heartbeat');

// Register background jobs
heartbeat.registerJob('dream-beacon-review', async () => {
  // Review queued dream beacons, group patterns, flag for founder
}, { interval: 4 * 60 * 60 * 1000 }); // every 4 hours

heartbeat.registerJob('profig-maintenance', async () => {
  // Scan profigs for stale data, engagement signals
}, { interval: 24 * 60 * 60 * 1000 }); // daily

heartbeat.registerJob('crystallization-scan', async () => {
  // Review conversation logs, identify patterns for deterministic handlers
}, { interval: 24 * 60 * 60 * 1000, budget: 5000 }); // daily, 5k token cap

// Start heartbeats after server is listening
heartbeat.start();
```

### Key Rules

- Heartbeat jobs must be **idempotent** — running twice produces the same result
- Each job gets its own token budget (see Budget Enforcement below)
- Jobs log start/end times and token usage to the metrics system
- A failed heartbeat does not crash the server — it logs and continues
- The founder can enable/disable individual heartbeats via admin

---

## 2. Atomic Budget Enforcement

### What It Is

Every LLM call gets tracked. Every agent (user-facing, background, admin) gets a budget. At 80% usage, a warning fires. At 100%, the agent pauses. No exceptions. No runaway costs.

### Why It Matters

Paul's concern: "I worry when I go public how much my BUDGET explodes." This is the safeguard. Without atomic budget enforcement, a popular product or a misbehaving conversation loop can burn hundreds of dollars in hours.

### Budget Hierarchy

```
Platform Budget ($500/mo cap)
├── User-Facing Agent ($300/mo)
│   ├── Per-conversation limit: $0.50 max
│   ├── Per-user daily limit: $2.00 max
│   └── Soft warning at 80% of any limit
├── Background Heartbeats ($100/mo)
│   ├── Per-job budget (set at registration)
│   └── Daily aggregate cap
├── Admin/Founder ($50/mo)
│   └── Protocol editing, dashboard queries
└── Reserve ($50/mo)
    └── Emergency buffer, never auto-spent
```

### Implementation

```javascript
// lib/budget.js

const fs = require('fs');
const path = require('path');

const BUDGET_FILE = path.join(__dirname, '..', 'data', 'budget.json');

function loadBudget() {
  if (!fs.existsSync(BUDGET_FILE)) {
    return {
      month: new Date().toISOString().slice(0, 7), // "2026-03"
      limits: {
        platform: 50000,     // $500 in cents
        userAgent: 30000,
        heartbeats: 10000,
        admin: 5000,
        reserve: 5000
      },
      spent: {
        userAgent: 0,
        heartbeats: 0,
        admin: 0
      },
      conversations: {},  // per-conversation tracking
      users: {},          // per-user daily tracking
      warnings: []
    };
  }
  const budget = JSON.parse(fs.readFileSync(BUDGET_FILE, 'utf8'));

  // Reset on new month
  const currentMonth = new Date().toISOString().slice(0, 7);
  if (budget.month !== currentMonth) {
    budget.month = currentMonth;
    budget.spent = { userAgent: 0, heartbeats: 0, admin: 0 };
    budget.conversations = {};
    budget.users = {};
    budget.warnings = [];
  }
  return budget;
}

function saveBudget(budget) {
  const dir = path.dirname(BUDGET_FILE);
  if (!fs.existsSync(dir)) fs.mkdirSync(dir, { recursive: true });
  fs.writeFileSync(BUDGET_FILE, JSON.stringify(budget, null, 2));
}

// Returns { allowed: bool, reason?: string, warning?: string }
function checkBudget(category, conversationId, userId, estimatedCostCents) {
  const budget = loadBudget();

  // Category limit
  if (budget.spent[category] + estimatedCostCents > budget.limits[category]) {
    return { allowed: false, reason: `${category} monthly budget exhausted` };
  }

  // Per-conversation limit (50 cents = 50 cost units)
  if (conversationId) {
    const convSpent = budget.conversations[conversationId] || 0;
    if (convSpent + estimatedCostCents > 50) {
      return { allowed: false, reason: 'Conversation budget limit reached ($0.50)' };
    }
  }

  // Per-user daily limit
  if (userId) {
    const today = new Date().toISOString().slice(0, 10);
    const userKey = `${userId}:${today}`;
    const userSpent = budget.users[userKey] || 0;
    if (userSpent + estimatedCostCents > 200) {
      return { allowed: false, reason: 'Daily user budget limit reached ($2.00)' };
    }
  }

  // 80% warning
  const categoryPercent = (budget.spent[category] + estimatedCostCents) / budget.limits[category];
  if (categoryPercent > 0.8) {
    return { allowed: true, warning: `${category} at ${Math.round(categoryPercent * 100)}% of monthly budget` };
  }

  return { allowed: true };
}

function recordSpend(category, conversationId, userId, costCents, metadata) {
  const budget = loadBudget();
  budget.spent[category] = (budget.spent[category] || 0) + costCents;

  if (conversationId) {
    budget.conversations[conversationId] = (budget.conversations[conversationId] || 0) + costCents;
  }

  if (userId) {
    const today = new Date().toISOString().slice(0, 10);
    const userKey = `${userId}:${today}`;
    budget.users[userKey] = (budget.users[userKey] || 0) + costCents;
  }

  saveBudget(budget);
}

module.exports = { checkBudget, recordSpend, loadBudget };
```

**Integration with lib/llm.js chat() function:**
```javascript
const { checkBudget, recordSpend } = require('./budget');

// Before making an API call:
const check = checkBudget('userAgent', conversationId, userId, estimatedCost);
if (!check.allowed) {
  return { text: `I've reached my conversation budget. ${check.reason}. Please try again later or contact the founder.`, actions: [] };
}
if (check.warning) {
  console.warn(`[budget] ${check.warning}`);
}

// After API call, log actual cost:
const inputTokens = response.usage.input_tokens;
const outputTokens = response.usage.output_tokens;
const costCents = Math.ceil((inputTokens * 0.003 + outputTokens * 0.015) / 10); // Sonnet pricing approx
recordSpend('userAgent', conversationId, userId, costCents, {
  model: 'claude-sonnet-4-20250514',
  inputTokens,
  outputTokens
});
```

### Key Rules

- Budget checks happen BEFORE every LLM call — atomic, no exceptions
- Token costs are estimated BEFORE the call (based on message size) and recorded AFTER (based on actual usage)
- The admin dashboard shows real-time budget status across all categories
- Budget resets on the 1st of each month
- The founder can override limits at any time via admin
- Budget data persists in `data/budget.json` (gitignored, lives on disk)

---

## 3. Task-to-Goal Traceability

### What It Is

Every piece of work — every artifact, every queued action, every heartbeat job result — traces back to a top-level goal. Nothing floats in isolation. You can always answer: "Why does this exist? What goal does it serve?"

### Why It Matters

Without traceability, autonomous agents produce orphaned work. Artifacts accumulate without context. Queue items pile up without priority signals. The founder can't tell what matters because nothing is connected to anything.

### Goal Hierarchy in Forkless

```
Company Goal: "Help people take control of their health through
              conversation-first, protocol-grounded experiences"
│
├── Product Goal: "Man vs Health — 4 health journeys"
│   │
│   ├── Journey Goal: "Allergy Management"
│   │   ├── Artifact: "Paul's allergy plan" (traces to this journey)
│   │   ├── Profig update: allergy data captured (traces here)
│   │   └── Queue item: "Research immunotherapy providers" (traces here)
│   │
│   ├── Journey Goal: "Nutrition Foundation"
│   │   ├── Artifact: "7-day meal plan" (traces here)
│   │   └── Dream Beacon: "I wish I could scan grocery receipts" (traces here)
│   │
│   └── Operational Goal: "Platform health"
│       ├── Heartbeat result: "Crystallization scan found 3 patterns" (traces here)
│       └── Budget alert: "User agent at 82% monthly budget" (traces here)
```

### Implementation

Add a `goalId` field to every entity:

**Artifacts:**
```javascript
// When storing an artifact, include the goal chain
storeArtifact({
  id: 'pauls-allergy-plan',
  type: 'markdown',
  subdir: 'docs',
  filename: 'pauls-allergy-plan.md',
  content: planContent,
  metadata: {
    title: "Paul's Allergy Management Plan",
    goalChain: ['company', 'mvh', 'allergy-management'],  // traces up
    conversationId: 'conv-abc123',
    userId: 'paul'
  }
});
```

**Queue items:**
```javascript
// When queueing an action, include why
queueAction({
  id: 'research-immunotherapy',
  description: 'Research immunotherapy providers in Paul\'s area',
  goalChain: ['company', 'mvh', 'allergy-management'],
  source: 'agent-conversation',
  conversationId: 'conv-abc123',
  priority: 'medium',
  status: 'pending'
});
```

**Heartbeat results:**
```javascript
// Background job outputs trace to operational goals
heartbeat.registerJob('crystallization-scan', async () => {
  const patterns = await scanForPatterns();
  for (const pattern of patterns) {
    await storeArtifact({
      id: `crystal-${pattern.hash}`,
      type: 'crystallization-report',
      metadata: {
        goalChain: ['company', 'platform-health', 'cost-optimization'],
        pattern: pattern.intent,
        frequency: pattern.count,
        estimatedSavings: pattern.savingsPerMonth
      }
    });
  }
});
```

### The Admin View

The admin dashboard should show a **goal tree** — a hierarchical view where the founder can:
- See all goals and their sub-goals
- See artifacts, queue items, and heartbeat results grouped by goal
- Identify goals with no recent activity (stalled journeys)
- Identify goals with heavy token spend (expensive goals)
- Re-prioritize by dragging goals up/down

### Key Rules

- Every artifact MUST have a `goalChain` in its metadata
- Every queue item MUST trace to at least one goal
- Heartbeat jobs declare which goal they serve at registration
- Orphaned work (no goalChain) is flagged in the admin dashboard
- Goals are defined in a `goals.json` file (or in project.json) — not hardcoded

---

## 4. Multi-Company Data Isolation

### What It Is

One Forkless deployment can serve multiple brands/companies with complete data isolation. Each company has its own protocols, profigs, artifacts, budget, and agent personality. They share the same infrastructure but never see each other's data.

### Why It Matters

This is the bridge between "I run my own product" (Phase 1) and "I deliver products to clients" (Dream in a Box, Phase 4). When you onboard a client, they get their own isolated space within your monolith before graduating to their own infrastructure.

### How It Maps to Forkless

Forkless already has the `project.json` pattern — a config file that defines brand, protocols, theme. Multi-company isolation extends this:

```
forkless/
├── companies/
│   ├── mvh/                        ← Man vs Health
│   │   ├── project.json
│   │   ├── protocols/internal/
│   │   ├── artifacts/
│   │   ├── users/                  ← MVH profigs only
│   │   ├── queue/
│   │   └── data/budget.json        ← MVH budget only
│   │
│   ├── good-vibes/                 ← Good Vibes (Phase 2)
│   │   ├── project.json
│   │   ├── protocols/internal/
│   │   ├── artifacts/
│   │   ├── users/
│   │   ├── queue/
│   │   └── data/budget.json
│   │
│   └── client-a/                   ← Dream in a Box client
│       ├── project.json
│       ├── protocols/internal/
│       ├── artifacts/
│       ├── users/
│       ├── queue/
│       └── data/budget.json
│
├── core/                           ← Shared platform code
├── lib/                            ← Shared libraries
└── capabilities/                   ← Shared capabilities
```

### Routing

Each company gets its own subdomain or path:
```
mvh.forkless.app         → loads companies/mvh/project.json
goodvibes.forkless.app   → loads companies/good-vibes/project.json
client-a.forkless.app    → loads companies/client-a/project.json
```

The server reads the hostname, loads the correct project.json, and scopes ALL data access to that company's directory. The LLM system prompt is built from that company's protocols. Profigs are read from that company's users/ folder. Artifacts are stored in that company's artifacts/ folder.

### Isolation Guarantees

```
┌────────────────────────────────────────────────────┐
│  SHARED (platform level)                           │
│  • Express server                                  │
│  • LLM wrapper (lib/llm.js)                       │
│  • Capability engine (lib/capabilities.js)         │
│  • Heartbeat scheduler                             │
│  • Budget enforcement engine                       │
├────────────────────────────────────────────────────┤
│  ISOLATED (company level)                          │
│  • project.json (brand config)                     │
│  • protocols/internal/ (agent knowledge)           │
│  • protocols/external/ (public protocols)          │
│  • users/ (profigs — NEVER cross-company)          │
│  • artifacts/ (generated content)                  │
│  • queue/ (action items)                           │
│  • data/budget.json (spending)                     │
│  • System prompt (built from company protocols)    │
└────────────────────────────────────────────────────┘
```

### Phase 1 → Phase 4 Migration Path

**Phase 1 (now):** Single company. project.json at root. No companies/ directory needed. Everything works as-is.

**Phase 2:** Second product (Good Vibes). Move MVH into companies/mvh/, create companies/good-vibes/. Server learns to route by hostname. Shared code stays in core/lib/.

**Phase 3:** Same pattern, more companies. Budget enforcement per company. Admin dashboard shows all companies.

**Phase 4 (Dream in a Box):** Client companies live in companies/client-x/. The monolith serves them all. When a client graduates, their companies/client-x/ directory exports to their own EC2 in an AWS Organizations child account. Clean migration because the isolation was there from the start.

### Key Rules

- Profigs NEVER cross company boundaries — a MVH user's data is invisible to Good Vibes
- Each company has its own independent budget
- Artifacts are company-scoped — deep links include the company slug
- The admin can see all companies (founder view), but users see only theirs
- Company creation is gated by the founder (no self-signup for new companies in Phase 1)

---

## Implementation Priority

These patterns don't all need to be built at once. Here's the order:

**Build NOW (Phase 1):**
1. **Budget Enforcement** — Add token tracking to every LLM call in lib/llm.js. This is the most immediately valuable pattern. Protects against cost surprises from day one.
2. **Task-to-Goal Traceability** — Add `goalChain` to artifact and queue metadata. Low effort, high long-term value.

**Build SOON (late Phase 1 / early Phase 2):**
3. **Heartbeat Pattern** — When you have background work to do (dream beacon review, crystallization scanning). Not needed until you have recurring users.

**Build LATER (Phase 2+):**
4. **Multi-Company Isolation** — When you're ready for a second product or first client. The project.json pattern already provides the scaffolding.

---

## Relationship to Other Docs

- [PROTOCOL-MATURITY.md](../../sovereign-streams/web4/PROTOCOL-MATURITY.md) (sovereign-streams) — The Strangler Pattern and crystallization theory. Budget Enforcement and Heartbeats implement the cost controls described there.
- [FUTURE-PHASES.md](./FUTURE-PHASES.md) — Multi-Company Isolation is the infrastructure foundation for Dream in a Box (Phase 4).
- [HANDOVER.md](./HANDOVER.md) — Phase 1 build order. Budget Enforcement and Traceability slot into Steps 3 (LLM Integration) and 4 (Transaction Layer) respectively.
- [architecture.md](./architecture.md) — The two-layer pattern. These operational patterns extend it with governance and cost control.

---

*Adapted from patterns observed in Paperclip (agent orchestration platform, March 2026). The corporate metaphor is theirs. The sovereign implementation is ours.*
