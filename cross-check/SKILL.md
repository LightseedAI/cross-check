---
name: cross-check
description: Have a second AI from a different lab (OpenAI Codex) review Claude's plan and code before it ships, then fix and re-review until it's clean. Use before committing or shipping a significant change, before a big build starts, after a long autonomous run, or when the user says "cross-check this", "second opinion", or "review before ship".
---

# Cross-check

Claude checking its own work is grading its own homework. Same model, same blind spots.

So the builder and the reviewer are different models. Claude builds. Codex reviews, read-only. Claude fixes what's real, and Codex looks again. Nothing ships with a P0 or P1 open.

## When to run

| Moment | What to review |
|---|---|
| Before a big build | The plan (step 1) |
| Before a commit or ship | The code (step 2) |
| After a long autonomous run | Everything since the base branch (step 2, `--base`) |
| Before anything customer-facing goes live | The code, focused on the risky parts |

Skip it for docs-only changes, scratch work and one-line fixes.

Check Codex is there first: `codex --version`. If it's missing, any model from another lab works. The point is a different family, not a vendor.

## 1. Review the plan (before building)

A bad plan reviewed after the build costs a rewrite. Review it first:

```bash
codex exec -s read-only --ephemeral --skip-git-repo-check -c model_reasoning_effort="high" \
  -o /tmp/cross-check-plan.md \
  "Review this plan adversarially. Find wrong assumptions, missing steps, edge cases it ignores and simpler approaches. Do not write code. Plan: <paste plan or path>" </dev/null
```

`--skip-git-repo-check` lets it run outside a repo. `</dev/null` stops it waiting for extra input when nothing is piped in.

Fold the real points into the plan before writing any code.

## 2. Review the code

Run it from the repo root. Pick the diff that matches what changed:

| Scope | Diff command |
|---|---|
| Work in progress | `git add -N . && git diff HEAD` (`add -N` makes new files show up) |
| A whole branch | `git diff main...HEAD` |
| One commit | `git show <sha>` |

Save the diff and check it isn't empty, then hand it to a read-only Codex with a focus line and structured output:

```bash
git diff main...HEAD > /tmp/cross-check.diff || { echo "diff failed: check the branch name"; exit 1; }
[ -s /tmp/cross-check.diff ] || { echo "empty diff: nothing to review"; exit 1; }
codex exec -s read-only --ephemeral -c model_reasoning_effort="high" \
  --output-schema ~/.claude/skills/cross-check/findings.schema.json \
  -o /tmp/cross-check.json \
  "Review this diff. Report only real bugs, security holes (including hardcoded secrets and keys) and broken edge cases. Ignore style. Must hold: <success criteria>." \
  < /tmp/cross-check.diff
```

A wrong branch name or empty diff would otherwise send Codex nothing, and it would report all clear.

Each finding comes back as severity (P0–P3), file, line, problem, why it breaks and a fix, plus an overall `ship` or `fix-first` verdict. Add the risky areas to the focus line (auth, payments, migrations).

**Quick version** when you just want a fast look:

```bash
codex review --uncommitted -c model_reasoning_effort="high"    # or --base main, or --commit <sha>
```

It's fine for a fast look, but it doesn't take a focus line, and in testing it **missed a hardcoded API key** that the structured review caught as P0. Use the structured review before shipping.

Not code (a page, a config, a script outside git)? Use step 1's command and point it at the file.

## 3. Triage

Go through every finding:

- **Real bug:** fix it, then re-run the test or check that proves the fix.
- **The models disagree:** read the code and reproduce it. The file decides, not either model.
- **Style opinion:** ignore it and keep the repo's existing style.
- **Can't tell:** ask the user. Don't guess.

## 4. Loop until clean

Re-review after fixing, and make sure the fixes are in the diff. Fixes are usually uncommitted, so from round 2 diff the working tree against the starting point:

| First round was | Round 2+ diff |
|---|---|
| Work in progress | `git add -N . && git diff HEAD` (same as before) |
| A whole branch | `git add -N . && git diff main` |
| One commit | `git add -N . && git diff <sha>^` |

Stop when there are no P0 or P1 findings left, or after **3 rounds**. If it's still not clean after 3, stop and show the user what's left. Endless loops usually mean the models disagree about intent, and that's a human call.

## 5. Report

Tell the user:
- what Codex caught
- what was fixed, and how it was proven
- what was dismissed, and why

Never drop a finding silently.

## Notes

- Reasoning effort is set to `high` on purpose. Many Codex configs default to low, which is fine for chat but too shallow for a review.
- Leave the model to your Codex config, or pass `-m <model>` to pin one. Model names change often, so this skill doesn't hardcode one.
- `--ephemeral` keeps review sessions out of your Codex history.
