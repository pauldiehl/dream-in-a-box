# Journey Blocks: Composable Journey Patterns

> "Journeys aren't custom implementations. They're compositions of proven blocks."

## The Insight

Five MVH journeys deconstructed into eight reusable blocks:

```
JumpStart:      presentation → simple_intake → recommendation(+agreement) → payment($97) → confirmation(+cal 1hr) → followup
PHO Walkthrough: presentation → recommendation → payment($49) → confirmation(+cal 30min) → followup
MCL Book:        presentation → recommendation → payment($5) → confirmation → convo_return
Nutrition Plan:  presentation → iterative_intake → plan_page(+signup) → followup(+notif_prefs)
Fitness Plan:    presentation → iterative_intake → plan_page(+signup) → followup(+notif_prefs)
```

The journeys aren't five different things. They're five orderings and parameterizations of the same building blocks. JumpStart and PHO Walkthrough differ by: price, meeting length, and whether intake/agreement exist. Nutrition and Fitness are the SAME journey with different domain content.

This pattern generalizes beyond MVH. Any dream's offerings decompose into these same blocks. A coaching program, a legal consultation, a creative service, a product purchase — all compositions of presentation, intake, recommendation, payment, confirmation, and followup.

---

## The Block Library

### Block: `presentation`

**Purpose:** The no-gatekeeping moment. Show the customer what we've got before they commit anything.

**Always first.** Every journey starts here. The customer sees a readonly artifact — an HTML/markdown page driven by JSON — that presents the offering. This is the "SHOW ME WHAT YOU GOT" moment.

**Parameters:**
| Param | Type | Description |
|-------|------|-------------|
| `offering_type` | string | What's being presented (service, plan, product, book) |
| `template` | string | Which presentation template to render |
| `default_config` | JSON | The plain-vanilla starting config for the artifact |

**Transition map:**
```
presentation → [simple_intake, iterative_intake, recommendation, payment, abandoned, escalated]
```

Presentation can go directly to payment (MCL book — no intake needed), to intake (JumpStart — need some info first), to iterative intake (Nutrition — need to build a custom plan), or to recommendation (PHO Walkthrough — just show the recommendation).

**Transaction layer artifact:** Readonly offering page with CTAs that route to conversation or next block.

---

### Block: `simple_intake`

**Purpose:** Quick information gathering. Not creating a custom plan — just collecting enough context for the next step.

**Use when:** The service requires some customer info before recommending or scheduling, but the offering itself doesn't change based on the answers. JumpStart needs to know basic health background for the consult — but the consult offering stays the same.

**Parameters:**
| Param | Type | Description |
|-------|------|-------------|
| `questions` | array | What to ask (can be protocol-defined or agent-directed) |
| `max_rounds` | number | Maximum back-and-forth before moving on (default: 3) |
| `required_fields` | array | Minimum data points before transition is valid |

**Transition map:**
```
simple_intake → [recommendation, payment, abandoned, escalated]
```

**How it works:** The agent asks questions conversationally. Each answer updates `state_data` on the journey. When `required_fields` are populated (or `max_rounds` reached), the agent transitions to the next block. No form. No multi-step wizard. Just conversation.

---

### Block: `iterative_intake`

**Purpose:** Multi-round customization. The customer and agent work together to CREATE something — a custom plan, a tailored recommendation, a designed product.

**Use when:** The offering IS the output of the intake. Nutrition Plan — the plan is built from the conversation. The customer might go back and forth several times ("I don't like those foods," "Can we add more protein?") before they're satisfied.

**Parameters:**
| Param | Type | Description |
|-------|------|-------------|
| `plan_type` | string | What's being built (nutrition_plan, fitness_plan, coaching_program) |
| `template` | string | Which plan template to use for generation |
| `refinement_ctas` | array | CTAs on the plan page that route back to conversation for changes |
| `satisfaction_signal` | string | How the customer indicates they're done (e.g., "looks good", clicks signup CTA) |

**Transition map:**
```
iterative_intake → [plan_page, iterative_intake, abandoned, escalated]
```

Note: `iterative_intake → iterative_intake` — the customer can loop. Agent generates a plan, presents it, customer asks for changes, agent regenerates. Each iteration updates the JSON config driving the plan artifact.

**How it works:**
1. Agent asks initial questions (dietary preferences, goals, schedule)
2. Agent generates plan config (JSON) and renders plan artifact
3. Customer views readonly plan page with refinement CTAs:
   - "I WANT DIFFERENT FOODS" → `/chat?action=change_foods`
   - "I WANT MORE VARIETY" → `/chat?action=meal_variety`
   - "LOOKS GOOD — SIGN ME UP" → transition to `plan_page` or `payment`
4. Each CTA returns to conversation with context pre-loaded
5. Agent updates JSON, regenerates artifact
6. Loop until satisfaction signal

**The critical difference from `simple_intake`:** Simple intake collects data. Iterative intake creates a product. The artifact exists during iterative intake and evolves. In simple intake, no artifact is shown until the next block.

---

### Block: `recommendation`

**Purpose:** Present a readonly recommendation page. May include a service agreement. Always includes a CTA to the next step (usually payment).

**Parameters:**
| Param | Type | Description |
|-------|------|-------------|
| `include_agreement` | boolean | Whether a service agreement is shown (default: false) |
| `agreement_template` | string | Which agreement template (if applicable) |
| `price` | number | Price shown on the page (informational, payment block handles transaction) |
| `cta_label` | string | What the action button says ("PROCEED TO PAYMENT", "GET STARTED", etc.) |
| `cta_target` | string | Which block the CTA leads to (usually `payment`) |

**Transition map:**
```
recommendation → [payment, abandoned, escalated]
```

If `include_agreement` is true:
```
recommendation → [agreement_signed, abandoned, escalated]
agreement_signed → [payment, abandoned, escalated]
```

**Transaction layer artifact:** Readonly recommendation page. Shows what the customer is getting, the price, and optionally a service agreement with a signature/consent CTA. The agreement CTA is a minimal fulfillment moment — one of the few interactive elements we allow.

---

### Block: `payment`

**Purpose:** Process payment. One of the minimal fulfillment moments.

**Parameters:**
| Param | Type | Description |
|-------|------|-------------|
| `amount_cents` | number | Payment amount in cents |
| `currency` | string | Currency code (default: 'USD') |
| `provider` | string | Payment provider ('stripe', 'square', etc.) |
| `description` | string | What appears on the charge |
| `receipt_template` | string | Which receipt template to generate after success |

**Transition map:**
```
payment → [confirmation, payment_failed, abandoned, escalated]
payment_failed → [payment, abandoned, escalated]
```

**How it works:** This is a concession to current reality — customers aren't ready to do payments purely through conversation yet. A minimal payment form (hosted by Stripe/Square checkout, not custom UI) handles the transaction. On success, a receipt artifact is generated and a transaction note is injected into the conversation.

**Side effects on success:**
- Generate receipt artifact (readonly PDF/HTML)
- Inject transaction note into conversation
- Transition to confirmation block
- If scheduler events are defined (e.g., appointment reminders), create them

---

### Block: `confirmation`

**Purpose:** Post-payment confirmation. May include scheduling.

**Parameters:**
| Param | Type | Description |
|-------|------|-------------|
| `include_scheduling` | boolean | Whether to show a Cal.com scheduling widget |
| `cal_duration_minutes` | number | Meeting duration if scheduling (15, 30, 60) |
| `cal_type` | string | Cal.com event type slug |
| `confirmation_template` | string | Which confirmation page template to render |
| `create_scheduler_events` | boolean | Whether to create reminder/followup events |

**Transition map:**
```
confirmation → [scheduled, followup, completed, escalated]
```

If `include_scheduling`:
```
confirmation → [scheduling, escalated]
scheduling → [scheduled, abandoned, escalated]
scheduled → [post_visit, missed, escalated]
```

**Transaction layer artifact:** Readonly confirmation page. Shows receipt summary, next steps, and optionally an embedded Cal.com scheduling widget (the scheduling iframe is one of our allowed interactive elements — it's third-party hosted, not custom UI).

**Side effects on scheduling:**
- Create `reminder` event at T-15min
- Create `notification` event at T-0
- Create `followup` event at T+60min (with callback to check completion)
- These are exactly the patterns from MVH-CASE-STUDY-JM-AND-SCHEDULER.md

---

### Block: `plan_page`

**Purpose:** Present a generated plan with a signup CTA. Used after iterative intake when the customer is satisfied.

**Parameters:**
| Param | Type | Description |
|-------|------|-------------|
| `plan_type` | string | Type of plan (nutrition, fitness, coaching) |
| `plan_config_id` | string | Reference to the JSON config in origin_store |
| `signup_cta_label` | string | What the signup button says |
| `signup_action` | string | What happens on signup (start conversation, create enrollment, etc.) |

**Transition map:**
```
plan_page → [enrollment, iterative_intake, abandoned, escalated]
```

Note: `plan_page → iterative_intake` — the customer can go back and refine even after the "final" plan is shown.

**Transaction layer artifact:** Readonly plan page driven by the JSON config that was iteratively refined. Minimal CTAs: "SIGN ME UP" and refinement options that route back to conversation.

---

### Block: `followup`

**Purpose:** Post-journey engagement. Notification preferences, check-ins, conversation continuation.

**Parameters:**
| Param | Type | Description |
|-------|------|-------------|
| `collect_notification_prefs` | boolean | Whether to ask about notification preferences |
| `notification_types` | array | Available notification types (meal_reminders, weekly_stats, etc.) |
| `schedule_first_event` | boolean | Whether to create the first event in a recursive chain |
| `first_event_type` | string | What the first scheduled event is |
| `first_event_delay_hours` | number | When the first event fires (hours from now) |

**Transition map:**
```
followup → [active, completed, escalated]
```

**How it works:** The agent asks about preferences conversationally. "Would you like daily meal reminders? What time works best?" Answers are stored in `state_data`. If `schedule_first_event` is true, the first event in a recursive chain is created — and from there, the scheduler takes over (one event creates the next, as per MVH case study).

---

## Composing Journeys from Blocks

A journey definition is just a block sequence with parameters:

```json
{
  "journey_type": "jumpstart",
  "display_name": "JumpStart Health Consult",
  "blocks": [
    {
      "block": "presentation",
      "params": {
        "offering_type": "service",
        "template": "consult_offering"
      }
    },
    {
      "block": "simple_intake",
      "params": {
        "questions": ["health_background", "current_medications", "goals"],
        "max_rounds": 3,
        "required_fields": ["health_background"]
      }
    },
    {
      "block": "recommendation",
      "params": {
        "include_agreement": true,
        "agreement_template": "health_consult_agreement",
        "price": 9700,
        "cta_label": "PROCEED TO PAYMENT"
      }
    },
    {
      "block": "payment",
      "params": {
        "amount_cents": 9700,
        "provider": "stripe",
        "description": "JumpStart Health Consult"
      }
    },
    {
      "block": "confirmation",
      "params": {
        "include_scheduling": true,
        "cal_duration_minutes": 60,
        "cal_type": "jumpstart-consult",
        "create_scheduler_events": true
      }
    },
    {
      "block": "followup",
      "params": {
        "collect_notification_prefs": false,
        "schedule_first_event": false
      }
    }
  ]
}
```

Compare PHO Walkthrough — same blocks, fewer steps, different params:

```json
{
  "journey_type": "pho_walkthrough",
  "display_name": "PHO Walkthrough",
  "blocks": [
    {
      "block": "presentation",
      "params": { "offering_type": "service", "template": "pho_walkthrough" }
    },
    {
      "block": "recommendation",
      "params": {
        "include_agreement": false,
        "price": 4900,
        "cta_label": "BOOK YOUR WALKTHROUGH"
      }
    },
    {
      "block": "payment",
      "params": { "amount_cents": 4900, "provider": "stripe", "description": "PHO Walkthrough" }
    },
    {
      "block": "confirmation",
      "params": {
        "include_scheduling": true,
        "cal_duration_minutes": 30,
        "cal_type": "pho-walkthrough",
        "create_scheduler_events": true
      }
    },
    {
      "block": "followup",
      "params": { "collect_notification_prefs": false }
    }
  ]
}
```

And the Nutrition Plan — different shape, iterative:

```json
{
  "journey_type": "nutrition_plan",
  "display_name": "The Man Plan - Nutrition",
  "blocks": [
    {
      "block": "presentation",
      "params": { "offering_type": "plan", "template": "nutrition_overview" }
    },
    {
      "block": "iterative_intake",
      "params": {
        "plan_type": "nutrition_plan",
        "template": "nutrition_plan_generator",
        "refinement_ctas": [
          { "label": "I WANT DIFFERENT FOODS", "action": "change_foods" },
          { "label": "I WANT MORE VARIETY", "action": "meal_variety" },
          { "label": "CHANGE MY MEAL TIMES", "action": "change_times" }
        ],
        "satisfaction_signal": "signup_cta_clicked"
      }
    },
    {
      "block": "plan_page",
      "params": {
        "plan_type": "nutrition",
        "signup_cta_label": "START MY PLAN",
        "signup_action": "start_enrollment"
      }
    },
    {
      "block": "followup",
      "params": {
        "collect_notification_prefs": true,
        "notification_types": ["morning_checkin", "meal_reminders", "weekly_stats"],
        "schedule_first_event": true,
        "first_event_type": "morning_checkin",
        "first_event_delay_hours": 12
      }
    }
  ]
}
```

---

## How Blocks Generate Transition Maps

The block sequence automatically produces the JM transition map. Each block knows its valid exit states. The composition engine chains them:

```
presentation.exits  → [next_block.entry, abandoned, escalated]
simple_intake.exits → [next_block.entry, abandoned, escalated]
payment.exits       → [next_block.entry, payment_failed, abandoned, escalated]
...
```

The transition map for JumpStart, auto-generated from blocks:

```js
const JUMPSTART_TRANSITIONS = {
  presentation:     ['simple_intake', 'abandoned', 'escalated'],
  simple_intake:    ['recommendation', 'abandoned', 'escalated'],
  recommendation:   ['agreement_review', 'abandoned', 'escalated'],
  agreement_review: ['payment', 'abandoned', 'escalated'],
  payment:          ['confirmation', 'payment_failed', 'abandoned', 'escalated'],
  payment_failed:   ['payment', 'abandoned', 'escalated'],
  confirmation:     ['scheduling', 'escalated'],
  scheduling:       ['scheduled', 'abandoned', 'escalated'],
  scheduled:        ['post_visit', 'missed', 'escalated'],
  missed:           ['scheduling', 'abandoned', 'escalated'],
  post_visit:       ['followup', 'escalated'],
  followup:         ['completed', 'escalated'],
  completed:        ['presentation'],
  abandoned:        ['presentation', 'simple_intake', 'recommendation', 'payment'],
  escalated:        ['presentation', 'simple_intake', 'recommendation', 'payment',
                     'confirmation', 'scheduling', 'scheduled', 'post_visit', 'followup']
};
```

**The founder never writes this.** The DDP Toolkit generates it from the block sequence. The MVH case study's manual transition maps become auto-generated artifacts.

---

## Integration with the Intelligence Layer

This is where blocks and vectors converge.

### What Gets Vectorized

Each block definition becomes a vector in the intelligence layer:

- `presentation(offering_type=service)` → vector
- `payment(amount_cents=9700, provider=stripe)` → vector
- `iterative_intake(plan_type=nutrition_plan)` → vector

Each COMPOSED JOURNEY becomes a vector:

- "paid consult with intake, agreement, scheduling" → vector that points to JumpStart-like composition
- "simple product purchase" → vector that points to MCL-like composition
- "custom plan with iterative refinement" → vector that points to Nutrition-like composition

### How the DDP Toolkit Uses It

Founder says: "I sell a 30-minute coaching session for $75."

1. Embed the description
2. Vector search: nearest journey pattern → PHO Walkthrough composition (paid service + scheduling + no intake)
3. Present to founder: "This looks like a paid consultation journey. Here's what I'd recommend..."
4. Founder confirms or adjusts
5. Generate journey JSON with parameterized blocks
6. Auto-generate transition map

The founder didn't define a journey from scratch. They described their offering and the system assembled a proven pattern. The 2+2 moment.

### How Fulfillment Uses It

Customer says: "I want to start eating healthier."

1. Embed the request
2. Vector search: nearest journey type → `nutrition_plan`
3. Start the customer in the `presentation` block of the nutrition journey
4. Serve the no-gatekeeping plan immediately
5. The block sequence handles everything from there

The agent doesn't decide what to do. The blocks tell it what to do. The vectors tell it which blocks to use. The agent is the executor, not the architect.

---

## Block Reuse Across Dreams

These blocks aren't MVH-specific. They're universal:

| Block | MVH Usage | Coach Kid Usage | Torty Usage | Any Dream |
|-------|-----------|----------------|-------------|-----------|
| `presentation` | Health offering | Coaching package | Belt grading program | Whatever you sell |
| `simple_intake` | Health background | Athlete profile | Skill level assessment | Quick context |
| `iterative_intake` | Nutrition plan builder | Training plan builder | Curriculum designer | Custom product creator |
| `recommendation` | Service agreement | Package selection | Program enrollment | Offer + CTA |
| `payment` | Stripe checkout | Stripe checkout | Stripe checkout | Any payment provider |
| `confirmation` | Cal.com scheduling | Cal.com scheduling | Cal.com scheduling | Booking + receipt |
| `plan_page` | Nutrition/fitness plan | Training schedule | Belt progression | Any generated plan |
| `followup` | Meal reminders | Session check-ins | Practice reminders | Ongoing engagement |

The blocks are the same. The parameters change. The domain content changes. The infrastructure doesn't.

---

## Development Impact

### Before Blocks (Today's Pain)

```
Founder: "I want to add a new $49 consultation offering"
Developer: manually writes transition map (~30 states)
Developer: manually writes side effects for each transition
Developer: manually builds presentation page
Developer: manually wires payment integration
Developer: tests everything end-to-end
Developer: finds 50% of edge cases don't work
Developer: spends 3 days fixing
```

### After Blocks

```
Founder: "I want to add a new $49 consultation offering"
DDP Toolkit: "That sounds like a paid consultation journey. 30 or 60 minutes?"
Founder: "30 minutes"
DDP Toolkit: generates journey JSON with blocks, auto-generates transition map
System: composes blocks, wires payment, creates scheduler events
Validation: each block is pre-tested, only parameters are new
```

The blocks are proven. They've been tested in MVH. They work. The only new thing is the parameters. That's a dramatically smaller surface area for bugs.

---

## Reference Documents

- **`docs/DOMAIN-DRIVEN-PROTOCOLS.md`** — DDPs define what gets built; blocks define how it's assembled
- **`docs/MVH-CASE-STUDY-JM-AND-SCHEDULER.md`** — The transition maps and side effects that blocks auto-generate
- **`docs/INTELLIGENCE-LAYER.md`** — Blocks are the vectorized units; journeys are vector compositions
- **`docs/INTELLIGENCE-LAYER-HANDOFF.md`** — Build instructions for the vector layer that powers block matching
