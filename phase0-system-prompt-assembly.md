# Phase 0: System Prompt Assembly Investigation

> **Path note:** After directory restructuring, `~/.hermes` will be the real directory. Currently it is a symlink to `~/.arch`.

## Executive Summary

The identity confusion between Oracle and Arch stems from **three compounding issues**:

1. **`file://` URIs in channel_prompts are NOT resolved** — the literal string `file://personas/arch.md` is passed to the LLM, not the file contents.
2. **APPEND merge behavior** — even if file resolution worked, channel_prompts are APPENDED to the base prompt, not replacing it. Both Oracle AND Arch identities would coexist.
3. **SOUL.md always defines Oracle** — it is the unconditional primary identity for ALL channels.

---

## Significance

This investigation revealed that persona files have NEVER been loaded by the gateway. For the entire lifetime of the fleet (April 13-26), all four agents have been running as Oracle with a literal file path string appended. The specialists (Arch, Eureka, Aurus) have never operated as designed. This changes the priority and framing of all architectural decisions — any assessment of shared-home vs isolation was based on agents that were never distinct.

---

## Exact Assembly Flow

### Step 1: Config Loading (`gateway/config.py` lines 589-594)

```python
# channel_prompts values are passed through as literal strings
bridged["channel_prompts"] = {str(k): v for k, v in channel_prompts.items()}
```

Config.yaml:
```yaml
discord:
  channel_prompts:
    '1493461055618285670': file://personas/arch.md    # <-- literal string, NOT resolved
    '1493461225567420416': file://personas/eureka.md
    '1493461326662467624': file://personas/aurus.md
```

Result: `config.extra["channel_prompts"]["1493461055618285670"]` = `"file://personas/arch.md"` (literal string).

**No file:// resolution exists anywhere in the codebase.** Tests confirm values are plain strings like `"Research mode"`.

---

### Step 2: Message Event Creation (`gateway/platforms/discord.py` line 2593)

When a Discord message arrives in channel 1493461055618285670 (#arch):

```python
return MessageEvent(
    text=text,
    ...
    channel_prompt=self._resolve_channel_prompt(channel_id, parent_id or None),
)
```

`_resolve_channel_prompt` calls `resolve_channel_prompt()` from `base.py`:

```python
def resolve_channel_prompt(config_extra, channel_id, parent_id=None):
    prompts = config_extra.get("channel_prompts") or {}
    for key in (channel_id, parent_id):
        if not key: continue
        prompt = prompts.get(key)
        if prompt is None: continue
        prompt = str(prompt).strip()
        if prompt: return prompt          # returns "file://personas/arch.md"
    return None
```

Result: `event.channel_prompt = "file://personas/arch.md"` (literal string).

---

### Step 3: Ephemeral Prompt Assembly (`gateway/run.py` lines 9688-9695)

```python
# Combine platform context, per-channel context, and the user-configured ephemeral system prompt.
combined_ephemeral = context_prompt or ""                        # session metadata
event_channel_prompt = (channel_prompt or "").strip()            # "file://personas/arch.md"
if event_channel_prompt:
    combined_ephemeral = (combined_ephemeral + "\n\n" + event_channel_prompt).strip()
if self._ephemeral_system_prompt:
    combined_ephemeral = (combined_ephemeral + "\n\n" + self._ephemeral_system_prompt).strip()
```

Result: `combined_ephemeral = "<session context>\n\nfile://personas/arch.md"`

This is passed to AIAgent as `ephemeral_system_prompt=combined_ephemeral` (line 9858).

---

### Step 4: Base System Prompt Build (`run_agent.py` _build_system_prompt, lines 4357-4522)

The base system prompt is assembled from layers:

```
Layer 1: SOUL.md content (from ~/.hermes/SOUL.md)    <-- "I am Oracle"
Layer 2: Tool guidance (memory, session_search, skills)
Layer 3: Nous subscription prompt
Layer 4: Tool use enforcement
Layer 5: system_message param (from gateway)
Layer 6: Persistent memory (MEMORY.md, USER.md)
Layer 7: Skills system prompt
Layer 8: Context files (AGENTS.md, .hermes.md, etc.)
Layer 9: Timestamp + session ID + model info
Layer 10: Platform hints (discord formatting)
```

**Critical note** (line 4435-4436):
```python
# Note: ephemeral_system_prompt is NOT included here. It's injected at
# API-call time only so it stays out of the cached/stored system prompt.
```

SOUL.md is loaded via `load_soul_md()` from `~/.hermes/SOUL.md` (line 4377):
```python
_soul_content = load_soul_md()
if _soul_content:
    prompt_parts = [_soul_content]    # First layer = Oracle identity
    _soul_loaded = True
```

Result: The cached system prompt starts with the full Oracle persona.

---

### Step 5: API Call-Time Injection (`run_agent.py` lines 9144-9148)

At every LLM API call, the ephemeral prompt is APPENDED:

```python
effective_system = self._cached_system_prompt or ""         # Oracle + layers 2-10
if self.ephemeral_system_prompt:
    effective_system = (effective_system + "\n\n" + self.ephemeral_system_prompt).strip()
#                               ^^^ APPEND, not REPLACE
if effective_system:
    api_messages = [{"role": "system", "content": effective_system}] + api_messages
```

---

## Final System Prompt for #arch Channel

```
[Layer 1 - SOUL.md from ~/.hermes/SOUL.md]
# SOUL.md — Oracle (Your Alter Ego)
...
**Name:** Oracle
**Role:** Alter Ego & Life Navigator
...
DELEGATION — You know when to bring in specialists:
   - Building, coding, experimenting -> Arch (#arch)
...
You operate in a single Discord server (DeepG) with 4 channels:
- #oracle — You. The user talks to you here.
- #arch — The builder. Code, infrastructure, DevOps, tools.
...

[Layers 2-10: tool guidance, memory, skills, context files, timestamp, platform hints]

[Ephemeral - context_prompt]
## Current Session Context
**Source:** Discord (channel: #arch)
**User:** Deepak

[Ephemeral - channel_prompt]
file://personas/arch.md

[/end of system prompt]
```

---

## Root Cause Analysis

### Issue 1: `file://` References Are Never Resolved

The config contains `file://personas/arch.md` but there is NO code path in the gateway that resolves file:// URIs for channel_prompts. The string is passed verbatim to the LLM.

Evidence:
- `resolve_channel_prompt()` in `base.py` does `str(prompt).strip()` — no URI handling
- `gateway/config.py` passes values through unchanged
- Test file (`test_config.py`) uses plain strings like `"Research mode"` — confirming the expected format is inline text, not file references
- No `file://` handling exists in any gateway prompt resolution code

The personas/arch.md file EXISTS at `~/.hermes/personas/arch.md` with proper Arch identity content, but it is never loaded.

### Issue 2: APPEND Semantics Create Dual Identity

Even if file:// resolution were added, the merge behavior is APPEND:

```python
effective_system = (effective_system + "\n\n" + self.ephemeral_system_prompt).strip()
```

This means:
- Layer 1: SOUL.md says "I am Oracle, NOT a coder"
- Ephemeral: channel_prompt says "I am Arch, I ship code"

The LLM receives conflicting identity instructions in the same system prompt.

### Issue 3: SOUL.md Is Global, Not Per-Channel

`load_soul_md()` reads from `$HERMES_HOME/SOUL.md` — a single file shared across all channels. There is no mechanism to load different SOUL.md files per channel. The SOUL.md also references the entire fleet structure, including Arch as a "separate agent," further confusing the LLM about its identity.

---

## What Each Channel Currently Gets

| Channel | channel_prompt value | What LLM sees | Effective identity |
|---------|---------------------|---------------|-------------------|
| #oracle (1465562736451911838) | None | Only SOUL.md | Oracle (correct) |
| #arch (1493461055618285670) | `file://personas/arch.md` | SOUL.md + literal string | Oracle + garbage |
| #eureka (1493461225567420416) | `file://personas/eureka.md` | SOUL.md + literal string | Oracle + garbage |
| #aurus (1493461326662467624) | `file://personas/aurus.md` | SOUL.md + literal string | Oracle + garbage |

Note: "Oracle + garbage" means the LLM received Oracle's full SOUL.md identity plus the literal string `file://personas/arch.md` — not the persona content.

---

## Recommendations for Multi-Agent Fleet

1. **Implement file:// resolution in `resolve_channel_prompt()`**: Add URI detection and file loading from `$HERMES_HOME/` relative paths.

2. **Change merge semantics for channel_prompts from APPEND to REPLACE-SOUL**: When a channel_prompt is present, it should REPLACE the SOUL.md identity layer, not append to it. This requires either:
   - A new `_build_system_prompt` mode that skips SOUL.md when channel_prompt is set
   - OR a separate per-channel SOUL.md loading mechanism

3. **Per-channel SOUL.md**: Support loading different SOUL.md files per channel, either through:
   - `channel_soul:` config key mapping channel IDs to SOUL files
   - OR treating channel_prompts as a full identity override that replaces the SOUL.md slot

4. **Separate fleet awareness from identity**: SOUL.md currently mixes identity ("I am Oracle") with fleet awareness (describing Arch, Eureka, Aurus). Fleet context should be injected separately so it doesn't create identity confusion when the agent IS one of the fleet members.

---

## Key Source Files

| File | Lines | Purpose |
|------|-------|---------|
| `~/.hermes/config.yaml` | 297-306 | channel_prompts config with file:// URIs |
| `~/.hermes/SOUL.md` | 1-83 | Oracle identity (always loaded as Layer 1) |
| `~/.hermes/personas/arch.md` | 1-41 | Arch persona (never loaded) |
| `~/.hermes/personas/eureka.md` | 1-40 | Eureka persona (never loaded) |
| `~/.hermes/personas/aurus.md` | 1-44 | Aurus persona (never loaded) |
| `gateway/platforms/base.py` | 955-982 | `resolve_channel_prompt()` - no file:// handling |
| `gateway/platforms/discord.py` | 2703-2706 | Discord adapter channel prompt resolution |
| `gateway/config.py` | 589-594 | Config loading - passes channel_prompts through |
| `gateway/run.py` | 9688-9695 | Ephemeral prompt assembly (APPEND behavior) |
| `gateway/run.py` | 9851-9858 | AIAgent construction with ephemeral_system_prompt |
| `run_agent.py` | 4357-4522 | `_build_system_prompt()` - SOUL.md as Layer 1 |
| `run_agent.py` | 9144-9148 | API call-time ephemeral APPEND |
| `agent/prompt_builder.py` | 932-957 | `load_soul_md()` - reads from HERMES_HOME |
