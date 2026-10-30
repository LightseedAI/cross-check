# cross-check

A Claude Code skill that makes Claude and OpenAI's Codex check each other's work.

When Claude checks its own code, it's grading its own homework. Same model, same blind spots. This skill splits every job between two AIs from different labs, so nobody signs off their own work.

## Install

1. Install the [Codex CLI](https://github.com/openai/codex) and sign in.
2. Copy the `cross-check` folder into `~/.claude/skills/`:

   ```bash
   git clone https://github.com/LightseedAI/cross-check.git
   mkdir -p ~/.claude/skills
   cp -r cross-check/cross-check ~/.claude/skills/
   ```

3. In Claude Code, say **"cross-check this feature"** to plan and build, or **"cross-check this"** to review code before you ship.

## What it does

1. **Claude writes the plan.** Goal, steps, acceptance checks and one proof command.
2. **Codex attacks it.** It returns APPROVED, REVISE or BLOCKED with ranked findings. They go back and forth for up to 3 rounds until Codex approves.
3. **Then they swap.** Codex builds it. A fresh, read-only Claude reviews the code.
4. Every problem comes back **ranked P0 to P3**, with the file, the line, why it breaks and the fix.
5. **The builder fixes, the reviewer checks again.** It loops until no P0 or P1 is left (max 3 rounds, then it hands back to you).
6. It tells you what was caught, fixed and dismissed.

| Mode | Use it for |
|---|---|
| `full` (default) | New feature or big change: plan, attack, build, review, fix |
| `review` | Code already written: Codex reviews, Claude fixes |
| `plan` | A plan or spec only |

Say `builder=claude` to keep the old order: Claude builds, Codex reviews.

Tested on Codex CLI 0.162. In testing, the plain `codex review` command missed a hardcoded API key. This skill's structured review caught it as P0.

Built by Darren Parker while shipping RealOffer solo.

## License

MIT
