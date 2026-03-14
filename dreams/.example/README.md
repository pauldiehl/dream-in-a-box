# Example Dream Structure

A dream is a self-contained product definition. This directory shows the expected layout.

```
my-dream/
├── dream.json          # Dream manifest (name, protocols, service requirements)
├── protocols/          # Dream-level protocols (extend node protocols)
│   └── custom-rules.md
├── content/            # Dream content, templates, assets
└── README.md           # What this dream is
```

## dream.json (planned format)

```json
{
  "name": "My Dream",
  "slug": "my-dream",
  "description": "What this dream does",
  "services": ["forkless", "greenspaces"],
  "protocols": "./protocols/",
  "verification": {
    "required": true,
    "minAge": 13,
    "parentalControls": true
  }
}
```

Dream-level protocols extend node-level protocols. They cannot override node rules (e.g., a dream cannot disable adult verification if the node requires it).
