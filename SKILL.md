---
name: unity-cli
description: Control Unity Editor and Unity Hub from the command line using the official `unity` CLI (https://unity.com/blog/meet-the-unity-cli) — list/install Unity editors and modules, create/open/manage Unity projects, run headless CI builds (Android, iOS, WebGL, Standalone, etc.), run EditMode/PlayMode tests, execute arbitrary batch-mode commands, and drive an already-running Unity Editor instance live via the Pipeline package (status/list/command) or via MCP. Use this skill whenever the user asks to build, test, run, open, install, automate, or script anything involving Unity or Unity Hub — including requests like "build my Unity project", "run my Unity tests", "install Unity 6000.0", "open this Unity project", "what Unity editors do I have installed", or "make Claude control my running Unity Editor" — even if they never say "CLI" or "unity" by name.
---

# Unity CLI

`unity` is Unity's official command-line tool for driving the Unity Editor, Unity Hub, and Unity Cloud without touching the GUI. It has two distinct capabilities that are easy to conflate — figure out which one the user actually wants before picking commands:

1. **Batch-mode automation** — spawn/control a Unity Editor *process* to build, test, or run a project headlessly (CI-style). Commands: `build`, `test`, `run`, `open`, `projects`, `editors`, `install`, `install-modules`.
2. **Live control of a running Editor** — send commands to a Unity Editor that is *already open* on the user's machine, via the Pipeline package's HTTP server. Commands: `status`, `list`, `command`/`cmd`, `pipeline`, `mcp`.

If the user has Unity Editor open on screen and wants Claude to reach into it, that's mode 2 and requires the Pipeline package. If they want a build artifact or a test report, that's mode 1 and works with zero Editor setup beyond having an editor version installed. **Don't underestimate mode 2** — depending on the Pipeline package version installed in the project, it commonly exposes not just narrow scene/GameObject tools but a command that executes arbitrary C# inside the running Editor (often named `eval`), which means it can do essentially anything the Editor GUI can do — create/modify/delete assets, prefabs, scenes, and project files, not just call a few safe accessors. See the dedicated workflow section below before using it.

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
| List tools a running Editor exposes (can be 100+, may include arbitrary-code-execution) | `unity list --project-path <path>` |
| Call a tool on a running Editor (positional args, not `key=value`) | `unity command <name> <arg1> <arg2>... --project-path <path>` |
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

## Workflow: controlling a running Editor live (Pipeline) — read this before using it

This is the most powerful thing this CLI does, and the command shape (`unity command <name> [args]`) undersells it. Depending on the Pipeline package version installed in the project, `unity list` can return dozens to 100+ commands — not just a handful of narrow, safe accessors. Commonly among them is a command (often named `eval`) that executes **arbitrary C# inside the running Editor process**. That means live mode can do essentially anything the Editor GUI can do: create, modify, or delete GameObjects, prefabs, assets, scenes, and project files, or run arbitrary logic — it is not limited to whatever a command's name suggests. Treat any `unity command` call, and `eval`-style ones especially, with the same weight as running arbitrary code on the user's machine, because that's what it is.

### The most common live task: did my script change actually compile?

If you edit a `.cs` file in a project with a live-connected Editor, editing the file isn't the finish line — Unity needs to recompile, and telling the user "please check the Unity console" is a cop-out when you can just ask the Editor yourself:
```
unity command recompile --project-path <path>
unity command recompile_status --project-path <path>   # poll until it reports done
unity command console --project-path <path>             # read for compiler errors/warnings
```
(exact names depend on what the connected project's Pipeline package registers — confirm via `unity list` — but a recompile-trigger, a recompile-status poll, and a console-read are the common shape.) Make this your default close-out step any time you touch a script in a project with a live Editor attached — it turns "I made the change" into "I made the change and confirmed it compiles," which is the actual thing the user wants.

### Setup

```
unity pipeline install --project-path <path>
```
One-time per project, but not a no-op: it modifies the project's package manifest and triggers a domain reload (the Editor will briefly lock up and recompile), same as installing any other package through the Package Manager UI. Fine to run without asking every time the user wants live control, but don't run it silently mid-task if the Editor is doing something else at that moment — a reload will interrupt it.

### Discover before you act

Don't rely on memory of a previous session — the command set is defined by whatever Pipeline package version that project has, not by anything fixed in the CLI:
```
unity status                                      # confirm an Editor is live, and on which port/project
unity list --project-path <path> --format json    # full command list this Editor exposes
```
This list can run to 100+ entries — don't let the whole human-readable table eat your context on every call. Use `--format json` and filter for the keyword you actually need (grep/jq for "prefab", "asset", "eval", "recompile", etc.) instead of reading the entire output, and once you've confirmed a command's name and signature within a session, you don't need to re-list before every subsequent call to it.

### Invoke a command

```
unity command <name> <arg1> <arg2>... --project-path <path>
```
Arguments are **positional**, not `key=value` flags — passing `key=value` gets forwarded as a literal string argument rather than parsed as a named parameter, and will fail or misbehave. If a command's argument order isn't obvious from its name, check its entry in `unity list`'s output before guessing.

Reading the response: with `--format json`, the value you actually want is nested at `data.result.result`. Its type varies per command — sometimes a plain string, sometimes an object — so parse defensively (check the type before assuming structure) instead of assuming one shape works for every command.

### Tips specifically for `eval`

- **Batch, don't loop.** One `eval` call that sets up several objects/fields beats a call per field — each round-trip has fixed overhead, and reaching for a narrow per-field setter 15 times when one script block does it in one shot is slower for no benefit.
- **Use fully-qualified type names** inside the eval'd code (`UnityEditor.AssetDatabase`, `UnityEngine.GameObject`, ...) rather than assuming a `using` is in scope — the snippet doesn't inherit the project's normal usings, and unqualified names are a common source of failures.
- **Call `UnityEditor.AssetDatabase.SaveAssets()` and `.Refresh()` at the end** of any eval that touches assets/prefabs. Without them, changes can look like they succeeded in-memory but won't actually persist to disk.
- **For anything longer than a couple lines, write the code to a file and use `eval_file`** (or whatever the equivalent is named — check `unity list`) instead of inlining C# as a shell argument. Multi-line code as an inline argument runs into shell-quoting problems fast, especially on Windows — see below.

### Scope of approval: once per task, not once per call

Get the user's go-ahead for the *scope* of what you're about to do, then work within it without re-confirming every individual call — a task like "wire up these 4 prefabs" is inherently a request for however many mutating calls that takes, and stopping to ask before each one turns a 15-step task into 15 approval prompts and doesn't actually make anything safer. What does warrant stopping to re-confirm:
- You're about to touch something **outside what was discussed** (a file/object/asset the user didn't mention).
- You're about to do something **irreversible beyond the task itself** — overwriting or deleting an asset that predates this task, as opposed to something you just created a moment ago as part of it.
- You're about to run `eval` for something structurally different from what was agreed (e.g. the task was "wire up prefabs" and you're now about to touch project settings).

In short: confirm the plan once, then execute it — reserve fresh confirmation for scope creep or genuinely irreversible edges, not for every command in an already-agreed-on task. Read-only commands (querying state, listing tools) never need confirmation.

For a persistent AI-agent connection instead of one-off CLI calls, `unity mcp configure <client>` writes an MCP server entry so the client (Claude Desktop, Claude Code, etc.) talks to the Editor directly over MCP. Run `unity mcp configure --list` to see supported clients, and `unity mcp` alone starts the stdio server manually if the user wants to wire it up to something not in that list. The same arbitrary-execution caution applies over MCP as over the CLI — the underlying command surface is the same.

## Windows/PowerShell quoting

The user is on Windows. Project paths and names with spaces (`"My Game"`) need double quotes, and PowerShell needs `--` handled carefully when passing raw Editor args through (`unity run . -- -nographics -quit`) — test that the `--` separator actually reaches the CLI rather than being swallowed, since PowerShell's argument parsing differs from bash here. If a command with `--` misbehaves, try wrapping the whole trailing args in a single quoted string via `--args` instead, where the command supports it (e.g. `unity build --args "-nographics -quit"`).

This applies doubly to multi-line C# passed to `eval`-style Pipeline commands (see above) — don't try to inline a multi-line script as a quoted shell argument on Windows, it's a quoting minefield. Write it to a temp file and use `eval_file` (or the project's equivalent) instead.

## Safety: what to confirm before running

Some `unity` operations are expensive or destructive — confirm with the user before running these rather than firing them off proactively:

- **`unity install` / `unity install-modules`** — downloads can be multiple GB and take a long time. Run with `--dry-run` first and show the user what would download before running for real, unless they've already explicitly named the exact version/modules they want installed.
- **`--allow-install` on build/test/run** — same reason; it silently triggers an install mid-command.
- **`unity uninstall`, `unity self-uninstall`, `unity projects remove`** — removes editors/CLI/registry entries. `projects remove` doesn't delete files from disk, but `uninstall` does remove an installed editor.
- **`--allow-dirty-build`** — bypasses the uncommitted-changes guard; only use if the user explicitly asks to build dirty.
- **`unity projects create --vcs github/gitlab`** — creates and pushes to a *new remote repository* under the user's account. Confirm the target namespace/visibility before running, same as any other action that creates shared/remote state.
- **`unity command <name>` against a live Editor (Pipeline)** — this is the one most likely to be underestimated. Any command that creates, modifies, or deletes GameObjects, prefabs, assets, scenes, or project files is changing the user's actual project, often without Undo support, and `eval`-style commands run arbitrary code with no guardrails at all. This doesn't mean confirming before every individual call — see "Scope of approval" in the Pipeline workflow section above for how to get task-level sign-off once and then execute without nagging, reserving re-confirmation for scope creep or genuinely irreversible edges (overwriting/deleting something that predates the task).
- **`unity pipeline install`** — modifies the project's package manifest and triggers a domain reload, interrupting whatever the Editor was doing at that moment. Low-stakes compared to the rest of this list, but not a true no-op either — avoid running it mid-task if the user is actively working in the Editor.

Full reference for every other command (`auth`, `cloud`, `hub`, `templates`, `config`, `modules`, `cache`, `analytics`, `logs`, `license`, `releases`, `changelog`, `env`, `diagnose`, `completion`, `shell`) is in [references/commands-reference.md](references/commands-reference.md).
