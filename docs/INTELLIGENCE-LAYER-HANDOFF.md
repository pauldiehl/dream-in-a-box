# Intelligence Layer — Build Handoff

> For OpenClaw / Claude Code: how to add a local vector intelligence layer to a Forkless-powered dream.

## What You're Building

A local vector database (sqlite-vss) integrated into the existing Express + SQLite stack. It sits alongside `journey_states` and `scheduler_events` in the same database. It gives the agent memory — the ability to recognize known patterns and serve proven responses instead of computing everything from scratch.

**This is additive, not destructive.** The JM engine, scheduler, conversation agent, and all existing code stay exactly as they are. The vector layer is a new intelligence source that the agent consults BEFORE calling Claude.

## Prerequisites

- Existing Forkless project with Express + SQLite (better-sqlite3)
- Node.js 18+
- Claude API access (existing)
- ~100MB disk space for the embedding model

## Phase 1: Infrastructure (Day 1)

### Step 1: Install Dependencies

```bash
npm install sqlite-vss onnxruntime-node
```

**sqlite-vss:** SQLite extension for vector similarity search. Adds virtual tables that support cosine similarity queries. Runs in-process, no separate service.

**onnxruntime-node:** Runs the embedding model locally. No API calls needed for embeddings.

### Step 2: Download Embedding Model

```bash
mkdir -p models/
# Download all-MiniLM-L6-v2 ONNX model (~80MB)
# This produces 384-dimensional vectors — small, fast, good for semantic similarity
curl -L -o models/model.onnx "https://huggingface.co/sentence-transformers/all-MiniLM-L6-v2/resolve/main/onnx/model.onnx"
curl -L -o models/tokenizer.json "https://huggingface.co/sentence-transformers/all-MiniLM-L6-v2/resolve/main/tokenizer.json"
```

**Why this model:** 384 dimensions (small vectors), runs on CPU in <50ms per embedding, good semantic quality for structured text. No GPU needed. 80MB total.

**Alternative:** If ONNX setup is problematic, start with Claude API embeddings (`voyage-3-lite` via Anthropic's partner). Slightly more expensive per embed but zero local setup. Migrate to local model when ready.

### Step 3: Create the Vector Tables

Add to your database initialization (wherever you create `journey_states` and `scheduler_events`):

```js
const db = require('better-sqlite3')('data/app.db');

// Load sqlite-vss extension
db.loadExtension('vector0');  // base vector operations
db.loadExtension('vss0');     // vector similarity search

// Origin store — the actual content (JSON configs, templates, patterns)
db.exec(`
  CREATE TABLE IF NOT EXISTS origin_store (
    id TEXT PRIMARY KEY,
    type TEXT NOT NULL,
    data JSON NOT NULL,
    created_at TEXT DEFAULT (datetime('now')),
    updated_at TEXT DEFAULT (datetime('now'))
  );
`);

// Vector index — semantic embeddings pointing to origin store
db.exec(`
  CREATE VIRTUAL TABLE IF NOT EXISTS vss_vectors USING vss0(
    embedding(384)
  );
`);

// Vector metadata — links vss rowid to origin_id and context
db.exec(`
  CREATE TABLE IF NOT EXISTS vector_meta (
    rowid INTEGER PRIMARY KEY,
    origin_id TEXT NOT NULL,
    category TEXT NOT NULL,
    label TEXT,
    confidence REAL DEFAULT 0.0,
    hit_count INTEGER DEFAULT 0,
    created_at TEXT DEFAULT (datetime('now')),
    FOREIGN KEY (origin_id) REFERENCES origin_store(id)
  );
`);
```

**Three tables:**
- `origin_store` — the actual content (JSON). The vector points here.
- `vss_vectors` — the vector index (sqlite-vss virtual table). Stores 384-dim embeddings.
- `vector_meta` — links vectors to origins with category, confidence, and usage tracking.

### Step 4: Create the Embedding Module

```js
// lib/embeddings.js
const { InferenceSession, Tensor } = require('onnxruntime-node');
const { readFileSync } = require('fs');

let session = null;
let tokenizer = null;

async function init() {
  if (session) return;
  session = await InferenceSession.create('./models/model.onnx');
  tokenizer = JSON.parse(readFileSync('./models/tokenizer.json', 'utf-8'));
}

// Simple tokenization (for MiniLM — production should use a proper tokenizer)
function tokenize(text, maxLength = 128) {
  // Simplified — use @xenova/transformers for proper tokenization if needed
  const tokens = text.toLowerCase().split(/\s+/).slice(0, maxLength);
  // Map to token IDs (simplified — proper implementation uses vocab lookup)
  return tokens;
}

async function embed(text) {
  await init();
  // Generate embedding using ONNX runtime
  // Returns Float32Array of 384 dimensions
  const inputIds = tokenize(text);
  const feeds = {
    input_ids: new Tensor('int64', BigInt64Array.from(inputIds.map(BigInt)), [1, inputIds.length]),
    attention_mask: new Tensor('int64', BigInt64Array.from(inputIds.map(() => 1n)), [1, inputIds.length]),
    token_type_ids: new Tensor('int64', BigInt64Array.from(inputIds.map(() => 0n)), [1, inputIds.length])
  };
  const results = await session.run(feeds);
  // Mean pooling over token embeddings
  const embeddings = results.last_hidden_state.data;
  const dim = 384;
  const vector = new Float32Array(dim);
  for (let i = 0; i < inputIds.length; i++) {
    for (let d = 0; d < dim; d++) {
      vector[d] += embeddings[i * dim + d];
    }
  }
  for (let d = 0; d < dim; d++) {
    vector[d] /= inputIds.length;
  }
  return vector;
}

module.exports = { embed, init };
```

**IMPORTANT NOTE:** The tokenization above is simplified. For production, use `@xenova/transformers` which provides proper tokenization for MiniLM. The embedding logic is correct — mean pooling over ONNX output. Adjust based on testing.

**Simpler alternative for Phase 1:** Skip local ONNX entirely. Use Claude/Voyage API for embeddings. Replace `embed()` with an API call. More expensive per embed but zero model setup complexity. Move to local model in Phase 2 when you've validated the concept.

---

## Phase 2: Core Intelligence Module (Day 2-3)

### Step 5: Create the Intelligence Layer

```js
// lib/intelligence.js
const { embed } = require('./embeddings');

const THRESHOLDS = {
  STATIC: 0.95,        // Serve cached response directly, no LLM
  DETERMINISTIC: 0.80, // Use cached context, light LLM refinement
  AGENTIC: 0.0         // Full LLM computation
};

/**
 * Store a knowledge entry: embed text, store origin data, link them.
 */
async function store(db, { text, category, label, data }) {
  const id = `${category}_${Date.now()}_${Math.random().toString(36).slice(2, 8)}`;

  // Store origin data
  db.prepare(`
    INSERT INTO origin_store (id, type, data) VALUES (?, ?, ?)
  `).run(id, category, JSON.stringify(data));

  // Generate and store embedding
  const vector = await embed(text);
  const info = db.prepare(`
    INSERT INTO vss_vectors (rowid, embedding) VALUES (NULL, ?)
  `).run(JSON.stringify(Array.from(vector)));

  // Store metadata linking vector to origin
  db.prepare(`
    INSERT INTO vector_meta (rowid, origin_id, category, label) VALUES (?, ?, ?, ?)
  `).run(info.lastInsertRowid, id, category, label || text.slice(0, 100));

  return id;
}

/**
 * Query the vector space: find nearest neighbors, return with tier classification.
 */
async function query(db, { text, category = null, limit = 5 }) {
  const vector = await embed(text);

  // Search for nearest neighbors
  let results;
  if (category) {
    // Category-filtered search
    results = db.prepare(`
      SELECT v.rowid, v.distance, m.origin_id, m.category, m.label, m.confidence, m.hit_count
      FROM vss_vectors v
      JOIN vector_meta m ON m.rowid = v.rowid
      WHERE vss_search(v.embedding, ?)
      AND m.category = ?
      LIMIT ?
    `).all(JSON.stringify(Array.from(vector)), category, limit);
  } else {
    results = db.prepare(`
      SELECT v.rowid, v.distance, m.origin_id, m.category, m.label, m.confidence, m.hit_count
      FROM vss_vectors v
      JOIN vector_meta m ON m.rowid = v.rowid
      WHERE vss_search(v.embedding, ?)
      LIMIT ?
    `).all(JSON.stringify(Array.from(vector)), limit);
  }

  // Convert distance to similarity (cosine: similarity = 1 - distance)
  // Classify into tiers
  return results.map(r => {
    const similarity = 1 - r.distance;
    let tier;
    if (similarity >= THRESHOLDS.STATIC) tier = 'static';
    else if (similarity >= THRESHOLDS.DETERMINISTIC) tier = 'deterministic';
    else tier = 'agentic';

    return {
      origin_id: r.origin_id,
      category: r.category,
      label: r.label,
      similarity,
      tier,
      confidence: r.confidence,
      hit_count: r.hit_count
    };
  });
}

/**
 * Retrieve origin data for a matched vector.
 */
function getOrigin(db, originId) {
  const row = db.prepare('SELECT * FROM origin_store WHERE id = ?').get(originId);
  return row ? { ...row, data: JSON.parse(row.data) } : null;
}

/**
 * Record a cache hit — tracks usage for confidence scoring.
 */
function recordHit(db, originId) {
  db.prepare(`
    UPDATE vector_meta SET hit_count = hit_count + 1 WHERE origin_id = ?
  `).run(originId);
}

/**
 * Store a request→response pair for semantic caching.
 */
async function cacheResponse(db, { request, response, journeyType = null }) {
  return store(db, {
    text: request,
    category: 'semantic_cache',
    label: request.slice(0, 100),
    data: {
      request,
      response,
      journey_type: journeyType,
      cached_at: new Date().toISOString()
    }
  });
}

module.exports = { store, query, getOrigin, recordHit, cacheResponse, THRESHOLDS };
```

### Step 6: Integrate with the LLM Loop

In your existing `lib/llm.js` (or equivalent), add the intelligence check BEFORE calling Claude:

```js
const intelligence = require('./intelligence');

async function chat(db, conversationId, userMessage, opts = {}) {
  // === INTELLIGENCE LAYER CHECK ===
  // Query vector space BEFORE calling Claude
  const matches = await intelligence.query(db, {
    text: userMessage,
    limit: 3
  });

  const topMatch = matches[0];

  if (topMatch && topMatch.tier === 'static') {
    // HIGH CONFIDENCE: serve cached response directly
    const origin = intelligence.getOrigin(db, topMatch.origin_id);
    intelligence.recordHit(db, topMatch.origin_id);

    console.log(`[intelligence] Static hit: "${topMatch.label}" (${topMatch.similarity.toFixed(3)})`);

    return {
      response: origin.data.response,
      source: 'cached',
      similarity: topMatch.similarity
    };
  }

  // Build context from deterministic matches
  let additionalContext = '';
  const deterministicMatches = matches.filter(m => m.tier === 'deterministic');
  if (deterministicMatches.length > 0) {
    const contexts = deterministicMatches.map(m => {
      const origin = intelligence.getOrigin(db, m.origin_id);
      intelligence.recordHit(db, m.origin_id);
      return origin?.data;
    }).filter(Boolean);

    additionalContext = `\n\nRelevant context from previous interactions:\n${JSON.stringify(contexts, null, 2)}`;
    console.log(`[intelligence] Deterministic context: ${deterministicMatches.length} matches`);
  }

  // === EXISTING LLM CALL ===
  // Append vector context to system prompt
  const systemPrompt = buildSystemPrompt(opts) + additionalContext;
  const response = await callClaude(systemPrompt, userMessage, opts);

  // === CACHE THE RESULT ===
  // Store this request→response pair for future matching
  await intelligence.cacheResponse(db, {
    request: userMessage,
    response: response.text,
    journeyType: opts.journeyType
  });

  return {
    response: response.text,
    source: 'computed',
    similarity: topMatch?.similarity || 0
  };
}
```

**Key integration points:**
- Check vectors BEFORE Claude call (short-circuit on high similarity)
- Inject deterministic matches as context FOR Claude call (grounding)
- Cache response AFTER Claude call (learning)

---

## Phase 3: Protocol Population (Day 4-5)

### Step 7: Populate from Existing Protocols

Create a one-time script (and heartbeat job for ongoing maintenance) that reads your protocol files and populates the vector space:

```js
// scripts/populate-vectors.js
const fs = require('fs');
const path = require('path');
const intelligence = require('../lib/intelligence');

async function populateFromProtocols(db, protocolsDir) {
  const files = fs.readdirSync(protocolsDir, { recursive: true })
    .filter(f => f.endsWith('.md') || f.endsWith('.json'));

  for (const file of files) {
    const content = fs.readFileSync(path.join(protocolsDir, file), 'utf-8');

    // Extract sections from markdown (## headers become individual vectors)
    const sections = content.split(/^## /m).filter(s => s.trim());

    for (const section of sections) {
      const firstLine = section.split('\n')[0].trim();
      const body = section.slice(firstLine.length).trim();

      await intelligence.store(db, {
        text: `${firstLine}: ${body.slice(0, 500)}`,
        category: 'protocol',
        label: firstLine,
        data: {
          file,
          section: firstLine,
          content: body,
          type: 'protocol_section'
        }
      });
    }
  }
}

// Also populate from journey transition maps
async function populateFromJourneys(db) {
  const journeys = db.prepare('SELECT DISTINCT journey_type FROM journey_states').all();

  for (const { journey_type } of journeys) {
    // Embed the journey type itself
    await intelligence.store(db, {
      text: `Customer wants ${journey_type.replace(/_/g, ' ')}`,
      category: 'journey_routing',
      label: journey_type,
      data: {
        journey_type,
        type: 'journey_route'
      }
    });
  }
}

// Populate from existing successful conversations (if you have them)
async function populateFromConversations(db) {
  const convos = db.prepare(`
    SELECT c.id, c.messages
    FROM conversations c
    JOIN journey_states js ON js.conversation_id = c.id
    WHERE js.state IN ('completed', 'active', 'follow_up')
    LIMIT 100
  `).all();

  for (const convo of convos) {
    const messages = JSON.parse(convo.messages || '[]');
    // Find the first user message (the initial request)
    const firstUserMsg = messages.find(m => m.role === 'user');
    // Find the first assistant response
    const firstAssistantMsg = messages.find(m => m.role === 'assistant');

    if (firstUserMsg && firstAssistantMsg) {
      await intelligence.cacheResponse(db, {
        request: firstUserMsg.text || firstUserMsg.content,
        response: firstAssistantMsg.text || firstAssistantMsg.content
      });
    }
  }
}
```

### Step 8: Add to Heartbeat

In your heartbeat/scheduler, add a maintenance job:

```js
// In your heartbeat registration (from OPERATIONAL-PATTERNS.md)
registerJob('vector-maintenance', async () => {
  // Re-embed any protocols modified since last run
  const lastRun = getLastRunTime('vector-maintenance');
  // ... check file modification times, re-embed if changed

  // Prune low-confidence, zero-hit vectors older than 30 days
  db.prepare(`
    DELETE FROM vector_meta
    WHERE hit_count = 0
    AND confidence < 0.5
    AND created_at < datetime('now', '-30 days')
  `).run();
  // Also delete corresponding vss_vectors and origin_store entries
}, { interval: 60 * 60 * 1000 }); // Every hour
```

---

## Phase 4: Journey Routing (Day 6-7)

### Step 9: Vector-Powered Journey Selection

When a new customer arrives and says something, use vector similarity to route them to the right journey:

```js
async function routeToJourney(db, customerMessage) {
  const matches = await intelligence.query(db, {
    text: customerMessage,
    category: 'journey_routing',
    limit: 3
  });

  if (matches.length === 0 || matches[0].similarity < 0.6) {
    // No confident match — let the agent ask clarifying questions
    return { journey_type: null, confidence: 0, needs_clarification: true };
  }

  const top = matches[0];
  const origin = intelligence.getOrigin(db, top.origin_id);

  return {
    journey_type: origin.data.journey_type,
    confidence: top.similarity,
    needs_clarification: false
  };
}
```

This powers the no-gatekeeping principle: customer says "I want a nutrition plan" → vector search → `nutrition_plan` journey → serve plain-vanilla plan immediately.

---

## Build Order Summary

| Phase | What | Time | Risk |
|-------|------|------|------|
| 1 | Install deps, create tables, embedding module | Day 1 | Low — additive, no changes to existing code |
| 2 | Intelligence module + LLM integration | Day 2-3 | Medium — touches the LLM loop, but as a pre-check, not a replacement |
| 3 | Protocol population + heartbeat maintenance | Day 4-5 | Low — read-only against existing data |
| 4 | Journey routing | Day 6-7 | Low — new feature, doesn't change existing routing |

**Total: ~1 week of focused work to have a functional intelligence layer.**

After Phase 4, observe. Watch the logs. See how many requests hit static vs deterministic vs agentic. Tune thresholds. The system teaches itself from there.

---

## What NOT To Do

- **Don't remove the JM engine.** Vectors route TO journeys. The JM engine manages state WITHIN journeys. Different jobs.
- **Don't embed full documents.** Meta vectors only. Short structured text. Keep vectors small and meaningful.
- **Don't chase perfect embeddings.** The MiniLM model is "good enough." Perfect is the enemy of shipped.
- **Don't expose vectors externally.** They're internal intelligence. External communication uses protocols.
- **Don't pre-optimize.** Start with the simplest integration (Phase 2), see what happens, then expand. The system improves itself through usage — trust the caching loop.
- **Don't use a cloud vector service.** sqlite-vss runs locally, in-process, sovereign. No vendor dependency. No data leaving the node.

---

## Success Criteria

After 1 week of running with the intelligence layer:

- [ ] `[intelligence] Static hit` appears in logs for repeated request patterns
- [ ] Token costs drop measurably for known request types
- [ ] Journey routing correctly identifies journey type >80% of the time
- [ ] The validation gap narrows — fewer "100% claimed, 50% working" moments
- [ ] Protocol changes propagate to vectors via heartbeat (no manual re-embedding)

After 1 month:

- [ ] >50% of customer requests hit static or deterministic tier
- [ ] New solution patterns emerge in the vector space that weren't explicitly programmed
- [ ] The 2+2 threshold is tuned to the dream's domain (might be 0.92 instead of 0.95)
- [ ] Test bots have populated the cache with common archetypes

---

## Technical Notes

**sqlite-vss compatibility:** Requires SQLite 3.41+. The `better-sqlite3` npm package should work. If the extension loading fails, try `better-sqlite3@latest` or build sqlite-vss from source.

**Alternative to sqlite-vss:** If extension loading is problematic, use `hnswlib-node` (pure JS approximate nearest neighbor) with vectors stored in a regular SQLite table as JSON arrays. Slightly less elegant but zero native dependency issues.

**Embedding model alternatives:**
- `all-MiniLM-L6-v2` — 384 dim, 80MB, recommended starting point
- `all-MiniLM-L12-v2` — 384 dim, 120MB, slightly better quality
- `bge-small-en-v1.5` — 384 dim, 130MB, newer, better for retrieval
- Claude/Voyage API — no local model, ~$0.0001 per embed, simplest setup

**Memory estimate:** 1,000 vectors × 384 dims × 4 bytes = 1.5 MB. Even 50,000 vectors = 75 MB. This is not a memory concern.
