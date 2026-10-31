<p align="center">
  <img src="logo.svg" width="280" alt="A mackerel">
</p>

<h1 align="center">mackerel</h1>

<p align="center">Free disk space on a Mac without losing work.</p>

---

Package managers keep every version they ever downloaded. Build folders grow with every branch, apps
cache what you looked at last month, and coding agents add a worktree with its own dependencies each
time they start. None of it is work you'd miss, and on a 512 GB Mac it fills the disk in days.

mackerel knows where about 80 tools keep that kind of data: package managers for the most used
languages, build tools, IDEs, browsers, design apps and chat apps. It measures what is there, shows you
the biggest items first, and cleans them when you say yes. It deletes only what the tool can rebuild
or download again.

```
  ><≋≋≋°>  mackerel  dry run

  ██████████████████████░░   39.7 GB free of 460.4 GB, 92% full

  Safe  refilled by its tool
  npm         npm cache                               3.4 GB  re-downloaded on next install
  cargo       Cargo registry cache                    1.6 GB  re-downloaded on next build
  agents      Finished agent worktrees                1.5 GB  clean, idle 24 h; branch kept
  spotify     Spotify cache                           679 MB  offline downloads are kept
  jetbrains   JetBrains IDE caches                    640 MB  rebuilt when the IDE opens
  slack       Slack caches                            535 MB  rebuilt when Slack opens
  ...
  chrome      Chrome cache                            2.1 GB  Chrome is running
  docker      Dangling Docker images, build cache          -  Docker is not running

  Deep  only with --deep
  rusttarget  Rust target folders                   129.5 GB  rebuilt on next cargo build
  projcache   Build caches in your projects           368 MB  rebuilt on next build

  Up to 10.1 GB can be freed. --deep adds 129.9 GB.
  58 more targets have nothing to free on this Mac. --all lists them.

  Check by hand  mackerel never deletes these
  · Ollama models: 43.7 GB. ollama list, then ollama rm the ones you don't use.

  Dry run: nothing was touched.
```

## Install

mackerel is one Bash script. It needs nothing beyond what macOS ships.

```sh
git clone https://github.com/emnlmn/mackerel.git ~/Code/mackerel
ln -s ~/Code/mackerel/mackerel ~/.local/bin/mackerel   # or any directory on your PATH
```

## Usage

```sh
mackerel                         # show what can be freed, ask, clean the safe tier
mackerel --dry-run               # show what can be freed and stop
mackerel --deep                  # the deep tier too
mackerel --only npm,docker       # just these targets
mackerel --yes --if-below 60     # unattended, and only if less than 60 GB is free
mackerel --help
```

| Option | |
|---|---|
| `-n`, `--dry-run` | show the table and stop; without `--deep` it also lists what `--deep` would add |
| `-d`, `--deep` | include the deep tier |
| `-a`, `--all` | also list targets with nothing to free, and why |
| `-y`, `--yes` | don't ask; required when there is no terminal |
| `--if-below <GB>` | do nothing unless free space is under this many GB |
| `--only <id,id>` | only these targets, from either tier |
| `-h`, `--help` | usage |
| `-v`, `--version` | version |

A run takes 10 to 30 seconds, mostly spent measuring folders. A status line shows which target it is
on, and nothing is deleted until you answer `y`.

## How it decides

**Two tiers.** The safe tier holds caches their tool refills on its own the next time it needs them. It
runs by default. The deep tier holds things that are still caches but cost more to get back: a full
Rust rebuild, browser binaries, model weights, old agent sessions, the Trash. It runs only with
`--deep`, or when you name a target in `--only`.

**It asks the tool where its cache is.** Where a tool can say (`npm config get cache`, `uv cache dir`,
`pnpm store path`, `brew --cache`, `go env GOCACHE`), mackerel asks. Otherwise it honours the tool's own
variable (`PIP_CACHE_DIR`, `CARGO_HOME`, `GRADLE_USER_HOME`, `HF_HOME`, `XDG_CACHE_HOME` and so on)
before falling back to the macOS default. A target whose folder isn't there doesn't apply to your Mac
and stays out of the table.

**It uses the tool's own cleanup when there is one.** `uv cache prune`, `pnpm store prune`,
`brew cleanup`, `conda clean`, `go clean -modcache`, `docker image prune`, `xcrun simctl delete
unavailable`. Those know which files are still in use. Everything else gets its folder emptied.

**It waits for open apps.** App caches (Slack, Chrome, VS Code, Figma, a JetBrains IDE and the rest)
are skipped while the app is running, because an open app keeps rewriting them. The table says which
app to quit.

**It finds your projects.** For worktrees and build folders it looks in `~/Code`, `~/Projects`,
`~/Developer`, `~/dev`, `~/src`, `~/workspace`, `~/repos`, `~/git` and `~/GitHub`, whichever exist, four
levels deep. Set `MACKEREL_CODE_DIR` to choose the folders yourself.

**It reports what it measured.** After each target it measures again, so the freed amount is what
actually left the disk.

## What it never deletes

- **Things people keep on purpose.** Docker volumes, Ollama and LM Studio models, iPhone backups,
  Xcode archives, Android emulators, Vagrant boxes, virtual machines and a large Downloads folder show
  up under "Check by hand" when they pass a gigabyte, with a hint on how to clean them. mackerel
  never touches them.
- **Agent worktrees that might be in use.** A worktree in `<repo>/.claude/worktrees` goes only when it
  is unlocked, has no uncommitted changes, and no file in it changed in the last 24 hours.
  `git worktree remove` deletes the folder; the branch and its commits stay.
- **Recent dependencies.** `staledeps` removes `node_modules` and virtualenvs only from projects where
  no file changed in 90 days.
- **Rust builds in progress.** `rusttarget` waits while `cargo` runs.
- **Logins and settings.** It empties cache folders, not app data. Hugging Face keeps your token,
  Poetry keeps its virtualenvs, Spotify keeps your offline downloads, pub keeps global tools.
- **Claude Code's memory.** `sessions` deletes `*.jsonl` transcripts only.
- **Anything outside your home folder.** Every folder it empties or removes is checked against `$HOME`
  first, even when a variable points somewhere else.
- **A stopped Docker.** mackerel never starts OrbStack, Docker Desktop or Colima just to clean them.

## Targets

`mackerel --dry-run --all` lists every target with its id and whether it applies to your Mac.

<details>
<summary>All 80 targets</summary>

**JavaScript and TypeScript**

| id | What | Notes |
|---|---|---|
| `npm` | npm cache | re-downloaded on next install |
| `pnpm` | pnpm store, unused packages | pnpm store prune |
| `yarn` | Yarn cache | re-downloaded on next install |
| `bun` | Bun cache | re-downloaded on next install |
| `deno` | Deno cache | re-downloaded on next run |
| `nodegyp` | node-gyp headers | re-downloaded on next native build |

**Python**

| id | What | Notes |
|---|---|---|
| `uv` | uv cache | drops entries nothing uses |
| `pip` | pip cache | re-downloaded on next install |
| `poetry` | Poetry cache | virtualenvs are kept |
| `pipenv` | Pipenv cache | re-downloaded on next install |
| `conda` | Conda package cache | conda clean --all; envs are kept |

**Java, Kotlin, Scala**

| id | What | Notes |
|---|---|---|
| `gradle` | Gradle caches | re-downloaded on next build |
| `coursier` | Coursier and Ivy caches | re-downloaded on next build |

**C#, F#**

| id | What | Notes |
|---|---|---|
| `nuget` | NuGet packages | re-downloaded on next restore |

**Go, Rust, PHP, Ruby**

| id | What | Notes |
|---|---|---|
| `gobuild` | Go build cache | rebuilt on next build |
| `cargo` | Cargo registry cache | re-downloaded on next build |
| `composer` | Composer cache | re-downloaded on next install |
| `ruby` | Bundler and Ruby installer caches | re-downloaded when needed |

**Swift, Objective-C, Xcode**

| id | What | Notes |
|---|---|---|
| `swiftpm` | Swift Package Manager cache | re-downloaded on next build |
| `cocoapods` | CocoaPods cache | re-downloaded on next pod install |
| `carthage` | Carthage cache | re-downloaded on next build |
| `derived` | Xcode DerivedData | rebuilt on next build |
| `xcodecache` | Xcode previews, simulator caches | rebuilt when needed |
| `sim` | Invalid iOS simulators | their runtime is gone |

**Dart, Elixir, Haskell, Perl, Lua, Zig, OCaml**

| id | What | Notes |
|---|---|---|
| `pub` | Dart and Flutter pub cache | global tools are kept |
| `hex` | Hex and rebar3 caches | re-downloaded on next deps.get |
| `cabal` | Cabal package cache | re-downloaded on next build |
| `cpanm` | cpanm build folder | Perl; only build leftovers |
| `luarocks` | LuaRocks cache | re-downloaded when needed |
| `zig` | Zig cache | rebuilt on next build |
| `opam` | opam download cache | re-downloaded when needed |

**C and C++**

| id | What | Notes |
|---|---|---|
| `ccache` | ccache | rebuilt on next compile |
| `sccache` | sccache | rebuilt on next compile |
| `vcpkg` | vcpkg binary cache | rebuilt on next install |

**Tools**

| id | What | Notes |
|---|---|---|
| `brew` | Homebrew downloads, old versions | brew cleanup |
| `docker` | Dangling Docker images, build cache | volumes are kept |
| `terraform` | Terraform plugin cache | re-downloaded on next init |
| `minikube` | minikube image cache | re-downloaded on next start |
| `android` | Android SDK cache | re-downloaded when needed |

**Editors**

| id | What | Notes |
|---|---|---|
| `jetbrains` | JetBrains IDE caches | rebuilt when the IDE opens |
| `vscode` | VS Code caches | rebuilt when VS Code opens |
| `cursor` | Cursor caches | rebuilt when Cursor opens |
| `windsurf` | Windsurf caches | rebuilt when Windsurf opens |
| `postman` | Postman caches | rebuilt when Postman opens |

**Browsers**

| id | What | Notes |
|---|---|---|
| `chrome` | Chrome cache | rebuilt as you browse |
| `arc` | Arc cache | rebuilt as you browse |
| `brave` | Brave cache | rebuilt as you browse |
| `edge` | Edge cache | rebuilt as you browse |
| `firefox` | Firefox cache | rebuilt as you browse |

**Design**

| id | What | Notes |
|---|---|---|
| `adobe` | Adobe media cache | rebuilt when the app opens |
| `figma` | Figma cache | rebuilt when Figma opens |
| `sketch` | Sketch cache | rebuilt when Sketch opens |

**Work and chat**

| id | What | Notes |
|---|---|---|
| `slack` | Slack caches | rebuilt when Slack opens |
| `teams` | Microsoft Teams cache | rebuilt when Teams opens |
| `discord` | Discord caches | rebuilt when Discord opens |
| `notion` | Notion caches | rebuilt when Notion opens |
| `linear` | Linear caches | rebuilt when Linear opens |
| `claude` | Claude desktop caches | rebuilt when Claude opens |
| `spotify` | Spotify cache | offline downloads are kept |

**Coding agents**

| id | What | Notes |
|---|---|---|
| `agents` | Finished agent worktrees | clean, idle 24 h; branch kept |

**Deep tier** (only with `--deep`)

| id | What | Notes |
|---|---|---|
| `projcache` | Build caches in your projects | rebuilt on next build |
| `rusttarget` | Rust target folders | rebuilt on next cargo build |
| `staledeps` | Dependencies of idle projects | untouched 90 days; reinstall to use |
| `maven` | Maven local repository | re-downloaded; mvn install your own again |
| `gradledists` | Gradle distributions | re-downloaded on next build |
| `gomod` | Go module cache | re-downloaded on next build |
| `stack` | Stack GHC installs and snapshots | re-downloaded on next build |
| `julia` | Julia compiled cache | recompiled on next load |
| `renv` | renv package cache | R; reinstalled on next restore |
| `conan` | Conan packages | rebuilt or re-downloaded |
| `konan` | Kotlin/Native toolchains | re-downloaded on next build |
| `playwright` | Playwright browsers | npx playwright install brings them back |
| `puppeteer` | Puppeteer browsers | re-downloaded on next install |
| `cypress` | Cypress binaries | re-downloaded on next install |
| `devsup` | Xcode device support | rebuilt when a device connects |
| `hf` | Hugging Face models and datasets | re-downloaded; your login is kept |
| `torch` | PyTorch hub models | re-downloaded on next load |
| `whisper` | Whisper models | re-downloaded on next run |
| `sessions` | Agent sessions older than 30 days | Claude Code, Codex; can't be resumed |
| `trash` | Trash | permanent |

</details>

## Run it every evening

Clean before overnight jobs start, and only when space is actually short:

```sh
crontab -e
```

```
0 21 * * *  $HOME/Code/mackerel/mackerel --yes --if-below 60
```

Under cron mackerel prints plain text, adds `~/.local/bin`, Cargo, Go and Homebrew to `PATH` so it can
find the tools, and refuses to run without `--yes`, since there is no one to ask. Two runs never
overlap: the second exits with code 3.

## Log

Every run appends to `~/Library/Logs/mackerel.log`: one line per target with what it freed, plus the
full output of each cleanup command. When a target fails, the details are there.

## Configuration

| Variable | Default | |
|---|---|---|
| `MACKEREL_CODE_DIR` | found on their own | project folders, colon separated |
| `MACKEREL_STALE_DAYS` | `90` | idle days before `staledeps` removes a project's dependencies |
| `MACKEREL_SESSION_DAYS` | `30` | how old an agent session must be before `sessions` deletes it |
| `NO_COLOR` | | turns colors off |

## Exit codes

| Code | Meaning |
|---|---|
| 0 | done, or nothing to do |
| 1 | at least one target failed |
| 2 | usage error |
| 3 | another run is in progress |

## Requirements

macOS with its built-in Bash 3.2. The script uses no associative arrays and no `mapfile` on purpose.
Every tool is optional: a target whose tool or folder is missing skips itself.

## Adding a target

Most targets are one line: an id, a tier, a label, a note, where the data lives, and optionally an app
that must be closed first.

```bash
reg mytool safe "mytool cache" "re-downloaded on next run" \
	'"${MYTOOL_CACHE:-$HOME/Library/Caches/mytool}"' 'MyTool|/MyTool\.app/'
```

The paths are shell text, evaluated when mackerel measures, so they can ask the tool
(`'"$(mytool cache-dir 2>/dev/null)"'`). By default a target is measured with `du` and cleaned by
emptying its folders. To use the tool's own cleanup, add a function:

```bash
mytool_clean() { mytool cache prune; }
```

`<id>_check` and `<id>_size` replace the other two steps the same way. An Electron app takes one
`electron` line with its support folder; mackerel knows which of its subfolders are cache.

## Why "mackerel"

In Italian, *sgombro* is a mackerel. It is also what you say when you clear things out: *sgombro la
cantina* means "I'm clearing out the basement." So a fish that cleans your disk was hard to resist.

For its first hour the tool was called sgombro. Then it got an English name and kept the fish.

## License

[MIT](LICENSE)
