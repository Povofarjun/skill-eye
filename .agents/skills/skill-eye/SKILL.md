---
name: skill-eye
description: >
  Evaluates any Claude Code skill against the user's actual workflow before they install it.
  Acts as a guardian: learns what the user truly does day-to-day, then tears apart the
  candidate skill to answer "will this actually help you?" and delivers a compact verdict
  (install as-is / modify-then-install / skip) with specific, copy-pasteable changes.
  Use when the user says "should I install X skill", "is this skill worth it for me",
  "evaluate this skill", "review skill X", "check if Y skill fits my workflow",
  or pastes a SKILL.md URL or GitHub repo path to assess.
argument-hint: "[--help] [--detailed] <skill-name|github-url|owner/repo>"
disable-model-invocation: true
---

# skill-eye — The Skill Guardian

Built on the AXI design principles (axi.md): token-efficient output, content-first behavior,
contextual next-step disclosure, structured errors, and intelligent truncation.

---

## Phase 0 — Content-First No-Args Behavior

If `$ARGUMENTS` is empty OR equals `--help`, do NOT silently fail or dump a help wall.
Show live state first, then compact usage.

Glob these paths to get real counts:
- `~/.claude/plugins/**/skills/*/SKILL.md`
- `~/.claude/skills/*/SKILL.md`

Output:
```
skill-eye v1.0
installed: <N> skills across <M> plugin directories
[last eval: <name> · <verdict> · <N> days ago]   ← include only if found in ~/.claude/history.jsonl

usage: /skill-eye <skill-name|github-url|owner/repo> [--detailed]

next: /skill-eye code-review         evaluate the installed code-review skill
next: /skill-eye verify              evaluate the installed verify skill
next: /skill-eye <owner>/<repo>      evaluate skills from a GitHub repo
```

Use actual glob counts — never show placeholder text like `<N>`.

---

## Phase 1 — Ambient Context Gathering

Gather silently BEFORE asking anything. Run all three in parallel:

1. Read `./CLAUDE.md` and `./.claude/CLAUDE.md` — extract tech stack, project type, conventions
2. Read `~/.claude/history.jsonl` (last 40 lines) — parse `display` field; find recurring topics,
   slash commands used, question patterns
3. Glob `~/.claude/plugins/**/skills/*/SKILL.md` + `~/.claude/skills/*/SKILL.md` — record names
   and descriptions of all installed skills (needed for redundancy check in Phase 3)

Then ask ONLY what you still don't know. Always ask:
- "What's your primary role and what do you build day-to-day?"
- "What are the 2–3 things you use Claude Code for most?"

Ask only if still unclear after reading passive context:
- Tech stack / languages / frameworks
- MCP integrations in regular use
- Biggest workflow friction

Synthesize into an internal User Profile (never displayed verbatim):
```
role / stack / daily_tasks[] / mcp_available[] / pain_points[] / typical_prompts[] / installed_skills[]
```

---

## Phase 2 — Acquire & Truncate the Target Skill

Resolve `$ARGUMENTS` (strip `--detailed` flag first, then process the rest):

**Named skill** (e.g. `code-review`):
Glob `~/.claude/plugins/**/skills/<name>/SKILL.md` and `~/.claude/skills/<name>/SKILL.md`.
If multiple matches, list all with their source paths and ask which.

**GitHub URL** (e.g. `https://github.com/owner/repo/blob/main/skills/foo/SKILL.md`):
Fetch the raw content. Convert blob URLs to raw.githubusercontent.com equivalents.

**owner/repo** (e.g. `supabase/agent-skills`):
Fetch `https://raw.githubusercontent.com/<owner>/<repo>/main/` and look for `skills/` directory.
List available skills, ask which to evaluate.

**No args / `--help`**: → Phase 0 above.

### Truncation

If SKILL.md body exceeds 150 lines, read the first 120 lines and note:
```
[body: 120/<total> lines shown — truncated; key sections extracted below]
```
Extract from those 120 lines: what Claude is instructed to do, tool references, external
dependencies, and implicit assumptions about the user/project. That's enough to evaluate.

### Empty State

If skill name not found anywhere:
```
skill-eye: 0 results for "<name>"
searched: ~/.claude/plugins/**/skills/<name>/SKILL.md
          ~/.claude/skills/<name>/SKILL.md

next: /skill-eye <github-url>    fetch SKILL.md directly from GitHub
next: /skill-eye <owner>/<repo>  browse all skills in a GitHub repo
```

### Structured Error

For any other failure (fetch error, parse error, ambiguous input):
```
skill-eye: error — <what failed>
detail: <specific reason>
next: <one corrective command>
```

---

## Phase 3 — Tear It Apart

Pre-compute before scoring:
- `installed_count`: total skills from Phase 1 glob
- `overlap_candidates`: count of installed skills whose description shares significant keywords
  with the target skill's description (used in report header, not for scoring)

Read `references/evaluation-rubric.md` for detailed scoring criteria and edge cases.

Score four dimensions 0–10 against the User Profile:

**1. Trigger Alignment** — how often would the skill's description trigger phrases match
the user's `typical_prompts`? Base on language match, not just domain match.
0 = never activates for this user · 10 = fires multiple times per week

**2. Tool & Dependency Fit** — does the user have the required tools, MCPs, external services?
Hard penalize (≤3) if a critical dependency is absent with no workaround.
0 = core deps absent · 10 = all deps present and actively used

**3. Task Relevance** — does this skill directly address a `daily_task` or `pain_point`?
Most important dimension. A well-built skill for the wrong problem is worthless.
0 = wrong domain · 10 = solves a named pain point

**4. Value Density** — if installed, would the user actually invoke it, or forget it exists?
Consider: frequency of the underlying problem + whether skill fires automatically.
0 = install-and-forget · 10 = used multiple times per week

Overall Fit Score = (Trigger + Tool + Task + Value) / 4  (1 decimal)

Redundancy: compare target description against all `installed_skills[]` descriptions.
Flag if significant trigger overlap found.

---

## Phase 4 — Compact Report + Verdict

Always output in this format. Compact structured lines, not markdown headers or emoji tables.

```
skill-eye · <name> · <source: local|url|repo>
profile:  <role> · <stack summary>
catalog:  <installed_count> installed · <overlap_candidates> potential overlap(s)
──────────────────────────────────────────────
behavior: <1 honest sentence: what the body actually instructs Claude to do>
claims:   "<trigger phrases verbatim from description field>"
fit_for:  <who this skill is actually built for — honest, not generous>
needs:    <tools/MCPs/services it requires>
assumes:  <hidden project/role assumptions, or "none detected">
──────────────────────────────────────────────
scores{trigger,tool,task,value}: X,X,X,X
fit: X.X/10 · redundancy: NONE | OVERLAPS: <skill-name> (<what overlaps>)
──────────────────────────────────────────────
<VERDICT BLOCK>
──────────────────────────────────────────────
<NEXT LINES>
```

---

### VERDICT: INSTALL AS-IS  (fit ≥ 7.0)

```
verdict: INSTALL AS-IS

gives you: <specific benefit framed in user's actual context>
           <second concrete benefit>
tip:       <one usage tip specific to user's stack or habits>

next: cp <found-path> ~/.claude/skills/<name>/SKILL.md   install now
next: /skill-eye <other-skill>                            evaluate another
```

---

### VERDICT: MODIFY THEN INSTALL  (fit 4.0–6.9)

```
verdict: MODIFY THEN INSTALL

works:    <what aligns with user's workflow>
mismatch: <what doesn't, and the specific reason>

change 1 — <one-phrase reason>
  REPLACE in description:
    "<original trigger phrase>"
  WITH:
    "<rewritten phrase matching user's actual language>"

change 2 — <one-phrase reason>
  ADD to body after "<section header or first line of section>":
    "<content specific to user's tech stack or workflow>"

change 3 — <one-phrase reason, only if genuinely needed>
  REMOVE section starting with: "<section header>"
  (this section assumes <thing> that isn't true for your setup)

next: edit <found-path>                                   apply changes above
next: cp <found-path> ~/.claude/skills/<name>/SKILL.md   then install
next: /skill-eye <other-skill>                            evaluate another
```

Max 3 changes. If more than 3 are needed, reconsider: the verdict should probably be SKIP.

---

### VERDICT: SKIP  (fit < 4.0)

```
verdict: SKIP

primary:   <most important mismatch — specific, not vague>
secondary: <second reason, if applicable>
need:      <what a skill would actually have to do to be useful to this user>
existing:  <installed skill that already covers this area, or "none found">

next: /skill-eye <better-fit-alternative>   try a more relevant skill
next: /skill-eye <other-skill>              evaluate something else
```

---

## `--detailed` Flag (opt-in expansion)

When `$ARGUMENTS` contains `--detailed`, append this section after the verdict + next lines:

```
── DETAILED BREAKDOWN ──────────────────────
trigger_analysis:
  user_phrases:   [<actual phrases extracted from history.jsonl>]
  skill_triggers: [<trigger phrases from description field>]
  overlap:        <specific matching phrases, or "none">

tool_analysis:
  required:   [<tools from allowed-tools frontmatter + body references>]
  available:  [<user's mcp_available from profile>]
  missing:    [<gap list, or "none">]

body_summary: <extended summary of what body actually instructs — not truncated>
[body: <N>/<total> lines shown]   ← if truncation applied
```

---

## Edge Cases

**No args / `--help`** → Phase 0 content-first output (never silence, never help wall)

**Skill found in multiple locations**: list all paths with source labels, ask which to evaluate

**Skill has only frontmatter, no body**: evaluate on what exists; note in `behavior:` field:
```
behavior: no body — frontmatter only; actual behavior in use is unknown
```

**Model-invoked skill** (no slash command / no `argument-hint`): note in report:
```
trigger note: model-invoked — fires automatically when Claude detects context match;
              trigger alignment harder to assess; base score on description match to
              user's typical conversation topics
```

**owner/repo with no `skills/` directory**: structured error with next steps to try the repo root URL

**Very large SKILL.md (e.g. skill-creator at 33KB)**: truncation applies; note line counts;
extract only what matters for evaluation — don't load the full file
