---
name: dedup-by-tag
description: How to tell whether a YouTrack ticket was already delivered - check for the pmb-delivered tag before starting work, add it after delivery, and never use a state as the marker.
metadata:
  tools: youtrack__search_issues youtrack__get_issue youtrack__add_tag worktree__list_worktrees worktree__open_worktree ask_owner
---

# De-duplicate dev work by tag

The marker for "this ticket has already been delivered (code pushed and a merge request opened)"
is the YouTrack tag **`pmb-delivered`**. It lives on the ticket itself, so it survives restarts,
other machines and other personas. It is the only durable marker.

## Never use a state as the marker

States are configured by an administrator and change for other reasons: QA moves a ticket to
ready-for-test or back, the owner closes it (close = close). A state that doubles as "already
done" either re-closes tickets or hides work. So:

- Do not decide "already delivered" from a state name.
- Do not move a ticket to a state just to mark it as handled.
- Only a tag (or a custom field, if the owner sets one up) is a de-duplication marker.

## Before starting work on a ticket

1. `youtrack__get_issue` and read `tags`.
2. **Has `pmb-delivered`:** it was already delivered. Do not start a new worktree, branch or merge
   request for it. Tell whoever asked that it was delivered, and give the merge request link from the
   delivery comment on the ticket (the line `Merge request: <url>`).
   - Exception: the owner explicitly asks for more work on it (for example QA sent it back). Then
     resume the existing work instead of starting over: find the worktree with
     `worktree__list_worktrees` (it is named after the ticket) and `worktree__open_worktree`. Push to
     the same branch, so the existing merge request picks up the change. Open a new merge request
     only if the old one is closed or merged. There is no tool to remove a tag; leave it on.
3. **No tag, but the ticket is resolved (`resolved: true`) or already in the project's
   ready-for-test state:** it is mid-workflow or finished, not new work. Do not start; say so and ask
   the owner if they want it reworked. (This is a read-only check, not a marker.)
4. **No tag, and a worktree for this ticket already exists** (`worktree__list_worktrees`): someone
   started it and did not finish. Resume it with `worktree__open_worktree`; do not create a second one.
5. Otherwise it is new work.

To list delivered tickets, search with `youtrack__search_issues`, e.g.
`project: PROJ tag: pmb-delivered #Unresolved`.

## After delivery

Add the tag once the merge request exists (not before, not when only a commit or a push happened):
`youtrack__add_tag(issueId, "pmb-delivered")`. Adding it again is harmless.

If `add_tag` fails or is refused, try once more. If it still fails, say so plainly in your report to
the owner ("the pmb-delivered tag is missing on PROJ-123"): without the tag the ticket can be picked
up and coded a second time.
