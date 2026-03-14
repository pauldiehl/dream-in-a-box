# CLAUDE.md — Persistent Context for AI Assistants

## What This Project Is

Dream in a Box (DIAB) is a sovereign cloud node specification and reference implementation for Web 4.0. It is the orchestration layer — the thing that boots, manages, and runs everything.

This is NOT an application. It is a runtime for applications ("dreams").

## Architecture — Four Layers

- **Layer 0 — Node Agent:** Background process. Boots infrastructure, enforces protocols, runs heartbeat jobs, manages budgets. Think of it as the OS kernel.
- **Layer 1 — Protocols:** Human-readable files (markdown/JSON) that define rules. Identity, verification, budget, syndication, dream lifecycle. The node agent reads and enforces these.
- **Layer 2 — Services:** Shared capabilities available to all dreams. Forkless (conversational commerce) and Greenspaces (community) are the first two. These are NOT standalone apps — they are node-level tools.
- **Layer 3 — Dreams:** The actual products/experiences/businesses. Each dream is a self-contained definition that declares what it needs. Dreams don't know about infrastructure.

## Critical Distinctions

- **Services vs Dreams:** Forkless is a SERVICE (available to every dream). "Plain Fun" or "Coach Kid" are DREAMS (products that use services). Never confuse the two.
- **Protocol-first:** Everything starts as a protocol before it becomes code. If you're about to write code, ask yourself: should this be a protocol definition first?
- **Sovereign:** Each node is independently owned. No central authority. No app store. Invite-only. Peer-to-peer payments. No middleman.
- **This repo vs sovereign-streams:** sovereign-streams holds Web 4.0 theory and philosophy. This repo is the concrete implementation. Import concepts from sovereign-streams; don't duplicate them.

## Operational Patterns (from docs/OPERATIONAL-PATTERNS.md)

Four core patterns govern the node agent:
1. **Heartbeat Pattern** — Background agents wake, check, act, sleep
2. **Atomic Budget Enforcement** — Per-agent/conversation/user token limits
3. **Task-to-Goal Traceability** — Every action traces to a top-level goal
4. **Multi-Company Data Isolation** — One node can serve multiple brands with complete isolation

## Key Decisions

- Check docs/decisions.md before making architectural choices
- No frameworks — vanilla everything, AI is the framework
- Protocol files are the source of truth, not code
- Start cheap, scale when needed

## Related Repos

- `forkless` — Conversational commerce engine (Layer 2 service)
- `greenspaces` — Community layer (Layer 2 service)
- `sovereign-streams` — Web 4.0 theory (the standards body)
- `protocol-explorer` — Dev tooling for protocol visualization
