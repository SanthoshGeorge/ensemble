# Ensemble

A working, drop-in personal cross-functional dev team for Claude Code: five
subagents that take a problem statement, scope it, implement it, test it, and
(on request) deploy it — with a cheap classification step up front that
decides which model tier each stage actually needs, so trivial work never
pays opus prices.

## What's in here

```
.claude/
  agents/
    triage.md      # haiku  — classifies complexity/risk, recommends a model per stage
    architect.md   # sonnet — turns the problem into a spec + task breakdown
    developer.md   # sonnet — implements the spec
    tester.md      # sonnet — writes + runs tests, routes failures back
    deployer.md    # haiku  — packages/deploys, dry-run by default
  skills/
    ensemble/
      SKILL.md      # the orchestrator: /ensemble "<problem statement>"
tools/
  cost_rates.json    # current per-model $/MTok, sourced from platform.claude.com/docs
  estimate_cost.py   # rough cost calculator for a pipeline run
```

## Where Ensemble lives, and reusing it across work and personal projects

This is just a folder of markdown and JSON files — there's no server, account,
or license binding it to one place. Claude Code looks for subagents and skills
in two tiers, checked in this order:

1. **Project-level** — a `.claude/` folder committed inside one specific repo.
   Only available in that project; the right choice for something a work team
   shares and reviews together in version control.
2. **User-level** — `~/.claude/agents/` and `~/.claude/skills/` on a given
   machine, under your account. Anything here is available in *every* project
   you open with Claude Code on that machine, work repos and personal repos
   alike, without copying it into each one.

For your case — reusing Ensemble across both work and personal projects — the
practical answer depends on whether "work" and "personal" mean different
folders on the *same* laptop, or genuinely separate machines:

- **Same machine, different repos:** install once at user level
  (`~/.claude/agents/`, `~/.claude/skills/`) and every project you open there,
  work or personal, has access to `/ensemble` automatically. This is the
  simplest setup and the one most people want.
- **Separate machines (e.g. a locked-down work laptop and a personal one):**
  the files have to physically exist on each machine — copy this same folder
  to `~/.claude/` on both. Nothing about Ensemble is tied to one device, so
  there's no re-authoring involved, just getting the files there. On a
  managed corporate laptop, check first whether your org has its own managed
  Claude Code settings — those take priority over anything in your personal
  `~/.claude/`, so if IT already defines subagents with the same names,
  those win.
- **Keeping one copy in sync across machines:** put this Ensemble folder in
  its own small private git repo, then clone it (or symlink its `agents/` and
  `skills/` subfolders into `~/.claude/agents/` and `~/.claude/skills/`) on
  every machine you use. That way a change you make once — a tweaked prompt,
  a new agent — propagates everywhere the next time you pull, the same way
  people manage dotfiles.

Either way, also make sure `tools/` (the cost estimator) is reachable from
wherever the skill runs — project root is simplest if you're doing the
project-level install; anywhere on your `PATH` or a fixed personal path works
for the user-level install.

## Use it

From any Claude Code session where Ensemble is installed:

```
/ensemble "Add a rate limiter to the /submit endpoint, 10 req/min per user"
```

What happens:

1. **Triage** (haiku) reads the request and returns a tier — trivial, standard,
   or complex — plus a risk level and a recommended model for each of the next
   four stages. This is the one call that runs on every single task, and it's
   what keeps the rest cheap: it's the only place complexity gets judged before
   money gets spent on it.
2. **Architect** (model chosen by triage) turns that into a concrete spec: what
   changes, what doesn't, edge cases, and a checklist for the developer. If
   triage flagged the task as complex or high-risk, the architect stops and
   asks you to sign off on the spec before anything gets built — a human review
   gate, not an auto-proceed.
3. **Developer** (model chosen by triage) implements the checklist, nothing
   more.
4. **Tester** (model chosen by triage) writes and runs tests, and says
   explicitly whether a failure is an implementation bug, a gap in the spec, or
   flakiness — so retries go back to the right stage instead of just trying
   again blindly. Two failures on the same issue escalates the model tier
   rather than repeating the same attempt a third time.
5. **Deployer** (haiku by default) packages the result. It defaults to a dry
   run — it describes exactly what it would do — and only executes for real if
   you asked for that explicitly ("...and deploy it to staging"). Anything
   complex/high-risk stops here for sign-off too, before a live step.

You get a final summary: what was built, what was tested, deploy status, and a
per-stage cost estimate so you can see which stage is actually driving spend.

## Why this is cost-efficient, specifically

The lever that matters most is the one that's easy to skip: **classify before
you spend, not after.** A single haiku call (a few cents at most) decides
whether the next four calls run on haiku, sonnet, or opus. Skipping triage
"to save a step" is the single most common way this kind of pipeline gets
expensive — every stage ends up defaulting to whatever model is safest to
assume, which is usually the most expensive one.

Three other things compound on top of that:

- **Tool scoping.** Each subagent only has the tools it actually needs
  (tester can't `Write`, deployer can't `Edit`). Smaller tool manifests mean
  smaller system prompts, which means fewer tokens on every single call.
- **Bounded retries with escalation, not repetition.** The pipeline caps
  developer/architect retries at 3 total and escalates the model tier after two
  same-cause failures, instead of hammering the same (wrong) approach at the
  same model repeatedly.
- **Scope discipline.** Each agent's prompt explicitly forbids doing another
  stage's job (architect doesn't code, developer doesn't invent requirements).
  Scope creep inside a single subagent call is quietly one of the most
  expensive things that can happen — it turns one bounded task into an
  open-ended one.

If you want to push further: turn on prompt caching for anything the pipeline
re-reads across stages (the spec, a large file, repo context) — Claude Code
does this automatically for repeated context within a session, and it's worth
watching in the Console since a 1-hour cache read costs roughly 1/20th of a
fresh read at current rates.

## Cost reference (as of 2026-09-13, USD per million tokens)

| Model  | Input | Output |
|--------|-------|--------|
| Haiku  | $1    | $5     |
| Sonnet | $2    | $10    |
| Opus   | $5    | $25    |

Source: [platform.claude.com/docs — pricing](https://platform.claude.com/docs/en/about-claude/pricing).
Kept in `tools/cost_rates.json` so the estimator stays correct if prices change —
edit that file, not the script.

## Extending this

A few natural next steps, especially given governance patterns like human
review gates, prompt evaluation standards, and data-privacy safeguards:

- **Formal gate logging.** Right now the human-gate stops are conversational
  (the pipeline pauses and asks). If you want an audit trail, have the
  orchestrator append each gate decision (who approved, when, what tier/risk)
  to a log file before continuing.
- **Domain-specific triage rules.** If you're routing work that touches
  eligibility or benefits calculations specifically, add an explicit
  high-risk trigger for those keywords in `triage.md` rather than relying on
  the model to infer it every time.
- **A sixth "reviewer" agent** that only runs on complex/high-risk tier, doing
  a second-opinion pass on the architect's spec before the developer starts —
  cheap insurance against one agent's blind spot, run only when the risk
  actually warrants the extra cost.
- **Swap in Claude Agent SDK** instead of Claude Code subagents if you want
  this running outside an interactive session (e.g. triggered from a ticket
  queue) — the same five prompts and the same triage-first structure port
  directly; you'd just drive the loop with the SDK's `query()` calls and your
  own model-selection logic instead of a Claude Code skill.
