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
