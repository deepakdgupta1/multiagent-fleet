# Multi-Agent Fleet — Solution Design & Architecture

> Status: DRAFT v2 (April 26, 2026)
> Repository: https://github.com/deepakdgupta1/multiagent-fleet
> Owner: Deepak (deep.g)

---

## 1. Problem Statement

We have a 4-agent fleet (Oracle, Arch, Eureka, Aurus) running on Discord + WhatsApp. It was assembled quickly over April 13-19 and has not been matured. The current state has real problems:

### Symptoms

1. **Identity confusion** — Arch had to reason through whether it was Oracle or Arch ("let me check the system prompt again... file://personas/arch.md"). The persona swap should be seamless. This was discovered because reasoning blocks leaked to Discord (see point 2), revealing the internal confusion.
2. **Reasoning blocks visible in Discord** — Arch exposed its raw reasoning block ("💭 Reasoning: ...") to the user. This is what surfaced symptom #1. Status: intentionally left as-is for now. The reasoning blocks provide useful transparency in a private single-user context. No harmful consequences have been identified in our setup. Revisit if/when the fleet becomes multi-user or public-facing.
3. **No isolation** — All 4 personas share one Hermes home (`~/.hermes` → `~/.arch`). One gateway process serves all channels. There is no tool, memory, or state isolation between agents.
4. **Missing cross-agent visibility** — Oracle cannot see what happened in Arch/Eureka/Aurus conversations without manual `session_search`. The proposed `fleet/activity.md` shared log was never implemented.
5. **3 of 4 agents are shells** — Oracle, Eureka, and Aurus have empty `skills/` directories. Only Arch (the shared home) has real skills.
6. **No inter-agent communication** — Oracle cannot programmatically post in other channels. The `discord_post` tool was discussed but never built.

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

### 3A. Identity Confusion — STATUS: Needs Investigation

**Symptom:** Arch reasoned about whether it was Oracle or Arch, visible in reasoning tokens.

**What we know:**
- The base SOUL.md says "I am Oracle" (it IS Oracle's SOUL.md)
- Channel prompts load via `file://personas/arch.md` for Arch's channel
- The agent must reconcile these two conflicting identity sources

**What we DON'T know (investigation needed):**
1. How does `channel_prompts` actually merge with the base system prompt? Does it REPLACE, APPEND, or PREPEND? This is in `gateway/platforms/base.py`.
2. What does the final assembled system prompt look like for each channel? We need to see the exact output.
3. Is SOUL.md even the problem, or is it the merge behavior?

**Investigation steps before any fix is designed:**
1. Read `gateway/platforms/base.py` — understand the channel_prompts merge logic
2. Read the gateway session handler — trace how SOUL.md + channel_prompt are assembled into the final system message
3. Send a test message to each Discord channel and log the exact system prompt each agent receives
4. Based on the actual assembled prompt, design the minimal fix

**No fix will be prescribed until this investigation is complete.**

### 3B. Reasoning Block Visibility — STATUS: Deferred

The reasoning blocks are visible in Discord. This is what surfaced the identity confusion.

**Assessment:** In our current context (private Discord server, single user), this is actually useful transparency. No harmful consequences identified. The reasoning blocks helped diagnose the real problem. Leaving as-is until a concrete harm is identified.

### 3C. No Agent Isolation — STATUS: Architectural Decision Needed

Everything runs from `~/.arch` (via `~/.hermes` symlink). Oracle, Eureka, and Aurus have home directories but they're decorative — the single gateway process uses Arch's config, Arch's skills, Arch's sessions, Arch's memories. The other homes exist but aren't wired to anything.

This is a deliberate architectural choice with real trade-offs (see Q1 in Section 5).

### 3D. Cross-Agent Blindness — STATUS: Known Gap

Oracle has no structured way to know what happened in specialist channels. The proposed `fleet/activity.md` was never created. `session_search` works but requires Oracle to actively search and is not automatic.

---

## 4. Open Questions

These require your decision before implementation can proceed.

### Q1: Agent Isolation Level

**Option A: Shared Home (Current State, Formalized)**
- One HERMES_HOME, one config.yaml, one set of sessions/memories
- Personas are overlays via channel_prompts
- Pros: Simple, no multi-process complexity, shared context, single maintenance surface
- Cons:
  - No privacy between agents — Oracle sees Arch's sessions and vice versa, no agent has a private scratchpad
  - Memory contamination — one agent's saved memories surface in another agent's context (e.g., Arch saving a debugging memory could pollute Aurus's finance answers)
  - Skill noise — all agents load the same skill set; Aurus doesn't need GitHub skills but gets them anyway (wasted tokens, potential confusion)
  - Single point of failure — gateway crash kills all 4 agents simultaneously; can't restart Arch without taking down Oracle
  - Config coupling — changes to config.yaml for one agent's behavior risk breaking others
  - No permission isolation — any agent can modify any other agent's state files, memories, sessions
  - Can't scale independently — if Arch needs a different model or more resources, that affects everyone
  - state.db contention — single SQLite database with concurrent writes from 4 agents through one process
  - Agents are masks on one face, not truly independent entities — this may be fine forever, or it may become a ceiling when you want Eureka to genuinely operate independently

**Option B: Separate Homes, Single Gateway**
- Gateway runs from one home, but persona files reference per-agent configs
- Each agent has its own skills/, memories/, sessions/ subdirectory
- Pros: Clean separation, agent-specific skill curation, independent state
- Cons:
  - Gateway needs modification to support multi-home routing
  - More complex — the channel_prompts mechanism was not designed for this
  - Shared context (Oracle seeing all) requires explicit bridging

**Option C: Multi-Process, Multi-Token**
- Each agent gets its own Discord bot token and gateway process
- Full isolation
- Pros: Complete independence, each agent can restart independently, true separation
- Cons: Requires 4 Discord bot tokens, 4x resource usage (~2.3GB memory per gateway), no shared context by default, significantly more operational complexity

**Status: Decision needed.**

### Q2: Channel Naming

Current Discord channels are named generically (#assistant, #code, #content, #money). Should we rename them to match agent names (#oracle, #arch, #eureka, #aurus)?

Pro: Reduces user confusion about which channel does what.
Con: Requires updating channel_prompts channel IDs in config.yaml (Discord generates new IDs on rename... actually, Discord channel renames preserve IDs, so this is a non-issue).

**Status: Low priority, straightforward yes/no.**

### Q3: Inter-Agent Communication

How should agents communicate with each other?

**Option A: Fleet Activity Log** — Shared `fleet/activity.md` file that all agents append to. Oracle reads it periodically.

**Option B: Webhook Posts** — Oracle uses Discord webhooks to post summaries in specialist channels.

**Option C: Delegate Only** — No direct agent-to-agent communication. Oracle delegates via `delegate_task` and synthesizes results.

**Status: Start with C. Revisit when cross-agent visibility becomes a real pain point.**

### Q4: Startup & Process Management

Current state: `start-agents.sh` launches 4 processes, but systemd only runs 1. The startup script is not used in production — it's dead code that describes an architecture we didn't end up using.

**Status: Remove the startup script, or update it to reflect reality (single gateway). Low priority.**

### Q5: Memory Scoping

Should agents share memories or have separate memory stores?

**Status: Depends on Q1. If shared home, then shared memories with agent-scoped tags. If separate homes, then separate stores with explicit sharing.**

### Q6: Skill Curation

3 of 4 agents have empty skills directories. Should each agent have its own curated skill set?

**Status: Depends on Q1 and Q3. If shared home with delegate-only, specialists may not need their own skills — Oracle delegates to Arch which has the full set. If specialists handle tasks directly in their channels, they need relevant skills loaded.**

---

## 5. What We Know vs. What We Assume

| Claim | Status | Confidence |
|-------|--------|-----------|
| Discord enforces 1-token-1-connection | Verified (April 13-19 sessions) | High |
| Single gateway is running via systemd | Verified (process list, systemctl) | High |
| channel_prompts uses `file://` references | Verified (config.yaml) | High |
| SOUL.md says "I am Oracle" | Verified (file contents) | High |
| The merge behavior is the cause of identity confusion | **Unverified — assumption** | Low |
| Shared home is the right architecture | **Unverified — preference** | Medium |
| Reasoning block visibility is harmless | Assessed for current context | Medium |
| Agent homes (~/.oracle, ~/.eureka, ~/.aurus) are unused | Verified (symlink structure, empty dirs) | High |

---

## 6. Implementation Plan

### Phase 0: Investigation (Before Any Code Changes)

| # | Task | Why | Complexity |
|---|------|-----|-----------|
| 0.1 | Read `gateway/platforms/base.py` — trace channel_prompts merge logic | Understand how SOUL.md + channel prompt are assembled | Easy |
| 0.2 | Log the exact final system prompt for each of the 4 channels | See what each agent actually receives | Easy |
| 0.3 | Based on 0.1-0.2, write up the actual cause of identity confusion | Replace assumptions with evidence | Easy |
| 0.4 | Design the minimal fix based on evidence | Right fix, not assumed fix | Easy |

### Phase 1: Critical Fixes (After Investigation)

| # | Task | Why | Complexity |
|---|------|-----|-----------|
| 1.1 | Fix identity confusion (approach TBD from Phase 0 findings) | Agents should know who they are | TBD |
| 1.2 | Verify end-to-end: all 4 channels respond with correct persona | Validates the fix | Easy |

### Phase 2: Structural Cleanup (After Phase 1)

| # | Task | Why | Complexity |
|---|------|-----|-----------|
| 2.1 | Create `fleet/` directory with shared state structure | Foundation for cross-agent coordination | Easy |
| 2.2 | Curate skills per persona (depends on Q1/Q6 decisions) | Each agent loads only relevant tools | Medium |
| 2.3 | Remove dead startup script or update to reflect reality | Eliminate confusion about how fleet starts | Easy |
| 2.4 | Audit and clean symlinks in agent homes | Remove decorative directories that serve no purpose | Easy |

### Phase 3: Maturation (Ongoing)

| # | Task | Why | Complexity |
|---|------|-----|-----------|
| 3.1 | Implement fleet activity log (if Q3 moves beyond delegate-only) | Cross-agent visibility | Medium |
| 3.2 | Add memory scoping with agent tags | Prevent cross-contamination | Medium |
| 3.3 | Rename Discord channels to match agent names (if Q2 = yes) | Clarity | Easy |
| 3.4 | Build `discord_post` tool for Oracle (if needed) | Programmatic cross-channel communication | Medium |
| 3.5 | Add persona-specific system prompt testing | Regression prevention | Medium |

---

## 7. File Structure (Current)

```
~/.hermes/                          # Single home (symlinked from ~/.arch)
├── SOUL.md                         # Oracle's identity (also serves as base for all agents)
├── config.yaml                     # Gateway config with channel_prompts
├── .env                            # Shared secrets
├── auth.json                       # Shared auth
├── agent-templates/
│   └── SHARED_FOUNDATION.md        # Universal values
├── personas/
│   ├── arch.md                     # Arch persona overlay
│   ├── eureka.md                   # Eureka persona overlay
│   └── aurus.md                    # Aurus persona overlay
├── skills/                         # 19+ skills (all agents share these)
├── sessions/                       # Shared sessions
├── memories/                       # Shared memories
├── whatsapp/                       # WhatsApp bridge (Oracle only)
└── hermes-agent/                   # Hermes source code

~/.oracle/                          # Unused — decorative home directory
~/.eureka/                          # Unused — decorative home directory
~/.aurus/                           # Unused — decorative home directory
```

> Note: The proposed file structure under shared-home vs separate-homes depends on Q1 outcome and will be specified after that decision.

---

## 8. Key Decisions Taken

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Architecture pattern | Single gateway, channel-routed | Discord 1-token constraint; only viable option with 1 token |
| Reasoning block visibility | Leave as-is for now | Useful transparency, no identified harm in private single-user context |
| Investigation before fix | Phase 0 required for SOUL.md/identity issue | Follow our own philosophy: reproduce → isolate → root cause → fix |
| Startup script | Dead code, to be removed or updated | Not used in production, describes abandoned multi-process approach |

## 9. Decisions Pending

| Decision | Depends On | Blocked By |
|----------|-----------|-----------|
| Agent isolation level (Q1) | Your call | Nothing — ready for decision |
| SOUL.md fix approach | Phase 0 investigation results | Need to understand merge behavior first |
| Memory scoping (Q5) | Q1 outcome | Agent isolation decision |
| Skill curation (Q6) | Q1 + Q3 outcomes | Agent isolation + comms decisions |
| Channel renaming (Q2) | Your call | Nothing — low priority |
| Startup script fate (Q4) | Your call | Nothing — low priority |

---

## 10. Reasoning & Rationale

**Why Phase 0 before Phase 1?** Our own working philosophy is explicit: reproduce → isolate → root cause → fix. I violated this in DRAFT v1 by prescribing "agent-neutral SOUL.md" as the fix before understanding how channel_prompts actually merges with SOUL.md. The right approach is to investigate first, then fix based on evidence. This is not slower — it prevents building the wrong fix.

**Why defer reasoning block stripping?** The reasoning block leak is what caught the identity confusion. In our context (private server, one user), it's a feature masquerading as a bug. If a concrete harm is identified later (e.g., sensitive internal reasoning exposed to guests, or token waste from sending reasoning to Discord), we'll revisit.

**Why is Q1 the most important open question?** It determines the shape of everything else. Memory scoping, skill curation, process management, file structure — all of these are downstream of whether we formalize shared-home or invest in separation. The cons of shared-home (listed under Q1) are real and should be weighed against the simplicity benefits.
