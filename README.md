<p align="center">
  <img src="logo.svg" width="280" alt="A mackerel">
</p>

<h1 align="center">mackerel</h1>

<p align="center">Free disk space on a Mac without losing work.</p>

---

Coding agents eat disk. Each one gets a git worktree with its own `node_modules`, package managers keep
every version they ever downloaded, and simulators, model weights and build caches pile up behind them.
On a 512 GB Mac that is a full disk every few days, and agents left running overnight start failing.

mackerel measures what can go, shows you, and cleans it when you say yes. It only deletes things that
can be rebuilt, and it leaves alone anything an agent might still be using.

```
  ><≋≋≋°>  mackerel  dry run

  █████████████████████░░░   43.6 GB free of 460.4 GB, 91% full

  Safe  rebuilds on its own
  uv         uv cache                                  167 MB  drops entries nothing uses
  npm        npm cache                                 3.4 GB  re-downloaded on next install
  swiftpm    Swift Package Manager cache                    -  nothing to free
  sim        Invalid iOS simulators                         -  nothing to free
  agents     Finished agent worktrees                  1.5 GB  clean, idle 24 h; branch kept
  docker     Dangling Docker images, build cache            -  OrbStack is off

  Deep  only with --deep
  next       Project .next caches                      368 MB  rebuilt on next start
  playwright Playwright browsers                            -  nothing to free
  ...

  Up to 5.0 GB can be freed. --deep adds 368 MB.

  Dry run: nothing was touched.
```

## Install

mackerel is one Bash script with no dependencies beyond what macOS ships.

```sh
git clone https://github.com/emnlmn/mackerel.git ~/Code/mackerel
ln -s ~/Code/mackerel/mackerel ~/.local/bin/mackerel   # or any directory on your PATH
```

## Usage

```sh
mackerel                         # show what can be freed, ask, clean the safe tier
mackerel --dry-run               # show what can be freed and stop
mackerel --deep                  # the deep tier too
mackerel --only uv,npm           # just these targets
mackerel --yes --if-below 60     # unattended, and only if less than 60 GB is free
mackerel --help
```

| Option | |
|---|---|
| `-n`, `--dry-run` | show the table and stop; without `--deep` it also lists what `--deep` would add |
| `-d`, `--deep` | include the deep tier |
| `-y`, `--yes` | don't ask; required when there is no terminal |
| `--if-below <GB>` | do nothing unless free space is under this many GB |
| `--only <id,id>` | only these targets, from either tier |
| `-h`, `--help` | usage |
| `-v`, `--version` | version |

Measuring takes a few seconds. A status line shows which target it is on, and nothing is deleted until
you answer `y`.

## What it cleans

Targets come in two tiers. The safe tier runs by default. The deep tier runs only with `--deep` or when
you name a target in `--only`.

| id | Tier | What | What you lose |
|---|---|---|---|
| `uv` | safe | uv cache, via `uv cache prune` | nothing; only unused entries go |
| `npm` | safe | npm cache, via `npm cache clean` | a re-download on the next install |
| `swiftpm` | safe | Swift Package Manager cache | a re-download on the next build |
| `sim` | safe | iOS simulators whose runtime is gone | nothing usable |
| `agents` | safe | finished worktrees in `<repo>/.claude/worktrees` | nothing; the branch stays in the repo |
| `docker` | safe | untagged Docker images and build cache | rebuild time |
| `next` | deep | `.next/cache` in your projects | a slower first start |
| `playwright` | deep | Playwright browsers | `npx playwright install` brings them back |
| `devsup` | deep | Xcode iOS device support | rebuilt when an iPhone connects |
| `hf` | deep | Hugging Face models in `~/.cache/huggingface` | a re-download |
| `litert` | deep | `~/.cache/litert-night` | rebuilt on next use |
| `sessions` | deep | Claude Code transcripts older than 30 days | those sessions can't be resumed |
| `trash` | deep | the Trash | everything in it, for good |

A target that doesn't apply to your machine (no Xcode, Docker not running, nothing to free) shows up
dimmed with the reason and is skipped.

## What it never touches

- **Docker volumes.** They often hold databases. If unused volumes take more than a gigabyte, mackerel
  lists them under "Check by hand" and leaves the decision to you.
- **Agent worktrees that might be in use.** A worktree goes only when it is unlocked, has no
  uncommitted changes, and no file in it changed in the last 24 hours. `git worktree remove` deletes
  the folder; the branch and its commits stay.
- **Claude Code's memory.** The `sessions` target deletes `*.jsonl` transcripts only.
- **Anything outside your home folder.** Every directory it empties is checked against `$HOME` first.
- **A stopped OrbStack or Docker Desktop.** mackerel never starts it just to clean it.

## Run it every evening

Clean before overnight agents start, and only when space is actually short:

```sh
crontab -e
```

```
0 21 * * *  $HOME/Code/mackerel/mackerel --yes --if-below 60
```

Under cron mackerel prints plain text, adds `~/.local/bin` and Homebrew to `PATH` so it can find `uv`
and `docker`, and refuses to run without `--yes`, since there is no one to ask. Two runs never overlap:
the second exits with code 3.

## Log

Every run appends to `~/Library/Logs/mackerel.log`: one line per target with what it freed, plus the
full output of each cleanup command. When a target fails, the details are there.

## Configuration

| Variable | Default | |
|---|---|---|
| `MACKEREL_CODE_DIR` | `~/Code` | where to look for agent worktrees and `.next` caches |
| `MACKEREL_SESSION_DAYS` | `30` | how old a Claude Code session must be before `sessions` deletes it |
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
`uv`, `npm`, Xcode and Docker are optional: each target checks for its tool and skips itself if the
tool is missing.

## Adding a target

A target is one `reg` line and three functions:

```bash
reg mytool safe "mytool cache" "re-downloaded on next run"

mytool_check() { command -v mytool >/dev/null || { echo "mytool not installed"; return 1; }; }
mytool_size()  { du_kb "$HOME/.cache/mytool"; }
mytool_clean() { empty_dir "$HOME/.cache/mytool"; }
```

`_check` prints why the target can't run, `_size` prints kilobytes, `_clean` does the work. mackerel
measures again after cleaning, so the freed amount it reports is real.

## Why "mackerel"

In Italian, *sgombro* is a mackerel. It is also what you say when you clear things out: *sgombro la
cantina* means "I'm clearing out the basement." So a fish that cleans your disk was hard to resist.

For its first hour the tool was called sgombro. Then it got an English name and kept the fish.

## License

[MIT](LICENSE)
