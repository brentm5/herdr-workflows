# Context

Glossary for herdr-workflows. Terms here are canonical — use them consistently in code, scripts, issues, and docs. This file is vocabulary only: no implementation details, no decisions-with-rationale (those go in `docs/adr/`).

## Terms

**Workflow**
: A common pattern of interacting with herdr, codified as a script so it's easier to invoke than re-deriving the raw `herdr` command sequence by hand. One workflow = one script in `bin/` (see the README's naming convention, `herdr-<verb>-<type>`). A workflow's identity is just its filename — no separate registry or listing. Workflows can call each other directly by name (e.g. `herdr-run-agent` invoked from inside `herdr-clean-worktree`), relying on `bin/` being on `PATH`.

**Workspace**
: herdr's top-level container. A workspace holds one or more tabs. (herdr's own term — see `herdr --skill`.)

**Tab**
: A subdivision of a workspace. A tab holds one or more panes.

**Pane**
: A terminal within a tab. A tab can be split into multiple panes, arranged horizontally or vertically. A pane exists whether or not it holds a running agent.

**Agent**
: A CLI coding agent (e.g. Claude Code, Codex) running inside a herdr pane. Distinct from **agent kind**.

**Agent kind**
: Which CLI tool an agent is — e.g. `claude`, `codex`, `gemini`. Specified via `--kind` at `herdr agent start` time. An attribute of an agent, not a separate entity.

**Worktree**
: In herdr's vocabulary (not a plain pass-through to `git worktree`): a **workspace** whose filesystem checkout is backed by a dedicated git worktree. Created/opened via `herdr worktree create` / `herdr worktree open`, which create both the git worktree and the herdr workspace wired to it together. Removed via `herdr worktree remove --workspace <id>`, which tears down both the git worktree and its containing workspace as one unit — not a raw `git worktree remove`. `herdr worktree remove` has its own `--force` flag, but its semantics are undocumented; this repo does not rely on it as a safety mechanism (see **Clean**, below).

**Clean** (a worktree)
: Distinct from the raw `herdr worktree remove` call. Cleaning is this repo's own operation: check whether the worktree's branch is merged into **trunk** *before* calling `herdr worktree remove`. The merge check is local-git-only (no `gh`/API calls) and is intentionally conservative: it uses plain ancestry (`git merge-base --is-ancestor <branch> <trunk>`), which cannot prove a squash- or rebase-merged branch is merged (no direct ancestry link survives those strategies). Rather than trying to detect those cases, the check simply **defaults to "not merged"** whenever ancestry doesn't confirm it — false negatives (an actually-merged branch reported as unmerged) are the expected, accepted behavior, not a bug to fix. On "not merged," the script refuses to proceed automatically and instead prompts the user to continue or abort. There is no `--force` flag; override happens only through that interactive prompt.

**Trunk**
: The repo's default branch (e.g. `main`). Not a distinct concept from "default branch" — same thing, this repo's preferred name for it. Overridable per invocation via a flag (no shared config mechanism yet — see Open Questions in the README).

**Session-required workflow**
: A workflow that only makes sense running inside an active herdr session (i.e. `$HERDR_WORKSPACE_ID`/`$HERDR_ENV` etc. are set — see herdr's injected env vars). Such a workflow must check for this precondition and **error out** if invoked outside a herdr session, rather than trying to bootstrap one. Not every workflow requires this (e.g. `herdr-create-worktree` may be the one that bootstraps a session from nothing) — it's a per-script property, checked explicitly.
