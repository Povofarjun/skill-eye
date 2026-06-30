# skill-eye

A Claude Code skill that acts as a guardian for your skill setup. Built on the [AXI design principles](https://axi.md/) — token-efficient output, content-first behavior, contextual next-step disclosure, and structured errors.

Before you install any skill, `skill-eye` learns your actual workflow and tells you honestly whether that skill will help you — or just clutter your setup. After an audit surfaces zombies, `--remove` cleans them out in one step. And when a new version ships, `--update` upgrades itself — no terminal knowledge required.

---

## What it does

When you run `/skill-eye <skill-name>`, it:

1. **Gathers ambient context silently** — reads your project's CLAUDE.md and command history before asking you anything. Asks only what it couldn't infer.

2. **Tears apart the skill** — reads the candidate SKILL.md, extracts what it actually does (not just what it claims), what tools it requires, and what kind of user it was built for. Truncates large files intelligently rather than dumping them whole.

3. **Pre-computes aggregates** — counts your installed skills and flags potential overlaps before showing the report.

4. **Delivers a compact verdict** — scores the skill on four dimensions and outputs a structured, token-efficient report (not a wall of emoji markdown):

```
skill-eye · code-review · local
profile:  full-stack developer · TypeScript, Next.js, Supabase
catalog:  42 installed · 1 potential overlap(s)
──────────────────────────────────────────────
behavior: runs a multi-agent diff review scoring each finding by confidence, then
          reports only high-confidence bugs and simplification opportunities
claims:   "review the current diff for correctness bugs and reuse/simplification"
fit_for:  developers who ship code frequently and want pre-push automated review
needs:    Bash, Read, Glob, Grep (all standard)
assumes:  git repo with uncommitted or staged changes present
──────────────────────────────────────────────
scores{trigger,tool,task,value}: 8,10,9,8
fit: 8.8/10 · redundancy: NONE
──────────────────────────────────────────────
verdict: INSTALL AS-IS

gives you: automated review of every diff before you push — catches real bugs,
           not style nitpicks
           confidence scoring means you only act on findings that matter
tip:       run /code-review before every PR; pair with /verify for full coverage

next: cp ~/.claude/plugins/.../code-review/SKILL.md ~/.claude/skills/code-review/SKILL.md   install now
next: /skill-eye verify                                                                       evaluate another
```

---

## Install

```bash
npx skills add povofarjun/skill-eye
```

Or manually: copy the `skills/skill-eye/` directory into `~/.claude/skills/skill-eye/`.

---

## Usage

```
/skill-eye                           Show installed skill count + usage (content-first)
/skill-eye --help                    Same as above
/skill-eye <skill-name>              Evaluate an installed skill by name
/skill-eye <github-url>              Evaluate a skill from a raw GitHub URL
/skill-eye <owner/repo>              Browse and evaluate skills in a GitHub repo
/skill-eye <skill-name> --detailed   Expand with trigger analysis + tool gap breakdown
/skill-eye --remove <skill-name>     Remove an installed skill cleanly
/skill-eye --remove <skill-name> --force   Remove without confirmation prompt
/skill-eye --update                  Check for and apply the latest version
/skill-eye --update --force          Update without confirmation prompt
```

**Examples:**

```
/skill-eye code-review
/skill-eye verify --detailed
/skill-eye https://github.com/supabase/agent-skills/blob/main/skills/supabase/SKILL.md
/skill-eye supabase/agent-skills
/skill-eye --remove verify-old
/skill-eye --remove caveman --force
/skill-eye --update
```

---

## Verdicts

| Verdict | Fit Score | What you get |
|---------|-----------|-------------|
| `INSTALL AS-IS` | ≥ 7.0 | Benefits + usage tip + install command |
| `MODIFY THEN INSTALL` | 4.0–6.9 | Exact copy-pasteable changes to make first |
| `SKIP` | < 4.0 | Why it misses + what you'd actually need |

Every response ends with `next:` lines — concrete commands to run, never a dead end.

---

## AXI Principles Applied

skill-eye is built on all 10 [AXI design principles](https://axi.md/):

| Principle | How skill-eye applies it |
|-----------|--------------------------|
| Token-efficient output | Compact `field: value` lines; no emoji tables |
| Minimal default schemas | 6 key fields by default; `--detailed` expands |
| Content truncation | Large SKILL.md files truncated at 120/150 lines with hint |
| Pre-computed aggregates | Installed count + overlap candidates shown before report |
| Definitive empty states | Explicit "0 results" with search paths shown |
| Structured errors | `error: … / detail: … / next: …` format |
| Ambient context | Reads CLAUDE.md + history silently before asking anything |
| Content first | No-args shows live state, not a help wall |
| Contextual disclosure | Every response ends with `next:` command lines |
| Consistent `--help` | `/skill-eye --help` = `/skill-eye` = content-first response |

---

## How it learns about you

skill-eye reads:
- **CLAUDE.md** in your current project (if it exists)
- **~/.claude/history.jsonl** — your last 40 commands to infer patterns

It then asks 2–3 targeted questions to fill remaining gaps. All analysis happens locally in your Claude Code session — no data leaves your machine.

---

## License

MIT © [povofarjun](https://github.com/povofarjun)
