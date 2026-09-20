# herdr-workflows

Bash scripts for common workflows on top of [herdr](https://herdr.dev) (a terminal
multiplexer for coding agents). Herdr's CLI is low-level — panes, tabs, agents,
worktrees. This repo is the higher-level layer: one command each for the things
I do over and over.

## Why

Doing these by hand means chaining several `herdr` calls (find/create a pane,
start an agent, wait, prompt, ...). These scripts wrap that up into one
command each, so the workflow is a single memorable entrypoint instead of
re-deriving the `herdr` incantation every time.

## Conventions (tentative)

- One script per workflow, named `herdr-<verb>-<type>` (e.g. `herdr-run-agent`,
  `herdr-create-worktree`, `herdr-delete-worktree`).
- Plain bash, `set -euo pipefail`.
- Dependencies: `bash`, `herdr`, `jq`, and potentially [`gum`](https://github.com/charmbracelet/gum)
  for interactive prompts/menus where a script needs to ask the user something.
- Scripts live in `bin/` in this repo. Not on `PATH` yet — at some point `bin/`
  gets added to `PATH` (or symlinked in) so the scripts are runnable from
  anywhere.
- Each script is a thin wrapper: shell out to `herdr`, do the polling/gluing,
  print something useful. No reimplementing what `herdr` already does.
- Tunables (agent kind, default base branch, etc.) are declared as constants
  near the top of each script rather than hardcoded inline, so they're easy to
  find and change later. See "Multi-agent support" below.

## Candidate workflows

- **`herdr-run-simplify`** — kick off a "simplify" session on the current
  branch/diff. Create or reuse a sibling pane, start an agent there, prompt it
  to run `/simplify` (or equivalent) against the current changes.
- **`herdr-create-worktree`** — create a new git worktree for a branch and open
  it in herdr (`herdr worktree create` + focus), so starting a new line of
  work is one command instead of the usual git dance.
- **`herdr-delete-worktree`** — clean up a worktree once it's done. Before
  removing anything, check whether the worktree's branch is merged into the
  trunk branch; if it isn't, stop and warn instead of deleting (see "Worktree
  cleanup safety" below).
- **`herdr-run-agent`** — kick off a new agent with a prompt in one shot:
  create/pick a pane, `herdr agent start`, then `herdr agent prompt <text>`.
  The building block a lot of the other scripts probably call into.
- **`herdr-create-workspace`** — set up a new herdr workspace with a standard
  set of tabs already open (e.g. editor/shell pane, agent pane, test runner),
  so a new project/task starts from a consistent layout instead of assembling
  tabs by hand each time.

## Multi-agent support

Today this is all Claude, but herdr supports other agent kinds (codex, gemini,
etc.), and we should design for that from the start rather than hardcode
`claude` everywhere. Concretely: each script declares its agent kind (and any
other agent-specific bits, like the simplify slash-command name) as a constant
near the top, e.g.

```bash
# --- config ---
AGENT_KIND="claude"
```

so swapping/adding a kind later is a one-line change per script instead of a
search-and-replace.

## Worktree cleanup safety

`herdr-delete-worktree` must not silently delete unmerged work. Before calling
`herdr worktree remove`:

1. Determine the worktree's branch and the trunk branch (default `main`,
   overridable via `--trunk`).
2. Check whether the branch is merged into trunk via
   `git merge-base --is-ancestor <branch> <trunk>`. This is intentionally
   conservative: it can't prove a squash- or rebase-merged branch is merged,
   so it defaults to "not merged" whenever ancestry doesn't confirm it.
3. If not merged (including that ambiguous case), prompt interactively to
   continue or abort instead of proceeding automatically. There is no
   `--force` flag; the interactive prompt is the only override.

## Open questions

- Where should config live (default branch base, pane layout preferences,
  etc.) — flags, env vars, or a dotfile — vs. the per-script constants above?
- For multi-agent support, is agent kind a per-script constant (above) or a
  shared config file all scripts read?
