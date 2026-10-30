---
name: cross-check
description: Two AIs from different labs (Claude and OpenAI Codex) check each other's work. Claude plans, Codex attacks the plan until it approves, then they swap jobs. Codex builds and Claude reviews the code, looping fix and re-review until it's clean. Use for a new feature or big change ("cross-check this feature", "plan and build with Codex"), or in review mode before shipping code that's already written ("cross-check this", "second opinion", "review before ship").
---

# Cross-check

Claude checking its own work is grading its own homework. Same model, same blind spots.

So nobody signs off their own work:

| Job | Default (`builder=codex`) | `builder=claude` |
|---|---|---|
| Write the plan | Claude | Claude |
| Attack the plan | Codex | Codex |
| Build | **Codex** | Claude |
| Review the code | **Claude** (fresh, read-only) | Codex |
| Fix | the builder | the builder |

The swap is the point. The model that planned doesn't build, and the model that built doesn't review.

## Pick a mode

| Situation | Mode | Steps |
|---|---|---|
| New feature or big change | `full` (default) | 1 to 7 |
| Code already written, about to ship | `review` | 6 and 7, Codex reviews |
| A plan or spec only | `plan` | 1 to 3 |

Skip it for docs-only changes, scratch work and one-line fixes.

Check Codex first: `codex --version` and `codex login status`. If it's missing, any model from another lab works. The point is a different family, not a vendor.

## 1. Settle the requirements

Read the code and docs the change touches. Then ask the user only the questions whose answers change what gets built (scope, behaviour on edge cases, what "done" means). Batch them in one message. Don't ask what the code already answers.

## 2. Write the plan

Write `PLAN.md` (or the path the user gives) with:
- **Goal:** one line
- **Steps:** files to touch and what changes in each
- **Acceptance checks:** observable results, each one testable
- **Proof command:** the one command that shows it works (`pytest -q`, `npm test`, a curl)
- **Out of scope**

## 3. Codex attacks the plan

```bash
codex exec -s read-only --ephemeral --skip-git-repo-check -c model_reasoning_effort="high" \
  --output-schema ~/.claude/skills/cross-check/plan-review.schema.json \
  -o /tmp/cross-check-plan.json \
  "Review PLAN.md adversarially before anything is built. Read the code it touches. Find wrong assumptions, missing steps, ignored edge cases, untestable acceptance checks and simpler approaches. Back every finding with evidence. Do not write code. Verdict APPROVED only if it is buildable as written with no P0 or P1 left." </dev/null
```

Then go back and forth:
- For each finding: fix the plan, or write down why it's wrong. The code decides, not either model.
- Re-run the review with a summary of last round's findings and what you did about each, so Codex checks the fixes instead of starting over.
- Stop at **APPROVED**, or after **3 rounds**. **BLOCKED** or still REVISE after 3 goes to the user. Never call it approved yourself.

Record the approval: `shasum -a 256 PLAN.md`. If the plan changes after that, it needs another review.

In `plan` mode, stop here and report.

## 4. Build

Start from a clean git tree (`git status --porcelain` is empty), or commit or stash first. Then the diff is only the builder's work. Note the starting commit: `BASE=$(git rev-parse HEAD)`.

**`builder=codex` (default):**

```bash
codex exec -s workspace-write -c model_reasoning_effort="high" \
  -o /tmp/cross-check-build.md \
  "Implement PLAN.md exactly. Stay inside its steps. Run the proof command and make it pass. Don't commit. Finish with a short list of files changed and anything you couldn't do." </dev/null
```

`workspace-write` lets Codex edit files in this repo only, with no network. Read its summary, then check `git status` for anything outside the plan.

**`builder=claude`:** build it yourself from the plan.

## 5. Prove it

Run the proof command yourself. Don't trust the builder's report. A failing proof goes back to the builder before any review.

## 6. Review the code (the other model)

Get the diff, including new files, and check it isn't empty:

```bash
git add -N . && git diff $BASE > /tmp/cross-check.diff
[ -s /tmp/cross-check.diff ] || { echo "empty diff: nothing to review"; exit 1; }
```

In `review` mode, pick the diff that matches what changed: work in progress `git add -N . && git diff HEAD`, a branch `git diff main...HEAD`, one commit `git show <sha>`.

**Codex built it, so Claude reviews.** Spawn a fresh subagent so the review doesn't inherit the planner's assumptions. Tell it to read `PLAN.md` and `/tmp/cross-check.diff`, open any file it needs, change nothing, and return findings as JSON in the shape of `~/.claude/skills/cross-check/findings.schema.json`: real bugs, security holes (including hardcoded secrets and keys) and broken edge cases only, no style.

**Claude built it (or `review` mode), so Codex reviews:**

```bash
codex exec -s read-only --ephemeral -c model_reasoning_effort="high" \
  --output-schema ~/.claude/skills/cross-check/findings.schema.json \
  -o /tmp/cross-check.json \
  "Review this diff against PLAN.md if there is one. Report only real bugs, security holes (including hardcoded secrets and keys) and broken edge cases. Ignore style. Must hold: <acceptance checks>." \
  < /tmp/cross-check.diff
```

Each finding has a severity (P0 to P3), file, line, problem, why it breaks and a fix, plus a `ship` or `fix-first` verdict. Add the risky areas to the focus line (auth, payments, migrations).

**Quick version:** `codex review --uncommitted` is fine for a fast look, but it takes no focus line, and in testing it **missed a hardcoded API key** that the structured review caught as P0. Use the structured review before shipping.

## 7. Triage, fix, loop

Go through every finding:
- **Real bug:** send it to the builder. For Codex, re-run the step 4 command with "Fix these findings, then run the proof command:" and the list. For Claude, fix it yourself. Then run the proof command again.
- **The models disagree:** read the code and reproduce it. The file decides.
- **Style opinion:** dismiss it.
- **Can't tell:** ask the user.

Rebuild the diff from `$BASE` and review again with the same reviewer. Stop when no P0 or P1 is left, or after **3 rounds**. If it's still not clean, stop and show the user what's left. An endless loop usually means the models disagree about intent, and that's a human call.

## Report

Tell the user:
- who planned, built and reviewed
- what each review caught (plan and code), what was fixed and how it was proven
- what was dismissed, and why
- the proof command result

Never drop a finding silently. Don't commit unless the user asks.

## Notes

- Reasoning effort is `high` on purpose. Many Codex configs default to low, which is too shallow for a review.
- Leave the model to your Codex config, or pass `-m <model>` to pin one. Model names change often, so this skill doesn't hardcode one.
- `--ephemeral` keeps review sessions out of your Codex history. `</dev/null` stops Codex waiting for input when nothing is piped in.
