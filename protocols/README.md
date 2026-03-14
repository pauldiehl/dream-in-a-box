# Default Protocols

This directory will contain the default protocol definitions that ship with every DIAB node.

**Planned protocols:**

- `identity.md` — Node ownership, user identity, verification requirements
- `budget.md` — Spending limits per dream, per user, per conversation
- `syndication.md` — Peer discovery and content sharing between consenting nodes
- `dream-lifecycle.md` — How dreams boot, pause, archive, and terminate
- `verification.md` — Adult verification, age gating, parental controls
- `monetization.md` — P2P payment rules, sovereign revenue only

Protocols are human-readable files. The node agent (Layer 0) reads and enforces them. Dreams can extend protocols but cannot override node-level rules.
