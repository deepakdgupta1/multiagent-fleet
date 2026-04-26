# Multi-Agent Fleet

A 4-agent AI fleet running on Discord + WhatsApp, powered by a single Hermes gateway process.

## Agents

| Agent | Role | Discord Channel |
|-------|------|----------------|
| **Oracle** | Alter ego & life navigator — strategy, coordination, values guardian | #oracle |
| **Arch** | Builder — code, infrastructure, DevOps, tools | #arch |
| **Eureka** | Knowledge engine — learning, research, content | #eureka |
| **Aurus** | Money engine — financial learning, tracking, experiments | #aurus |

## Architecture

Single Hermes gateway process routes Discord channels to agent personas via `channel_prompts`. All agents share one gateway home (`~/.hermes`) with persona overlays per channel. WhatsApp bridge connects Oracle for direct conversation.

See [ARCHITECTURE.md](ARCHITECTURE.md) for the full design document.

## Documentation

| Document | Purpose |
|----------|---------|
| [ARCHITECTURE.md](ARCHITECTURE.md) | Full architecture, root causes, decisions, implementation plan |
| [phase0-system-prompt-assembly.md](phase0-system-prompt-assembly.md) | Phase 0 investigation: why personas never loaded (3 compounding bugs) |
| [SHARING-ISOLATION-ANALYSIS.md](SHARING-ISOLATION-ANALYSIS.md) | Trial framework: shared home hypotheses, falsification criteria, data collection |
| [MIGRATION-GUIDE.md](MIGRATION-GUIDE.md) | Ubuntu → macOS migration with portability audit |

## Current Status

- **Phase 0 (Investigation):** COMPLETE — personas have never loaded due to 3 bugs in gateway code
- **Phase 1 (Persona Fix):** NEXT — spec being written for Arch to implement
- **Phase 2 (Directory Restructuring):** Pending — rename `~/.arch` → `~/.hermes`, wire agent CLI homes
- **Phase 3 (Trial Week):** Pending — observe fleet with correct personas, collect H1-H5 data
- **Host:** Ubuntu Desktop (moving to Mac Mini as 24/7 host after architecture stabilizes)

## Fleet Infrastructure

```
Discord DeepG Server
├── #oracle  → Oracle (alter ego, WhatsApp bridge)
├── #arch    → Arch (builder)
├── #eureka  → Eureka (knowledge)
└── #aurus   → Aurus (money)

WhatsApp self-chat → Oracle
```

Gateway: Hermes Agent, single process, systemd service (launchd on macOS).
