---
name: dev-task
description: How a dev task uses a git worktree - cut a branch from the base named in the ticket, work only in that worktree, commit, push, then open the merge request against the recorded base. Never merge.
metadata:
  tools: create_worktree open_worktree list_worktrees worktree_status worktree_diff commit push remove_worktree
---

# Do a dev task in a worktree

1. **Know the base.** The base branch is the one named in the ticket or in the announcement that
   assigned the task (for example `release/2.3`). Do not guess it and do not assume `main`. If no
   base is named, stop and ask the owner which branch to cut from.
2. **Pick a name.** One worktree per task, named after the ticket: letters, digits, `.`, `_`, `-`,
   no slashes, e.g. `PROJ-123-fix-login`. The name is also the branch name.
3. **Create or resume.** `create_worktree(name, baseBranch)`. It fetches the remote, cuts the branch
   from the remote base and records that base. If it says the worktree already exists, the task was
   started before: `open_worktree(name)` instead. Both return the path, branch and base, and
   propose the worktree as this task's workspace.
4. **Work only there.** Read, edit and run commands only inside the returned path. Never edit the
   main checkout or another task's worktree.
5. **Check your change.** `worktree_status` and `worktree_diff` show what changed against the base.
   Build and test before committing.
6. **Commit.** `commit(name, message)` stages everything in the worktree. The message says what
   changed and why, and starts with the ticket id (e.g. `PROJ-123: Fix crash on login`). If the
   owner turned on the optional English-only writes setting (off by default), it must be English:
   a message with Chinese, Japanese or Korean characters is then rejected. It must not mention claude or codex and must not contain
   Co-Authored-By or "Generated with" lines; such messages are rejected.
7. **Push.** `push(name)` pushes the branch (never force, never the base) and returns the branch and
   the recorded base.
8. **Open the merge request.** With the gitlab plugin's `create_merge_request`: `sourceBranch` = the
   branch, `targetBranch` = the recorded base returned by `push`. Never a different target. Never
   merge it, and never ask anyone to merge it on your behalf: a human reviews and merges.
9. **Clean up only when told.** `remove_worktree` refuses while there are uncommitted changes. Do not
   pass `discardChanges` unless the owner asked to throw the work away.
