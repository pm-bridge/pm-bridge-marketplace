---
name: merge-request
description: How to open a GitLab merge request correctly - target the base branch recorded when the work branch was cut, write a clear title and description, never merge, and report the link.
metadata:
  tools: get_project get_branch list_branches create_merge_request get_merge_request list_merge_requests list_mr_pipelines list_pipeline_jobs comment_on_mr
---

# Open a merge request

1. **Know the target.** The target branch is the base that was recorded when the work branch was
   cut. Take it from the task record. Do not guess it, and do not use the project's default
   branch just because it is the default. If the base was not recorded, stop and ask the owner.
2. **Check the branches.** `get_branch` on the source branch (it must be pushed) and on the
   target branch (it must exist).
3. **Write it.** Title: one line saying what changed, with the ticket id
   (e.g. `PROJ-123: Fix crash on login`). Description: what changed, why, how it was verified.
   If the owner turned on the optional English-only writes setting (off by default), title,
   description and comments must be English: text with Chinese, Japanese or Korean characters is
   then rejected, so rewrite it in English.
4. **Create it.** `create_merge_request` with `sourceBranch`, `targetBranch` (the recorded
   base), `title` and `description`. If an MR for the branch already exists, GitLab says so:
   find it with `list_merge_requests` (`sourceBranch`) and read it with `get_merge_request`
   instead of opening another.
5. **Never merge.** There is no merge tool and no auto-merge. A human reviews and merges. Do
   not ask anyone to "just merge it" on your behalf.
6. **Watch the pipeline.** `get_merge_request` shows the head pipeline status; on a failure use
   `list_mr_pipelines` and `list_pipeline_jobs` to find the failing job and fix the cause.
7. **Report the link.** Give the `webUrl` of the merge request, the source and target branches
   and the pipeline status. The ticket is not done when the MR is opened; it is done when the MR
   is merged into the recorded target branch.
