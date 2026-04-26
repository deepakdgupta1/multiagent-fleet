# Multi-Agent Fleet — Solution Design & Architecture

> Status: DRAFT v3 (April 27, 2026)
> Repository: https://github.com/deepakdgupta1/multiagent-fleet
> Owner: Deepak (deep.g)

---

## 1. Problem Statement

We have a 4-agent fleet (Oracle, Arch, Eureka, Aurus) running on Discord + WhatsApp. It was assembled quickly over April 13-19 and has not been matured. Phase 0 investigation (April 26-27) revealed that **personas have never loaded correctly** — all agents have been running as Oracle for the fleet's entire lifetime.

### Symptoms

1. **Identity confusion** — Arch had to reason through whether it was Oracle or Arch ("let me check the system prompt again... file://personas/arch.md"). The persona swap should be seamless. Root cause identified: `file://` URIs are never resolved, the literal string is passed to the LLM (see [phase0-system-prompt-assembly.md](phase0-system-prompt-assembly.md)).
2. **Reasoning blocks visible in Discord** — Arch exposed its raw reasoning block ("💭 Reasoning: ...") to the user. This is what surfaced symptom #1. **Status: Intentionally kept as-is.** The reasoning blocks provide useful transparency in a private single-user context. No harmful consequences identified. Revisit if/when the fleet becomes multi-user or public-facing.
3. **No isolation** — All 4 personas share one Hermes home (`~/.hermes` → `~/.arch`). One gateway process serves all channels. There is no tool, memory, or state isolation between agents.
4. **Missing cross-agent visibility** — Oracle cannot see what happened in Arch/Eureka/Aurus conversations without manual `session_search`. The proposed `fleet/activity.md` shared log was never implemented.
5. **3 of 4 agent homes are shells** — Oracle, Eureka, and Aurus have home directories but they are decorative — the single gateway process uses Arch's config, skills, sessions, and memories. Plan: wire them as proper CLI entry points (see Section 6, Phase 2).
6. **No inter-agent communication** — Oracle cannot programmatically post in other channels. The `discord_post` tool was discussed but never built.
7. **Personas never loaded** (discovered April 26) — For the fleet's entire lifetime, all agents received Oracle's SOUL.md as their identity. The channel_prompts `file://` references were passed as literal strings. See [phase0-system-prompt-assembly.md](phase0-system-prompt-assembly.md).

---

## 2. Current Architecture (As-Is)

```
┌─────────────────────────────────────────────────────────┐
│                   Single Hermes Process                  │
│            (systemd: hermes-gateway.service)             │
│                  HERMES_HOME=~/.hermes                   │
│                   (symlink → ~/.arch)                    │
│                                                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────┐ │
│  │  Oracle   │  │   Arch   │  │  Eureka  │  │ Aurus  │ │
│  │ #assistant│  │  #code   │  │ #content │  │ #money │ │
│  │ +WhatsApp │  │          │  │          │  │        │ │
│  └──────────┘  └──────────┘  └──────────┘  └────────┘ │
│         Persona swap via channel_prompts in config.yaml │
│         file://personas/{arch,eureka,aurus}.md          │
│         ** BROKEN: file:// never resolved **            │
│         ** ALL agents receive Oracle's SOUL.md **       │
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
| Gateway process | 1 (systemd service on Ubuntu Desktop) |
| HERMES_HOME | `~/.hermes` → `~/.arch` (symlink; rename to `~/.hermes` real dir planned) |
| Discord bot tokens | 1 (shared) |
| WhatsApp bridge | 1 (Node.js/Baileys, Oracle only) |
| Persona mechanism | `channel_prompts` with `file://` references — **BROKEN (never resolved)** |
| Persona loading | All agents receive Oracle's SOUL.md. Persona files exist but are never read. |
| Agent homes | `~/.oracle` (has state.db, cron, WhatsApp session, skills, SOUL.md), `~/.arch` (active gateway home), `~/.aurus`/`~/.eureka` (SOUL.md + config.yaml only) |
| Startup script | `~/.arch/bin/start-agents.sh` (dead code, not used by systemd) |
| Config | `~/.arch/config.yaml` (shared, single file) |
| Planned host | Mac Mini (macOS, 24/7 operation, launchd) — see [MIGRATION-GUIDE.md](MIGRATION-GUIDE.md) |

### Discord Channel Map

| Channel | ID | Persona File | Actually Loaded |
|---------|----:|-------------|----------------|
| #assistant (Oracle) | 1465562736451911838 | (default SOUL.md = Oracle) | Oracle (correct) |
| #code (Arch) | 1493461055618285670 | `file://personas/arch.md` | Oracle + literal string "file://personas/arch.md" |
| #content (Eureka) | 1493461225567420416 | `file://personas/eureka.md` | Oracle + literal string "file://personas/eureka.md" |
| #money (Aurus) | 1493461326662467624 | `file://personas/aurus.md` | Oracle + literal string "file://personas/aurus.md" |

### Agent Home Contents (Current)

| | Oracle | Arch (active gateway home) | Eureka | Aurus |
|--|--------|------|--------|-------|
| SOUL.md | Yes | Yes (base, always loaded) | Yes | No |
| config.yaml | Yes (unused by gateway) | Yes (active) | Yes (unused) | Yes (unused) |
| skills/ | 27 dirs (symlinked from ~/.arch) | 26 dirs (active) | empty | empty |
| sessions/ | Yes | Yes | empty | empty |
| memories/ | Yes | Yes | empty | empty |
| state.db | Yes (own) | Yes | no | no |
| SHARED_FOUNDATION.md | symlink | source | symlink | symlink |
| .env | symlink→~/.arch | source | symlink→~/.arch | symlink→~/.arch |

---

## 3. Root Cause Analysis

### 3A. Identity Confusion & Persona Loading — STATUS: Root Cause Identified

**Symptom:** All agents run as Oracle. Persona files are never loaded.

**Root causes (3 compounding bugs, all in gateway code):**

1. **`file://` URIs never resolved** — `resolve_channel_prompt()` in `base.py` does `str(prompt).strip()` with no URI handling. The literal string `file://personas/arch.md` is passed to the LLM, not the file contents.
2. **APPEND merge semantics** — `channel_prompts` are APPENDED after SOUL.md at API call time, not replacing it. Even if file resolution worked, both Oracle AND Arch identities would coexist in the system prompt.
3. **SOUL.md is global** — `load_soul_md()` reads from `$HERMES_HOME/SOUL.md` for ALL channels. There is no per-channel SOUL.md mechanism.

**Investigation:** Complete. Full technical trace in [phase0-system-prompt-assembly.md](phase0-system-prompt-assembly.md).

**Fix specification:** Pending. Three fixes needed:
- Implement `file://` resolution in `resolve_channel_prompt()`
- Change merge semantics from APPEND to REPLACE-SOUL when channel_prompt is present
- Support per-channel SOUL.md loading

**No fix will be applied until specification is complete and reviewed.**

### 3B. Reasoning Block Visibility — STATUS: Deferred

The reasoning blocks are visible in Discord. This is what surfaced the identity confusion.

**Assessment:** In our current context (private Discord server, single user), this is actually useful transparency. No harmful consequences identified. The reasoning blocks helped diagnose the real problem. Leaving as-is until a concrete harm is identified.

### 3C. No Agent Isolation — STATUS: Trial Planned

Everything runs from `~/.arch` (via `~/.hermes` symlink). Oracle, Eureka, and Aurus have home directories but they're not wired to the gateway.

**Decision:** Trial week with shared gateway home + per-agent CLI homes. Personas have never worked, so we observe behavior with correct personas first before architecting isolation. See [SHARING-ISOLATION-ANALYSIS.md](SHARING-ISOLATION-ANALYSIS.md) for trial framework (hypotheses H1-H5).

### 3D. Cross-Agent Blindness — STATUS: Known Gap

Oracle has no structured way to know what happened in specialist channels. `session_search` works but requires Oracle to actively search and is not automatic.

**Decision:** Start with delegate-only (Option C). Revisit when cross-agent visibility becomes a real pain point.

---

## 4. Open Questions

### Q1: Agent Isolation Level — STATUS: Trial Decided

**Decision: Proceed with shared gateway home for trial week.** Rationale:
- Personas have never worked correctly — premature to architect isolation before observing correct persona behavior
- Trial tests 5 hypotheses (H1-H5) to validate or falsify shared home
- **Likely end state is isolation** based on three arguments:
  1. **Token efficiency**: Lean agents with relevant-only skills = better reasoning + lower token cost. Skills loading is the biggest token waste source.
  2. **Fleet as infrastructure**: The fleet is central infrastructure for ALL future projects. Getting isolation right has exponential ROI.
  3. **Quality**: No cross-contamination between domains. Arch shouldn't see Aurus's financial context. Aurus shouldn't load GitHub skills.
- But: we don't have evidence yet because agents have never been distinct. The trial provides that evidence.

See [SHARING-ISOLATION-ANALYSIS.md](SHARING-ISOLATION-ANALYSIS.md) for the complete trial framework.

### Q2: Channel Naming — STATUS: Low Priority

Current channels (#assistant, #code, #content, #money) should be renamed to (#oracle, #arch, #eureka, #aurus). Discord channel renames preserve IDs — no config breakage.

### Q3: Inter-Agent Communication — STATUS: Delegate-Only (Option C)

No direct agent-to-agent communication. Oracle delegates via `delegate_task` and synthesizes results. Revisit when cross-agent visibility becomes a real pain point.

### Q4: Startup Script — STATUS: Dead Code, To Remove

`~/.arch/bin/start-agents.sh` describes a multi-process architecture that was never used. Systemd runs a single gateway directly. Remove during Phase 2.

### Q5: Memory Scoping — STATUS: Depends on Trial Results

If shared home is validated: shared memories with agent-scoped tags.
If isolation is needed: separate stores with explicit sharing for Oracle's cross-domain visibility.

### Q6: Skill Curation — STATUS: Depends on Trial Results

If shared home is validated: skill filtering within shared skills/ directory.
If isolation is needed: curated skill subsets per agent home. This is the biggest token efficiency win under isolation.

---

## 5. What We Know vs. What We Assume

| Claim | Status | Confidence |
|-------|--------|-----------:|
| Discord enforces 1-token-1-connection | Verified (April 13-19 sessions) | High |
| Single gateway running via systemd | Verified (process list, systemctl) | High |
| channel_prompts uses `file://` references | Verified (config.yaml) | High |
| `file://` references are NEVER resolved | Verified (code trace in phase0 doc) | High |
| SOUL.md says "I am Oracle" | Verified (file contents) | High |
| Personas have NEVER loaded — all agents run as Oracle | Verified (full assembly trace) | High |
| The merge behavior (APPEND + global SOUL.md) causes identity confusion | Verified (3 compounding bugs traced) | High |
| Agent homes (~/.oracle, ~/.eureka, ~/.aurus) are currently unused by gateway | Verified (gateway uses ~/.hermes only) | High |
| Agent homes CAN serve as CLI entry points | Verified (HERMES_HOME=~/.agent hermes) | High |
| Shared home is the right architecture for trial week | Assumption — testing via H1-H5 | Medium |
| Isolation is the likely end state | Informed judgment — token efficiency + quality arguments | Medium-High |
| Reasoning block visibility is harmless | Assessed for current context only | Medium |
| Fleet will move to Mac Mini as 24/7 host | Planned, not yet executed | High |

---

## 6. Implementation Plan

### Phase 0: Investigation — COMPLETE

| # | Task | Status | Finding |
|---|------|--------|---------|
| 0.1 | Trace channel_prompts merge logic in `gateway/platforms/base.py` | Done | `file://` URIs passed as literal strings |
| 0.2 | Trace ephemeral prompt assembly in `gateway/run.py` | Done | APPEND semantics, not REPLACE |
| 0.3 | Trace SOUL.md loading in `run_agent.py` | Done | Global single file, no per-channel support |
| 0.4 | Write up root cause and minimal fix design | Done | 3 compounding bugs, see [phase0-system-prompt-assembly.md](phase0-system-prompt-assembly.md) |

### Phase 1: Persona Loading Fix — NEXT

| # | Task | Why | Complexity |
|---|------|-----|-----------:|
| 1.1 | Write specification for 3-bug fix (file:// resolution, REPLACE semantics, per-channel SOUL.md) | Clear implementation target for Arch | Medium |
| 1.2 | Arch implements the fix in gateway code | Agents should know who they are | Medium |
| 1.3 | End-to-end verification: all 4 channels respond with correct persona | Validates the fix works | Easy |

### Phase 2: Directory Restructuring — After Phase 1

| # | Task | Why | Complexity |
|---|------|-----|-----------:|
| 2.1 | Rename `~/.arch` → `~/.hermes` (real directory) | Eliminate confusing symlink | Easy |
| 2.2 | Create `~/.arch` as backward-compat symlink → `~/.hermes` | Don't break any hardcoded references | Easy |
| 2.3 | Wire `~/.oracle`, `~/.eureka`, `~/.aurus` as CLI entry points (own SOUL.md, config.yaml) | Per-agent CLI access | Medium |
| 2.4 | Fix relative symlinks for .env, auth.json, SHARED_FOUNDATION.md in agent homes | Portability | Easy |
| 2.5 | Remove dead startup script (`~/.arch/bin/start-agents.sh`) | Eliminate confusion | Easy |
| 2.6 | Restart gateway, verify all channels + CLI access work | Validates restructuring | Easy |

### Phase 3: Trial Week — After Phase 2

Run the fleet with correct personas for 1 week. Collect data for hypotheses H1-H5 per [SHARING-ISOLATION-ANALYSIS.md](SHARING-ISOLATION-ANALYSIS.md).

| # | Task | Why | Complexity |
|---|------|-----|-----------:|
| 3.1 | Begin trial: observe all 4 channels daily | Collect H1-H5 data | Easy |
| 3.2 | Log identity coherence, cross-contamination, quality per interaction | Evidence for Q1 decision | Easy |
| 3.3 | Instrument token usage per channel (if feasible) | H3 data | Medium |
| 3.4 | Test CLI access per agent home | H5 data | Easy |

### Phase 4: Architecture Decision — After Trial

Based on trial results (see decision matrix in [SHARING-ISOLATION-ANALYSIS.md](SHARING-ISOLATION-ANALYSIS.md)):

| Trial Outcome | Action |
|---------------|--------|
| H1 or H2 falsified | Isolation becomes next priority |
| H3 falsified only | Lean toward isolation, or skill curation within shared home |
| H4 falsified only | Operational fix, not architectural change |
| H5 falsified | Fix CLI home wiring |
| None falsified | Shared home validated, isolation deferred |

### Phase 5: Mac Mini Migration — After Architecture Stabilizes

Move fleet from Ubuntu Desktop to Mac Mini (24/7 host). See [MIGRATION-GUIDE.md](MIGRATION-GUIDE.md).

### Phase 6: Maturation — Ongoing

| # | Task | Why | Complexity |
|---|------|-----|-----------:|
| 6.1 | Curate skills per persona (based on Q6 decision) | Each agent loads only relevant tools | Medium |
| 6.2 | Add memory scoping with agent tags (based on Q5 decision) | Prevent cross-contamination | Medium |
| 6.3 | Rename Discord channels to match agent names (Q2) | Clarity | Easy |
| 6.4 | Implement fleet activity log or `discord_post` (if Q3 evolves) | Cross-agent visibility | Medium |
| 6.5 | Add persona regression tests | Prevent future breakage | Medium |

---

## 7. Target File Structure (Post-Restructuring)

After Phase 2 directory restructuring:

```
~/.hermes/                          # REAL directory (renamed from ~/.arch)
├── SOUL.md                         # Oracle's identity (loaded for #oracle channel)
├── config.yaml                     # Gateway config with channel_prompts
├── .env                            # Shared secrets [REDACTED]
├── auth.json                       # Shared auth [REDACTED]
├── agent-templates/
│   └── SHARED_FOUNDATION.md        # Universal values (symlinked to agent homes)
├── personas/
│   ├── arch.md                     # Arch persona (loaded for #arch channel after fix)
│   ├── eureka.md                   # Eureka persona (loaded for #eureka channel after fix)
│   └── aurus.md                    # Aurus persona (loaded for #aurus channel after fix)
├── skills/                         # 19+ skills (shared during trial; per-agent post-trial?)
├── sessions/                       # Shared sessions (during trial)
├── memories/                       # Shared memories (during trial)
├── whatsapp/                       # WhatsApp bridge (Oracle only)
└── hermes-agent/                   # Hermes source code

~/.arch                             # Backward-compat symlink → ~/.hermes

~/.oracle/                          # CLI entry point for Oracle
├── SOUL.md                         # Oracle's identity
├── config.yaml                     # Oracle-specific config
├── .env                            # → ../.hermes/.env (relative symlink)
├── auth.json                       # → ../.hermes/auth.json (relative symlink)
├── SHARED_FOUNDATION.md            # → ../.hermes/agent-templates/SHARED_FOUNDATION.md
├── skills/                         # Oracle's curated skills (post-trial)
└── (state.db, cron/, whatsapp session data — currently exists)

~/.eureka/                          # CLI entry point for Eureka
├── SOUL.md                         # Eureka's identity
├── config.yaml                     # Eureka-specific config
├── .env                            # → ../.hermes/.env (relative symlink)
├── auth.json                       # → ../.hermes/auth.json (relative symlink)
├── SHARED_FOUNDATION.md            # → ../.hermes/agent-templates/SHARED_FOUNDATION.md
└── skills/                         # Eureka's curated skills (post-trial)

~/.aurus/                           # CLI entry point for Aurus
├── SOUL.md                         # Aurus's identity
├── config.yaml                     # Aurus-specific config
├── .env                            # → ../.hermes/.env (relative symlink)
├── auth.json                       # → ../.hermes/auth.json (relative symlink)
├── SHARED_FOUNDATION.md            # → ../.hermes/agent-templates/SHARED_FOUNDATION.md
└── skills/                         # Aurus's curated skills (post-trial)
```

> Note: During trial week, agent homes are CLI-only access points. The gateway runs from `~/.hermes`. If the trial leads to isolation (Option B), agent homes become the primary operational homes for each agent with the gateway routing to them.

---

## 8. Key Decisions Taken

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Architecture pattern | Single gateway, channel-routed | Discord 1-token constraint; only viable option with 1 token |
| Reasoning block visibility | Leave as-is for now | Useful transparency, no identified harm in private single-user context |
| Investigation before fix | Phase 0 completed | Followed philosophy: reproduce → isolate → root cause → fix. Found 3 compounding bugs. |
| Q1 trial approach | Shared gateway home + per-agent CLI homes | Personas never worked — observe correct behavior before architecting isolation |
| Q1 likely end state | Isolation (Option B) | Lean agents, token efficiency, fleet as infrastructure. But evidence first. |
| Directory structure | Rename `~/.arch` → `~/.hermes`, wire agent homes | Eliminate confusing symlink, enable CLI access per agent |
| Cross-platform portability | Required (Ubuntu → macOS) | Mac Mini will be 24/7 host. All paths relative. |
| Q3 inter-agent comms | Delegate-only (Option C) | Start simple, evolve when pain points emerge |
| Q4 startup script | Remove (dead code) | Not used in production, describes abandoned multi-process approach |
| Persona fix scope | Fix all 3 bugs (file:// resolution, REPLACE semantics, per-channel SOUL.md) | All three must be fixed for personas to work correctly |

## 9. Decisions Pending

| Decision | Depends On | Status |
|----------|-----------|--------|
| Full isolation (Option B vs staying shared) | Trial week results (H1-H5) | Trial after Phase 1-2 |
| Memory scoping (Q5) | Isolation decision | Blocked by trial |
| Skill curation (Q6) | Isolation decision | Blocked by trial |
| Channel renaming (Q2) | Nothing — ready when convenient | Low priority |
| Mac Mini migration timing | Architecture stability | After trial decision |

---

## 10. Reasoning & Rationale

**Why Phase 0 before Phase 1?** Our working philosophy is explicit: reproduce → isolate → root cause → fix. The investigation revealed that the problem was not what we initially assumed (merge behavior alone) but three compounding bugs. The fix must address all three. Designing the fix before the investigation would have missed bugs 1 and 3.

**Why trial before isolation?** The fleet has operated with zero persona differentiation for its entire lifetime. We have no data on how well shared home works when agents actually have distinct identities. Architecting isolation without this data risks solving the wrong problem. That said, the token efficiency and quality arguments for isolation are strong — the trial is about getting evidence, not about being agnostic on the outcome.

**Why is token efficiency the strongest argument for isolation?** The skills section alone is thousands of tokens loaded into every agent's context. Aurus loads GitHub workflow, code review, and security reviewer skills that are completely irrelevant to financial learning. This wastes tokens and pollutes reasoning. Under isolation, each agent loads only relevant skills. Lean context = better reasoning + lower cost. Since the fleet is infrastructure for all future projects, this compounds exponentially.

**Why defer reasoning block stripping?** The reasoning block leak is what caught the identity confusion. In our context (private server, one user), it's a feature masquerading as a bug. If a concrete harm is identified later, we'll revisit.

**Why keep agent homes instead of removing them?** They enable CLI access per agent (`HERMES_HOME=~/.aurus hermes`). This gives us a second access mode beyond Discord, useful for debugging and direct agent interaction. The cost is minimal (a few files per home) and they're already mostly set up.
