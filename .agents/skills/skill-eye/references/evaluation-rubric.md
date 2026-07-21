# skill-eye Evaluation Rubric

Reference for Phase 3 scoring. Load this when scoring is ambiguous or an edge case arises.

---

## Trigger Alignment — Detailed Scoring

The goal: would this skill's description cause it to fire (or would the user remember to invoke it)
based on how they actually communicate with Claude?

**What to look at in the skill's description:**
- Specific quoted phrases the user must say
- Topic keywords
- Task verbs (e.g. "migrate", "deploy", "refactor")

**What to compare against:**
- `typical_prompts` in the User Profile (extracted from history.jsonl)
- The user's `daily_tasks` phrasing

**Scoring calibration:**
| Score | Meaning |
|-------|---------|
| 9–10 | User's history contains near-identical trigger phrases; skill would fire daily |
| 7–8  | User's history contains semantically similar prompts; skill fires several times/week |
| 5–6  | Trigger phrases are plausible for this user but not evidenced in history |
| 3–4  | Trigger phrases describe work the user does but they'd use different words |
| 1–2  | Trigger phrases are in a domain the user rarely touches |
| 0    | Trigger phrases describe work this user never does |

**Common mistakes to avoid:**
- Don't give high alignment just because a skill SOUNDS relevant to the user's role
- Base it on language match, not domain match — "write unit tests" vs "add test coverage" are different triggers
- For model-invoked skills, be more conservative: Claude has to recognize the context, which is harder

---

## Tool & Dependency Fit — Detailed Scoring

**Hard blockers (score ≤ 3 automatically):**
- Skill requires an MCP server the user doesn't have and it's central to the skill's function
- Skill requires a specific CLI tool the user's OS/setup doesn't have
- Skill expects a specific file structure (e.g. a monorepo pattern) the user's projects don't use

**Soft issues (deduct 1–2 points each):**
- Optional MCP that would improve results but skill still works without it
- Tool is available but user doesn't use it regularly
- Script dependencies (scripts/ directory) that need separate setup

**Scoring calibration:**
| Score | Meaning |
|-------|---------|
| 9–10 | All tools present and actively used in user's workflow |
| 7–8  | All required tools present; some optional ones missing |
| 5–6  | One secondary dependency missing; skill partially useful |
| 3–4  | One key dependency missing; user would need to add it |
| 1–2  | Multiple dependencies missing; significant setup required |
| 0    | Core dependency absent with no practical workaround |

---

## Task Relevance — Detailed Scoring

This is the most important dimension. A skill can have perfect trigger alignment and all tools
present, but if it doesn't solve a real problem the user has, it's still worthless.

**Questions to answer:**
1. Which of the user's `daily_tasks` does this skill address? (directly, not tangentially)
2. Does it address a named `pain_point`?
3. Would the user's work be meaningfully faster/better with this skill vs. without it?

**Scoring calibration:**
| Score | Meaning |
|-------|---------|
| 9–10 | Directly addresses a top daily task or a named pain point |
| 7–8  | Addresses a regular but secondary task |
| 5–6  | Addresses something the user does occasionally (monthly) |
| 3–4  | Adjacent to their work — same domain but not their specific problem |
| 1–2  | Different role or discipline; they'd rarely encounter this task |
| 0    | Completely wrong domain for this user |

**Watch out for:** skills that address aspirational work vs. actual work. A developer who
says "I want to do more infrastructure work" but whose history shows only frontend tasks
should get a low task relevance score for a Terraform skill.

---

## Value Density — Detailed Scoring

Would they actually use it, or just install it and forget it?

**Factors that increase value density:**
- Skill fires automatically (model-invoked) so no remembering needed
- Short, contained interactions (not long multi-step flows) are easier to build habits around
- Skill saves significant time per use (not just 10 seconds)
- Skill addresses a pain point the user mentioned explicitly

**Factors that decrease value density:**
- User-invoked skill (they have to remember to type it)
- Skill is only useful in setup/initialization scenarios (low frequency)
- Skill addresses a task the user already has a fast way to do
- Skill has a steep learning curve to use correctly

**Scoring calibration:**
| Score | Meaning |
|-------|---------|
| 9–10 | Would be invoked multiple times per week; high ROI per use |
| 7–8  | Would be invoked several times per month reliably |
| 5–6  | Would be used but user might forget it exists |
| 3–4  | Would be used occasionally but mostly forgotten |
| 1–2  | Would likely be installed and never used within 2 weeks |
| 0    | No realistic usage scenario given this user's patterns |

---

## Redundancy Check — How to Flag

**When to flag redundancy:**
- Target skill's description trigger phrases significantly overlap with an installed skill's description
- Both skills produce similar outputs (e.g. two code-review skills)
- One skill is a strict subset of the other

**How to report it:**
```
Redundancy: OVERLAPS WITH: <installed-skill-name>
            Both trigger on: <overlapping phrases>
            Overlap type:    <identical | partial | output-level>
            Recommendation:  <keep installed / prefer new / use both for different cases>
```

**When NOT to flag redundancy:**
- Skills share keywords but address genuinely different tasks
- One is model-invoked, other is user-invoked (different activation patterns)
- Skills target different phases of the same workflow

---

## Skill Anti-Patterns to Call Out

When you spot these in the target skill, mention them in the "What this skill actually does" section:

**Vague triggers**: description uses generic phrases like "when working with code" or "for development tasks"
→ Note: "Triggers broadly — may activate in unintended situations"

**Missing tool declaration**: body uses Bash/MCP calls but `allowed-tools` frontmatter is absent
→ Note: "No tool restrictions declared — will request all permissions on first use"

**Assumed project structure**: body hardcodes paths like `src/`, `packages/`, `apps/`
→ Note: "Assumes specific project layout — may not apply to your structure"

**Overly broad scope**: skill tries to do 5+ unrelated things
→ Note: "Wide scope makes it unpredictable — it may do the right thing or go off-script"

**Description ≠ body**: what the description claims the skill does doesn't match the body's instructions
→ Note: "Description and behavior misaligned — triggers may not match actual output"

---

## Risk Assessment — Detailed Signals

Reference for the Risk Assessment section (used by Phase 3, Phase 6, Phase 10). Read this when a
signal match is ambiguous — the goal is to catch real capability surface without crying wolf on
routine, well-guarded skills.

**R1 exfil-surface (`Bash`/`Write`/`Edit` + a network tool in `allowed-tools`)**
- Don't flag just because a skill fetches its *own* update (e.g. skill-eye's own `--update`
  fetches from GitHub via Bash `curl` and writes locally) — that's the same signal by definition,
  which is correct: it's disclosed and documented, not hidden. R1 is a *disclosure* flag, not a
  verdict — plenty of legitimate skills trip it (anything that fetches a URL and saves the
  result). What matters downstream is whether R1 combines with R3 or R4.
- Do flag more heavily when the network tool's target is user-controlled/dynamic (e.g. "fetch the
  URL the user pastes") rather than a fixed, single first-party endpoint.

**R2 injection-surface (fetch remote content, then act on instructions found inside it)**
- The tell is language like "follow the instructions in the fetched page", "execute what the
  file says", or a skill that treats fetched content as anything beyond *data to summarize/
  display*. A skill that fetches a SKILL.md and **evaluates** it (reads description, checks
  tools) is fine — it's treating the content as data. A skill that fetches a webpage and then
  says "do what it says" is the injection surface.
- This is the only signal that alone forces `critical` — indirect prompt injection is the most
  severe class because it lets a third party (whoever controls the fetched content) issue
  instructions the user never wrote and never saw.

**R3 unguarded-destructive (destructive op with no nearby confirm gate)**
- Look for the gate textually near the destructive mention, not just anywhere in the file — a
  global "we sometimes ask for confirmation" doesn't cover a specific `rm -rf` three sections
  away with no local gate.
- A destructive op inside a **template of what NOT to do**, or inside a code comment explaining a
  past incident, is not a live capability — don't flag documentation-only mentions.
- `--force` flags that explicitly *bypass* a stated confirmation are still fine if the
  confirmation exists as the default path — the gate exists, force just skips it deliberately.

**R4 credential-surface (reads/exports credential paths or secret-shaped strings)**
- Distinguish "the skill's own `allowed-tools` justification text mentions `.env` as an example
  of what it *doesn't* touch" (not a hit) from "the body actually greps/cats a credential path"
  (a hit). Read the surrounding sentence, not just the keyword match.
- A skill that reads `.env` to check whether a *variable name* is set (existence check) is a
  lower-severity version of this than one that echoes or transmits the *value*.

**R5 silent-trigger (model-invoked + write/exec power)**
- This is the weakest signal on its own (`medium`) — plenty of legitimate model-invoked skills
  need Bash for read-only inspection. It matters mainly in combination: a model-invoked skill
  that's *also* R1 or R3 is materially riskier than a user-invoked one with the same body, because
  the user never explicitly asked for it to run.

**False-positive discipline:** when a signal match is a keyword appearing in prose/documentation
about the skill rather than in an actual instruction to the agent, don't count it. When genuinely
unsure whether a match is live instruction vs. discussion, note it as tripped but say so plainly
in the signal reason (e.g. "R4: mentions `.env` — verify this is a live read, not documentation")
rather than silently dropping it — false negatives are worse than a flagged line the user can
dismiss in two seconds.

---

## Modify-Then-Install: Writing Good Change Instructions

When issuing Verdict B, the changes must be specific enough to copy-paste.

**Good change instruction:**
```
Change 1 — your trigger phrases don't match its description
In the frontmatter, REPLACE:
  description: "Use when creating API endpoints or REST services"
WITH:
  description: "Use when creating API endpoints, REST services, GraphQL resolvers,
    or when the user asks to 'add a route', 'create an endpoint', or 'build an API'"
```

**Bad change instruction (too vague):**
```
Update the description to match your workflow.
```

Always explain WHY each change is needed in one short phrase before the change block.
Limit to 3 changes maximum — if more than 3 are needed, reconsider whether the verdict
should be SKIP instead.
