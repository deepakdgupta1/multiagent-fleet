# Multi-Agent Fleet — Solution Design & Architecture

> Status: DRAFT v1 (April 26, 2026)
> Repository: https://github.com/deepakdgupta1/multiagent-fleet
> Owner: Deepak (deep.g)

---

## 1. Problem Statement

We have a 4-agent fleet (Oracle, Arch, Eureka, Aurus) running on Discord + WhatsApp. It was assembled quickly over April 13-19 and has not been matured. The current state has real problems:

### Symptoms

1. **Persona leakage** — Arch exposed its raw reasoning block ("💭 Reasoning: ...") to the user in Discord. The agent's internal thought process was broadcast verbatim.
2. **Identity confusion** — Arch had to reason through whether it was Oracle or Arch ("let me check the system prompt again... file://personas/arch.md"). The persona swap should be seamless.
3. **No isolation** — All 4 personas share one Hermes home (`~/.hermes` → `~/.arch`). One gateway process serves all channels. There is no tool, memory, or state isolation between agents.
4. **Missing cross-agent visibility** — Oracle cannot see what happened in Arch/Eureka/Aurus conversations without manual `session_search`. The proposed `fleet/activity.md` shared log was never implemented.
5. **3 of 4 agents are shells** — Oracle, Eureka, and Aurus have empty `skills/` directories. Only Arch (the shared home) has real skills.
6. **No inter-agent communication** — Oracle cannot programmatically post in other channels. The `discord_post` tool was discussed but never built.
7. **Reasoning format not controlled** — Gateway doesn't suppress or format reasoning tokens before sending to Discord.

---

## 2. Current Architecture (As-Is)

```
┌─────────────────────────────────────────────────────────┐
│                   Single Hermes Process                  │
│            (systemd: hermes-gateway.service)             │
│                  HERMES_HOME=~/.hermes                   │
│                      (symlink → ~/.arch)                 │
│                                                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────┐ │
│  │  Oracle   │  │   Arch   │  │  Eureka  │  │ Aurus  │ │
│  │ #assistant│  │  #code   │  │ #content │  │ #money │ │
│  │ +WhatsApp │  │          │  │          │  │        │ │
│  └──────────┘  └──────────┘  └──────────┘  └────────┘ │
│         Persona swap via channel_prompts in config.yaml │
│         file://personas/{arch,eureka,aurus}.md          │
└─────────────────────────────────────────────────────────┘
         │                          │
         │ Discord (1 bot token)    │ WhatsApp (bridge.js)
         │                          │
    ┌────▼────┐              ┌──────▼──────┐
    │ Discord │              │  WhatsApp   │
    │  DeepG  │              │  self-chat  │
    │  Server │              │   bridge    │
    └─────────┘              └─────────────┘
```

### Key Facts

| Item | Value |
|------|-------|
| Gateway process | 1 (systemd service) |
| HERMES_HOME | `~/.hermes` → `~/.arch` (symlink) |
| Discord bot tokens | 1 (shared) |
| WhatsApp bridge | 1 (Node.js/Baileys, Oracle only) |
| Persona mechanism | `channel_prompts` with `file://` references |
| Agent homes | `~/.oracle`, `~/.arch`, `~/.aurus`, `~/.eureka` |
| Startup script | `~/.arch/bin/start-agents.sh` (NOT used by systemd) |
| Config | `~/.arch/config.yaml` (shared, single file) |

### Discord Channel Map

| Channel | ID | Persona File |
|---------|----:|-------------|
| #assistant (Oracle) | 1465562736451911838 | (default SOUL.md = Oracle) |
| #code (Arch) | 1493461055618285670 | `file://personas/arch.md` |
| #content (Eureka) | 1493461225567420416 | `file://personas/eureka.md` |
| #money (Aurus) | 1493461326662467624 | `file://personas/aurus.md` |

### Agent Home Contents

| | Oracle | Arch | Eureka | Aurus |
|--|--------|------|--------|-------|
| SOUL.md | Yes | Yes (base) | Yes | No |
| config.yaml | Yes (unused) | Yes (active) | Yes (unused) | Yes (unused) |
| skills/ | 27 dirs (from ~/.arch) | 26 dirs (active) | empty | empty |
| sessions/ | Yes | Yes | empty | empty |
| memories/ | Yes | Yes | empty | empty |
| SHARED_FOUNDATION.md | symlink | source | symlink | symlink |
| .env | symlink→~/.arch | source | symlink→~/.arch | symlink→~/.arch |

---

## 3. Root Cause Analysis

### 3A. Reasoning Block Leakage

The Hermes gateway sends the LLM's complete response to Discord, including reasoning/thinking tokens. The Discord platform adapter (`gateway/platforms/discord.py`) does not strip `reasoning` content from assistant messages before sending.

**Fix:** The Discord adapter's `send()` method needs to filter out reasoning blocks from the response payload before posting to Discord.

### 3B. Identity Confusion

Arch's persona is loaded as a `file://` channel prompt overlay at message time. But:
- The base SOUL.md says "Oracle" everywhere
- The agent has to reconcile Oracle's SOUL.md with Arch's overlay on every message
- The reasoning tokens expose this internal reconciliation process

**Fix:** Channel prompts should completely override (not merge with) the base persona when a channel-specific persona is active. Or: the base SOUL.md should be agent-neutral, with channel prompts providing the full identity.

### 3C. No Agent Isolation

Everything runs from `~/.arch` (via `~/.hermes` symlink). Oracle, Eureka, and Aurus have home directories but they're decorative — the single gateway process uses Arch's config, Arch's skills, Arch's sessions, Arch's memories. The other homes exist but aren't wired to anything.

**Fix:** Either (a) accept shared-home as the architecture and formalize it, or (b) separate into true independent homes. This is a fundamental architectural decision (see Section 5).

### 3D. Cross-Agent Blindness

Oracle has no structured way to know what happened in specialist channels. The proposed `fleet/activity.md` was never created. `session_search` works but requires Oracle to actively search and is not automatic.

**Fix:** Shared activity log + event-driven notifications (see Section 6).

---

## 4. Proposed Architecture (To-Be)

### Architecture Decision: Shared Home with Persona Routing

Given Discord's 1-token-1-connection constraint, we continue with the **Single Gateway + Channel Routing** pattern. But we formalize and mature it.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Single Gateway Process                        │
│                  HERMES_HOME=~/.hermes                          │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              Persona Router (config.yaml)                │   │
│  │   channel_prompts: {channel_id → persona_override}      │   │
│  │   Default: Oracle SOUL.md                               │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              Shared Infrastructure Layer                 │   │
│  │  • skills/       (curated per-agent via persona files)  │   │
│  │  • sessions/     (shared, searchable via session_search)│   │
│  │  • memories/     (shared, with agent-scoped entries)    │   │
│  │  • fleet/        (cross-agent coordination)             │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐  │
│  │  Oracle   │  │   Arch   │  │  Eureka  │  │    Aurus     │  │
│  │ #oracle   │  │  #arch   │  │ #eureka  │  │   #aurus    │  │
│  │ +WhatsApp │  │          │  │          │  │              │  │
│  └──────────┘  └──────────┘  └──────────┘  └──────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### Key Changes from Current State

1. **Base SOUL.md becomes agent-neutral** — No "I am Oracle" in the base file. Oracle gets its identity from the default channel (no override). Specialists get theirs from channel prompts.
2. **Reasoning blocks are stripped** in the Discord adapter before sending.
3. **Fleet coordination layer** — `fleet/` directory with shared state.
4. **Persona files are self-contained** — Each persona file includes everything the agent needs, not just domain info.
5. **Skills are persona-aware** — Persona files specify which skills to load, not the global config.

---

## 5. Open Questions

These require your decision before implementation can proceed.

### Q1: Agent Isolation Level

**Option A: Shared Home (Current, Formalized)**
- One HERMES_HOME, one config.yaml, one set of sessions/memories
- Personas are thin overlays via channel_prompts
- Pros: Simple, no multi-process complexity, shared context
- Cons: No privacy between agents, skill conflicts, one agent's mess affects all

**Option B: Separate Homes, Single Gateway**
- Gateway runs from one home, but persona files reference per-agent configs
- Each agent has its own skills/, memories/, sessions/ subdirectory
- Pros: Clean separation, agent-specific skill curation
- Cons: More complex routing, gateway needs modification to support multi-home

**Option C: Multi-Process, Multi-Token**
- Each agent gets its own Discord bot token and gateway process
- Full isolation
- Pros: Complete independence, each agent can restart independently
- Cons: Requires 4 Discord bot tokens, 4x resource usage, no shared context

**Recommendation:** Option A (formalized shared home). The current setup works; it just needs structure. The real problems (reasoning leakage, identity confusion) are code fixes, not architecture changes.

### Q2: Channel Naming

Current Discord channels are named generically (#assistant, #code, #content, #money). Should we rename them to match agent names (#oracle, #arch, #eureka, #aurus)?

**Recommendation:** Yes. Reduces user confusion about which channel does what.

### Q3: Inter-Agent Communication

How should agents communicate with each other?

**Option A: Fleet Activity Log** — Shared `fleet/activity.md` file that all agents append to. Oracle reads it periodically.

**Option B: Webhook Posts** — Oracle uses Discord webhooks to post summaries in specialist channels.

**Option C: Delegate Only** — No direct agent-to-agent communication. Oracle delegates via `delegate_task` and synthesizes results.

**Recommendation:** Start with C (delegate only) — it already works. Add A when cross-agent visibility becomes a real pain point.

### Q4: Startup & Process Management

Current: `start-agents.sh` launches 4 processes, but systemd only runs 1. The startup script is not used in production.

**Recommendation:** Decide one way. Either (a) use systemd with the single-gateway approach, or (b) use the startup script. Don't maintain both. Systemd is the right answer for production.

### Q5: Memory Scoping

Should agents share memories or have separate memory stores?

**Recommendation:** Shared memories with agent-scoped tags. Each memory entry includes which agent created it. Oracle can see all; specialists see their own + Oracle's.

### Q6: Skill Curation

3 of 4 agents have empty skills directories. Should each agent have its own curated skill set?

**Recommendation:** Yes, but implement via persona files, not separate directories. The persona file specifies which skills to load (subset of the shared skills/ directory).

---

## 6. Implementation Plan

### Phase 1: Critical Fixes (This Week)

| # | Task | Why | Complexity |
|---|------|-----|-----------|
| 1.1 | Patch Discord adapter to strip reasoning blocks from responses | Stops the bleeding — no more internal thoughts exposed | Medium |
| 1.2 | Make base SOUL.md agent-neutral | Eliminates identity confusion at the root | Easy |
| 1.3 | Make persona files self-contained with full identity | Each agent knows exactly who it is without reconciliation | Easy |
| 1.4 | Verify end-to-end: all 4 channels respond with correct persona | Validates the fixes work | Easy |

### Phase 2: Structural Cleanup (Next Week)

| # | Task | Why | Complexity |
|---|------|-----|-----------|
| 2.1 | Create `fleet/` directory with shared state structure | Foundation for cross-agent coordination | Easy |
| 2.2 | Curate skills per persona (Arch=full, Oracle=strategic, Eureka=research, Aurus=finance) | Each agent loads only relevant tools | Medium |
| 2.3 | Remove dead startup script or wire it properly | Eliminate confusion about how fleet starts | Easy |
| 2.4 | Audit and clean symlinks in agent homes | Remove decorative directories that serve no purpose | Easy |

### Phase 3: Maturation (Ongoing)

| # | Task | Why | Complexity |
|---|------|-----|-----------|
| 3.1 | Implement fleet activity log | Cross-agent visibility | Medium |
| 3.2 | Add memory scoping with agent tags | Prevent cross-contamination | Medium |
| 3.3 | Rename Discord channels to match agent names | Clarity | Easy (requires Discord admin) |
| 3.4 | Build `discord_post` tool for Oracle | Programmatic cross-channel communication | Medium |
| 3.5 | Add persona-specific system prompt testing | Regression prevention | Medium |
| 3.6 | Document the architecture in this repo | Knowledge preservation | Easy |

---

## 7. File Structure (Proposed)

```
~/.hermes/                          # Single home (symlinked from ~/.arch)
├── SOUL.md                         # Agent-neutral base identity
├── config.yaml                     # Gateway config with channel_prompts
├── .env                            # Shared secrets
├── auth.json                       # Shared auth
├── agent-templates/
│   └── SHARED_FOUNDATION.md        # Universal values (unchanged)
├── personas/
│   ├── oracle.md                   # Oracle's full persona (loaded for #oracle)
│   ├── arch.md                     # Arch's full persona (loaded for #arch)
│   ├── eureka.md                   # Eureka's full persona (loaded for #eureka)
│   └── aurus.md                    # Aurus's full persona (loaded for #aurus)
├── fleet/
│   ├── activity.md                 # Cross-agent activity log
│   ├── agent-manifest.yaml         # Agent definitions, channels, capabilities
│   └── health/                     # Health checks and status
├── skills/                         # Shared skill library
│   ├── all 19+ skills...           # Curated skill set
│   └── ...                         # Persona files specify which to load
├── sessions/                       # Shared sessions (searchable)
├── memories/                       # Shared memories (agent-scoped)
├── whatsapp/                       # WhatsApp bridge (Oracle only)
└── hermes-agent/                   # Hermes source code
```

---

## 8. Key Decisions Taken

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Architecture pattern | Single gateway, channel-routed | Discord 1-token constraint; simplest working model |
| Agent isolation | Shared home, persona-overlay | Formalize what works; fix code bugs separately |
| Persona loading | Channel prompt = complete identity | No reconciliation with base SOUL.md needed |
| Cross-agent comms | Delegate-only (start) | Already works; add structured comms later |
| Process management | systemd (single service) | Production-grade; startup script is legacy |
| Memory model | Shared, agent-tagged | Oracle sees all; specialists see own + Oracle's |

---

## 9. Reasoning & Rationale

**Why not multi-process/multi-token?** It's the "right" architecture in theory — full isolation, independent scaling — but it solves problems we don't have yet. The actual bugs (reasoning leakage, identity confusion) are code-level fixes in the single-process model. Adding 4x infrastructure before fixing the basics is doing the wrong thing right.

**Why formalize shared home instead of fighting it?** The shared home has been de facto reality since April 19. The separate agent directories (`~/.oracle`, `~/.eureka`, `~/.aurus`) exist but aren't wired to anything. Rather than building complex routing to give each agent its own home, we accept the shared model and add structure (persona files, skill curation, memory tags) to make it work cleanly.

**Why agent-neutral SOUL.md?** The current SOUL.md says "I am Oracle" everywhere. When Arch loads and sees "I am Oracle" in SOUL.md plus "I am Arch" in the channel prompt, it has to reconcile — and that reconciliation is visible in reasoning tokens. Making SOUL.md neutral (or removing it entirely in favor of per-channel personas) eliminates this at the source.

**Why Phase 1 before Phase 2?** The reasoning block leak is a user-facing embarrassment. It needs to stop immediately. Structural cleanup matters but can wait a week.
