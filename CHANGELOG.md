# Changelog

All notable changes to skill-eye are documented here.

## 0.2.7 — 2026-07-21

- Added: static Risk Assessment — every evaluate, `--audit`, and `--inspect` run now scores a
  skill's capability surface (`low`/`medium`/`high`/`critical`) against five signals: exfil
  surface (local-power tool + network tool declared together), indirect prompt-injection surface
  (fetches remote content and acts on instructions found inside it), unguarded destructive
  commands (no confirmation gate near `rm -rf`/`--force`/etc.), credential-path access, and
  silent model-invoked triggers with write/exec power. Purely textual/static — never executes or
  fetches anything to test a signal.
- Changed: `--audit` gains a `flagged(<n>)` bucket and a `flagged:` section listing any installed
  skill at `high`/`critical` risk, independent of its active/dormant/zombie status — a skill you
  use daily can still be a risk you haven't reviewed.
- Changed: Phase 0's no-args dashboard now shows a one-line `⚠ risk:` warning when any installed
  skill is flagged `high`/`critical`, computed via the same batched grep pass as the existing
  working-condition check (no added tool-call budget).
- Changed: evaluation verdicts are now risk-aware — `risk = critical` caps the verdict at
  `MODIFY THEN INSTALL` regardless of fit score, with a required change addressing the flagged
  capability; `risk = high` still allows `INSTALL AS-IS` but forces a mandatory `caution:` line.
- Docs: `references/evaluation-rubric.md` gains a full false-positive-discipline guide for each
  risk signal; README documents the new scoring dimension.

## 0.2.6 — 2026-07-14

- Fixed: added explicit `allowed-tools: Bash, Read, Write, Edit, Glob, Grep` to SKILL.md frontmatter. Its absence meant the skill declared no tool boundary at all — automated security scanners (Snyk via `npx skills add`) flagged this as Critical Risk, since an agent-skill markdown file with unrestricted implicit tool access, `rm -rf`/`Remove-Item` destructive commands, and a self-overwriting `curl`+Write update mechanism is exactly the pattern those scanners are built to catch. The six tools listed are the actual, complete set the skill body uses — nothing was added or removed from its real behavior.
- Docs: added a "Security & Permissions" section to README.md spelling out exactly what each destructive/network-capable mode does, so users can make an informed install decision instead of discovering it via a scanner warning.

## 0.2.5 — 2026-07-14

- Fixed: GitHub username casing normalized to lowercase `povofarjun` throughout SKILL.md (mixed casing in `raw.githubusercontent.com` URLs used by `--update`).
- Docs: README.md rewritten for accuracy — full usage table covering `--discover`, `--audit`/`--audit --prune`, `--batch`, `--inspect` (previously undocumented), agent-agnostic support with a per-agent invocation table, corrected manual-install path, working-condition badge documentation, and an accurate network-call disclosure.
- Added: `LICENSE` file (MIT) — previously only claimed in README/plugin.json with no file present.

## 0.2.4 — 2026-07-01

- Fixed: `--update` Phase 9 rewritten to fetch and overwrite via the Write tool directly (`curl` + `Write`), removing the dependency on `npx skills update` as the sole update path. `--force` now correctly bypasses the version check, not just the confirmation prompt.

## 0.2.3 — 2026-07-01

- Fixed: `--force` previously only skipped the confirmation step; the "already up to date" exit in Step 1 fired before `--force` had any effect. `--force` now bypasses the version check entirely.

## 0.2.2 — 2026-07-01

- Fixed: versioning reverted from the short-lived `0.3.0` scheme back to `0.2.x` (see below) at user request. `--update` project-local path fixed.

## 0.3.0 — 2026-07-01 (superseded — see 0.2.2)

- Redesigned skill-eye as an agent-agnostic skill monitor: agent detection (claude/codex/opencode/pi/grok), multi-agent path resolution, rich Phase 0 overview with working-condition badges (healthy/degraded/broken), graceful degradation when history/network is unavailable, and a new `--inspect <name>` mode exposing a skill's execution anatomy.
- This version number was reverted the same day after a live regression report (`--update` broken); the feature set shipped instead under the 0.2.x line starting at 0.2.2.

## 0.2.1 — 2026-06-30

- Added: npx-first self-update with a direct file-write fallback across all install paths.

## 0.2.0 — 2026-06-30

- Refactored: AXI output deepening — fixed installed-skill count inconsistency and argument routing.

## 0.1.1 — 2026-06-30

- Added: `--discover`, `--audit`, `--batch` modes.
- Added: `--remove` and `--update` modes.
- Fixed: `argument-hint` YAML quoting (square brackets were invalid unquoted).
- Fixed: install path corrected to `.agents/skills/` (the path `npx skills add` actually uses).

## 1.0 — 2026-06-30

- Initial release.
