# Fleet Sharing & Isolation Analysis

> Status: DRAFT v1 (April 26, 2026)
> Context: Trial week framework for testing shared-home architecture with correct persona loading

---

## 1. Trial Week: Shared Home with Working Personas

### 1A. SHARED — Static (on disk)

| What | Details | Impact of sharing |
|------|---------|-------------------|
| HERMES_HOME | One directory (`~/.hermes` → `~/.arch`) for all agents | Any file corruption affects all agents equally |
| config.yaml | Single config file. Discord routing, tool permissions, model settings all in one place | Config change for one agent risks breaking others. No per-agent model tuning. |
| .env | API keys, bot tokens, all secrets | Neutral — all agents need the same API access. Efficient. |
| auth.json | Authentication state | Neutral. |
| hermes-agent/ | Source code | Neutral — one codebase to maintain, one to update. |
| skills/ | All 19+ skills in one directory | **Negative.** Every agent loads every skill. Aurus loads GitHub workflow, code review, security reviewer. Arch loads nothing finance-specific because none exist. Token waste + reasoning pollution. |
| SHARED_FOUNDATION.md | Universal values document | **Positive.** All agents share the same behavioral contract. This is correct sharing. |
| state.db | Single SQLite database for all sessions, memories, checkpoints | **Negative under load.** Write contention possible. One corrupted DB = all agents lose state. |
| personas/ | All 4 persona files in one directory | Neutral — they're selectively loaded per channel. |

### 1B. SHARED — Dynamic (at runtime)

| What | Details | Impact of sharing |
|------|---------|-------------------|
| Gateway process | One process, one crash domain | **Negative.** Gateway restart kills all 4 agents mid-conversation. Can't upgrade Arch without taking down Oracle. |
| Discord connection | One bot token, one websocket | Constraint, not choice. Discord enforces this. |
| System prompt layers 2-10 | Tool guidance, memory injection, skills prompt, context files, platform hints | **Mixed.** All agents get the same tool instructions regardless of relevance. Arch gets memory injection tips it doesn't need differently than Aurus does. |
| Memory store | All agents read/write the same memory files | **Negative.** If Arch saves "debugging Docker networking issue" and Aurus is later asked about portfolio allocation, that debugging memory might surface in Aurus's context. Cross-domain noise. Also: no agent has private thoughts. If you tell Arch something in confidence, Oracle can see it. |
| Session history | All conversations in one session store, fully searchable by any agent | **Positive for Oracle** (can see the whole board). **Negative for specialist independence** (agents see context from domains they shouldn't care about). |
| Skills prompt | Every skill's system prompt fragment loaded into context | **Negative.** The skills section alone is thousands of tokens. Most are irrelevant to most agents. This is the biggest source of token waste in shared home. |

### 1C. ISOLATED — Static

| What | Details | Impact of isolation |
|------|---------|---------------------|
| Persona identity | Each channel loads its own persona file | **Positive.** Agent knows exactly who it is. No reconciliation needed. |
| Discord channel mapping | Each agent tied to one channel | **Positive.** Clean routing, no ambiguity. |

### 1D. ISOLATED — Dynamic

| What | Details | Impact of isolation |
|------|---------|---------------------|
| Active conversation thread | Each channel has its own conversation | **Positive.** No cross-talk during active sessions. |
| Ephemeral system prompt | Assembled per-channel | **Positive.** Channel-specific context only. |

---

## 2. If We Move to Isolation (Option B: Separate Homes)

### 2A. What becomes additionally isolated

| What | Change from shared | Impact |
|------|-------------------|--------|
| config.yaml | Each agent gets its own | **Positive.** Per-agent model selection, tool permissions, feature flags. Arch can use a code-focused model. Aurus can be locked to terminal-only tools. |
| skills/ | Each agent gets a curated subset | **Positive.** This is the biggest win. Arch loads GitHub, code-review, devops, test-engineer. Aurus loads web-search, terminal, maybe a future finance skill. Eureka loads research, web. Oracle loads strategic-review, delegation, web. Lean context = better reasoning + lower token cost. |
| sessions/ | Separate conversation stores | **Mixed.** Clean domain context, but Oracle can't see Arch's sessions directly — needs explicit fetching. Coordination overhead. |
| memories/ | Separate memory files | **Positive for quality.** No cross-contamination. **Negative for Oracle's visibility.** Oracle needs a mechanism to access specialist memories for synthesis. |
| state.db | Separate SQLite per agent | **Positive.** No write contention, no single-point-of-failure for state. |
| SOUL.md | Per-agent identity file | **Positive.** No identity confusion possible. Agent-neutral base not needed. |

### 2B. What stays shared even under isolation

| What | Why it stays shared | Impact |
|------|---------------------|--------|
| .env / secrets | All agents need same API access | Neutral. Symlinks or shared file. |
| hermes-agent/ source code | One codebase, one version | Neutral. Don't duplicate. |
| Discord bot token | Still one gateway process | Constraint remains. |
| Gateway process | Option B is separate homes, single process | **Same crash domain.** This doesn't change under Option B. Only Option C (multi-process) fixes this. |
| SHARED_FOUNDATION.md | Values should be universal | **Positive.** Symlinked to all homes. |

---

## 3. Summary: Sharing vs. Isolation Verdict

### Sharing HELPS (keep shared):
- Secrets and auth (efficiency)
- Source code (maintenance)
- Foundation values (consistency)
- Oracle's access to specialist context (seeing the whole board)

### Sharing HURTS (isolate these):
- Skills loading (token waste + reasoning pollution — biggest impact)
- Memory store (cross-domain contamination)
- Config (per-agent tuning)
- Session history (domain noise)

### Sharing is a constraint, not a choice:
- Gateway process (only Option C fixes this)
- Discord connection (only multi-token fixes this)

---

## 4. Trial Week Hypotheses

The trial tests whether the "sharing hurts" items cause measurable problems with correct personas loaded.

### Hypotheses

**H1: Identity Coherence** — Each agent correctly assumes its persona identity (no confusion episodes).

**H2: No Cross-Contamination** — One agent's context does not meaningfully degrade another agent's domain-specific output quality.

**H3: Token Efficiency** — Shared skill loading does not cause token waste exceeding 20% of total context per agent interaction.

**H4: Availability** — Shared state (single process, single DB) does not cause availability issues for any agent more than once per week.

### Falsification Criteria

| Hypothesis | Falsified by |
|-----------|-------------|
| H1 (identity coherence) | Any agent expressing uncertainty about its identity, or responding with another agent's domain behavior |
| H2 (no cross-contamination) | Aurus referencing debugging context, Arch giving financial opinions, any agent using skills/knowledge clearly from another agent's domain inappropriately |
| H3 (token efficiency) | Measured: skills + system prompt tokens that are irrelevant to the active agent exceed 20% of total context window used per interaction |
| H4 (availability) | More than 1 gateway incident per week caused by one agent's workload affecting another's |

### Decision Matrix

| Outcome | Action |
|---------|--------|
| H1 or H2 falsified | Isolation becomes next priority (cross-contamination is the ceiling) |
| H3 falsified only | Lean toward isolation, but could be solved with skill curation within shared home |
| H4 falsified only | Operational fix, not architectural change |
| None falsified | Shared home validated, isolation deferred |

### Data Collection

Manual log per interaction:

```
| Date | Channel | Agent | Identity OK? | Cross-bleed? | Quality 1-5 | Notes |
```

Programmatic (Arch instruments gateway):
- System prompt token count per channel
- Skill relevance ratio (relevant vs total loaded)
- Memory entry origin per agent interaction

### Double-Loop Review

At trial end, beyond the data:
- "What did we not measure that mattered?"
- "Did shared-home subtly shape agent behavior in ways we didn't notice because we weren't looking?"
