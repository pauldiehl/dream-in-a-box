# MVH Case Study: Journey Mappings & Embedded Scheduler

> Two reusable patterns from Man vs Health that DIAB should adopt as first-class infrastructure.

---

## Part 1: Journey Mappings (JM) as First-Class Protocol Structures

### The Problem

Every conversational AI product eventually needs to track where a customer is in a multi-step process. Most teams reach for one of two bad options:

1. **Status flags on a user record** — `user.onboarding_complete`, `user.paid`, `user.scheduled`. Grows into a mess of booleans that don't compose.
2. **Workflow engines** — Temporal, Step Functions, etc. Powerful but massive overkill for most use cases, and they own your execution model.

MVH needed something in between: a state machine that tracks customer progress through complex journeys, validates transitions, fires side effects, and surfaces actionable items to business users. It needed to be dead simple — a single SQLite table and ~200 lines of JavaScript.

### How MVH Represents Journeys

Every customer journey is a finite state machine. The `journey_states` table is the single source of truth:

```sql
CREATE TABLE journey_states (
  id TEXT PRIMARY KEY,
  user_id TEXT NOT NULL,
  conversation_id TEXT,          -- 1 journey ↔ 1 customer conversation
  admin_conversation_id TEXT,    -- separate admin convo for this journey
  journey_type TEXT NOT NULL,    -- 'medical_consult', 'nutrition_plan', 'fitness_plan'
  state TEXT NOT NULL,
  prev_state TEXT,
  state_data JSON,               -- arbitrary per-state payload
  entered_at TEXT,
  updated_at TEXT,
  resolved_at TEXT,
  created_at TEXT DEFAULT (datetime('now'))
);
```

Key design decisions:

- **`state_data` is a JSON blob.** Each state can store whatever it needs — scheduled appointment time, customer name from a booking form, prescription plan details. No schema migration when you add a field.
- **`prev_state` enables undo.** When a customer is `escalated` and the admin resolves it, the system knows where to send them back.
- **`conversation_id` ties the journey to a conversation.** One journey, one conversation. The conversation IS the customer's portal for that journey.
- **`admin_conversation_id`** gives the business user a separate conversation context for the same journey — internal discussion that the customer never sees.

### Per-Journey-Type Transition Maps

This is where it gets interesting. Different journey types have completely different lifecycles, but they share the same table and the same transition engine.

**Medical Consult** — a transactional journey with many states:

```js
const MEDICAL_CONSULT_TRANSITIONS = {
  intake:       ['consenting', 'abandoned', 'escalated'],
  consenting:   ['paying', 'abandoned', 'escalated'],
  paying:       ['scheduling', 'abandoned', 'escalated'],
  scheduling:   ['scheduled', 'abandoned', 'escalated'],
  scheduled:    ['post_visit', 'missed', 'escalated'],
  missed:       ['scheduling', 'abandoned', 'escalated'],
  post_visit:   ['rx_pending', 'follow_up', 'escalated'],
  rx_pending:   ['rx_payment', 'follow_up', 'escalated'],
  rx_payment:   ['rx_ordered', 'abandoned', 'escalated'],
  rx_ordered:   ['follow_up', 'escalated'],
  follow_up:    ['completed', 'intake'],
  completed:    ['intake'],
  abandoned:    ['intake', 'consenting', 'paying', 'scheduling', 'scheduled'],
  escalated:    ['intake', 'consenting', 'paying', 'scheduling', 'scheduled',
                 'post_visit', 'rx_pending', 'rx_payment', 'rx_ordered', 'follow_up']
};
```

**Nutrition Plan** — a long-running engagement journey with fewer states:

```js
const NUTRITION_PLAN_TRANSITIONS = {
  presentation: ['enrollment', 'abandoned', 'escalated'],
  enrollment:   ['active', 'abandoned', 'escalated'],
  active:       ['follow_up', 'stalled', 'escalated'],
  stalled:      ['active', 'abandoned', 'escalated'],
  follow_up:    ['active', 'graduated', 'escalated'],
  graduated:    ['completed'],
  completed:    [],
  abandoned:    ['presentation', 'enrollment'],
  escalated:    ['presentation', 'enrollment', 'active', 'follow_up']
};
```

Notice the patterns:
- **Every state can reach `escalated`** — the customer can always request a human.
- **`escalated` can return to any prior state** — the admin resolves the escalation and puts the journey back on track.
- **`abandoned` can restart** — customers come back. The system doesn't punish them.
- **Terminal states** (`completed`) have no outbound transitions, or loop back to `intake` for repeat journeys.

### The Transition Validator

The core function is ~20 lines. It looks up the transition map for the journey type, checks if the requested transition is valid, and either executes or rejects:

```js
function transitionState(journeyId, newState, opts = {}) {
  const db = getDb();
  const journey = db.prepare('SELECT * FROM journey_states WHERE id = ?').get(journeyId);
  if (!journey) return { success: false, error: 'Journey not found' };

  const transitions = getTransitions(journey.journey_type);
  const valid = transitions[journey.state];
  if (!valid || !valid.includes(newState)) {
    return { success: false, error: `Cannot transition from ${journey.state} to ${newState}` };
  }

  const now = new Date().toISOString();
  const stateData = opts.stateData ? JSON.stringify(opts.stateData) : journey.state_data;

  db.prepare(`
    UPDATE journey_states
    SET state = ?, prev_state = ?, state_data = ?, entered_at = ?, updated_at = ?, resolved_at = ?
    WHERE id = ?
  `).run(newState, journey.state, stateData, now, now, opts.resolve ? now : null, journeyId);

  // Fire side effects
  runSideEffects(journey, newState, opts);

  const updated = db.prepare('SELECT * FROM journey_states WHERE id = ?').get(journeyId);
  return { success: true, journey: updated };
}
```

This is a pattern that generalizes perfectly. Any DIAB dream with a multi-step customer process gets this for free.

### Side Effects on State Transitions

The `runSideEffects` function is a switch statement on the new state. When a journey transitions, things happen automatically:

```js
function runSideEffects(journey, newState, opts) {
  switch (newState) {
    case 'escalated': {
      // Inject tx note into customer convo: "A team member has been notified"
      injectTransactionNote(db, journey.conversation_id,
        '👋 CONNECTING — A team member has been notified and will be with you shortly.');
      // Log admin notification
      db.prepare(`INSERT INTO notification_log ...`).run(...);
      break;
    }
    case 'abandoned': {
      // Schedule a nudge 72 hours from now
      createEvent({
        event_type: 'followup',
        resource_type: 'journey',
        resource_id: journey.id,
        conversation_id: journey.conversation_id,
        fire_at: new Date(Date.now() + 72 * 60 * 60 * 1000).toISOString(),
        callback: JSON.stringify({
          type: 'check_abandoned',
          journeyId: journey.id,
          previousState: journey.state
        })
      });
      break;
    }
    case 'rx_payment': {
      // Alert admin if unpaid after 2 days
      createEvent({
        event_type: 'followup',
        fire_at: new Date(Date.now() + 2 * 24 * 60 * 60 * 1000).toISOString(),
        callback: JSON.stringify({ type: 'check_rx_payment', journeyId: journey.id })
      });
      break;
    }
    case 'follow_up': {
      // 7-day check-in event
      createEvent({
        event_type: 'item_received',
        fire_at: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000).toISOString(),
        ...
      });
      break;
    }
  }
}
```

This is the integration point between JMs and the scheduler. State transitions CREATE scheduler events. The scheduler fires them later. No polling the database to detect "has this customer been idle?" — the event was pre-created at the moment the state changed.

### How the Admin Queue Derives from JM State

The admin queue is a real-time view that queries `journey_states` and produces a priority-sorted list of items needing attention. No separate queue table. No message broker. Just a SQL query:

```js
const QUEUE_STATES = [
  'escalated', 'missed', 'post_visit', 'rx_pending', 'rx_payment',
  'rx_ordered', 'scheduled', 'abandoned', 'stalled', 'enrollment', 'presentation'
];

function getAdminQueue() {
  const journeys = db.prepare(`
    SELECT js.*, u.name as user_name, u.phone as user_phone, u.email as user_email
    FROM journey_states js
    JOIN users u ON u.id = js.user_id
    WHERE js.state IN (${QUEUE_STATES.map(() => '?').join(',')})
    ORDER BY js.entered_at ASC
  `).all(...QUEUE_STATES);
  // ... priority sorting, situation labels, overdue detection
}
```

Each state maps to a priority level and a human-readable situation label:

| State | Priority | Situation Label |
|-------|----------|----------------|
| `escalated` | 1 | 🔴 ESCALATED |
| `missed` | 2 | 🟠 MISSED |
| `post_visit` | 3 | 🟡 POST-VISIT |
| `stalled` | 3 | 🟠 STALLED |
| `rx_pending` | 4 | 🟡 RX PENDING |
| `rx_payment` | 5 (2 if overdue) | 🔵 RX PAYMENT (🟠 if overdue) |
| `scheduled` | 7 (5 if <2hr away) | 🔵 SCHEDULED |
| `abandoned` | 8 | ⚪ ABANDONED |

The queue dynamically adjusts priority based on time. An `rx_payment` that's been sitting for 2+ days gets bumped from priority 5 to priority 2 (🟠 RX OVERDUE). A `scheduled` appointment within 2 hours gets bumped from 7 to 5. The queue reflects urgency, not just state.

### How This Maps to DDP Concepts

| DDP Concept | MVH Implementation |
|-------------|-------------------|
| **Aggregate Root** | Journey Mapping (the `journey_states` row) |
| **Domain Events** | State transitions (`transitionState()` calls) |
| **Side Effects** | `runSideEffects()` — notifications, scheduler events, escalation alerts |
| **Bounded Context** | Journey type — `medical_consult` and `nutrition_plan` share infrastructure but have completely different state maps |
| **Artifact** | Transaction notes injected into conversations (readonly records of what happened) |

The JM row IS the aggregate root. Nothing outside the JM decides what state the customer is in. The transition map IS the domain logic. The side effects ARE the domain events.

### What DIAB Should Adopt

1. **The `journey_states` table** — exactly as shown. One row per customer journey. JSON `state_data` for flexibility.
2. **Per-type transition maps** — a plain JS object (or JSON file) defining valid transitions for each journey type. A dream's DDP Toolkit conversation produces this as part of journey mapping.
3. **The `transitionState()` validator** — never mutate state directly. Always go through the validator. This is your consistency boundary.
4. **Side effect hooks** — `runSideEffects()` as the single place where "when X happens, do Y" lives. Not scattered across endpoints.
5. **Queue derivation from state** — no separate queue table. The queue IS a view over journey states with priority logic.
6. **Chat mode on journeys** — MVH tracks `chatMode` ('agent' vs 'direct') on the journey's `state_data`, controlling whether customer messages go through the LLM or straight to an admin. DIAB dreams with human handoff need this.

---

## Part 2: The Embedded Scheduler (Replacing Temporal)

### Why Temporal Was Overkill

Temporal is a workflow orchestration platform. It's fantastic for distributed systems with complex retry logic, saga patterns, and multi-service coordination. It's also:

- A separate service to deploy, monitor, and maintain
- A Java/Go ecosystem with a steep learning curve
- Complete overkill for "send a reminder in 15 minutes" or "check if this payment came in after 2 days"

MVH's scheduling needs are simple:
- Fire a reminder before an appointment
- Nudge a customer who went inactive
- Check if a payment was received after N days
- Send daily meal reminders for a nutrition plan

All of these are: **"do something at a future time, optionally check a condition first."** That's it. One SQLite table and a `setInterval` loop.

Most DIAB dreams will have similar needs. Unless a dream is orchestrating multi-service distributed transactions (unlikely for a small business), Temporal adds complexity without value.

### The `scheduler_events` Table

```sql
CREATE TABLE scheduler_events (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  event_type TEXT NOT NULL,        -- 'item_received', 'reminder', 'notification', 'followup'
  resource_type TEXT NOT NULL,     -- 'visit', 'lab_order', 'journey', 'plan'
  resource_id TEXT NOT NULL,       -- FK to the resource
  conversation_id TEXT NOT NULL,   -- which conversation to inject into
  user_id TEXT,
  status TEXT DEFAULT 'pending',   -- 'pending', 'fired', 'skipped'
  fire_at TEXT NOT NULL,           -- ISO datetime: when this event should fire
  fired_at TEXT,                   -- when it actually fired
  callback TEXT,                   -- JSON: conditional logic before firing
  parent_event_id INTEGER,         -- chains: which event created this one
  note_html TEXT,                  -- the transaction note that was injected
  created_at TEXT DEFAULT (datetime('now'))
);

CREATE INDEX idx_sched_events_status ON scheduler_events(status);
CREATE INDEX idx_sched_events_fire ON scheduler_events(fire_at);
```

Key design decisions:

- **`fire_at`** is the only scheduling primitive. No cron expressions. No recurrence rules. Just "fire at this time." Recurrence is handled by chaining (one event creates the next).
- **`callback`** is a JSON payload describing a condition to check before firing. If the condition fails, the event is skipped (or a follow-up is created).
- **`parent_event_id`** tracks event chains. You can trace the full lifecycle: "this reminder was created by that booking, which created this follow-up, which created this admin escalation."
- **`conversation_id`** ties every event to a conversation. When the event fires, its output goes into that conversation as a transaction note.

### Event Types

| Type | Purpose | Example |
|------|---------|---------|
| `reminder` | Alert before a scheduled event | "Your Medical Consult is in 15 minutes" |
| `notification` | Alert at event start time | "Your Medical Consult is starting now! Join Meeting →" |
| `followup` | Conditional check after something should have happened | "Looks like you missed your appointment. Reschedule?" |
| `item_received` | Resource is ready for customer | "Your lab requisition is ready. Take this to LabCorp." |
| `admin_escalation` | Internal alert (admin-only, not customer-facing) | "Patient missed appointment and hasn't rescheduled" |

### The 5-Minute Polling Cycle

The scheduler is embedded in the Express server process. No separate service. No message queue. No worker pool.

```js
// In server.js startup:
const { startScheduler } = require('./app/scheduler');
startScheduler({ intervalMs: 5 * 60 * 1000 });

// Inside startScheduler:
function startScheduler(opts = {}) {
  const intervalMs = opts.intervalMs || 5 * 60 * 1000;
  
  // Run once immediately on startup
  runSchedulerCycle({ verbose: true });
  
  // Then every 5 minutes
  setInterval(() => {
    runSchedulerCycle({ verbose: true });
  }, intervalMs);
}
```

The cycle itself is trivial:

```js
function runSchedulerCycle() {
  const now = new Date().toISOString();
  
  const dueEvents = db.prepare(`
    SELECT * FROM scheduler_events
    WHERE status = 'pending' AND fire_at <= ?
    ORDER BY fire_at ASC
  `).all(now);

  for (const event of dueEvents) {
    processEvent(db, event, now);
  }
}
```

That's the entire scheduler. Query pending events where `fire_at` has passed. Process each one. Done.

**Why 5 minutes?** It's a balance. A 15-minute reminder that fires 1-4 minutes late is fine. A daily meal reminder that fires a few minutes late is fine. If a dream needs sub-minute precision, reduce the interval — but most don't.

### The Callback Mechanism

This is where the scheduler gets smart. A callback is a JSON payload on the event that says "before you fire, check this condition."

**Example: Check if a visit was completed before sending a follow-up:**

```js
// When a visit is booked, create a followup event with a callback:
createEvent({
  event_type: 'followup',
  resource_type: 'visit',
  resource_id: visitId,
  conversation_id: convoId,
  fire_at: new Date(scheduledTime + 60 * 60 * 1000).toISOString(), // 1 hour after
  callback: JSON.stringify({
    type: 'check_visit_completed',
    typeName: 'Medical Consult',
    escalationLevel: 1,
    escalateAfterMs: 60 * 60 * 1000  // chain: admin escalation 1 hour later
  })
});
```

When this event fires, the scheduler evaluates the callback:

```js
function evaluateCallback(db, event, callback) {
  switch (callback.type) {
    case 'check_visit_completed': {
      const visit = db.prepare('SELECT * FROM visits WHERE id = ?').get(event.resource_id);
      
      if (visit.status === 'completed') {
        // Visit happened — don't nag
        return { fire: false, reason: 'visit completed' };
      }
      if (visit.status === 'cancelled') {
        return { fire: false, reason: 'visit cancelled' };
      }
      
      // Visit didn't happen — fire a "you missed it" note to customer
      // AND chain an admin escalation event for 1 hour later
      return {
        fire: true,
        noteOverride: 'It looks like you missed your appointment. Would you like to reschedule?',
        createFollowup: {
          event_type: 'admin_escalation',
          fire_at: new Date(Date.now() + callback.escalateAfterMs).toISOString(),
          callback: JSON.stringify({
            type: 'check_visit_completed',
            escalationLevel: 2  // next level: admin-only notification
          })
        }
      };
    }
  }
}
```

The callback can:
1. **Suppress the event** (`fire: false`) — the condition isn't met, skip it
2. **Override the note** — use a different message than the default template
3. **Chain a follow-up event** — create another event for later (recursive scheduling)
4. **Escalate** — same callback type with a higher `escalationLevel` changes behavior (level 1 = message customer, level 2 = notify admin only)

### Recursive Event Chaining

This is the critical pattern for ongoing engagement like the Man Plan nutrition protocol.

**The principle: one event schedules the next. There is no master list of future events.**

From the [MAN-PLAN-NUTRITION-PROTOCOL.md](./MAN-PLAN-NUTRITION-PROTOCOL.md):

```
Morning check-in completes → schedules first meal reminder
First meal reminder fires → schedules next meal reminder
Last meal reminder fires → schedules evening daily log
Evening daily log completes → schedules tomorrow's morning check-in
Sunday evening log completes → schedules Monday morning stats collection
Monthly review trigger → schedules next monthly review
```

Why this works better than pre-scheduling:

1. **No cron bomb.** You don't have 1,000 future events for 1,000 customers sitting in a table. Each customer has exactly ONE pending event at any time.
2. **Cancellation is trivial.** Customer says "stop nagging me" → don't schedule the next event. The chain simply stops. No events to find and delete.
3. **Frequency changes are instant.** Customer says "text me at 6 AM instead of 7" → the next scheduled event uses the new time. No cron expression to edit.
4. **State-aware.** Each event fires in the context of the current journey state. If the customer went `stalled`, the chain can adapt its behavior.

### Conversation Injection

When a scheduler event fires, it injects a transaction note into the customer's conversation:

```js
function injectTransactionNote(db, convoId, noteHtml) {
  const convo = db.prepare('SELECT messages FROM conversations WHERE id = ?').get(convoId);
  const messages = JSON.parse(convo.messages || '[]');

  messages.push({
    role: 'transaction',
    text: noteHtml,
    ts: Date.now()
  });

  db.prepare("UPDATE conversations SET messages = ?, updated_at = datetime('now') WHERE id = ?")
    .run(JSON.stringify(messages), convoId);
}
```

Transaction notes render differently from agent or user messages in the chat UI — they're system messages with icons, styled as cards. They carry links (reschedule, join meeting, view receipt) but are readonly. The customer sees them in their conversation timeline alongside the regular chat.

The note templates are simple functions:

```js
const NOTE_TEMPLATES = {
  reminder: (ctx) =>
    `⏰ REMINDER — Your ${ctx.typeName} is in ${ctx.minutesUntil} minutes with ${ctx.doctorName} · Join Meeting`,

  notification: (ctx) =>
    `🔔 STARTING NOW — Your ${ctx.typeName} is beginning! · Join Now`,

  followup: (ctx) =>
    `❓ FOLLOW UP — ${ctx.description} · Reschedule`,

  item_received: (ctx) =>
    `📋 READY — ${ctx.description}`
};
```

### Customer Control

The nutrition protocol defines how customers control their notification chain:

| Customer Says | System Does |
|---------------|-------------|
| "Stop meal reminders" | Exclude `meal_reminder` from notification types. Chain stops for that type. |
| "Stop nagging me" / "Turn everything off" | Set `notifications_enabled: false`. No next event is scheduled. |
| "Just do weekly check-ins" | Set notification types to `["weekly_stats"]` only. |
| "Start sending me reminders again" | Re-enable types. Schedule the next event in the chain. |
| "Text me at 6 AM instead" | Update the time preference. Next event uses new time. |

The key insight: **cancellation = stop scheduling the next event.** There are no events to find and delete. The chain terminates naturally.

### What the Scheduler Handles in Production

**Abandoned customer nudges:**
- JM transitions to `abandoned` → side effect creates a followup event at +72 hours
- Callback checks if customer has re-engaged; if not, sends a casual nudge
- No escalation chain — one nudge, then leave them alone

**Overdue payment alerts:**
- JM transitions to `rx_payment` → side effect creates a followup event at +2 days
- Callback checks if payment was received; if not, logs an admin notification
- The admin queue also detects this independently (overdue calculation on `entered_at`)

**Follow-up check-ins:**
- JM transitions to `follow_up` → side effect creates an `item_received` event at +7 days
- When it fires, injects a check-in prompt into the conversation

**Appointment reminders:**
- Visit is booked → `createVisitEvents()` creates three events:
  1. `reminder` at T-15 minutes
  2. `notification` at T-0
  3. `followup` at T+60 minutes (with callback to check visit completion)

```js
function createVisitEvents(opts) {
  const { visitId, conversationId, userId, scheduledAt, typeName, reminderMinutes } = opts;
  const scheduledDate = new Date(scheduledAt);

  // 1. Reminder: 15 min before
  createEvent({
    event_type: 'reminder',
    resource_type: 'visit',
    resource_id: visitId,
    conversation_id: conversationId,
    user_id: userId,
    fire_at: new Date(scheduledDate.getTime() - reminderMinutes * 60000).toISOString()
  });

  // 2. Notification: at start time
  createEvent({
    event_type: 'notification',
    resource_type: 'visit',
    resource_id: visitId,
    conversation_id: conversationId,
    user_id: userId,
    fire_at: scheduledDate.toISOString()
  });

  // 3. Followup: 1 hour after (with conditional callback)
  createEvent({
    event_type: 'followup',
    resource_type: 'visit',
    resource_id: visitId,
    conversation_id: conversationId,
    user_id: userId,
    fire_at: new Date(scheduledDate.getTime() + 60 * 60000).toISOString(),
    callback: JSON.stringify({
      type: 'check_visit_completed',
      typeName,
      escalationLevel: 1,
      escalateAfterMs: 60 * 60 * 1000
    })
  });
}
```

**Daily meal reminders (nutrition plan):**
- Recursive chain as described above
- Each event in the chain reads the customer's current config (meal times, notification preferences) before scheduling the next one
- If the customer changed their preferences between events, the next event respects the new settings immediately

### How It Wires Into the Express App

The scheduler starts as a side effect of `app.listen()`:

```js
app.listen(PORT, () => {
  // Start JM Scheduler (embedded mode)
  const schedulerConfig = projectConfig.scheduler || {};
  if (schedulerConfig.enabled !== false) {
    startScheduler({
      intervalMs: schedulerConfig.intervalMs || 5 * 60 * 1000,
      verbose: true
    });
  }
});
```

Events are created by application code at the point of action. When a visit is booked:

```js
app.post('/api/schedule/book', async (req, res) => {
  const result = await cal.bookAppointment({ ... });
  
  if (result.success && convoId) {
    // Create visit record
    db.prepare(`INSERT INTO visits ...`).run(...);
    
    // Create scheduler events (reminder, notification, followup)
    createVisitEvents({
      visitId, conversationId: convoId, userId,
      scheduledAt: result.start, typeName: 'Medical Consult',
      reminderMinutes: 15
    });
    
    // Transition journey → scheduled
    transitionState(journey.id, 'scheduled', { stateData: { ... } });
  }
});
```

The booking endpoint does three things in sequence: creates a visit record, creates scheduler events, and transitions the journey state. All synchronous. All in one request. No distributed coordination needed.

### What DIAB Should Adopt

1. **The `scheduler_events` table** — exactly as shown. Single table, indexed on `status` and `fire_at`.
2. **The embedded `setInterval` pattern** — no separate service. If the Express server is running, the scheduler is running. If it crashes, both restart together.
3. **The callback mechanism** — JSON payloads that define conditional logic before firing. This is what makes the scheduler smart without making it complex.
4. **Recursive chaining** — one event creates the next. No master schedule. No cron expressions. Cancellation = stop chaining.
5. **Conversation injection** — events don't send emails or push notifications (though they could). They inject messages into the customer's conversation, keeping everything in one place.
6. **The `parent_event_id` chain** — trace any event back to its origin. Debugging "why did the customer get this message?" becomes trivial.
7. **Event creation at the point of action** — don't scan the database looking for things to schedule. Create events when the triggering action happens (booking, state transition, resource creation).

---

## How These Patterns Connect

The JM and scheduler aren't independent systems. They form a feedback loop:

```
Customer action
  → JM state transition (validated by transition map)
    → Side effects fire (runSideEffects)
      → Scheduler events created (createEvent)
        → Time passes...
          → Scheduler fires event (runSchedulerCycle)
            → Transaction note injected into conversation
              → Customer sees update, takes action
                → JM state transition (loop)
```

The JM is the state authority. The scheduler is the time authority. Together they handle the full lifecycle of a customer journey without any external dependencies, background workers, or message queues.

For DIAB, these two patterns cover the operational core of any dream that has:
- Multi-step customer processes (JM)
- Time-sensitive follow-ups (scheduler)
- Business user visibility into customer progress (admin queue from JM state)
- Proactive customer communication (scheduler → conversation injection)

Both patterns are <500 lines of JavaScript total, backed by two SQLite tables, running inside the same Express process. That's the level of simplicity DIAB should target.
