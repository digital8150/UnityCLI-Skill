# unity-cli

A [Claude Code / Claude](https://claude.com/claude-code) skill that teaches Claude how to drive Unity from the command line using Unity's official [`unity` CLI](https://unity.com/blog/meet-the-unity-cli).

With this skill installed, Claude can:

- Inspect installed Unity editors, Hub-registered projects, and live Editor instances (`unity editors -i`, `unity projects list`, `unity status`)
- Create and open Unity projects (`unity projects new/create/open`)
- Run headless CI builds for any platform (Android, iOS, WebGL, Standalone, etc.) via `unity build`
- Run EditMode/PlayMode tests and summarize the resulting NUnit report
- Run arbitrary batch-mode Editor commands via `unity run`
- Install Unity editors and modules (with dry-run/confirmation guardrails before large downloads)
- Talk to an **already-running** Unity Editor live, through the Unity Pipeline package (`unity list` / `unity command` / `unity status`) or via MCP (`unity mcp configure`)

## Install

Copy this repository into your Claude skills directory (or `git clone` it there directly):

```
git clone https://github.com/digital8150/UnityCLI-Skill.git ~/.claude/skills/unity-cli
```

Requires the [`unity` CLI](https://unity.com/blog/meet-the-unity-cli) to already be installed and on `PATH`.

## Structure

- [`SKILL.md`](SKILL.md) — core workflows, quick-reference command table, and safety guardrails
- [`references/commands-reference.md`](references/commands-reference.md) — full flag reference for every `unity` subcommand
