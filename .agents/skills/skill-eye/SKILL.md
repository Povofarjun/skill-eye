---
name: skill-eye
description: >
  Evaluates any AI agent skill against your actual workflow before you install it, and monitors
  the health of the skills you already have. Agent-agnostic: works under claude, codex, opencode,
  pi, and grok — it detects the running agent, resolves skill paths accordingly, and speaks the
  agent's own invocation syntax. Acts as a guardian: learns what you truly do day-to-day, then
  tears apart the candidate skill to answer "will this actually help you?" and delivers a compact
  verdict (install as-is / modify-then-install / skip) with specific, copy-pasteable changes.
  Use when the user says "should I install X skill", "is this skill worth it for me",
  "evaluate this skill", "review skill X", "check if Y skill fits my workflow",
  or pastes a SKILL.md URL or GitHub repo path to assess.
  Also use when the user says "what skills should I install", "recommend skills for me",
  "discover skills", "audit my skills", "which of my skills am I not using",
  "inspect skill X", "show skill anatomy", "is this skill working",
  "evaluate all skills in this repo", or "batch evaluate".
argument-hint: "[--help] [--discover] [--audit] [--batch] [--inspect <name>] [--detailed] [--remove [--force]] [--update [--force]] <skill-name|github-url|owner/repo>"
allowed-tools: Bash, Read, Write, Edit, Glob, Grep
disable-model-invocation: true
version: 0.2.6
---

# skill-eye — The Skill Guardian

AXI protocol: token-efficient output, content-first, contextual `next:` lines, structured errors.

---

## Agent Detection

Run this once, before anything else. It sets four variables used by every phase: `AGENT`, `AGENT_PREFIX`, `HISTORY_PATH`, `history_available`. Never emit an error solely because detection is inconclusive — `unknown` is a valid result and all features continue via graceful degradation.

Check signals in this order; **first match wins**:

| # | Signal | AGENT | AGENT_PREFIX | HISTORY_PATH |
|---|--------|-------|--------------|--------------|
| 1 | `HARNESS` env var set | value of `HARNESS` | prefix for that value (below) | `HARNESS_HISTORY` if set, else per-agent default |
| 2 | Glob `~/.claude/history.jsonl` returns a result | `claude` | `/` | `~/.claude/history.jsonl` |
| 3 | `CODEX_HOME` env set OR `~/.codex/` dir exists | `codex` | `$` | (none) |
| 4 | `OPENCODE_HOME` env set OR `~/.opencode/` dir exists | `opencode` | `\` | (none) |
| 5 | `PI_HOME` env set OR `~/.pi/` dir exists | `pi` | `>` | (none) |
| 6 | `GROK_HOME` env set OR `~/.grok/` dir exists | `grok` | `~` | (none) |
| 7 | no match | `unknown` | `/` | (none) |

`AGENT_PREFIX` by agent name: `claude` → `/` · `codex` → `$` · `opencode` → `\` · `pi` → `>` · `grok` → `~` · `unknown` or unrecognized `HARNESS` value → `/`.

**History path resolution:** if `HARNESS_HISTORY` env var is set, HISTORY_PATH = its value (overrides detection). Else if AGENT = `claude`, HISTORY_PATH = `~/.claude/history.jsonl`. Else HISTORY_PATH = `` (empty).

**history_available:** `true` when HISTORY_PATH is non-empty AND the file exists; otherwise `false`.

Anti: never exit, never return a raw error, when AGENT resolves to `unknown` or when `HARNESS` holds an unrecognized value — proceed with the universal path set and `AGENT_PREFIX = /`.

---

## Path Resolution

Single source of path truth. The active skill-path set is a function of `AGENT`. This set is used by **every** phase that locates skills (Phase 0 overview, Phase 2 acquire, Phase 6 audit, Phase 8 remove, Phase 9 self-locate, Phase 10 inspect).

**Always included (all agents):**
```
~/.agents/skills/*/SKILL.md
./.agents/skills/*/SKILL.md
```

**Included only for the detected agent:**

| AGENT | Additional paths |
|-------|------------------|
| `claude` | `~/.claude/skills/*/SKILL.md` · `~/.claude/plugins/*/skills/*/SKILL.md` |
| `codex` | `~/.codex/skills/*/SKILL.md` |
| `opencode` | `~/.opencode/skills/*/SKILL.md` |
| `pi` | `~/.pi/skills/*/SKILL.md` |
| `grok` | `~/.grok/skills/*/SKILL.md` |
| `unknown` | (none beyond the universal two) |

Rules:
- **Run all active path globs in parallel** (not sequentially).
- A path that does not exist yields **zero results**, never an error condition.
- **Deduplicate by skill name** — the directory basename containing SKILL.md. Same name in multiple paths = 1 unique skill.
- `N` = count of unique skill names. `M` = count of active paths that returned ≥1 match.
- Anti: when AGENT = `codex` (or any non-claude), `~/.claude/` paths are **not** in the glob set.
- Anti: never surface a raw permission-denied error — wrap any access failure as a structured note (`note: <path> unreadable — skipped`) and continue.

---

## History Abstraction

All history access flows through this one block. When `history_available = true`, read the **last 40 lines** of HISTORY_PATH and parse the `display` field of each line to extract recurring commands, topics, and skill invocations. When `history_available = false`, every history-dependent output degrades cleanly:

| Consumer | Degradation when `history_available = false` |
|----------|-----------------------------------------------|
| Phase 0 | last-used column shows `—` for every skill |
| Phase 1 | ask both standard profile questions (no ambient inference) |
| Phase 5 (discover) | skip the history-read step; proceed with profile questions only |
| Phase 6 (audit) | classify by quick-fit only; drop the history component of active/dormant/zombie |

**Never** output a file-not-found error for a history path. When HISTORY_PATH is empty or the file is absent, that is simply `history_available = false` — a displayable condition, not an exception.

---

## Working-Condition Check

Reusable procedure. Called from Phase 0, Phase 6, and Phase 10. Given a SKILL.md path (or a batch of them):

1. Read frontmatter — extract the `allowed-tools` list.
2. Grep the body for tool references: `Bash`, `Read`, `Write`, `Edit`, `Glob`, `Grep`, `WebFetch`, `WebSearch`, `mcp__*`.
3. **Standard tools** — always treated as available: `Read`, `Write`, `Edit`, `Glob`, `Grep`, `Bash`.
4. **MCP tools** (`mcp__*`) — flagged as "may require MCP server"; not assumed present.

Classification:

| Badge | Condition |
|-------|-----------|
| `healthy` | every tool in `allowed-tools` is a standard tool or confirmed available |
| `degraded` | all `allowed-tools` present, but the body references ≥1 non-standard tool that is **not** in `allowed-tools` (optional tool missing) |
| `broken` | ≥1 tool listed in `allowed-tools` is non-standard and not confirmed available |
| `parse-error` | frontmatter YAML is malformed and cannot be read |

Budget: across **all** skills in a single Phase 0 run, use **≤3 Glob+Grep calls total**. Strategy: batch the frontmatter reads and body scans (one Grep over the resolved SKILL.md set for `allowed-tools`, one for body tool references) rather than per-skill calls.

Anti: never mark a skill `broken` when it uses only standard tools. Working-condition analysis is **static** — it never executes a skill and never makes a network call.

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
| `--inspect` | `<name>` | Phase 10, name = second token |
| `--inspect` | (absent) | error: `missing skill name — usage: <AGENT_PREFIX>skill-eye --inspect <name>` |
| `--remove` | (absent) | error: `missing skill name — usage: <AGENT_PREFIX>skill-eye --remove <name>` |
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
next: <AGENT_PREFIX>skill-eye    show usage
```

---

## Phase 0 — Overview (no-args / --help)

Content-first: show the skill list before asking anything.

**Step 1 — Detect.** Run Agent Detection and Path Resolution.

**Step 2 — Glob.** Run all active paths in parallel. Deduplicate by skill name. N = unique names, M = paths with ≥1 match.

**Step 3 — Condition.** Run the Working-Condition Check across the resolved set (batch, ≤3 Glob+Grep calls total).

**Step 4 — History.** When `history_available = true`, read history (History Abstraction) for last-used dates per skill.

**Step 5 — Version check** (silent, max 3s, skip on any failure): Fetch `https://raw.githubusercontent.com/povofarjun/skill-eye/main/.claude-plugin/plugin.json`. Parse `version`. If remote > local frontmatter `version:`, set update_available = true.

Output:
```
skill-eye v<version> [agent: <AGENT>]
[update: v<remote> available — run <AGENT_PREFIX>skill-eye --update]   ← only if update_available
installed: <N> skills across <M> directories

name              type    condition  last used    description
──────────────────────────────────────────────────────────────
<name>            user    healthy    2 days ago   <description truncated to 55 chars>
<name>            model   healthy    never        <description>
<name>            user    degraded   —            <description>
<name>            user    broken     3 days ago   <description>
──────────────────────────────────────────────────────────────
[last eval: <name> · <verdict> · <N> days ago]   ← only if found in history

usage: <AGENT_PREFIX>skill-eye <skill-name|github-url|owner/repo> [--detailed]
       <AGENT_PREFIX>skill-eye --inspect <name>   show skill anatomy
       <AGENT_PREFIX>skill-eye --discover          recommend skills for your workflow
       <AGENT_PREFIX>skill-eye --audit             review all installed skills
       <AGENT_PREFIX>skill-eye --update            update to latest version

next: <AGENT_PREFIX>skill-eye --discover    find skills matched to your workflow
next: <AGENT_PREFIX>skill-eye --audit       check which installed skills you actually use
next: <AGENT_PREFIX>skill-eye --inspect <first-skill-name>   inspect a skill
```

Rules:
- **Sort:** healthy first, degraded second, broken last (`parse-error` after broken).
- **type:** `user` if the skill declares `argument-hint`, else `model`.
- **last used:** date string (`2 days ago`) when history available and skill seen; `never` when history available but skill unseen; `—` when no history file.
- **last-eval line** shown only when found in history.
- Use real counts and the actual frontmatter version — never hardcode either.
- **Empty state (N = 0):**
  ```
  skill-eye v<version> [agent: <AGENT>]
  installed: 0 skills

  next: <AGENT_PREFIX>skill-eye --discover        find skills for your workflow
  next: <AGENT_PREFIX>skill-eye <owner>/<repo>    browse a skills repo
  ```

Anti: never crash on malformed SKILL.md frontmatter — show `parse-error` in the condition column and continue. Never output more than 4 lines per skill. Never show 0 skills when skills exist in non-claude paths.

---

## Phase 1 — Ambient Context Gathering

Runs only when Phase 2+ is invoked; Phase 0 skips it.

Run in parallel before asking anything:
1. Read `./CLAUDE.md` and `./.claude/CLAUDE.md` — extract tech stack, project type, conventions.
2. History Abstraction — when `history_available = true`, parse last 40 lines for recurring commands and topics. When `false`, skip this read entirely.
3. Reuse the Phase 0 glob results for installed skills (no re-glob) for the redundancy check.

Ask only what passive context didn't answer. Always ask:
- "What's your primary role and what do you build day-to-day?"
- "What are the 2–3 things you use <AGENT> for most?"   ← substitute the detected agent name

Ask if still unclear: tech stack · MCP integrations · biggest friction.

When `history_available = false`: ask both standard questions above; do not attempt ambient inference.

Store internally as: `role / stack / daily_tasks[] / mcp_available[] / pain_points[] / typical_prompts[] / installed_skills[]`. Never re-ask for information already present in CLAUDE.md.

---

## Phase 2 — Acquire & Truncate the Target Skill

**Named skill** (`<name>`): glob the active Path Resolution paths with `<name>` in place of `*`. Multiple matches → list paths, ask which.

**GitHub URL** (`https://...`): Fetch raw content. Convert blob URLs to raw.githubusercontent.com.

**owner/repo**: Fetch via GitHub API tree — find `skills/` directory, list available skills, ask which to evaluate.

**Truncation:** If body > 150 lines, read first 120 and note `[body: 120/<total> lines shown]`. Extract: what the agent is instructed to do, tool references, dependencies, assumptions.

**Not found** (paths shown are the actual agent-aware set that was searched):
```
skill-eye: 0 results for "<name>"
searched: <each active Path Resolution path with <name> substituted, one per line>

next: <AGENT_PREFIX>skill-eye <github-url>    fetch directly from GitHub
next: <AGENT_PREFIX>skill-eye <owner>/<repo>  browse skills in a repo
```

**Other errors:**
```
skill-eye: error — <what failed>
detail: <specific reason>
next: <one corrective command>
```

Never expose raw HTTP status codes — wrap in the structured error format.

---

## Phase 3 — Tear It Apart

Pre-compute:
- `installed_count`: N from Path Resolution.
- `overlap_candidates`: installed skills whose description shares significant keywords with target.

Read `references/evaluation-rubric.md` for scoring edge cases.

Score 0–10 against User Profile:
1. **Trigger Alignment** — how often do skill triggers match `typical_prompts`? (0 = never · 10 = multiple times/week). When `history_available = false`, score on description vs. `daily_tasks` only.
2. **Tool Fit** — does user have required tools/MCPs? Hard-penalize ≤3 if a critical dep is absent. Extract deps from both `allowed-tools` frontmatter AND body text references. (0 = core deps missing · 10 = all present and in use)
3. **Task Relevance** — does skill address a `daily_task` or `pain_point`? Most important. (0 = wrong domain · 10 = solves named pain point)
4. **Value Density** — would user actually invoke it, or forget it? (0 = install-and-forget · 10 = used multiple times/week)

Store internally (do not output):
```
scores: trigger=<T> tool=<To> task=<Ta> value=<V>
fit: <(T+To+Ta+V)/4, 1 decimal>
redundancy: NONE | OVERLAPS: <skill-name> (<what overlaps>)
verdict: INSTALL AS-IS | MODIFY THEN INSTALL | SKIP
```
Never auto-assume an MCP is available unless the user confirmed it in Phase 1. → proceed to Phase 4.

---

## Phase 4 — Compact Report + Verdict

```
skill-eye · <name> · <local|url|repo>
profile:  <role> · <stack>
catalog:  <installed_count> installed · <overlap_candidates> potential overlap(s)
──────────────────────────────────────────────
behavior: <1 honest sentence: what the body actually instructs the agent to do>
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

next: cp <found-path> ~/.agents/skills/<name>/SKILL.md   install now
next: <AGENT_PREFIX>skill-eye <other-skill>              evaluate another
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

next: edit <found-path>                                  apply changes
next: cp <found-path> ~/.agents/skills/<name>/SKILL.md   then install
next: <AGENT_PREFIX>skill-eye <other-skill>              evaluate another
```
Max 3 changes. If more needed → reconsider as SKIP.

**SKIP** (fit < 4.0):
```
verdict: SKIP

primary:   <most important mismatch — specific>
secondary: <second reason if applicable>
need:      <what a skill would need to do to be useful>
existing:  <installed skill covering this area, or "none found">

next: <AGENT_PREFIX>skill-eye <better-fit>    try more relevant skill
next: <AGENT_PREFIX>skill-eye <other-skill>   evaluate something else
```

No verdict block uses language like "might", "probably", or "seems". No verdict block outputs without ≥1 `next:` line.

---

## `--detailed` Flag

Append after verdict when detailed_mode = true:
```
── DETAILED BREAKDOWN ──────────────────────
trigger_analysis:
  user_phrases:   [<from history / daily_tasks>]
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

**Step 1 — Context:** History Abstraction — when `history_available = true`, read last 40 lines (parse `display`); when `false`, **skip the history read** and proceed with profile questions only. Glob active paths for installed skills. Ask the 2 standard profile questions if role/stack unknown.

**Step 2 — Scan in parallel:**
1. Installed skills with trigger alignment ≥ 6 not in recent history → "hidden gems" (only meaningful when history available).
2. Community repos — GitHub API tree, find SKILL.md files, fetch first 20 lines each:
   - `kunchenguid/axi` · `multica-ai/andrej-karpathy-skills` · `supabase/agent-skills` · `povofarjun/skill-eye` · `anthropics/claude-code`

**Step 3 — Score and rank:** Quick fit = (trigger alignment + task relevance) / 2. Skip fit < 3.0. Skip already installed AND in recent history. Show top 5.

```
skill-eye discover · <role> · <stack>
searched: <N> installed · <M> community repos · <total> candidates
──────────────────────────────────────────────
<fit>/10  <name>    <source>    <one-line value>
<fit>/10  <name>    <source>    <one-line value>
[up to 5 rows]
──────────────────────────────────────────────
next: <AGENT_PREFIX>skill-eye <rank-1>          full evaluation of top pick
next: <AGENT_PREFIX>skill-eye <rank-2>          full evaluation of #2
next: <AGENT_PREFIX>skill-eye <repo> --batch    explore a repo in depth
```

`--detailed`: append all candidates below 3.0 with one-line skip reason each. Never recommend a skill with fit ≥ 6 that is already installed and active.

---

## Phase 6 — Audit Mode (`--audit`)

**Step 1 — Context:** History Abstraction (graceful when absent). Glob active paths for installed skills.

**Step 2 — Score + condition every installed skill:**
For each: read description + first 30 body lines. Check history for name/trigger appearance. Run the Working-Condition Check (reuse the section) to get a `condition` badge per skill.
Quick fit = (trigger alignment + task relevance) / 2.

Classify:
- `active` — quick fit ≥ 6.0 OR appears in history
- `dormant` — quick fit 3.0–5.9 AND not in history
- `zombie` — quick fit < 3.0 AND not in history
- `redundant` — trigger overlap with another installed skill

When `history_available = false`: classify `active`/`dormant`/`zombie` by **quick-fit only** (no history component). A model-invoked skill is never classed `zombie` on explicit-invocation history alone.

**Step 3 — Coverage gaps:** Scan `typical_prompts` (or skill types when no history) for recurring verb+object pairs (3+ times) with no installed skill match. Report at most 3.

```
skill-eye audit · <N> skills
──────────────────────────────────────────────
active(<n>)  dormant(<n>)  zombie(<n>)  redundant(<n>)  healthy(<n>)  degraded(<n>)  broken(<n>)
──────────────────────────────────────────────
active:    <name> [healthy] · <name> [degraded]
dormant:   <name> [healthy] · <name> [broken]
zombie:    <name> [healthy] · <name> [healthy]
redundant: <name-a> ≈ <name-b>  (both trigger on "<phrase>")
──────────────────────────────────────────────
top removals:
  1. <name> — <specific reason>
  2. <name> — <specific reason>

gaps:
  1. "<recurring pattern>" → no skill covers this; try: <AGENT_PREFIX>skill-eye --discover
  2. "<pattern>" → try: <AGENT_PREFIX>skill-eye <known-skill-or-repo>
──────────────────────────────────────────────
next: <AGENT_PREFIX>skill-eye --remove <zombie>    remove top zombie
next: <AGENT_PREFIX>skill-eye --discover           find skills for gaps
next: <AGENT_PREFIX>skill-eye --audit --detailed   full per-skill breakdown
```

`--detailed`: append full per-skill table: `<name>  fit: X.X  status: <status>  condition: <badge>  last seen: <N> days ago | never | —`

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
   next: <AGENT_PREFIX>skill-eye --audit    verify remaining set
   ```
Never remove a skill without confirmation unless `--force`.

---

## Phase 7 — Batch Mode (`<owner>/<repo> --batch`)

**Step 1 — Context:** History Abstraction for user profile (graceful when absent).

**Step 2 — Discover:** GitHub API tree: `https://api.github.com/repos/<owner>/<repo>/git/trees/main?recursive=1`. Find all SKILL.md files. Fetch first 25 lines each (frontmatter only — never fetch full body).

Zero found:
```
skill-eye: error — no skills found in <owner>/<repo>
detail: checked .agents/skills/, skills/, external_plugins/
next: <AGENT_PREFIX>skill-eye <raw-url-to-SKILL.md>    evaluate a specific file
```

**Step 3 — Score:** Quick fit = (trigger + task relevance) / 2. Sort descending.

```
skill-eye batch · <owner>/<repo> · <N> skills
──────────────────────────────────────────────
rank  skill    fit    verdict              next
1     <name>   X.X    INSTALL AS-IS        <AGENT_PREFIX>skill-eye <name>
2     <name>   X.X    MODIFY THEN INSTALL  <AGENT_PREFIX>skill-eye <name>
3     <name>   X.X    SKIP                 low fit
──────────────────────────────────────────────
next: <AGENT_PREFIX>skill-eye <rank-1>                full evaluation of top pick
next: <AGENT_PREFIX>skill-eye --batch <other>/<repo>  batch evaluate another repo
```

`--detailed`: append one-line reason per row. Never crash on malformed frontmatter → show `parse-error` in the verdict column.

---

## Phase 8 — Remove Mode (`--remove <name>`)

**Step 1 — Locate** (agent-aware Path Resolution). Glob IN PARALLEL:
1. `~/.agents/skills/<name>/SKILL.md` → type: `npx-global`
2. `./.agents/skills/<name>/SKILL.md` → type: `npx-project` (current directory)
3. [claude only] `~/.claude/plugins/*/<name>/SKILL.md` → type: `native-plugin`
4. [claude only] `~/.claude/skills/<name>/SKILL.md` → type: `standalone`
5. [other agents] `~/<agent-home>/skills/<name>/SKILL.md` → type: `standalone`  (e.g. `~/.codex/skills/<name>/` when AGENT = codex)
6. Try case-insensitive if exact match fails.

Zero matches (list the actual searched paths for the detected agent):
```
skill-eye: 0 results for "<name>"
searched: <each active Path Resolution path with <name> substituted, one per line>

next: <AGENT_PREFIX>skill-eye --audit    scan all installed skills
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
| standalone | `rm -rf <resolved-path>` (Windows: `Remove-Item -Recurse -Force "<resolved-path>"`) |
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
next: <AGENT_PREFIX>skill-eye --audit           verify your skill set
next: <AGENT_PREFIX>skill-eye --remove <other>  remove another skill
```
Only report `cleaned` if file was found and modified. Never delete the parent plugin directory for `native-plugin`.

---

## Phase 9 — Self-Update Mode (`--update [--force]`)

How a Claude Code agent updates a skill: it fetches the new SKILL.md from GitHub using
Bash curl, then uses the **Write tool** to overwrite each install path directly. No package
manager is required. The Write tool is always available and is the only reliable write path.

**Step 1 — Remote version** (max 3s, skip-on-failure):

Run Bash:
```bash
curl -fsSL "https://raw.githubusercontent.com/povofarjun/skill-eye/main/.claude-plugin/plugin.json"
```
Parse `version` field → `remote_version`.
Read `local_version` from this file's own frontmatter `version:` field.

If curl exits non-zero or returns empty:
```
skill-eye: error — could not reach GitHub
detail: <curl error or empty response>
next: <AGENT_PREFIX>skill-eye --update         retry when online
next: <AGENT_PREFIX>skill-eye --update --force  skip version check and force overwrite
```
Stop.

When `force_mode = false` AND `remote_version ≤ local_version`:
```
skill-eye v<local_version> · already up to date

next: <AGENT_PREFIX>skill-eye --update --force  overwrite anyway (clears stale installs)
next: <AGENT_PREFIX>skill-eye --audit           audit your skill set
```
Stop. **When `force_mode = true`, skip this exit entirely — always continue to Step 2.**

**Step 2 — Locate install paths** (Glob all in parallel):
```
~/.agents/skills/skill-eye/SKILL.md
./.agents/skills/skill-eye/SKILL.md
~/.claude/skills/skill-eye/SKILL.md
~/.claude/plugins/*/skills/skill-eye/SKILL.md
```
Collect every path that resolves to an existing file. Store as `install_locations[]`.

Zero found:
```
skill-eye: error — cannot locate any install of skill-eye
detail: searched ~/.agents/, ./.agents/, ~/.claude/skills/, ~/.claude/plugins/
next: npx -y skills add povofarjun/skill-eye    reinstall from scratch
```
Stop.

**Step 3 — Confirm** (skip entirely when `force_mode = true`):
```
skill-eye update · v<local_version> → v<remote_version>
──────────────────────────────────────────────
<one line per install_location:>
path: <path>
──────────────────────────────────────────────
confirm? (y/N)
```
On N or no response: stop with `cancelled.`

**Step 4 — Fetch new SKILL.md**:

Run Bash:
```bash
curl -fsSL "https://raw.githubusercontent.com/povofarjun/skill-eye/main/.agents/skills/skill-eye/SKILL.md"
```
Store the full response body as `new_content`. Do **not** truncate or modify it.

If curl exits non-zero or returns empty:
```
skill-eye: error — could not download new SKILL.md
detail: <curl error or empty response>
next: <AGENT_PREFIX>skill-eye --update    retry when online
```
Stop. Do not attempt writes with empty content.

**Step 5 — Write to each install path**:

For each path in `install_locations`:
- Use the **Write tool** to write `new_content` to the exact path.
- Write tool replaces the file contents in full — no append, no merge.
- If Write fails for a path: record `failed: <path> — <reason>` and continue to the next path.

**Step 6 — Verify**:

For each path written in Step 5:
- Use the **Read tool** to read the first 20 lines of the file.
- Check the `version:` line in the frontmatter.
- If value = `remote_version` → mark ✓
- If value ≠ `remote_version` → mark ✗ (write may have failed silently)

**Report**:
```
skill-eye update · done
v<local_version> → v<remote_version>
──────────────────────────────────────────────
<one line per install_location:>
path:   <path>
result: updated ✓  |  failed ✗ — <reason>
──────────────────────────────────────────────
[warning: <N> path(s) still at old version — see above]   ← only if any ✗
next: <AGENT_PREFIX>skill-eye          see what's new
next: <AGENT_PREFIX>skill-eye --audit  audit your skill set
```

If every path shows ✗:
```
skill-eye: error — no paths updated successfully
detail: Write tool may not have permission, or paths are read-only
next: npx -y skills add povofarjun/skill-eye   reinstall from scratch
```

---

## Phase 10 — Inspect Mode (`--inspect <name>`)

Expose a skill's execution anatomy without running it. Never execute or simulate the skill's logic. Without `--detailed`, output never exceeds 20 lines.

**Step 1 — Locate:** Use agent-aware Path Resolution (same lookup as Phase 2 named skill). Multiple matches → list, ask which. Zero matches → the Phase 2 not-found structured error with the actual searched paths.

**Step 2 — Read SKILL.md.** Extract:
- `name`, `version`, `description` (frontmatter)
- `argument-hint` (full, not truncated) — if absent: type = `model`
- `allowed-tools` list (frontmatter)
- `disable-model-invocation` flag
- Body section headers (all `## ` lines)
- Body tool references (Bash, Read, Write, Edit, Glob, Grep, WebFetch, WebSearch, `mcp__*`)

**Step 3 — Condition:** Run the Working-Condition Check for this one skill.

**Step 4 — History:** When `history_available = true`, check history for last-used.

Output (max 20 lines without `--detailed`):
```
skill-eye inspect · <name> · [agent: <AGENT>]
──────────────────────────────────────────────
version:    <v>
type:       <user-invoked | model-invoked>
invoked:    <AGENT_PREFIX><name>
triggers:   <comma-separated phrases from description, max 80 chars>
──────────────────────────────────────────────
argument-hint: <full value, or "not declared">
allowed-tools: <list from frontmatter, or "not declared">
body-tool-refs: <tools mentioned in body beyond allowed-tools, or "none">
──────────────────────────────────────────────
condition:  <healthy | degraded | broken> — <one-line reason>
deps:       available: <list> | missing: <list or "none">
──────────────────────────────────────────────
body-sections: <## Section headers found, comma-separated>
last-used:  <N days ago | never | — (no history)>
──────────────────────────────────────────────
next: <AGENT_PREFIX>skill-eye <name>           full evaluation
next: <AGENT_PREFIX>skill-eye --audit          audit all skills
```

`--detailed`: append the full body structure — every `## ` heading plus the first line of each section:
```
── BODY STRUCTURE ──────────────────────────
## <heading>
  <first line of section>
## <heading>
  <first line of section>
[...]
```

Anti: never execute or simulate the skill. Never exceed 20 lines without `--detailed`.
