# worktree plugin

A PM Bridge plugin that gives a dev task its own git worktree: cut a branch from a recorded base,
work there, commit, push. It is a standard Claude Code plugin: one MCP stdio server plus one
skill. **No tool merges, rebases onto, or pushes to the base branch.**

```
.claude-plugin/plugin.json   name, version, description, author, userConfig
.mcp.json                    server "worktree": node ${CLAUDE_PLUGIN_ROOT}/dist/server.js
skills/dev-task/SKILL.md     the skill worktree:dev-task
src/ test/                   the server and its tests
```

## Build and run

```sh
npm install
npm run build   # tsc -> dist/server.js (not committed)
npm test        # builds, then runs unit tests and the stdio end-to-end test (spawned from .mcp.json)
```

The host runs the server from `.mcp.json`. `dist/` is not committed: run `npm install && npm run build` once after checking out.

## Configuration

Read once at startup from the process environment. `.mcp.json` maps each variable to a
`${user_config.KEY}` placeholder that the host (or Claude Code) fills from the owner's settings;
`plugin.json` `userConfig` declares each KEY (title, description, `required`, `sensitive`).

| userConfig key → variable | kind | required | |
|---|---|---|---|
| `repo_path` → `WORKTREE_REPO_PATH` | text | yes | absolute path of the git repository the team works on |
| `worktrees_dir` → `WORKTREE_DIR` | text | no | absolute folder for task worktrees; default `<WORKTREE_REPO_PATH>/.pmb-worktrees` |
| `remote` → `WORKTREE_REMOTE` | text | no | git remote to fetch from and push to; default `origin` |
| `english_only_writes` → `WORKTREE_ENGLISH_ONLY_WRITES` | on/off | no | default **off**. When on, commit messages with Chinese/Japanese/Korean characters are rejected (only `true` turns it on) |

Missing or invalid config does not stop the server: every tool call returns an error that names the
variable (e.g. `not configured: missing WORKTREE_REPO_PATH`) and never echoes a value. A placeholder
left unexpanded (`${user_config.worktrees_dir}`) counts as missing, so an optional variable falls back to its
default.

When `WORKTREE_DIR` is inside the repository, `create_worktree` adds it to the repository's
`.git/info/exclude` so the main checkout stays clean.

## Tools

| tool | read/write | what it does |
|---|---|---|
| `create_worktree` | write | `name`, `baseBranch`: fetch, cut branch `name` from `<WORKTREE_REMOTE>/<baseBranch>` (local `baseBranch` when there is no remote) in `<WORKTREE_DIR>/<name>`, record the base; returns `{ path, branch, base, from }` + workspace proposal |
| `open_worktree` | read | `{ path, branch, base }` of an existing task worktree + workspace proposal |
| `list_worktrees` | read | name, path, branch, base, dirty, ahead/behind the base (as last fetched) |
| `worktree_status` | read | `status --short --branch` + diff stat against the merge base with the base |
| `worktree_diff` | read | diff against the merge base with the base, truncated to `maxBytes` (default 50000) |
| `commit` | write | stage everything in the worktree and record a commit with the user's git identity |
| `push` | write | push the branch to the remote with upstream; returns branch and recorded base |
| `remove_worktree` | write | remove the worktree; optional `deleteBranch`, `discardChanges` |

Read tools set `annotations.readOnlyHint = true`; the host treats everything else as a write.

### The base

`create_worktree` records the base in the repository config as `branch.<name>.pmbBase =
<baseBranch>`. It is shared by all worktrees of the repository and survives restarts. Every tool
returns it as `base`; `push` returns it so the merge request (gitlab `create_merge_request`)
targets exactly the branch the work was cut from. The new branch is created with `--no-track`, so
it never has the base as its upstream.

### The workspace proposal

`create_worktree` and `open_worktree` put the worktree's real absolute path in the tool result's
`_meta`:

```json
{ "pmb/workspace": { "path": "/abs/path/to/repo/.pmb-worktrees/PROJ-123-fix" } }
```

The host validates it (exists, inside the owner's workspace, not protected) and then confines the
task's file and command tools to it. The plugin only proposes.

## Hard rules (enforced in the plugin)

- **Never touch the base.** No merge, rebase or force tool or parameter. `push` pushes an explicit
  `refs/heads/<name>:refs/heads/<name>` refspec without `--force`, so a diverged remote is rejected
  by git. `commit` and `push` refuse when the worktree is not on its own branch, when no base is
  recorded, or when the branch is the recorded base of any task (stacked work).
  `remove_worktree` never deletes such a branch.
- **Safe names.** A worktree name is a flat slug `[A-Za-z0-9][A-Za-z0-9._-]{0,99}`: no slashes,
  `..`, whitespace, leading dash, trailing `.` or `.lock`. Base branch names may contain `/` but
  no git-forbidden characters, `..`, `@{` or leading dashes.
- **Commit messages.** English only when `english_only_writes` is on (off by default; same
  deterministic CJK check as the youtrack plugin). Always: no
  `claude` / `codex` (case-insensitive), no `Co-Authored-By:` / `Generated with` / robot-emoji
  lines. Rejected before anything is staged.
- **No silent data loss.** `remove_worktree` refuses with uncommitted changes unless
  `discardChanges: true`; `deleteBranch` refuses (before removing anything) when the branch has
  commits that are neither pushed nor merged, and deletes with `branch -d`, never `-D`.
- **Only task worktrees.** Tools act on `<WORKTREE_DIR>/<name>` only, and only when it is a linked
  worktree of the configured repository; the main checkout is never adopted.
- **Git safely.** Git runs with `execFile` (no shell) and `-C <path>`, prompts disabled, English
  messages, a 50 s timeout. Errors are `git <command> failed: <git's stderr>` with any
  `user:password@` / `token@` in URLs stripped. Remote URLs are never returned.

## Skill

- `skills/dev-task/SKILL.md` (`worktree:dev-task`) - how a dev task uses the worktree, end to end.
