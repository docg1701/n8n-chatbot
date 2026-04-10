# Claude Code Instruction Quality Guide

Reference document for auditing and improving all Claude Code instruction files (CLAUDE.md, subagent definitions, checkpoint prompts). Synthesized from official Anthropic documentation, Claude Code docs, and prompt engineering best practices for Claude 4.x models.

---

## 1. The Context Budget Problem

Claude Code's system prompt contains ~50 built-in instructions. Research shows frontier LLMs can reliably follow **150-200 instructions total** before quality degrades uniformly. Every instruction in your CLAUDE.md, subagent prompts, and checkpoint files competes for that same attention budget.

**The core constraint**: as the context window fills, model performance degrades ("context rot"). The goal is to find the **smallest set of high-signal tokens** that maximize the likelihood of correct behavior.

### What This Means for Pipeline Instruction Files

- Your CLAUDE.md (231 lines) + pipeline-supervisor.md (937 lines) + checkpoint prompts (~3,000+ lines total) are fighting for attention against each other AND against the built-in system prompt.
- When files are too long, Claude **ignores rules** — it doesn't announce it's ignoring them, it just stops following them.
- The solution is NOT adding more instructions. It's making existing instructions higher-signal and moving low-signal content out of the attention path.

### Priority Hierarchy (How Claude Resolves Conflicts)

1. **System prompt** (built-in Claude Code instructions) — highest structural weight
2. **CLI flags** (`--system-prompt`, `--agent`) — override everything below
3. **CLAUDE.md** (project root) — loaded every session, treated as context not configuration
4. **`~/.claude/CLAUDE.md`** (global) — applies to all sessions
5. **Files loaded via READ** (checkpoint prompts) — compete with accumulated conversation context
6. **Memory files** — auto-injected, lowest weight

**Key insight from Anthropic**: CLAUDE.md has lower **structural** priority than the system prompt, but gains **informational** weight when it provides rationale (WHY something matters). Instructions with explanations are followed more reliably than bare directives.

---

## 2. CLAUDE.md Best Practices

### Official Recommendations (from code.claude.com/docs/en/best-practices)

**Target under 200 lines per CLAUDE.md file.** For each line, ask: "Would removing this cause Claude to make mistakes?" If not, cut it.

**Bloated CLAUDE.md files cause Claude to ignore your actual instructions.**

| Include | Exclude |
|---------|---------|
| Bash commands Claude can't guess | Anything Claude can figure out by reading code |
| Code style rules that differ from defaults | Standard language conventions Claude already knows |
| Testing instructions and preferred test runners | Detailed API documentation (link to docs instead) |
| Repository etiquette (branch naming, PR conventions) | Information that changes frequently |
| Architectural decisions specific to your project | Long explanations or tutorials |
| Developer environment quirks (required env vars) | File-by-file descriptions of the codebase |
| Common gotchas or non-obvious behaviors | Self-evident practices like "write clean code" |

### Progressive Disclosure Strategy

Instead of cramming everything into CLAUDE.md, use **external references**:

```markdown
# CLAUDE.md (lean)
See @docs/pipeline-rules.md for pipeline execution rules.
See @tools/README.md for checkpoint prompt architecture.
```

The `@path/to/import` syntax tells Claude to load files on demand. This keeps CLAUDE.md lean while making detailed rules accessible.

### Emphasis and Adherence

You can tune instructions by adding emphasis — `IMPORTANT`, `CRITICAL`, `YOU MUST`, `NEVER` — to improve adherence. But use sparingly: if everything is "CRITICAL", nothing is.

**Claude 4.6 is more responsive to system prompts than previous models.** Instructions that were needed for weaker models may now cause **overtriggering**. Where you might have said `"CRITICAL: You MUST use this tool when..."`, use more normal prompting like `"Use this tool when..."`.

### What Makes Instructions "Stick"

From Anthropic's prompt engineering guide:

1. **Provide rationale (WHY)**: "Don't run FFmpeg commands in parallel — FFmpeg saturates CPU/RAM and parallel runs will crash" sticks better than "Don't run FFmpeg commands in parallel."

2. **Tell Claude what TO DO, not what NOT to do**: Instead of "Do not use markdown", say "Write in flowing prose paragraphs." Instead of "Don't invent JSON fields", say "Use ONLY the fields defined in the schema below."

3. **Use examples**: 3-5 concrete examples are more effective than abstract rules. Examples function as "high-density information transfer — pictures worth a thousand words for LLMs."

4. **Use XML tags for structure**: Wrap different content types in distinct tags (`<schema>`, `<rules>`, `<examples>`) so Claude can parse unambiguously.

5. **Provide verification criteria**: Claude performs dramatically better when it can check its own work. Include tests, expected outputs, or validation commands.

### Anti-Patterns

- **Don't auto-generate with `/init`**: The output is mediocre. Manual curation is essential.
- **Don't use CLAUDE.md as a linter**: Use hooks for deterministic enforcement.
- **Don't include code style guidelines**: Claude learns patterns from existing code.
- **Don't include task-specific instructions**: These distract when irrelevant. Use skills or checkpoint prompts instead.
- **Don't duplicate information across files**: A rule stated in CLAUDE.md, repeated in pipeline-supervisor.md, and again in a checkpoint prompt wastes tokens and creates divergence risk.

---

## 3. Subagent Definitions Best Practices

### Official Documentation (from code.claude.com/docs/en/sub-agents)

Subagents receive **ONLY their system prompt** (plus basic environment details like working directory). They do NOT receive the full Claude Code system prompt. This means:

- They are NOT subject to the built-in "be concise" / "avoid over-engineering" directives
- They are a **clean slate** — everything they need must be in their markdown file
- They CANNOT spawn other subagents

### Required Frontmatter Fields

Only `name` and `description` are required:

```yaml
---
name: my-agent        # lowercase letters and hyphens
description: When Claude should delegate to this agent
---
```

### All Available Frontmatter Fields

| Field | Required | Description |
|-------|----------|-------------|
| `name` | Yes | Unique identifier, lowercase with hyphens |
| `description` | Yes | When Claude should delegate — this is what the orchestrator reads |
| `tools` | No | Allowlist of tools. Inherits all if omitted |
| `disallowedTools` | No | Denylist, removed from inherited/specified list |
| `model` | No | `sonnet`, `opus`, `haiku`, full model ID, or `inherit` (default) |
| `permissionMode` | No | `default`, `acceptEdits`, `dontAsk`, `bypassPermissions`, `plan` |
| `maxTurns` | No | Maximum agentic turns before stopping |
| `skills` | No | Skills to preload into context at startup |
| `mcpServers` | No | MCP servers available to this subagent |
| `hooks` | No | Lifecycle hooks scoped to this subagent |
| `memory` | No | Persistent memory scope: `user`, `project`, or `local` |
| `background` | No | Set `true` to always run as background task |
| `isolation` | No | Set `worktree` for isolated git worktree |

### Design Principles

1. **Each subagent should excel at ONE specific task** — don't combine research + generation + validation in one agent.

2. **Write detailed descriptions** — Claude uses the description to decide when to delegate. Vague descriptions cause wrong delegation or no delegation.

3. **Limit tool access** — grant only necessary permissions. A read-only analyzer should NOT have Write/Edit tools.

4. **The system prompt body IS the complete instruction set** — everything the subagent needs must be in the markdown body. No external references will be automatically loaded.

5. **Use `skills` field to inject additional context** — if a subagent needs to follow a checkpoint protocol, preload it as a skill rather than hoping it will READ the file.

### Subagent Prompt Quality Checklist

For each subagent definition, verify:

- [ ] **Description clearly states WHEN** to delegate (not just what it does)
- [ ] **Tools are minimally scoped** — no unnecessary Write/Edit access for read-only agents
- [ ] **Model is explicitly set** — don't rely on `inherit` for critical tasks
- [ ] **Input is clearly defined** — what data will be provided by the orchestrator
- [ ] **Output format is explicitly specified** — with schema or example
- [ ] **Rules are stated positively** — what TO do, not just what NOT to do
- [ ] **Error handling is defined** — what to do if input is missing/corrupt
- [ ] **The prompt is self-contained** — agent doesn't need to READ external files to understand its task

---

## 4. Checkpoint Prompts (Files Loaded via READ)

### The Challenge

Checkpoint prompts are loaded mid-conversation via the READ tool. By the time Claude reads them, the context window already contains:

- The full CLAUDE.md (~12KB)
- The pipeline-supervisor.md (~42KB, if loaded)
- All prior conversation history
- Tool outputs from previous phases

This means checkpoint prompts compete with potentially **100KB+ of existing context**. Rules buried in a 500-line checkpoint prompt will be followed less reliably than rules at the top.

### Structural Recommendations

**Put critical rules at the TOP and BOTTOM of the file** — research shows LLMs recall information better from the beginning and end of documents ("primacy and recency effects"). Rules buried in the middle are most likely to be ignored.

**Use clear section hierarchy:**

```markdown
## CRITICAL RULES (read first)
[The 3-5 most important rules that MUST NOT be violated]

## Task
[What to do]

## Input
[What data to read]

## Output Format
[Exact schema with example]

## Quality Checklist
[Self-verification steps]

## REMINDERS (read last)
[Restate the critical rules from the top]
```

### Schema Enforcement

When a checkpoint prompt defines a JSON schema, compliance depends on:

1. **Providing a complete, literal example** — not just field names, but a full JSON object with realistic values. Examples are followed more reliably than abstract schema descriptions.

2. **Listing every required field explicitly** — don't assume Claude will infer required fields from the example. State them: "The following fields are REQUIRED: field_a, field_b, field_c."

3. **Providing verification instructions** — "After generating the JSON, verify: (1) all required fields are present, (2) no extra fields were added, (3) value types match the schema."

4. **Embedding the schema near the output instruction** — don't define the schema at line 50 and the "write the file" instruction at line 400. Place them together.

### Anti-Patterns for Checkpoint Prompts

- **Don't repeat CLAUDE.md rules** — the checkpoint prompt should contain ONLY checkpoint-specific instructions. General rules belong in CLAUDE.md.
- **Don't include extensive background/context** — Claude already has the context from prior phases. The checkpoint prompt should focus on the TASK.
- **Don't use vague compliance language** — "Follow the protocol" is meaningless. Specify WHICH rules from the protocol matter most for THIS step.
- **Don't bury the output schema** — if the file is 500 lines and the schema is at line 350, Claude may not follow it. Put it in the first 50 lines or the last 20.

---

## 5. Making Claude Follow Schemas Exactly

This is the core problem described by the user. Here are the specific techniques from Anthropic's documentation:

### Technique 1: Structured Outputs (API-level)

For programmatic use, Claude supports `--output-format json` with `--json-schema` for guaranteed schema compliance. However, this only works for CLI/API invocations, not for files written during interactive sessions.

### Technique 2: Inline Schema + Example + Verification

For files written during checkpoint execution:

```markdown
## Output: Write to `.temp/{video_id}/segments/{seg}_seo.json`

The file MUST contain exactly this structure:

<schema>
{
  "segment_id": "seg_001",
  "video_type": "horizontal",
  "titles": [
    {
      "rank": 1,
      "label": "T1",
      "text": "Primary Title Here",
      "char_count": 52,
      "strategy": "Why this title was chosen"
    }
  ],
  "description": "Full description text...",
  "tags": ["tag1", "tag2", "tag3"],
  "category": "Education"
}
</schema>

<required_fields>
- segment_id (string, from input)
- video_type (string: "horizontal" or "shorts")
- titles (array of exactly 3 objects, each with: rank, label, text, char_count, strategy)
- description (string, 150-200 words for horizontal, 50-80 words for shorts)
- tags (array of 15-20 strings, no duplicates)
- category (string, default "Education")
</required_fields>

<verification>
Before writing the file, check:
1. titles array has exactly 3 entries
2. Each title has all 5 fields (rank, label, text, char_count, strategy)
3. labels are exactly "T1", "T2", "T3"
4. tag count is between 15 and 20
5. No duplicate tags
6. description word count is in range
7. No fields were added beyond those listed above
</verification>
```

### Technique 3: Examples Over Rules

Instead of listing 20 rules about title formatting, provide 3 complete examples:

```markdown
<examples>
<example type="horizontal">
{
  "titles": [
    {"rank": 1, "label": "T1", "text": "This Knee MRI Shows Something Doctors Rarely See", "char_count": 49, "strategy": "Curiosity gap + rarity signal"},
    {"rank": 2, "label": "T2", "text": "Knee MRI: Rare Meniscal Root Tear Diagnosis", "char_count": 45, "strategy": "Search-focused with medical keyword"},
    {"rank": 3, "label": "T3", "text": "The Scan That Changed Everything for This Patient", "char_count": 50, "strategy": "Narrative hook + emotional angle"}
  ]
}
</example>
</examples>
```

### Technique 4: Hooks for Deterministic Enforcement

For rules that MUST be enforced 100% of the time, don't rely on instructions — use hooks:

```json
{
  "hooks": {
    "PostToolUse": [{
      "matcher": "Write",
      "hooks": [{
        "type": "command",
        "command": "./scripts/validate-seo-json.sh"
      }]
    }]
  }
}
```

A validation script that runs AFTER every file write is more reliable than any instruction. The script can check schema compliance, field counts, value ranges, etc.

---

## 6. The System Prompt Conflict

### The Problem

Claude Code's built-in system prompt contains directives that conflict with pipeline needs:

| System Prompt Directive | Pipeline Need |
|------------------------|---------------|
| "Be extra concise" | Complete JSON output with all fields |
| "Avoid over-engineering" | Follow exact multi-step protocols |
| "Keep solutions simple" | Complex checkpoint choreography |
| "Don't add features beyond what was asked" | Write all schema fields even when some seem redundant |
| "If you can say it in one sentence, don't use three" | Verbose descriptions, tags, timestamps |

### Mitigation Strategies

**Strategy 1: Rationale-Based Override**

CLAUDE.md instructions gain informational weight when they explain WHY:

```markdown
## Pipeline Data Completeness

Pipeline JSON outputs (SEO files, frame selections, checkpoint data) are consumed
by downstream scripts that expect EVERY field. Missing fields cause script crashes
that require re-running hours of work. Conciseness rules apply to conversation
with the user — NEVER to pipeline data files.
```

This works because the system prompt says "These instructions OVERRIDE any default behavior" about CLAUDE.md. Adding rationale makes the override more likely to be respected.

**Strategy 2: Subagent Isolation**

Subagents do NOT receive the full Claude Code system prompt. They only receive their own markdown body. This means subagents are NOT subject to:

- "Be extra concise"
- "Avoid over-engineering"
- "Keep solutions simple"

**This is why the 3 existing subagents (seo-generator, frame-selector, segment-analyzer) likely work better than the orchestrator** — they operate in a clean context without the conflicting system prompt.

**Strategy 3: Hooks for Critical Compliance**

For the most critical schema compliance, use PostToolUse hooks to validate JSON files after they're written. This is deterministic — no LLM involved — and catches deviations regardless of what the model decided to do.

**Strategy 4: Reduce Orchestrator Burden**

Move as much generation work as possible to subagents (which have clean context) and reduce the orchestrator's role to:

1. Reading state
2. Running commands
3. Spawning subagents
4. Presenting results to the user
5. Writing approval decisions

The less the orchestrator generates complex structured output, the fewer opportunities for deviation.

---

## 7. Audit Checklist for Instruction Files

Use this checklist to evaluate each of the 24 pipeline instruction files:

### A. Signal-to-Noise Ratio

- [ ] Every line serves a purpose — no filler, no obvious instructions
- [ ] No duplicated content across files
- [ ] No information that Claude can derive from reading code
- [ ] Total line count is proportional to the task complexity

### B. Structure and Readability

- [ ] Critical rules are at the TOP and BOTTOM (not buried in the middle)
- [ ] Clear section hierarchy with headers
- [ ] Output schemas are placed NEAR the output instruction (not 300 lines away)
- [ ] XML tags or markdown headers separate instructions from context from examples

### C. Schema Compliance

- [ ] Every JSON output has a complete literal example (not just field names)
- [ ] Required fields are explicitly listed
- [ ] Self-verification instructions are provided
- [ ] Value constraints (character counts, word counts, array lengths) are stated with the schema

### D. Instruction Quality

- [ ] Instructions say what TO DO (not just what NOT to do)
- [ ] Critical instructions include rationale (WHY)
- [ ] 3-5 examples are provided for complex outputs
- [ ] Emphasis (CRITICAL, MUST, NEVER) is used sparingly and only for true invariants
- [ ] No vague meta-instructions ("follow the protocol exactly") — specific rules instead

### E. Context Efficiency

- [ ] File doesn't repeat rules from CLAUDE.md
- [ ] File doesn't include background information that's already in the conversation
- [ ] Progressive disclosure is used where possible (@imports, external references)
- [ ] Total file size is proportional to the value it provides

### F. Conflict Awareness

- [ ] No instructions that contradict the system prompt without providing rationale
- [ ] No instructions that contradict other instruction files
- [ ] Subagent definitions are self-contained (don't rely on external files being loaded)
- [ ] Generation-heavy tasks are delegated to subagents (which have clean context)

---

## 8. Recommended Architecture

Based on the research, here is the optimal instruction architecture for a complex pipeline like youtube-guru:

### Layer 1: CLAUDE.md (< 200 lines)

- Identity and role (5 lines)
- Pipeline overview with phase names only (10 lines)
- Command reference table (20 lines)
- Checkpoint file mapping table (15 lines)
- Key file locations (10 lines)
- Parallel execution rules with rationale (10 lines)
- Critical rules that apply to EVERY session (10 lines)
- Data completeness override with rationale (5 lines)

### Layer 2: Pipeline Supervisor (loaded once per session, < 300 lines)

- State management protocol
- Step-by-step workflow (commands only, not detailed rules)
- Edge case handling
- Communication guidelines

### Layer 3: Checkpoint Prompts (loaded on demand, < 200 lines each)

- Task-specific instructions only
- Output schema with complete example
- Self-verification checklist
- NO repeated general rules from CLAUDE.md

### Layer 4: Subagent Definitions (isolated context)

- Self-contained system prompts
- Complete schema in the markdown body
- Preloaded skills for any protocols they need to follow
- Minimal tool access

### Enforcement Layer: Hooks (deterministic)

- PostToolUse validation scripts for critical JSON schemas
- PreToolUse guards for dangerous operations

---

## Sources

1. [Best Practices for Claude Code](https://code.claude.com/docs/en/best-practices) — Official Claude Code documentation
2. [Create Custom Subagents](https://code.claude.com/docs/en/sub-agents) — Official subagent configuration reference
3. [Prompting Best Practices (Claude 4.x)](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/claude-4-best-practices) — Official prompt engineering for Claude Opus 4.6 / Sonnet 4.6
4. [Effective Context Engineering for AI Agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) — Anthropic engineering blog
5. [Advanced Tool Use](https://www.anthropic.com/engineering/advanced-tool-use) — Anthropic engineering blog on tool patterns
6. [Increase Output Consistency](https://platform.claude.com/docs/en/test-and-evaluate/strengthen-guardrails/increase-consistency) — Official consistency techniques
7. [Giving Claude a Role with System Prompts](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/system-prompts) — Official system prompt guidance
8. [Writing a Good CLAUDE.md](https://www.humanlayer.dev/blog/writing-a-good-claude-md) — HumanLayer practical analysis
9. [CLAUDE.md vs System Prompt Priority](https://docs.bswen.com/blog/2026-03-15-claudemd-vs-system-prompt-priority/) — Priority hierarchy analysis
10. [System Prompt Plan Mode Override Issue](https://github.com/anthropics/claude-code/issues/30634) — Known issue with instruction conflicts
11. [Skill Authoring Best Practices](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices) — Official skill design patterns
12. [Claude Code Advanced Patterns Webinar](https://www.anthropic.com/webinars/claude-code-advanced-patterns) — Subagents, MCP, and scaling
