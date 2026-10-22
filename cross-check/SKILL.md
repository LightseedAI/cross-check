---
name: cross-check
description: Have a second AI model from a different lab review Claude's work before it ships. Use before committing or shipping a significant code change, after a long autonomous run, or when the user says "cross-check this", "second opinion", or "review before ship".
---

# Cross-check

Claude checking its own work is grading its own homework. Same model, same blind spots.

So the builder and the reviewer should be different models. Claude Code builds. A model from another lab (OpenAI's Codex CLI by default) reviews the changes. Nothing ships until the findings are dealt with.

## When to run

- Before committing or shipping a significant code change
- After a long autonomous run (an overnight build, a multi-hour loop)
- Before anything customer-facing goes live

Skip it for docs-only changes, scratch work and one-line fixes.

## Steps

1. **Run the review.** In a git repo:

   ```bash
   codex exec review
   ```

   To target a specific range:

   ```bash
   codex exec "Review the diff between main and HEAD. Find real bugs: logic errors, security holes, broken edge cases. Ignore style. Report each finding as file:line plus why it breaks."
   ```

   For files outside a repo (a page, a script, a config):

   ```bash
   codex exec "Review <path> adversarially. Assume it is broken and show how. Check it against: <what it is supposed to do>."
   ```

2. **Triage every finding.**
   - Real bug: fix it, then re-run whatever proves the fix works.
   - The two models disagree: read the code. The file decides, not either model.
   - Style opinions: ignore them and keep the repo's existing style.

3. **Report back.** What the reviewer caught, what was fixed, and what was dismissed and why. Never drop a finding silently.

## Notes

- No Codex? Any model from a different lab works. The point is a different family, not a particular vendor.
- `codex exec` runs non-interactively, so it fits inside a Claude Code session.
