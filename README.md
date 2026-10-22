# cross-check

A Claude Code skill that gets a second AI from a different lab to review Claude's work before you ship it.

When Claude checks its own code, it's grading its own homework. Same model, same blind spots. This skill hands the review to OpenAI's Codex instead, then makes Claude deal with every finding.

## Install

1. Install the [Codex CLI](https://github.com/openai/codex) and sign in.
2. Copy the `cross-check` folder into `~/.claude/skills/`:

   ```bash
   git clone https://github.com/LightseedAI/cross-check.git
   cp -r cross-check/cross-check ~/.claude/skills/
   ```

3. In Claude Code, say **"cross-check this"** before you ship.

## What it does

- Runs `codex exec review` on your changes
- Fixes real bugs, ignores style nitpicks
- Tells you what was caught, fixed and dismissed

Built by Darren Parker while shipping RealOffer solo.

## License

MIT
