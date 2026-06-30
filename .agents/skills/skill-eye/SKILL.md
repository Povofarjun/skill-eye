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
argument-hint: "[--help] [--discover] [--audit] [--batch] [--detailed] [--remove [--force]] [--update [--force]] <skill-name|github-url|owner/repo>"
disable-model-invocation: true
version: 0.2.1
---

# skill-eye — The Skill Guardian

AXI protocol: token-efficient output, content-first, contextual `next:` lines, structured errors.

---

## Argument Parser

Tokenize `$ARGUMENTS` by whitespace. Execute in order:

**Step 1 — Extract modifiers** (remove from token list, set as flags):
- `--force` → force_mode = true
- `--detailed` → detailed_mode = true

**Step 2 — Match remaining tokens:**

| First token | Second token | Route |
|-------------|-------------|-------|
| (empty) or `--help` | — | Phase 0 |
| `--discover` | — | Phase 5 |
| `--audit` | `--prune` | Phase 6 prune mode |
| `--audit` | — | Phase 6 |
| `--remove` | (absent) | error: `missing skill name — usage: /skill-eye --remove <name>` |
| `--remove` | `<name>` | Phase 8, name = second token |
| `--update` | — | Phase 9 |
| `--batch` | `<repo>` | Phase 7, repo = second token |
| `<repo>` | `--batch` | Phase 7, repo = first token |
| starts with `https://` | — | Phase 2, target = URL |
| contains `/` | — | Phase 2, target = owner/repo |
| anything else | — | Phase 2, target = skill name |

**Step 3 — No match:**
```
skill-eye: error — unknown argument: <token>
next: /skill-eye    show usage
```

---

## Glob Standard (apply everywhere)

Three absolute paths only. Never use `./` (CWD-relative). Never use `**` (recursive).
```
~/.agents/skills/*/SKILL.md
~/.claude/plugins/*/skills/*/SKILL.md
~/.claude/skills/*/SKILL.md
```
After globbing: **deduplicate by skill name** (directory containing SKILL.md). Same name in multiple paths = 1 skill. N = unique names. M = paths with ≥1 match.

---

## Phase 0 — Content-First (no-args / --help)

Glob using Glob Standard. Count unique skill names (N) and paths with matches (M).

**Version check (silent, max 3s, skip on any failure):**
Fetch `https://raw.githubusercontent.com/povofarjun/skill-eye/main/.claude-plugin/plugin.json`.
Parse `version`. Compare to frontmatter `version:`. If remote > local: update_available = true.

```
skill-eye v<frontmatter version>
[update: v<remote> available — run /skill-eye --update]   ← only if update_available
installed: <N> skills across <M> directories
[last eval: <name> · <verdict> · <N> days ago]            ← only if found in ~/.claude/history.jsonl

usage: /skill-eye <skill-name|github-url|owner/repo> [--detailed]
       /skill-eye --discover    recommend skills for your workflow
       /skill-eye --audit       review all installed skills
       /skill-eye --update      update to latest version

next: /skill-eye --discover    find skills matched to your workflow
next: /skill-eye --audit       check which installed skills you actually use
next: /skill-eye --update      update skill-eye to latest version
```

Use real counts. Use actual frontmatter version. Never hardcode either.

---

## Phase 1 — Ambient Context Gathering

Run all three in parallel before asking anything:
1. Read `./CLAUDE.md` and `./.claude/CLAUDE.md` — extract tech stack, project type, conventions
2. Read `~/.claude/history.jsonl` last 40 lines — parse `display` field for recurring commands and topics
3. Glob (Glob Standard) — record names and descriptions of all installed skills for redundancy check

Ask only what passive context didn't answer. Always ask:
- "What's your primary role and what do you build day-to-day?"
- "What are the 2–3 things you use Claude Code for most?"

Ask if still unclear: tech stack · MCP integrations · biggest friction

Store internally as: `role / stack / daily_tasks[] / mcp_available[] / pain_points[] / typical_prompts[] / installed_skills[]`

---

## Phase 2 — Acquire & Truncate the Target Skill

**Named skill** (`<name>`): Glob Standard with `<name>` in place of `*`. Multiple matches → list paths, ask which.

**GitHub URL** (`https://...`): Fetch raw content. Convert blob URLs to raw.githubusercontent.com.

**owner/repo**: Fetch `https://raw.githubusercontent.com/<owner>/<repo>/main/` — find `skills/` directory, list available skills, ask which to evaluate.

**Truncation:** If body > 150 lines, read first 120 and note `[body: 120/<total> lines shown]`. Extract: what Claude is instructed to do, tool references, dependencies, assumptions.

**Not found:**
```
skill-eye: 0 results for "<name>"
searched: ~/.agents/skills/<name>/
          ~/.claude/plugins/*/<name>/
          ~/.claude/skills/<name>/

next: /skill-eye <github-url>    fetch directly from GitHub
next: /skill-eye <owner>/<repo>  browse skills in a repo
```

**Other errors:**
```
skill-eye: error — <what failed>
detail: <specific reason>
next: <one corrective command>
```

---

## Phase 3 — Tear It Apart

Pre-compute:
- `installed_count`: N from Glob Standard
- `overlap_candidates`: installed skills whose description shares significant keywords with target

Read `references/evaluation-rubric.md` for scoring edge cases.

Score 0–10 against User Profile:
1. **Trigger Alignment** — how often do skill triggers match `typical_prompts`? (0 = never · 10 = multiple times/week)
2. **Tool Fit** — does user have required tools/MCPs? Hard penalize ≤3 if critical dep absent. (0 = core deps missing · 10 = all present and in use)
3. **Task Relevance** — does skill address a `daily_task` or `pain_point`? Most important. (0 = wrong domain · 10 = solves named pain point)
4. **Value Density** — would user actually invoke it, or forget it? (0 = install-and-forget · 10 = used multiple times/week)

Store internally (do not output):
```
scores: trigger=<T> tool=<To> task=<Ta> value=<V>
fit: <(T+To+Ta+V)/4, 1 decimal>
redundancy: NONE | OVERLAPS: <skill-name> (<what overlaps>)
verdict: INSTALL AS-IS | MODIFY THEN INSTALL | SKIP
```
→ proceed to Phase 4.

---

## Phase 4 — Compact Report + Verdict

```
skill-eye · <name> · <local|url|repo>
profile:  <role> · <stack>
catalog:  <installed_count> installed · <overlap_candidates> potential overlap(s)
──────────────────────────────────────────────
behavior: <1 honest sentence: what the body actually instructs Claude to do>
claims:   "<trigger phrases verbatim from description>"
fit_for:  <who this skill is built for — honest>
needs:    <tools/MCPs/services required>
assumes:  <hidden assumptions, or "none detected">
──────────────────────────────────────────────
scores{trigger,tool,task,value}: <T>,<To>,<Ta>,<V>
fit: <X.X>/10 · redundancy: <NONE | OVERLAPS: name (what)>
──────────────────────────────────────────────
<VERDICT BLOCK>
──────────────────────────────────────────────
<NEXT LINES>
```

**INSTALL AS-IS** (fit ≥ 7.0):
```
verdict: INSTALL AS-IS

gives you: <specific benefit in user's context>
           <second concrete benefit>
tip:       <one usage tip for user's stack or habits>

next: cp <found-path> ~/.claude/skills/<name>/SKILL.md   install now
next: /skill-eye <other-skill>                            evaluate another
```

**MODIFY THEN INSTALL** (fit 4.0–6.9):
```
verdict: MODIFY THEN INSTALL

works:    <what aligns with user's workflow>
mismatch: <what doesn't, specific reason>

change 1 — <one-phrase reason>
  REPLACE in description: "<original>" → "<rewritten for user's language>"

change 2 — <one-phrase reason>
  ADD after "<section>": "<content for user's stack>"

change 3 — <only if needed>

next: edit <found-path>                                   apply changes
next: cp <found-path> ~/.claude/skills/<name>/SKILL.md   then install
next: /skill-eye <other-skill>                            evaluate another
```
Max 3 changes. If more needed → reconsider as SKIP.

**SKIP** (fit < 4.0):
```
verdict: SKIP

primary:   <most important mismatch — specific>
secondary: <second reason if applicable>
need:      <what a skill would need to do to be useful>
existing:  <installed skill covering this area, or "none found">

next: /skill-eye <better-fit>    try more relevant skill
next: /skill-eye <other-skill>   evaluate something else
```

---

## `--detailed` Flag

Append after verdict when detailed_mode = true:
```
── DETAILED BREAKDOWN ──────────────────────
trigger_analysis:
  user_phrases:   [<from history.jsonl>]
  skill_triggers: [<from description field>]
  overlap:        <matching phrases, or "none">

tool_analysis:
  required:   [<from allowed-tools + body refs>]
  available:  [<user's mcp_available>]
  missing:    [<gap list, or "none">]

body_summary: <extended summary — not truncated>
[body: <N>/<total> lines shown]   ← if truncation applied
```

---

## Edge Cases

**Skill found in multiple locations** → list all paths with labels, ask which to evaluate.

**Frontmatter only, no body:**
```
behavior: no body — frontmatter only; actual behavior in use is unknown
```

**Model-invoked skill** (no `argument-hint`):
```
trigger note: model-invoked — fires on context match; trigger alignment scored on description vs. user topics
```

**owner/repo with no `skills/` directory** → structured error with next step to try repo root URL.

**Very large SKILL.md** → truncation applies; note line counts; extract only what matters.

---

## Phase 5 — Discover Mode (`--discover`)

**Step 1 — Context:** Read `~/.claude/history.jsonl` last 40 lines (parse `display`). Glob Standard for installed skills. Ask the 2 standard profile questions if role/stack unknown.

**Step 2 — Scan in parallel:**
1. Installed skills with trigger alignment ≥ 6 not in recent history → "hidden gems"
2. Community repos — GitHub API tree, find SKILL.md files, fetch first 20 lines each:
   - `kunchenguid/axi` · `multica-ai/andrej-karpathy-skills` · `supabase/agent-skills` · `Povofarjun/skill-eye` · `anthropics/claude-code`

**Step 3 — Score and rank:** Quick fit = (trigger alignment + task relevance) / 2. Skip fit < 3.0. Skip already installed AND in recent history. Show top 5.

```
skill-eye discover · <role> · <stack>
searched: <N> installed · <M> community repos · <total> candidates
──────────────────────────────────────────────
<fit>/10  <name>    <source>    <one-line value>
<fit>/10  <name>    <source>    <one-line value>
[up to 5 rows]
──────────────────────────────────────────────
next: /skill-eye <rank-1>    full evaluation of top pick
next: /skill-eye <rank-2>    full evaluation of #2
next: /skill-eye <repo> --batch    explore a repo in depth
```

`--detailed`: append all candidates below 3.0 with one-line skip reason each.

---

## Phase 6 — Audit Mode (`--audit`)

**Step 1 — Context:** Read `~/.claude/history.jsonl` last 40 lines. Glob Standard for installed skills.

**Step 2 — Score every installed skill:**
For each: read description + first 30 body lines. Check history.jsonl for name/trigger appearance.
Quick fit = (trigger alignment + task relevance) / 2.

Classify:
- `active` — quick fit ≥ 6.0 OR appears in history
- `dormant` — quick fit 3.0–5.9 AND not in history
- `zombie` — quick fit < 3.0 AND not in history
- `redundant` — trigger overlap with another installed skill

**Step 3 — Coverage gaps:** Scan `typical_prompts` for recurring verb+object pairs (3+ times) with no installed skill match. Report at most 3.

```
skill-eye audit · <N> skills
──────────────────────────────────────────────
active (<n>):    <name> · <name>
dormant (<n>):   <name> · <name>
zombie (<n>):    <name> · <name>
redundant (<n>): <name-a> ≈ <name-b>  (both trigger on "<phrase>")
──────────────────────────────────────────────
top removals:
  1. <name> — <specific reason>
  2. <name> — <specific reason>

gaps:
  1. "<recurring pattern>" → no skill covers this; try: /skill-eye --discover
  2. "<pattern>" → try: /skill-eye <known-skill-or-repo>
──────────────────────────────────────────────
next: /skill-eye --remove <zombie>    remove top zombie
next: /skill-eye --discover           find skills for gaps
next: /skill-eye --audit --detailed   full per-skill breakdown
```

`--detailed`: append full per-skill table: `<name>  fit: X.X  status: <status>  last seen: <N> days ago | never`

**Prune mode** (`--audit --prune`):
1. Run Steps 1–3, identify zombies.
2. Show list + ask confirmation:
   ```
   skill-eye prune · <N> zombies
   ──────────────────────────────────────────────
   <name>   <path>   last seen: never
   ──────────────────────────────────────────────
   remove all <N>? (y/N)
   ```
3. On confirm: run Phase 8 per zombie with force_mode = true. On item failure: record, continue.
4. Final:
   ```
   skill-eye prune · done
   removed: <N> · <M> manifest entries cleaned
   failed:  <K> — <name> (<reason>)   ← only if K > 0
   next: /skill-eye --audit    verify remaining set
   ```

---

## Phase 7 — Batch Mode (`<owner>/<repo> --batch`)

**Step 1 — Context:** Read `~/.claude/history.jsonl` last 40 lines for user profile.

**Step 2 — Discover:** GitHub API tree: `https://api.github.com/repos/<owner>/<repo>/git/trees/main?recursive=1`. Find all SKILL.md files. Fetch first 25 lines each (frontmatter only).

Zero found:
```
skill-eye: error — no skills found in <owner>/<repo>
detail: checked .agents/skills/, skills/, external_plugins/
next: /skill-eye <raw-url-to-SKILL.md>    evaluate a specific file
```

**Step 3 — Score:** Quick fit = (trigger + task relevance) / 2. Sort descending.

```
skill-eye batch · <owner>/<repo> · <N> skills
──────────────────────────────────────────────
rank  skill    fit    verdict              next
1     <name>   X.X    INSTALL AS-IS        /skill-eye <name>
2     <name>   X.X    MODIFY THEN INSTALL  /skill-eye <name>
3     <name>   X.X    SKIP                 low fit
──────────────────────────────────────────────
next: /skill-eye <rank-1>               full evaluation of top pick
next: /skill-eye --batch <other>/<repo> batch evaluate another repo
```

`--detailed`: append one-line reason per row.

---

## Phase 8 — Remove Mode (`--remove <name>`)

**Step 1 — Locate:** Glob these IN PARALLEL:
1. `~/.agents/skills/<name>/SKILL.md` → type: `npx-global`
2. `./.agents/skills/<name>/SKILL.md` → type: `npx-project` (current directory)
3. `~/.claude/plugins/*/<name>/SKILL.md` → type: `native-plugin`
4. `~/.claude/skills/<name>/SKILL.md` → type: `standalone`
5. Try case-insensitive if exact match fails.

Zero matches:
```
skill-eye: 0 results for "<name>"
searched: ~/.agents/skills/<name>/
          ./.agents/skills/<name>/
          ~/.claude/plugins/*/<name>/
          ~/.claude/skills/<name>/

next: /skill-eye --audit    scan all installed skills
```

Multiple matches → list all paths with type labels, ask: `which scope? (1/<N>)`

**Step 2 — Confirm** (skip if force_mode = true):
```
skill-eye remove · <name> · <type>
path:      <directory>
last seen: <N> days ago | never | unknown
install:   <type>
──────────────────────────────────────────────
<type-note>
──────────────────────────────────────────────
confirm? (y/N)
```
Type notes: `npx-global`: removes from `~/.agents/skills/` and cleans manifests · `npx-project`: removes from `./.agents/skills/` and cleans project manifests · `standalone`: removes directory only · `native-plugin`: ⚠ managed by plugin system — confirm removes SKILL.md only; verify plugin state after.

**Step 3 — Execute:**
Check npx: Bash: `command -v npx 2>/dev/null` · PowerShell: `Get-Command npx -ErrorAction SilentlyContinue`

If npx absent (npx types only):
```
skill-eye: error — npx not found
detail: removal requires npx (Node.js ≥18)
next: install Node.js from https://nodejs.org, then retry
      or manually: rm -rf ~/.agents/skills/<name>/
```

| type | command |
|------|---------|
| npx-global | `npx -y skills remove <name> --global --yes 2>&1` then verify `~/.agents/skills/<name>/` gone |
| npx-project | `npx -y skills remove <name> --yes 2>&1` then verify `./.agents/skills/<name>/` gone |
| standalone | `rm -rf ~/.claude/skills/<name>/` (Windows: `Remove-Item -Recurse -Force "$env:USERPROFILE\.claude\skills\<name>"`) |
| native-plugin | Delete `<resolved-skill-directory>/` only — not its parent. Add: `verify plugin state in Claude Code.` |

Surface any non-zero exit as:
```
skill-eye: error — <what failed>
detail: <reason or exit code>
next: <corrective command>
```

**Step 4 — Lockfile cleanup** (npx types only):
Search `~/` and `~/.agents/` (global) or `./` (project) for `.skills.json` and `skills-lock.json`.
For each found: parse JSON; exact match on `"<name>"` then suffix-scan keys ending `"/<name>"`; delete matching key; write back. If malformed: `manifest: <file> malformed — skipped`. Never delete the whole file. Never run `npx skills install`.

**Step 5 — Report:**
```
skill-eye remove · <name> · done
removed: <path>
manifest: <.skills.json cleaned · skills-lock.json cleaned | none found | malformed — skipped>
──────────────────────────────────────────────
next: /skill-eye --audit           verify your skill set
next: /skill-eye --remove <other>  remove another skill
```
Only report `cleaned` if file was found and modified.

---

## Phase 9 — Self-Update Mode (`--update`)

**Step 1 — Remote version:** Fetch `https://raw.githubusercontent.com/povofarjun/skill-eye/main/.claude-plugin/plugin.json`. Parse `version`. Read local from frontmatter `version:`.

Fetch fails:
```
skill-eye: error — could not reach GitHub
detail: check your network connection
next: /skill-eye --update    retry when online
```

Already up to date:
```
skill-eye v<version> · already up to date

next: /skill-eye --audit    audit your skill set
next: /skill-eye            return to dashboard
```
Stop.

**Step 2 — Scope:** Glob in parallel to find all install locations:
1. `~/.agents/skills/skill-eye/SKILL.md` → type: `npx-global`
2. `~/.claude/skills/skill-eye/SKILL.md` → type: `standalone`

If neither found:
```
skill-eye: error — cannot locate own install path
next: reinstall: npx skills add povofarjun/skill-eye
```

Build `install_locations` list with all found paths and their types.

**Step 3 — Confirm** (skip if force_mode = true):
```
skill-eye update · v<local> → v<remote>
──────────────────────────────────────────────
<for each install_location>
path:   <install-path>
method: <npx (with file-write fallback) | direct file write>
</for each>
──────────────────────────────────────────────
confirm? (y/N)
```

**Step 4 — Execute:** For each install location in `install_locations`:

*npx-global path:*
Check npx (same check as Phase 8). If npx present:
```bash
npx -y skills update skill-eye --global --yes 2>&1
```
If npx absent OR npx exits non-zero → fall through to Step 4b (direct file write for this path), and note `method: direct write (npx unavailable)` in report.

*standalone path OR fallback:* (Step 4b — Direct file write)
```
Fetch https://raw.githubusercontent.com/povofarjun/skill-eye/main/.agents/skills/skill-eye/SKILL.md
Write content to <install-path>
```
If fetch fails:
```
skill-eye: error — could not fetch latest SKILL.md from GitHub
detail: check your network connection
next: /skill-eye --update    retry when online
```

**Step 5 — Verify + report:**
For each updated path, re-read frontmatter `version:`.

All paths updated:
```
skill-eye update · done
v<old> → v<new>
──────────────────────────────────────────────
<for each install_location>
path:   <install-path>
method: <npx | direct write | direct write (npx unavailable)>
</for each>
──────────────────────────────────────────────
next: /skill-eye          see what's new
next: /skill-eye --audit  audit your skill set
```

Any path still at old version after update attempt:
```
skill-eye: warning — <path> still at v<old>
next: npx -y skills add povofarjun/skill-eye   reinstall to force latest
```
