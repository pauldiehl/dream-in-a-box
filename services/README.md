# Node Services (Layer 2)

Services are shared capabilities available to every dream on the node. They are NOT standalone applications.

## Core Services

### Forkless
Conversational commerce engine. Provides the Agent Layer (chat interface) and Transaction Layer (artifact rendering) to every dream. Lives in its own repo and is imported/referenced here.

**Repo:** [github.com/pauldiehl/forkless](https://github.com/pauldiehl/forkless)

### Greenspaces
Community and environment layer. Shared spaces, collaboration, environmental context. Early concept stage.

## Adding Services

New services can be added to a node. A service must:
- Respect node-level protocols (budget, identity, verification)
- Expose capabilities that any dream can use
- Not require dreams to know about infrastructure
- Register with the node agent for heartbeat monitoring
