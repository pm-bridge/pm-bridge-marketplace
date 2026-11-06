# dev-loop plugin

A PM Bridge plugin with **skills only** - no MCP server, no process, no configuration. It is a
standard Claude Code plugin with no `.mcp.json`:

```
.claude-plugin/plugin.json             name, version, description, author
skills/deliver-dev-ticket/SKILL.md     dev-loop:deliver-dev-ticket
skills/mr-completion/SKILL.md          dev-loop:mr-completion
skills/dedup-by-tag/SKILL.md           dev-loop:dedup-by-tag
test/skills.test.mjs                   the drift test
```

Its skills
tell a persona how to run the dev ticket loop with the tools of other plugins. It replaces a
delivery pipeline that was originally hard-coded in the app.

## Skills

| skill | what it is for |
|---|---|
| `dev-loop:deliver-dev-ticket` | From a triaged dev ticket to a delivered merge request: read the ticket and its screenshots, pick the base branch, work in a worktree, verify, commit, push, open the MR against the recorded base, comment on the ticket, move the ticket to ready-for-test, tag it, log time. |
| `dev-loop:mr-completion` | Whether a delivered ticket is really done: only when GitLab confirms its MR merged into the recorded base. Then one comment; the ticket is closed only on the owner's explicit instruction. Meant for a scheduled task (Settings -> Scheduled tasks). |
| `dev-loop:dedup-by-tag` | The `pmb-delivered` tag is the only "already delivered" marker: check it before starting, add it after the MR exists, never use a state for this. |

## What it needs

The persona needs these plugins enabled and their tools granted (write tools are off by default):

- **youtrack** - `get_issue`, `list_attachments`, `get_attachment`, `search_issues`,
  `list_project_states`, `add_comment`, `change_state`, `close_issue`, `add_tag`, `add_work_item`
- **gitlab** - `get_project`, `list_branches`, `get_branch`, `create_merge_request`,
  `get_merge_request`, `list_mr_pipelines`, `list_pipeline_jobs`
- **worktree** - `create_worktree`, `open_worktree`, `list_worktrees`, `worktree_status`,
  `worktree_diff`, `commit`, `push`
- core tools - `read_file`, `write_file`, `search_files`, `list_dir`, `run_command`, `recall`,
  `remember`, `ask_owner`, `ask_colleague`

Each skill's front matter `metadata.tools` (a space-separated string, since Agent Skills
`metadata` values are strings) lists exactly what it uses, as host names (`<plugin>__<tool>`).
This is documentation; permissions still come from the grants table.

Device or UI verification (screenshots) is not part of these skills: `deliver-dev-ticket` hands it to
a tester persona, who uses their own skill and returns evidence from their own verification run.

## Rules these skills encode

- Optional English-only writes: if the owner turned on `english_only_writes` ("English-only writes")
  in the youtrack, gitlab and worktree plugins, everything written there is English and the tools
  reject Chinese, Japanese or Korean text. It is off by default.
- MR target = the base recorded when the branch was cut; nothing ever merges.
- Delivery moves a ticket to ready-for-test, picked from the project's unresolved states; if none
  fits, the state is left alone - never closed as a fallback.
- Completion = GitLab says merged into the recorded base. Not open, not green, not reviewed.
- close = close, and only on the owner's explicit word (`close_issue`, always a resolved state).
- De-duplication by the `pmb-delivered` tag, never by state.
- Verification evidence comes from the verification run, never from ticket attachments.
- needs-info only for information the reporter really did not give.

## Tests

```sh
npm test    # node --test, no dependencies, no install needed
```

`test/skills.test.mjs` checks the manifest (`.claude-plugin/plugin.json`, valid name, author, no MCP server,
no `.mcp.json`, no `userConfig` or `pmb` extras), each `skills/<name>/SKILL.md` against the Agent Skills rules (name = folder name,
description length, only known front matter keys), that every tool in `metadata.tools` and every
`<plugin>__<tool>` named in a skill body is registered in that plugin's `src/tools.ts` in this repo,
that referenced skills (`<plugin>:<skill>`, read from each plugin's `skills/*/SKILL.md`) exist,
and that every plugin's skills in this repo use only the Agent Skills front matter keys with their
tools in `metadata.tools`.
Renaming or removing a tool in another plugin fails this test.
