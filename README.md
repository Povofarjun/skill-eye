# skill-eye

A skill guardian for AI coding agents. Built on the [AXI design principles](https://axi.md/) — token-efficient output, content-first behavior, contextual next-step disclosure, and structured errors.

Before you install any skill, `skill-eye` learns your actual workflow and tells you honestly whether that skill will help you — or just clutter your setup. It also monitors the health of the skills you already have, finds the ones you never use, and cleans up or updates itself, all with no terminal knowledge required.

**Agent-agnostic.** skill-eye detects which harness you're running — Claude Code, Codex, opencode, pi, or grok — and adapts its paths and invocation syntax automatically. The examples below use Claude Code's `/skill-eye` syntax; see [Supported Agents](#supported-agents) for the others.

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

Beyond one-off evaluation, `/skill-eye` with no arguments gives you a live health dashboard of everything installed:

```
skill-eye v0.2.6 [agent: claude]
installed: 5 skills across 2 directories

name              type    condition  last used    description
──────────────────────────────────────────────────────────────
code-review       user    healthy    2 days ago   review diffs for bugs and simplifications
verify            user    healthy    5 days ago   exercise a change end-to-end before commit
data-viz          model   healthy    never        chart/graph/dashboard design guidance
old-linter        user    degraded   —            lints staged files (optional MCP missing)
caveman           user    broken     30 days ago  writes commit messages in a dead style
──────────────────────────────────────────────────────────────
usage: /skill-eye <skill-name|github-url|owner/repo> [--detailed]
       /skill-eye --inspect <name>   show skill anatomy
       /skill-eye --discover          recommend skills for your workflow
       /skill-eye --audit             review all installed skills
       /skill-eye --update            update to latest version

next: /skill-eye --discover    find skills matched to your workflow
next: /skill-eye --audit       check which installed skills you actually use
next: /skill-eye --inspect code-review   inspect a skill
```

Every skill is tagged with a **working-condition badge** — `healthy` (all required tools available), `degraded` (an optional tool referenced in the body is missing), or `broken` (a declared dependency is unavailable) — computed via static analysis, never by executing the skill.

---

## Install

```bash
npx skills add povofarjun/skill-eye
```

Or manually: copy the `.agents/skills/skill-eye/` directory (including `references/`) into your agent's skills folder — `~/.claude/skills/skill-eye/` for Claude Code, `~/.codex/skills/skill-eye/` for Codex, and so on. See [Supported Agents](#supported-agents) for the full path per harness.

---

## Usage

```
/skill-eye                                 Show installed skill health dashboard (content-first)
/skill-eye --help                          Same as above
/skill-eye <skill-name>                    Evaluate an installed skill by name
/skill-eye <github-url>                    Evaluate a skill from a raw GitHub URL
/skill-eye <owner/repo>                    Browse and evaluate skills in a GitHub repo
/skill-eye <skill-name> --detailed         Expand with trigger analysis + tool gap breakdown
/skill-eye --discover                      Recommend skills matched to your workflow
/skill-eye --audit                         Classify every installed skill: active / dormant / zombie / redundant
/skill-eye --audit --prune                 Audit, then remove all zombies after one confirmation
/skill-eye --inspect <skill-name>          Show a skill's execution anatomy (tools, triggers, condition)
/skill-eye <owner/repo> --batch            Evaluate every skill in a repo, ranked by fit
/skill-eye --remove <skill-name>           Remove an installed skill cleanly
/skill-eye --remove <skill-name> --force   Remove without confirmation prompt
/skill-eye --update                        Check for and apply the latest version
/skill-eye --update --force                Update without confirmation prompt (also fixes stale installs)
```

**Examples:**

```
/skill-eye code-review
/skill-eye verify --detailed
/skill-eye https://github.com/supabase/agent-skills/blob/main/skills/supabase/SKILL.md
/skill-eye supabase/agent-skills
/skill-eye supabase/agent-skills --batch
/skill-eye --discover
/skill-eye --audit --prune
/skill-eye --inspect code-review
/skill-eye --remove verify-old
/skill-eye --remove caveman --force
/skill-eye --update
```

Running under Codex? Same commands, `$` prefix: `$skill-eye --audit`, `$skill-eye --discover`, etc. — see below.

---

## Supported Agents

skill-eye detects the running harness automatically (first match wins: `HARNESS` env var → Claude Code → Codex → opencode → pi → grok → `unknown`) and adapts its invocation prefix and skill-path resolution accordingly. No configuration needed.

| Agent | Invocation prefix | Skill path |
|-------|-------------------|------------|
| Claude Code | `/skill-eye` | `~/.claude/skills/skill-eye/` or `~/.claude/plugins/*/skills/skill-eye/` |
| Codex | `$skill-eye` | `~/.codex/skills/skill-eye/` |
| opencode | `\skill-eye` | `~/.opencode/skills/skill-eye/` |
| pi | `>skill-eye` | `~/.pi/skills/skill-eye/` |
| grok | `~skill-eye` | `~/.grok/skills/skill-eye/` |
| any other / unrecognized | `/skill-eye` | `~/.agents/skills/skill-eye/` (universal path, always checked) |

`~/.agents/skills/` and `./.agents/skills/` (project-local) are always checked regardless of agent — this is where `npx skills add` installs by default.

If your harness has no command history to learn from (only Claude Code's `~/.claude/history.jsonl` is read today), skill-eye degrades gracefully: it asks two direct profile questions instead of inferring from history, and every history-dependent output (last-used dates, "hidden gems" in `--discover`) shows a clean `—` instead of erroring.

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
| Content first | No-args shows a live health dashboard, not a help wall |
| Contextual disclosure | Every response ends with `next:` command lines |
| Consistent `--help` | `/skill-eye --help` = `/skill-eye` = content-first response |

---

## How it learns about you

skill-eye reads, when available:
- **CLAUDE.md** in your current project
- **`~/.claude/history.jsonl`** (Claude Code only) — your last 40 commands, to infer patterns without asking

It then asks 2–3 targeted questions to fill remaining gaps. skill-eye's only network calls are to GitHub, and only in these cases: every run of the no-args dashboard does a silent version check (capped at 3s, skipped on any failure); evaluating a skill by URL/name/repo, `--discover`, and `--batch` fetch the target skill's public source with no stated timeout; `--update` checks and downloads its own release with a 3s cap on the version check, but if that check or the download itself fails, it stops and shows you a structured error rather than failing silently. No data about your prompts, files, or profile is ever sent anywhere.

---

## Security & Permissions

skill-eye is a skill *manager* — it needs real system access to do its job, and you should know exactly what before installing it. Declared tools (`allowed-tools` in SKILL.md frontmatter): `Bash`, `Read`, `Write`, `Edit`, `Glob`, `Grep`. No MCP servers, no credentials, nothing beyond what's listed.

What that access is used for, concretely:

- **`--remove`** deletes skill directories (`rm -rf` / `Remove-Item -Recurse -Force`) and edits `.skills.json`/`skills-lock.json`. It always shows the exact path before deleting and asks for confirmation unless you pass `--force`.
- **`--update`** fetches its own SKILL.md from `raw.githubusercontent.com/povofarjun/skill-eye` over HTTPS and overwrites your local install with the Write tool — no build step, no code execution, just a file replaced with the published Markdown. There's no cryptographic signature check beyond HTTPS/GitHub's own integrity guarantees; if you don't trust GitHub's transport, don't run `--update` and instead re-clone and diff manually.
- **`--remove`/`--update`** shell out to `npx skills remove`/`npx -y skills add` for npx-managed installs — meaning it invokes the `npx skills` package manager on your behalf, scoped to the one skill you named.
- Evaluating a skill (by name, URL, or repo), `--discover`, and `--batch` fetch **public** GitHub content (the target skill's own SKILL.md) — never your prompts, files, or profile.

None of this runs unattended: every destructive action shows you the path/command first and asks `confirm? (y/N)` unless you explicitly pass `--force`. If you want zero-trust, read `.agents/skills/skill-eye/SKILL.md` yourself before installing — it's plain-English instructions to an agent, not compiled code, so there's nothing hidden that isn't in that file.

## Changelog

See [CHANGELOG.md](CHANGELOG.md) for release history.

## License

MIT © [povofarjun](https://github.com/povofarjun) — see [LICENSE](LICENSE).
