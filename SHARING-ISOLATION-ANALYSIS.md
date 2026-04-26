# Fleet Sharing & Isolation Analysis

> Status: DRAFT v2 (April 27, 2026)
> Context: Trial week framework — shared gateway home with per-agent CLI homes
> Critical qualifier: Personas have NEVER loaded correctly (Phase 0 finding). The trial tests shared-home behavior ONLY after persona loading is fixed. Any conclusions drawn before personas load correctly are premature and invalid.

---

## 0. Why This Trial Matters Differently Now

Phase 0 revealed that persona/identity loading has never worked correctly. This changes the nature of the trial fundamentally:

- **We are NOT testing "does shared home work?" in the abstract.** We already know shared home has been the only mode that has ever existed.
- **We ARE testing "does shared home work once personas actually load?"** — a question we cannot answer yet because the baseline (correct persona loading) has never been achieved.
- **Premature architecture decisions are a risk.** Moving to full isolation before personas have ever loaded correctly would be optimizing for a problem we haven't actually observed under correct conditions.

The trial is the first time any of these agents will operate with their intended identities active. That is the real experiment.

---

## 1. Trial Week Framework: Shared Gateway + Per-Agent CLI Homes

The trial uses a hybrid architecture:
- **Shared gateway home** (`~/.hermes`) — single gateway process, single state store, single Discord connection
- **Per-agent CLI homes** (`~/.oracle`, `~/.eureka`, `~/.aurus`) — each with its own SOUL.md and config.yaml for direct CLI access

Note: `~/.hermes` is the real directory (post-restructuring rename). `~/.arch` will be maintained as a backward-compat symlink pointing to `~/.hermes`.

### 1A. SHARED — Static (on disk)

| What | Details | Impact of sharing |
|------|---------|-------------------|
| HERMES_HOME | One directory (`~/.hermes`, real dir; `~/.arch` as backward-compat symlink) for gateway | Any file corruption affects all agents equally via the gateway |
| config.yaml (gateway) | Single gateway config. Discord routing, tool permissions, model settings all in one place | Config change for one agent risks breaking others. No per-agent model tuning at the gateway level. |
| .env | API keys, bot tokens, all secrets ([REDACTED]) | Neutral — all agents need the same API access. Efficient. |
| auth.json | Authentication state | Neutral. |
| hermes-agent/ | Source code | Neutral — one codebase to maintain, one to update. |
| skills/ | All 19+ skills in one directory | **Negative.** Every agent loads every skill via gateway. Aurus loads GitHub workflow, code review, security reviewer. Hermes loads nothing finance-specific because none exists. Token waste + reasoning pollution. |
| SHARED_FOUNDATION.md | Universal values document | **Positive.** All agents share the same behavioral contract. This is correct sharing. |
| state.db | Single SQLite database for all sessions, memories, checkpoints | **Negative under load.** Write contention possible. One corrupted DB = all agents lose state. |
| personas/ | All 4 persona files in one directory | Neutral — they're selectively loaded per channel. |

### 1B. SHARED — Dynamic (at runtime)

| What | Details | Impact of sharing |
|------|---------|-------------------|
| Gateway process | One process, one crash domain | **Negative.** Gateway restart kills all 4 agents mid-conversation. Can't upgrade one agent without taking down all. |
| Discord connection | One bot token, one websocket | Constraint, not choice. Discord enforces this. |
| System prompt layers 2-10 | Tool guidance, memory injection, skills prompt, context files, platform hints | **Mixed.** All agents get the same tool instructions regardless of relevance. Hermes gets memory injection tips it doesn't need differently than Aurus does. |
| Memory store | All agents read/write the same memory files | **Negative.** If Hermes saves "debugging Docker networking issue" and Aurus is later asked about portfolio allocation, that debugging memory might surface in Aurus's context. Cross-domain noise. Also: no agent has private thoughts. If you tell Hermes something in confidence, Oracle can see it. |
| Session history | All conversations in one session store, fully searchable by any agent | **Positive for Oracle** (can see the whole board). **Negative for specialist independence** (agents see context from domains they shouldn't care about). |
| Skills prompt | Every skill's system prompt fragment loaded into context | **Negative.** The skills section alone is thousands of tokens. Most are irrelevant to most agents. This is the biggest source of token waste in shared home. |

### 1C. ISOLATED — Static

| What | Details | Impact of isolation |
|------|---------|---------------------|
| Agent CLI homes | `~/.oracle`, `~/.eureka`, `~/.aurus` each with own SOUL.md and config.yaml | **Positive.** Each agent is a proper CLI entry point with its own identity and configuration. Per-agent model selection, tool permissions, feature flags. |
| Persona identity | Each channel loads its own persona file; each CLI home has its own SOUL.md | **Positive.** Agent knows exactly who it is. No reconciliation needed. Dual-mode identity (Discord + CLI) reinforced. |
| Discord channel mapping | Each agent tied to one channel | **Positive.** Clean routing, no ambiguity. |

### 1D. ISOLATED — Dynamic

| What | Details | Impact of isolation |
|------|---------|---------------------|
| Active conversation thread | Each channel has its own conversation | **Positive.** No cross-talk during active sessions. |
| Ephemeral system prompt | Assembled per-channel (gateway) or per-invocation (CLI) | **Positive.** Channel/agent-specific context only. |
| CLI invocation | Each agent home provides isolated CLI entry | **Positive.** Per-agent context window, no gateway dependency. Tests whether agent identities hold outside Discord. |

---

## 2. If We Move to Isolation (Option B: Separate Homes)

### 2A. What becomes additionally isolated

| What | Change from trial hybrid | Impact |
|------|--------------------------|--------|
| config.yaml | Each agent gets its own (not just CLI homes but gateway-level) | **Positive.** Full per-agent model selection, tool permissions, feature flags. Hermes can use a code-focused model. Aurus can be locked to terminal-only tools. |
| skills/ | Each agent gets a curated subset | **Positive.** This is the biggest win. Hermes loads GitHub, code-review, devops, test-engineer. Aurus loads web-search, terminal, maybe a future finance skill. Eureka loads research, web. Oracle loads strategic-review, delegation, web. Lean context = better reasoning + lower token cost. |
| sessions/ | Separate conversation stores | **Mixed.** Clean domain context, but Oracle can't see other agents' sessions directly — needs explicit fetching. Coordination overhead. |
| memories/ | Separate memory files | **Positive for quality.** No cross-contamination. **Negative for Oracle's visibility.** Oracle needs a mechanism to access specialist memories for synthesis. |
| state.db | Separate SQLite per agent | **Positive.** No write contention, no single-point-of-failure for state. |
| SOUL.md | Per-agent identity file (already in place for CLI homes) | **Positive.** No identity confusion possible. Agent-neutral base not needed. |

### 2B. What stays shared even under isolation

| What | Why it stays shared | Impact |
|------|---------------------|--------|
| .env / secrets ([REDACTED]) | All agents need same API access | Neutral. Symlinks or shared file. |
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

### Likely End State: Isolation

A strong case exists for isolation as the likely end state:
- **Lean agents with relevant-only skills = better reasoning + lower token cost.** This is not marginal — it is the single largest architectural lever on agent quality.
- **The fleet is central infrastructure for ALL future projects**, not just one repo or one workflow. That scope justifies the upfront investment in proper isolation.
- **Per-agent CLI homes are already in place** during the trial. The scaffolding for isolation partially exists.
- **But: trial first.** Agents have never worked correctly (personas never loaded). It would be premature to architect full isolation without evidence from correctly-loaded agents operating under shared-home conditions. The trial exists to validate the problem before committing to the solution.

---

## 4. Trial Week Hypotheses

The trial tests whether the "sharing hurts" items cause measurable problems — **with correct personas loaded for the first time**.

### Hypotheses

**H1: Identity Coherence** — Each agent correctly assumes its persona identity (no confusion episodes), on both Discord (via gateway) and CLI (via agent home).

**H2: No Cross-Contamination** — One agent's context does not meaningfully degrade another agent's domain-specific output quality.

**H3: Token Efficiency** — Shared skill loading does not cause token waste exceeding 20% of total context per agent interaction.

**H4: Availability** — Shared state (single process, single DB) does not cause availability issues for any agent more than once per week.

**H5: CLI Access** — Per-agent CLI homes (`~/.oracle`, `~/.eureka`, `~/.aurus`) load SOUL.md correctly and produce identity-coherent responses independent of the gateway.

### Falsification Criteria

| Hypothesis | Falsified by |
|-----------|-------------|
| H1 (identity coherence) | Any agent expressing uncertainty about its identity, or responding with another agent's domain behavior — on Discord or CLI |
| H2 (no cross-contamination) | Aurus referencing debugging context, Hermes giving financial opinions, any agent using skills/knowledge clearly from another agent's domain inappropriately |
| H3 (token efficiency) | Measured: skills + system prompt tokens that are irrelevant to the active agent exceed 20% of total context window used per interaction |
| H4 (availability) | More than 1 gateway incident per week caused by one agent's workload affecting another's |
| H5 (CLI access) | Any agent home failing to load its SOUL.md, or producing identity-incoherent output when invoked via CLI |

### Decision Matrix

| Outcome | Action |
|---------|--------|
| H1 or H2 falsified | Isolation becomes next priority (cross-contamination is the ceiling) |
| H3 falsified only | Lean toward isolation, but could be solved with skill curation within shared home |
| H4 falsified only | Operational fix, not architectural change |
| H5 falsified | Fix CLI home configuration before drawing architectural conclusions |
| None falsified | Shared home validated, isolation deferred |

---

## 5. Data Collection

### Discord Access (via gateway)

Manual log per interaction:

```
| Date | Channel | Agent | Identity OK? | Cross-bleed? | Quality 1-5 | Notes |
```

Programmatic (Hermes instruments gateway):
- System prompt token count per channel
- Skill relevance ratio (relevant vs total loaded)
- Memory entry origin per agent interaction

### CLI Access (via agent homes)

Manual log per invocation:

```
| Date | Agent Home | SOUL.md Loaded? | Identity OK? | Quality 1-5 | Notes |
```

Checks:
- SOUL.md loads correctly from `~/.oracle`, `~/.eureka`, `~/.aurus`
- Agent responds with correct persona identity when invoked via CLI
- No dependency on gateway state for CLI access
- config.yaml per agent home is respected

---

## 6. Double-Loop Review

At trial end, beyond the data:
- "What did we not measure that mattered?"
- "Did shared-home subtly shape agent behavior in ways we didn't notice because we weren't looking?"
- "Did CLI access reveal anything about agent identity that Discord access didn't?"
- "Now that personas load correctly for the first time, are we asking the right questions about architecture — or are we still reacting to Phase 0 artifacts?"

---

## Key Source Files

| File | Path | Purpose |
|------|------|---------|
| Gateway config | `~/.hermes/config.yaml` | Shared gateway configuration |
| Foundation values | `~/.hermes/SHARED_FOUNDATION.md` | Universal behavioral contract |
| Agent source | `~/.hermes/hermes-agent/` | Gateway and agent codebase |
| Secrets | `~/.hermes/.env` | API keys and tokens ([REDACTED]) |
| Oracle home | `~/.oracle/SOUL.md`, `~/.oracle/config.yaml` | Oracle CLI identity and config |
| Eureka home | `~/.eureka/SOUL.md`, `~/.eureka/config.yaml` | Eureka CLI identity and config |
| Aurus home | `~/.aurus/SOUL.md`, `~/.aurus/config.yaml` | Aurus CLI identity and config |

Note: `~/.arch` is a backward-compat symlink to `~/.hermes`. All references to `~/.arch/` in existing tooling should resolve correctly. Update tooling to use `~/.hermes` as canonical path.
