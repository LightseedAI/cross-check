# cross-check

A Claude Code skill that gets a second AI from a different lab to review Claude's work before you ship it.

When Claude checks its own code, it's grading its own homework. Same model, same blind spots. This skill hands the review to OpenAI's Codex instead, then makes Claude deal with every finding.

## Install

1. Install the [Codex CLI](https://github.com/openai/codex) and sign in.
2. Copy the `cross-check` folder into `~/.claude/skills/`:

   ```bash
   git clone https://github.com/LightseedAI/cross-check.git
   mkdir -p ~/.claude/skills
   cp -r cross-check/cross-check ~/.claude/skills/
   ```

3. In Claude Code, say **"cross-check this"** before you ship.

## What it does

- **Reviews the plan** before Claude writes any code
- **Reviews the code** with a read-only Codex at high reasoning effort, focused on bugs, security holes and hardcoded secrets
- Returns **structured findings** (P0–P3, file, line, why it breaks, fix) and a ship / fix-first verdict
- **Loops** fix → re-review until no P0/P1 is left (max 3 rounds, then it hands back to you)
- Tells you what was caught, fixed and dismissed

Tested on Codex CLI 0.162. In testing, the plain `codex review` command missed a hardcoded API key. This skill's structured review caught it as P0.

Built by Darren Parker while shipping RealOffer solo.

## License

MIT
