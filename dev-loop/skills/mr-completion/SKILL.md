---
name: mr-completion
description: How to decide whether a delivered dev ticket is really done - only when GitLab confirms its merge request merged into the base branch recorded at delivery; then comment on the ticket once. Suits a scheduled task.
metadata:
  tools: youtrack__search_issues youtrack__get_issue youtrack__add_comment youtrack__close_issue gitlab__get_merge_request worktree__list_worktrees ask_owner
---

# Confirm completion from the merge

A merge request that is open, has a green pipeline or was reviewed is **not** completion. A ticket
counts as done only when GitLab reports its merge request **merged into the branch recorded as the
base when the work branch was cut**. Nothing is inferred locally. PM Bridge never merges: a human
does, and this skill only notices it afterwards.

This skill fits a scheduled task: the owner creates one in **Settings → Scheduled tasks** that sends
a persona a message such as "Run dev-loop:mr-completion for project PROJ" every hour. Each run is
independent and safe to repeat.

## 1. Find the delivered tickets

`youtrack__search_issues` with `tag: pmb-delivered #Unresolved` (add `project: PROJ` when the task
names a project). Tickets QA or the owner already resolved do not appear and need nothing.

## 2. Read what was recorded at delivery

For each ticket, `youtrack__get_issue` and find the delivery comment (written by the
dev-loop:deliver-dev-ticket skill). Take from it:

- `Merge request:` the URL, and `MR iid:` the number;
- `GitLab project:` the project path;
- `Target branch:` the recorded base.

If the work branch's worktree still exists, `worktree__list_worktrees` shows its recorded `base` too.
If the two disagree, do not decide: report it (step 4, "ask a human").

If a ticket already has a comment containing `Merge confirmed by GitLab`, it was handled by an
earlier run: skip it, so the comment is written only once.

## 3. Ask GitLab

`gitlab__get_merge_request(project, iid)`. Then decide exactly like this:

| GitLab says | Verdict |
|---|---|
| the call failed (network, token, 5xx) | **wait** - an outage says nothing; never conclude "not merged" or "merged" from it |
| no merge request record on the ticket | **wait** - list it in your report |
| `merged: false`, `state: closed` | **ask a human** - closed without merging: abandoned or to be reopened? |
| `merged: false`, `draft: true` | **wait** (still a draft) |
| `merged: false`, `state: locked` | **wait** |
| `merged: false`, `state: opened` | **wait** (not merged yet) |
| `merged: true`, no recorded target branch | **ask a human** - cannot check where it went |
| `merged: true`, `targetBranch` missing | **ask a human** |
| `merged: true`, `targetBranch` != recorded base | **ask a human** - merged into another branch is not this delivery |
| `merged: true`, `targetBranch` == recorded base | **complete** |

Compare branch names exactly (trim spaces only).

## 4. Act on the verdict

- **wait:** do nothing to the ticket. The next run checks again.
- **ask a human:** do not touch the ticket. Put it in your report with the reason (one line each).
  Use `ask_owner` only if the owner has to choose something now.
- **complete:** add one comment with `youtrack__add_comment`:

  ```
  Merged into `<target branch>`.
  Merge commit: `<mergeCommitSha>`
  MR: <webUrl>
  Merge confirmed by GitLab, not inferred locally.
  ```

  **State:** close the ticket only when the owner has explicitly said to close it - for this ticket,
  or in the scheduled task's own message (for example "... and close the tickets whose MR merged").
  Then use `youtrack__close_issue` with the same comment as its `comment` (instead of
  `add_comment`): it always picks a state whose `isResolved` flag is true, never a needs-info state
  (close = close). Never use `change_state` to close, and never pick a state by its name.
  Without such an instruction, leave the state as it is (the ticket stays in ready-for-test for QA)
  and end the comment with: `Left in ready-for-test for QA verification.`

## 5. Report

One line per ticket: id, verdict, and for complete what you wrote (comment only, or closed and into
which state). Do not report a ticket as done unless its verdict was complete.
