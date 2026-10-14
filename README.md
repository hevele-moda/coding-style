# coding-style

A coding style guide for writing code that is easy to read, change and delete, packaged as an [Agent Skill](https://agentskills.io) so AI coding agents can follow it. It has general principles for any language, plus guides for TypeScript and React.

It fills gaps and never overrides your project. Where your project has its own conventions, those win. Rules marked *(opinion)* are preferences you can drop.

Start with [the rules in one screen](skills/coding-style/SKILL.md#the-rules-in-one-screen).

## Install

### Claude Code

Copy or symlink the skill folder into your personal skills directory, or into a project's:

```bash
git clone https://github.com/hevele-moda/coding-style.git
ln -s "$PWD/coding-style/skills/coding-style" ~/.claude/skills/coding-style      # every project
ln -s "$PWD/coding-style/skills/coding-style" .claude/skills/coding-style        # one project
```

### Agents without skill support

Add this to your `AGENTS.md`, `CLAUDE.md` or equivalent, pointing at wherever you cloned the repo:

```markdown
Follow the coding style in path/to/coding-style/skills/coding-style/SKILL.md.
Before writing or reviewing code, read it and the guides it lists for the language you are using.
```

## What is inside

| File | Covers |
| --- | --- |
| [SKILL.md](skills/coding-style/SKILL.md) | The rules in one screen, which guide to read when, and how this guide yields to your project |
| [general_principles.md](skills/coding-style/references/general_principles.md) | Separation of concerns, naming, comments, simplicity, errors, state, testing |
| [typescript.md](skills/coding-style/references/typescript.md) | Types, modeling, nulls, modules, async, validation |
| [react.md](skills/coding-style/references/react.md) | Components, state, effects, hooks, forms, data fetching, testing |

The agent loads only the skill's description up front. It reads the guides when it is writing or reviewing code, so the full guide costs nothing until it is needed.

## License

[MIT](LICENSE)
