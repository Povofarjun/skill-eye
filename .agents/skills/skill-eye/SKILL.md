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
  Also use when the user says "what skills should I install", "recommend skills for me",
  "discover skills", "audit my skills", "which of my skills am I not using",
  "evaluate all skills in this repo", or "batch evaluate".
argument-hint: "[--help] [--discover] [--audit] [--batch] [--detailed] <skill-name|github-url|owner/repo>"
disable-model-invocation: true
---

# skill-eye — The Skill Guardian

Built on the AXI design principles (axi.md): token-efficient output, content-first behavior,
contextual next-step disclosure, structured errors, and intelligent truncation.

---

## Mode Detection Router

Check `$ARGUMENTS` before anything else. Strip `--detailed` first (it's an expansion flag
that appends to any mode's output), then route on what remains:

| Argument pattern | Route to |
|-----------------|----------|
| empty or `--help` | Phase 0 — Content-First |
| `--discover` | Phase 5 — Discover Mode |
| `--audit` | Phase 6 — Audit Mode |
| `<repo> --batch` or `--batch <repo>` | Phase 7 — Batch Mode |
| anything else (skill name, URL, owner/repo) | Phases 1–4 — Standard Evaluation |

---

## Phase 0 — Content-First No-Args Behavior

If `$ARGUMENTS` is empty OR equals `--help`, do NOT silently fail or dump a help wall.
Show live state first, then compact usage.

Glob these paths to get real counts:
- `~/.claude/plugins/**/skills/*/SKILL.md`
- `~/.claude/skills/*/SKILL.md`

Output:
```
skill-eye v1.1
installed: <N> skills across <M> plugin directories
[last eval: <name> · <verdict> · <N> days ago]   ← include only if found in ~/.claude/history.jsonl

usage: /skill-eye <skill-name|github-url|owner/repo> [--detailed]
       /skill-eye --discover          recommend skills for your workflow
       /skill-eye --audit             review all installed skills
       /skill-eye <owner>/<repo> --batch   evaluate every skill in a repo

next: /skill-eye --discover          find skills matched to your workflow
next: /skill-eye --audit             check which installed skills you actually use
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

When `$ARGUMENTS` contains `--detailed`, append this section after the verdict + next lines.
Works with all modes (standard evaluation, --discover, --audit, --batch).

For standard evaluation:
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

---

## Phase 5 — Discover Mode (`--discover`)

**Purpose:** The user doesn't know what skill to try next. Recommend best-fit skills from
known sources, pre-scored against their actual workflow. Solves the cold-start problem.

### Step 1 — Build User Profile
Run Phase 1 (ambient context) silently. Ask the 2 standard questions if profile is unclear.

### Step 2 — Scan Sources (parallel)

1. **Already-installed skills not fully utilized**: Glob installed skills, identify any with
   trigger alignment ≥ 6 against user profile that the user has never invoked (per history).
   These are "hidden gems" — already available, never discovered.

2. **Community repos** — fetch the GitHub API tree for each, find all SKILL.md files,
   fetch only the first 20 lines (frontmatter only) to get name + description for scoring.
   Skip files that match already-installed skills.

   Repos to scan:
   - `kunchenguid/axi` — `.agents/skills/`
   - `multica-ai/andrej-karpathy-skills` — `skills/`
   - `supabase/agent-skills` — `skills/`
   - `Povofarjun/skill-eye` — `.agents/skills/`
   - `anthropics/claude-code` — check for any `skills/` directories

### Step 3 — Score and Rank
Score each candidate on trigger alignment + task relevance only (2-dimension quick score).
Skip: already installed AND appearing in recent history (user knows about it).
Skip: fit < 3.0 (irrelevant noise).
Sort descending. Show top 5 with verdict.

### Step 4 — Output

```
skill-eye discover · profile: <role> · <stack>
searched: <N> installed (hidden gems) + <M> community repos · <total> candidates
──────────────────────────────────────────────
rank  skill                 source                fit    verdict
1     <name>                <owner/repo|local>    X.X    INSTALL AS-IS
2     <name>                <owner/repo>          X.X    MODIFY THEN INSTALL
3     <name>                <owner/repo>          X.X    INSTALL AS-IS
──────────────────────────────────────────────
next: /skill-eye <rank-1-name>              full evaluation of top pick
next: /skill-eye <rank-2-name>             full evaluation of #2
next: /skill-eye --discover --detailed     show all candidates + near-misses
next: /skill-eye <owner>/<repo> --batch    explore a specific repo
```

### `--detailed` expansion for discover

When `--detailed` is present, show all candidates including those below 3.0:
```
── ALL CANDIDATES ──────────────────────────
near-misses (fit < 3.0 — skipped):
  <name> (<source>) — fit: X.X — <one-line reason skipped>
  ...
```

---

## Phase 6 — Audit Mode (`--audit`)

**Purpose:** Review ALL installed skills against the user's actual history. Surface zombies
(install-and-forget), redundant pairs, and workflow gaps — without the user asking about
each skill one by one.

### Step 1 — Build User Profile
Run Phase 1 (ambient context). This gives `installed_skills[]` and `typical_prompts[]`.

### Step 2 — Score Every Installed Skill

For each installed SKILL.md:
1. Read description + first 30 lines of body
2. Check history.jsonl: does the skill name, its slash command, or its trigger phrases
   appear in the `display` field of any recent entry?
3. Score trigger alignment against `typical_prompts` (rubric dimension 1)
4. Score task relevance against `daily_tasks` (rubric dimension 3)
5. Quick fit = (trigger + task) / 2

Classify each skill:
- `active`    — quick fit ≥ 6.0 OR appears in recent history
- `dormant`   — quick fit 3.0–5.9 AND not in recent history
- `zombie`    — quick fit < 3.0 AND not in history (install-and-forget)
- `redundant` — trigger phrases overlap significantly with another installed skill

### Step 3 — Find Coverage Gaps

Scan `typical_prompts[]` for recurring task patterns with no installed skill match.
Look for: verb + object pairs appearing 3+ times in history that no skill description covers.
Report at most 3 gaps. For each, suggest a known skill or repo to fill it.

### Step 4 — Output

```
skill-eye audit · <N> skills scanned · <date>
profile: <role> · <stack>
──────────────────────────────────────────────
active (<N>):    <name> · <name> · <name>
dormant (<N>):   <name> · <name>
zombie (<N>):    <name> · <name> · <name>
redundant (<N>): <name-a> ≈ <name-b>  (both trigger on "<phrase>")
──────────────────────────────────────────────
top removals:
  1. <name> — <specific reason: never triggered / wrong domain / overlaps with X>
  2. <name> — <specific reason>

coverage gaps:
  1. "<recurring prompt pattern>" → no installed skill handles this
     try: /skill-eye --discover
  2. "<pattern>" → try: /skill-eye <known-skill-or-repo>
──────────────────────────────────────────────
next: /skill-eye <zombie-name>           evaluate before removing
next: /skill-eye --audit --detailed      full per-skill breakdown
next: /skill-eye --discover              find skills for the gaps above
```

### `--detailed` expansion for audit

When `--detailed` is present, append a full per-skill inventory table:
```
── FULL SKILL INVENTORY ────────────────────────
<name>    fit: X.X  status: active    last seen: <N> days ago
<name>    fit: X.X  status: zombie    last seen: never
<name>    fit: X.X  status: redundant with <name>  trigger: "<phrase>"
...
```

---

## Phase 7 — Batch Mode (`<owner>/<repo> --batch`)

**Purpose:** Evaluate ALL skills in a GitHub repo in one pass. No asking which one — score
all of them and return a ranked table. For users browsing a repo and wanting a full picture
before committing to individual evaluations.

### Step 1 — Build User Profile
Run Phase 1 (ambient context) silently. Ask standard questions only if profile is missing.

### Step 2 — Discover All Skills in Repo

Use GitHub API tree endpoint:
`https://api.github.com/repos/<owner>/<repo>/git/trees/main?recursive=1`

Find all SKILL.md files. Common paths:
- `.agents/skills/*/SKILL.md`
- `skills/*/SKILL.md`
- `external_plugins/*/skills/*/SKILL.md`

For each SKILL.md found, fetch only the first 25 lines (frontmatter + opening body lines).
That's enough to score. Do NOT fetch full bodies.

If 0 SKILL.md files found anywhere in the tree:
```
skill-eye: error — no skills found in <owner>/<repo>
detail: checked .agents/skills/, skills/, external_plugins/ — 0 SKILL.md files
next: /skill-eye <raw-url-to-SKILL.md>   evaluate a specific file directly
```

### Step 3 — Score All Skills (Quick 2D)

Quick fit = (trigger alignment + task relevance) / 2 for each skill.
Apply standard verdict thresholds to quick fit score.
Sort descending by quick fit.

### Step 4 — Output

```
skill-eye batch · <owner>/<repo> · <N> skills found
profile: <role> · <stack>
──────────────────────────────────────────────
rank  skill                 fit    verdict              next
1     <name>                X.X    INSTALL AS-IS        /skill-eye <name>
2     <name>                X.X    MODIFY THEN INSTALL  /skill-eye <name>
3     <name>                X.X    SKIP                 low fit
──────────────────────────────────────────────
next: /skill-eye <rank-1-name>               full evaluation of top pick
next: /skill-eye <rank-2-name>               full evaluation of #2
next: /skill-eye --batch <other>/<repo>      batch evaluate another repo
```

### `--detailed` expansion for batch

When `--detailed` is present, append a one-line reason after each table row:
```
  → <trigger mismatch | tool gap | task irrelevant | strong fit — specific reason>
```
