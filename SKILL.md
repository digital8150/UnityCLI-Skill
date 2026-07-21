---
name: unity-cli
description: Control Unity Editor and Unity Hub from the command line using the official `unity` CLI (https://unity.com/blog/meet-the-unity-cli) — list/install Unity editors and modules, create/open/manage Unity projects, run headless CI builds (Android, iOS, WebGL, Standalone, etc.), run EditMode/PlayMode tests, execute arbitrary batch-mode commands, and drive an already-running Unity Editor instance live via the Pipeline package (status/list/command) or via MCP. Use this skill whenever the user asks to build, test, run, open, install, automate, or script anything involving Unity or Unity Hub — including requests like "build my Unity project", "run my Unity tests", "install Unity 6000.0", "open this Unity project", "what Unity editors do I have installed", or "make Claude control my running Unity Editor" — even if they never say "CLI" or "unity" by name.
---

# Unity CLI

`unity` is Unity's official command-line tool for driving the Unity Editor, Unity Hub, and Unity Cloud without touching the GUI. It has two distinct capabilities that are easy to conflate — figure out which one the user actually wants before picking commands:

1. **Batch-mode automation** — spawn/control a Unity Editor *process* to build, test, or run a project headlessly (CI-style). Commands: `build`, `test`, `run`, `open`, `projects`, `editors`, `install`, `install-modules`.
2. **Live control of a running Editor** — send commands to a Unity Editor that is *already open* on the user's machine, via the Pipeline package's HTTP server. Commands: `status`, `list`, `command`/`cmd`, `pipeline`, `mcp`.

If the user has Unity Editor open on screen and wants Claude to poke around inside it (move GameObjects, call methods, inspect scene state), that's mode 2 and requires the Pipeline package. If they want a build artifact or a test report, that's mode 1 and works with zero Editor setup beyond having an editor version installed.

Every command accepts `--format json` for machine-parseable output and `--non-interactive` to prevent prompts from hanging a script — always add both when running commands on the user's behalf so failures surface as errors instead of stalled prompts.

## Before doing anything: orient yourself

Run these three (cheap, read-only) checks before acting on a Unity request — they tell you what's actually available so you don't guess at editor versions or project paths:

```
unity doctor                     # CLI environment health check
unity editors -i                 # installed editor versions (and which is default)
unity status                     # live Editor instances currently running (port, project, version, PID)
unity projects list              # projects registered in Hub
```

`unity status` returning an empty table means no Editor is currently running — live-control commands (`list`, `command`, `mcp`) will fail until the user opens one (see below).

## Command quick-reference

| Task | Command |
|---|---|
| List installed editors | `unity editors -i` |
| List/download available editor releases | `unity editors -r` or `unity releases` |
| Install an editor version | `unity install <version> [-m <module>...] --accept-eula` |
| List projects Hub knows about | `unity projects list` |
| Create a new project | `unity projects new <name> --editor-version <ver> --template <id>` |
| Open a project in the Editor | `unity open <project>` or `unity projects open <pattern>` |
| Headless CI build | `unity build --target <target> --execute-method <Class.Method>` |
| Run tests | `unity test [project] --mode EditMode\|PlayMode` |
| Run project in batch mode with custom Editor args | `unity run [project] -- -executeMethod X -quit` |
| See which Editors are live right now | `unity status` |
| List tools a running Editor exposes | `unity list --project-path <path>` |
| Call a tool on a running Editor | `unity command <name> [args...] --project-path <path>` |
| Wire up an AI agent (e.g. this CLI) to talk to a running Editor | `unity mcp configure claude` |

Full flag tables for every command are in [references/commands-reference.md](references/commands-reference.md). Read it before constructing a command you're not confident about — this CLI has a lot of platform-specific flags (Android signing, versioning strategy, module IDs) that are easy to get subtly wrong from memory.

## Workflow: headless build (most common ask)

Unity has no built-in command-line build feature — every build needs a static C# method in the project (e.g. `public static void Build()` calling `BuildPipeline.BuildPlayer`) that `--execute-method` invokes. **Check the project for an existing build script first** (search for `BuildPipeline.BuildPlayer` or an `Editor/` folder with a `Build`-ish class name) rather than assuming one exists or inventing a method name.

```
unity build --target StandaloneWindows64 --execute-method Builder.PerformBuild
unity build ./MyGame --target Android --execute-method Builder.AndroidBuild -o ./out/app.apk --allow-install
unity build --target WebGL --execute-method Builder.WebGLBuild --editor-version 6000.0
```

Notes:
- `--target` and `--execute-method` are both required — there's no sane default for either.
- Add `--allow-install` if the project's required editor version (read from `ProjectVersion.txt`) isn't installed yet; otherwise the build fails asking you to install it manually. Warn the user before this triggers a multi-GB download rather than letting it run silently — see Safety below.
- The build refuses to run against a dirty working tree by default (uncommitted changes). That's intentional (reproducible builds); only pass `--allow-dirty-build` if the user explicitly says they're fine building uncommitted changes.
- For Android signing (`--android-keystore-base64`, `--android-keystore-password`, etc.), the CLI itself warns these values land in shell history / CI logs in plaintext. Prefer environment variables or a secrets manager over typing them inline in a command the user will see in their terminal history. See the [Android-specific flags](references/commands-reference.md#build) in the command reference for the full flag set.
- Default log location is `<project>/Logs/build-<target>-<timestamp>.log` — if a build fails, read that file (or don't pass `--no-tail`, so it streams to stdout) before guessing at the cause.

## Workflow: tests

```
unity test                                      # current directory, default test platform
unity test ./MyProject --mode EditMode --output ./results/edit.xml
unity test . --filter "MyNamespace.MyTests"
```

Results are written as an NUnit XML report (default `test-results.xml`) — after running, read that file to summarize pass/fail counts for the user rather than only relaying CLI stdout, since the interesting detail (which specific test failed and why) lives in the XML.

## Workflow: projects and editors

```
unity projects new MyGame --editor-version 6000.0.30f1 --template com.unity.template.3d --open
unity projects create MyGame --vcs github --git-token-stdin   # also creates+links a GitHub repo
unity open "My Game"                            # fuzzy/glob match against Hub registry, falls back to filesystem path
unity install 6000.0.30f1 -m android --accept-eula --dry-run  # ALWAYS dry-run first, see Safety
```

`unity open`/`unity projects open` resolve the argument in this order: exact Hub name/path → glob pattern → fuzzy title match → filesystem path. If a project name the user gives you doesn't match anything, don't assume you should create it — ask, or check `unity projects list` for the closest match.

## Workflow: controlling a running Editor live (Pipeline)

This is what lets Claude actually reach *into* an open Unity Editor instead of just batch-processing it. It requires the Unity Pipeline package installed in the target project:

```
unity pipeline install --project-path <path>     # one-time setup per project
unity pipeline list                               # shows every Editor instance + its Pipeline status
unity status                                      # confirm the Editor is up and which port/project
unity list --project-path <path>                  # discover what commands/tools this Editor exposes
unity command <tool-name> [args...] --project-path <path>   # invoke one
```

`unity command` with no command name lists available commands too — useful when you don't know what the connected Editor's Pipeline package registers. **Always run `unity list` (or bare `unity command`) before guessing a command name** — the set of available commands depends on what the project's Pipeline package version registers, not on anything fixed in the CLI itself, so don't assume a command exists from memory.

For a persistent AI-agent connection instead of one-off CLI calls, `unity mcp configure <client>` writes an MCP server entry so the client (Claude Desktop, Claude Code, etc.) talks to the Editor directly over MCP. Run `unity mcp configure --list` to see supported clients, and `unity mcp` alone starts the stdio server manually if the user wants to wire it up to something not in that list.

## Windows/PowerShell quoting

The user is on Windows. Project paths and names with spaces (`"My Game"`) need double quotes, and PowerShell needs `--` handled carefully when passing raw Editor args through (`unity run . -- -nographics -quit`) — test that the `--` separator actually reaches the CLI rather than being swallowed, since PowerShell's argument parsing differs from bash here. If a command with `--` misbehaves, try wrapping the whole trailing args in a single quoted string via `--args` instead, where the command supports it (e.g. `unity build --args "-nographics -quit"`).

## Safety: what to confirm before running

Some `unity` operations are expensive or destructive — confirm with the user before running these rather than firing them off proactively:

- **`unity install` / `unity install-modules`** — downloads can be multiple GB and take a long time. Run with `--dry-run` first and show the user what would download before running for real, unless they've already explicitly named the exact version/modules they want installed.
- **`--allow-install` on build/test/run** — same reason; it silently triggers an install mid-command.
- **`unity uninstall`, `unity self-uninstall`, `unity projects remove`** — removes editors/CLI/registry entries. `projects remove` doesn't delete files from disk, but `uninstall` does remove an installed editor.
- **`--allow-dirty-build`** — bypasses the uncommitted-changes guard; only use if the user explicitly asks to build dirty.
- **`unity projects create --vcs github/gitlab`** — creates and pushes to a *new remote repository* under the user's account. Confirm the target namespace/visibility before running, same as any other action that creates shared/remote state.

Full reference for every other command (`auth`, `cloud`, `hub`, `templates`, `config`, `modules`, `cache`, `analytics`, `logs`, `license`, `releases`, `changelog`, `env`, `diagnose`, `completion`, `shell`) is in [references/commands-reference.md](references/commands-reference.md).
