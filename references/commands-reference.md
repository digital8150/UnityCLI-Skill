# Unity CLI — full command reference

Generated from `unity --help` and per-command `--help` output (CLI version 1.0.0-beta.2). Run `unity <command> --help` to double-check flags for anything not detailed here, since this is a beta CLI and flags can change between versions — treat this file as a fast-reference, not a substitute for `--help` when precision matters (e.g. before a build that will take a long time to fail).

Every command also accepts these global options: `-V/--version`, `--format <human|json|tsv>`, `--no-banner`, `--non-interactive`, `--quiet`, `--verbose`, `--proxy <url>`, `--proxy-disable`, `--log-proxy`, `--no-log-proxy`.

## Table of contents
- [build](#build)
- [test](#test)
- [run](#run)
- [open](#open)
- [projects](#projects)
- [editors](#editors)
- [install / install-modules](#install--install-modules)
- [command / list / status / pipeline](#command--list--status--pipeline-live-editor-control)
- [mcp](#mcp)
- [templates](#templates)
- [auth](#auth)
- [config](#config)
- [doctor / diagnose / logs / env](#doctor--diagnose--logs--env)
- [Other commands (brief)](#other-commands-brief)

## build

`unity build [options] [project]` — runs the Editor in batch mode with common CI flags. Project defaults to the current directory.

Required: `--target <target>` (e.g. `StandaloneWindows64`, `StandaloneOSX`, `Android`, `iOS`, `WebGL`), `--execute-method <Class.Method>` (Unity has no built-in CLI build — this static method must exist in the project and perform the actual `BuildPipeline.BuildPlayer` call).

Common options:
- `--build-target-group <group>` — passed as `-buildTargetGroup`
- `-o, --output-path <path>` — passed as `-buildOutput`; the `executeMethod` is responsible for actually writing there
- `-l, --log-file <path>` — default `<project>/Logs/build-<target>-<timestamp>.log`
- `--editor-version <version>` — override the version read from `ProjectVersion.txt`
- `-e, --editor-path <path>` — use a specific Editor binary instead of resolving by version
- `-a, --architecture <x86_64|arm64>`
- `--args <string>` — extra args passed to Unity (shell-split)
- `--no-tail` — don't stream the log to stdout live
- `--allow-install` — auto-install the required Editor version if missing (confirm with user first — see SKILL.md Safety)
- `--versioning-strategy <semantic|tag|custom|none>` (default `none`) + `--build-version <version>` (only used with `custom`)
- `--allow-dirty-build` — skip the uncommitted-changes guard (default: build refuses if the working tree is dirty)

Android-specific (only apply with `--target Android`):
- `--android-export-type <apk|aab|android-studio-project>` (default `apk`)
- `--android-keystore-base64 <content>`, `--android-keystore-password <pass>`, `--android-key-alias <alias>`, `--android-key-alias-password <pass>` — signing. **CLI warns these appear in shell history/CI logs** — prefer env vars.
- `--android-target-sdk-version <N>`
- `--android-symbol-type <none|public|debugging>` (default `none`)
- `--android-version-code <N>`

Examples:
```
unity build --target StandaloneWindows64 --execute-method Builder.PerformBuild
unity build ./MyGame --target Android --execute-method Builder.AndroidBuild --output-path ./out/app.apk
unity build "My Game" --target WebGL --execute-method Builder.WebGLBuild --editor-version 6000.0
unity build --target iOS --execute-method Builder.iOSBuild --allow-install --no-tail
unity build --target Android --execute-method Builder.Build --android-export-type aab --android-keystore-base64 <b64> --android-keystore-password secret --android-key-alias mykey
```

## test

`unity test [options] [project]` — runs EditMode/PlayMode tests, writes an NUnit XML report.

- `--mode <EditMode|PlayMode>` — omit to run the Editor's default test platform
- `--filter <pattern>` — only tests whose name matches
- `--output <path>` (default `test-results.xml`)
- `--editor-version`, `-e/--editor-path`, `-a/--architecture`, `--allow-install` — same semantics as `build`
- `--timeout <seconds>` — kill the Unity process after this long (disabled by default; set this for CI/unattended runs so a hung test doesn't hang forever)

Examples:
```
unity test
unity test ./MyProject --mode EditMode
unity test "My Game" --mode PlayMode --output ./results/play.xml
unity test . --filter "MyNamespace.MyTests" --editor-version 6000.0
unity test . -- -nographics
```

## run

`unity run [options] [project]` — runs the project in batch mode, forwarding everything after `--` to the Editor as raw args. Use this for anything `build`/`test` don't cover (custom `-executeMethod` calls that aren't a build, asset-database refresh scripts, etc.).

- `--editor-version`, `-e/--editor-path`, `-a/--architecture`, `--allow-install`, `--timeout <seconds>` — same as `test`

Examples:
```
unity run
unity run ./MyProject -- -executeMethod Builder.Build
unity run "My Game" --editor-version 6000.0.0f1 -- -nographics -quit
unity run . --allow-install -- -logFile ./build.log -quit
```

## open

`unity open [options] [project]` — opens a project in the Editor GUI (not headless) with the right Editor version.

- `--editor-version`, `-e/--editor-path`, `-a/--architecture`
- `--build-target <target>`, `--build-target-group <group>` — passed through to Unity
- `--args <string>`

Argument resolution: exact Hub name/path → glob pattern (multiple matches → picker) → filesystem path.

```
unity open
unity open "My Game"
unity open "My Game*"
unity open /path/to/project
unity open . --editor-version 6000.0.0f1
```

`unity projects open [pattern]` is the Hub-registry-aware sibling — same flags, but resolution order also includes a fuzzy title match step (exact → glob → fuzzy → filesystem path) before falling back to a raw path.

## projects

`unity projects` (alias `p`) manages the Hub project registry.

| Subcommand | Purpose |
|---|---|
| `list [pattern]` | list registered projects |
| `add <paths...>` | register existing project folder(s) |
| `remove <paths...>` | unregister (does **not** delete files) |
| `info [pathOrName]` | detailed metadata |
| `create <name>` | create + register a new project; supports `--cloud`, `--cloud-org`, `--cloud-project`, `--vcs github\|gitlab`, `--git-namespace`, `--git-repo`, `--git-visibility private\|public\|internal` (default private), `--git-default-branch`, `--git-token`/`--git-token-stdin`, `--no-initial-commit`, `--git-lfs` |
| `new <name>` | create a project non-interactively, CI-friendly, no Hub registration weirdness; `--path`, `--editor-version`, `--template`, `--open`, `--json` |
| `open [pattern]` | open by name/fuzzy/glob/path |
| `pin <pattern>` / `unpin <pattern>` | favorite/unfavorite |
| `require [pathOrName]` | ensure the project's required Editor version is installed, installing if needed |
| `link` / `unlink` | connect/disconnect a local project to a Cloud or VCS link |
| `upgrade [pathOrName]` | upgrade a project to a different Editor version |
| `export` / `import` | dump/restore the Hub project list as JSON |

`create` examples:
```
unity projects create MyGame --editor-version 6000.0.30f1
unity projects create MyGame --template com.unity.template.3d --path ~/UnityProjects
unity projects create MyGame --cloud
unity projects create MyGame --vcs github --git-token-stdin
```

## editors

`unity editors` (alias `e`) manages installed/available Editor versions.

- `-i, --installed` — list installed editors; `-r, --releases` — list downloadable releases
- `--json`, `--verbose` (full module names/paths), `-w/--watch`

| Subcommand | Purpose |
|---|---|
| `add <path...>` | register an existing Editor install with Hub |
| `default [version]` | show/set the default Editor version |
| `list` | installed editors or available releases |
| `info <version>` | release details for a version |
| `module` | manage modules of installed editors — `list <version>`, `add <version>`, `refresh <version>` |
| `install-path`/`ip` | show/change global editor install directory |
| `path <version>` | print install directory for a version |
| `upgrade [editor]` | upgrade an installed editor to latest patch in its release line |

## install / install-modules

`unity install [version]` (alias `i`) — installs an Editor version (omit version for an interactive picker).
- `-m, --module <id...>` — modules to install alongside (repeatable)
- `--cm/--no-cm` (`--childModules`) — auto-include/exclude submodules
- `-f, --force` — reinstall even if present
- `-y, --yes` — auto-accept first match, no confirmation
- `--accept-eula` — accept all module license agreements automatically
- `--dry-run` — **show what would download without installing** — always run this first for anything not explicitly pre-approved by the user
- `--resume` — resume an interrupted cached download
- `--no-elevate` — skip the Windows UAC elevation helper (installs to protected paths will just fail instead of prompting)

`unity install-modules` (alias `im`) — installs/lists modules for an *already-installed* editor. No args = full interactive mode.
- `-e, --editor-version <version>`, `-m, --module <id...>`, `-l, --list`, `--all`
- `--reinstall`, `-f/--force` (force implies reinstall + skip confirmation + include submodules)
- `--retries <n>` — retry failed downloads/verifications with backoff (0 disables)
- same `--dry-run`, `--accept-eula`, `--cm/--no-cm`, `--no-elevate` as `install`

## command / list / status / pipeline (live Editor control)

These talk to a **currently running** Unity Editor over the Pipeline package's local HTTP server — not batch mode. **Read the "read this before using it" callout in SKILL.md first** — this command surface commonly includes arbitrary-code-execution (an `eval`-style command), not just narrow, safe accessors.

- `unity status [--port <n>] [--project <substring>]` — live table of every connected Editor: port, state, project, version, PID. Empty output = nothing running (or Pipeline not installed in the open project).
- `unity list [--project-path <path>] [--runtime <name>] [--runtime-path <path>]` — lists commands/tools the connected Editor's Pipeline package registers. Run this before calling `command` on anything you haven't confirmed exists — depending on the package version, this list can run to 100+ entries. Add `--format json` and filter for the keyword you need (grep/jq) rather than reading the full human-readable table into context every time; once you've confirmed a command within a session you don't need to re-list before calling it again.
- `unity command <name> [args...]` (alias `cmd`) — invokes a registered command on the Editor. Bare `unity command` (no name) also lists available commands. `--project-path` auto-detects if omitted; `--runtime`/`--runtime-path` target a Player build instead of the Editor; `--timeout <seconds>` (default 30).
  - **Arguments are positional**, e.g. `unity command eval "return 1+1;" --project-path <path>` — not `key=value` pairs. Passing `key=value` gets forwarded as a literal string argument to the command instead of being parsed as a named parameter, and will fail or produce a confusing error. If a command's parameter order isn't obvious from its name, look for it in `unity list`'s output before guessing.
  - **Response shape**: with `--format json`, the actual return value is nested at `data.result.result`. Its type is command-specific — sometimes a plain string, sometimes a JSON object — so parse it defensively (check the type at runtime) rather than assuming one shape applies to every command.
  - **A common project often exposes `recompile` / `recompile_status` / `console`-style commands** (exact names vary by Pipeline version — confirm via `unity list`) — use them after any script edit to verify the change actually compiled instead of asking the user to check the Unity console themselves.
  - **`eval`-style commands**: batch multiple changes into one call rather than one call per field; use fully-qualified type names (e.g. `UnityEditor.AssetDatabase`) since the snippet doesn't inherit the project's normal `using`s; call `AssetDatabase.SaveAssets()` + `.Refresh()` at the end of anything touching assets or the change may not persist to disk; and for multi-line scripts, write to a file and use `eval_file` (or the project's equivalent) instead of inlining as a shell argument — especially on Windows, where quoting multi-line code inline is unreliable.
  - **Mutating commands are often not undoable.** Some integrate with Unity's Undo system (Ctrl+Z works); many explicitly don't, and anything code-execution-based (`eval`) bypasses Undo entirely. Get the user's go-ahead for the scope of a task once, then execute the mutating calls it requires without re-confirming each one — reserve fresh confirmation for scope creep or edits to something that predates the task (overwriting/deleting existing data). See "Scope of approval" in SKILL.md.
- `unity pipeline` (alias `pipe`) manages the Pipeline package itself:
  - `install [--project-path <path>] [--force] [--package-version <version>]` — install the package into a project (one-time setup, required before `list`/`command`/`mcp` will work for that project). Modifies the project's manifest and triggers a domain reload — avoid running it while the Editor is mid-task on something else.
  - `upgrade [--project-path <path>]`
  - `list` — every Editor instance + its Pipeline package status
  - `list-versions` — available package versions

```
unity pipeline install --project-path /path/to/project
unity pipeline list
unity status
unity list --project-path /path/to/project
unity command <tool-name> arg1 arg2 --project-path /path/to/project
```

If `unity command`/`unity list` errors with "No Unity Editor instances found with reachable Pipeline servers," it means either no Editor is open, or the open project doesn't have the Pipeline package installed yet — run `unity pipeline install` first, then reopen/keep the project open.

## mcp

`unity mcp [options] [command]` — MCP server/client wiring for talking to a running Editor as an AI agent tool, instead of one-off CLI calls.

- Bare `unity mcp` starts the stdio MCP server for the current/specified project.
- `--project-path <path>` — locate the Editor by project path (and with `configure`, pin the config entry to that project)
- `--runtime <version>`, `--runtime-path <path>` — target a Player runtime instead
- `configure [client]` — write an MCP server config entry for a specific AI client (e.g. `claude`). `configure --list` shows all supported clients.

```
unity mcp                                   # start the MCP stdio server
unity mcp --project-path /path/to/MyProject # start server scoped to one project
unity mcp configure claude                  # wire up Claude Desktop
unity mcp configure --list                  # show all supported clients
```

## templates

`unity templates` (alias `t`) — browse/inspect/manage project templates (the `--template` values used by `projects create`/`projects new`).

| Subcommand | Purpose |
|---|---|
| `list` | templates available for an Editor version |
| `info <name>` | details for one template |
| `create <project-path>` | turn an existing project into a custom template |
| `delete <name>` | delete a user-generated template |
| `location` | get/set/reset the storage path for custom templates |
| `edit <name>` | edit a custom template's metadata |

## auth

`unity auth` (alias `a`) — Unity account session, needed for Cloud-linked operations (`projects create --cloud`, etc.).

| Subcommand | Purpose |
|---|---|
| `login` | opens a browser to log in |
| `status` | show current login state |
| `logout` | log out and delete stored credentials |

## config

`unity config` — persistent CLI configuration.

| Subcommand | Purpose |
|---|---|
| `proxy [url]` | view/change configured proxy |
| `update-check [state]` | enable/disable background update checks |

## doctor / diagnose / logs / env

- `unity doctor [--tail <n>]` — environment health check + recent log lines (default 20). Run this first when something's misbehaving.
- `unity diagnose` — one-shot, paste-safe (redacted) diagnostic output, meant for pasting into a support request.
- `unity logs [options]` — read or tail Hub log files.
- `unity env` — print Unity Hub environment paths and versions.

## Other commands (brief)

Run `unity <command> --help` for full flags on any of these — they're lower-frequency so aren't detailed above:

- `analytics` — manage analytics/telemetry consent
- `bug` — file a bug report directly to Unity's bug reporter
- `cache` — manage the CLI's download cache
- `changelog` — show release notes for the installed CLI
- `cloud` — manage Unity Cloud organizations/projects
- `completion <shell>` — print a shell completion script
- `editor` — singular alias surface for editor install management (see `editors` above for the full command set)
- `hub` — manage the Unity Hub application itself
- `install-path`/`ip` — get/set the global Editor install path (also under `editors`)
- `language`/`lang` — show/change the CLI's display language (CLI currently defaults to Korean on this machine — pass `--format` output stays consistent across languages, but human-readable text won't)
- `license` — show the Unity license activated on this machine
- `modules` — list/manage Editor modules (also under `editors module`)
- `releases` — list available Unity releases (also under `editors -r`)
- `self-uninstall` — uninstall the `unity` CLI itself (deletes binary + env files) — confirm with the user, this is destructive to their tooling
- `shell` — interactive REPL that runs multiple commands against one already-started process (useful for scripting many operations without repeated process startup cost)
- `uninstall`/`u [version]` — uninstall an installed Editor version — confirm with the user first
- `upgrade` — upgrade the `unity` CLI itself to the latest version
