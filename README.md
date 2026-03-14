# Dream in a Box (DIAB)

**An open sovereign cloud node for Web 4.0.**

A Dream in a Box is a self-contained runtime that boots, manages, and scales everything needed to run one or more *dreams* — products, experiences, businesses — without middlemen, without platforms, without giving up ownership.

Each node is sovereign. Invite-only. Peer-to-peer. No central registry. No app store.

## What's Inside a Node

```
┌─────────────────────────────────────────────────┐
│                  DREAM IN A BOX                  │
│                 (sovereign node)                 │
│                                                  │
│  ┌─────────────────────────────────────────────┐ │
│  │              DREAMS (Layer 3)                │ │
│  │                                              │ │
│  │  Plain Fun  ·  Coach Kid  ·  Good Vibes     │ │
│  │  Yomo  ·  Torty  ·  Man vs Health  · ...    │ │
│  │                                              │ │
│  │  (products, experiences, businesses)         │ │
│  └──────────────────┬──────────────────────────┘ │
│                     │ uses                       │
│  ┌──────────────────▼──────────────────────────┐ │
│  │            SERVICES (Layer 2)                │ │
│  │                                              │ │
│  │  Forkless ─── conversational commerce        │ │
│  │  Greenspaces ─ community + environment       │ │
│  │  (+ future services)                         │ │
│  │                                              │ │
│  │  Built-in or added. Available to ALL dreams. │ │
│  └──────────────────┬──────────────────────────┘ │
│                     │ governed by                 │
│  ┌──────────────────▼──────────────────────────┐ │
│  │           PROTOCOLS (Layer 1)                │ │
│  │                                              │ │
│  │  identity · verification · budget            │ │
│  │  syndication · dream lifecycle · peer disc.  │ │
│  │                                              │ │
│  │  Files. Markdown or JSON. Human-readable.    │ │
│  └──────────────────┬──────────────────────────┘ │
│                     │ enforced by                 │
│  ┌──────────────────▼──────────────────────────┐ │
│  │          NODE AGENT (Layer 0)                │ │
│  │                                              │ │
│  │  Boots the node. Manages infrastructure.     │ │
│  │  Runs background jobs. Enforces protocols.   │ │
│  │  Heartbeat. Budget. Traceability. Isolation. │ │
│  └─────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────┘
```

## Key Principles

**Protocol-first.** Everything is a protocol before it's code. Protocols are files — readable, versionable, portable. The node agent reads them and enforces them.

**Sovereign by default.** Each node is owned. No platform takes a cut. No central authority approves what runs on it. Monetization happens at the sovereign level only — direct peer-to-peer.

**Services, not frameworks.** Forkless and Greenspaces aren't apps you deploy. They're capabilities that come with the node. Every dream gets conversational commerce and community tooling the same way every OS gets a filesystem.

**Dreams are the product.** The node exists to serve dreams. A dream is a self-contained product definition — its protocols, its content, its rules. Dreams don't know about infrastructure. They declare what they need; the node handles the rest.

**Invite-only, not public.** A DIAB node serves the people it's meant to serve. Discovery happens through syndication between consenting nodes, not through search engines or app stores.

## Project Structure

```
dream-in-a-box/
├── docs/                    # Architecture, decisions, patterns
│   ├── architecture.md      # The four-layer model
│   ├── decisions.md         # Decision log
│   └── OPERATIONAL-PATTERNS.md  # Heartbeat, budget, traceability, isolation
├── protocols/               # Default protocol definitions
│   └── (coming soon)
├── services/                # Node services (Forkless, Greenspaces, etc.)
│   └── (imports/references)
├── dreams/                  # Example dream definitions
│   └── .example/
└── node-agent/              # Layer 0 — the background agent (future)
```

## Status

**Phase: Project Definition.** This repo anchors the concrete implementation of Web 4.0's sovereign cloud node concept. Theory and philosophy live in [sovereign-streams](https://github.com/sovereign-streams). This is where theory becomes runnable.

## Related Projects

| Project | Role | Relationship |
|---------|------|-------------|
| **Forkless** | Conversational commerce engine | Node service (Layer 2) |
| **Greenspaces** | Community + environment layer | Node service (Layer 2) |
| **sovereign-streams** | Web 4.0 theory + specifications | The standards body |
| **protocol-explorer** | Protocol visualization | Dev tooling |
| **Plain Fun** | Open gaming platform | Example dream |
| **Coach Kid** | Youth coaching platform | Candidate dream |
| **Good Vibes** | Wellness platform | Candidate dream |
| **Yomo** | (TBD) | Candidate dream |
| **Torty** | Martial arts platform | Candidate dream |
| **Man vs Health** | Health journeys | First dream (via Forkless) |

## License

TBD — likely open source with sovereignty protections.
