# gitlab plugin

A PM Bridge plugin that gives personas GitLab projects, branches, merge requests and pipelines.
It is a standard Claude Code plugin: one MCP stdio server plus one skill. **It has no merge tool.**

```
.claude-plugin/plugin.json      name, version, description, author, userConfig
.mcp.json                       server "gitlab": node ${CLAUDE_PLUGIN_ROOT}/dist/server.js, env from ${user_config.KEY}
skills/merge-request/SKILL.md   the skill gitlab:merge-request
src/ test/                      the server and its tests
```

## Build and run

```sh
npm install
npm run build   # tsc -> dist/server.js (not committed)
npm test        # builds, then runs unit tests and the stdio end-to-end test
```

The host runs the server from `.mcp.json`. `dist/` is not committed: run
`npm install && npm run build` once after checking out.

The stdio end-to-end test spawns the built server exactly as `.mcp.json` describes it (placeholders
filled like the host does), checks that the `${user_config.*}` keys `.mcp.json` uses equal `plugin.json`
`userConfig` (official fields only, `token` sensitive), and checks that an unset variable is a config error rather than a crash.

## Configuration

Read once at startup from the process environment. `.mcp.json` maps each variable to a
`${user_config.KEY}` placeholder that the host (or Claude Code) fills from the owner's settings;
`plugin.json` `userConfig` declares each KEY (title, description, `required`, `sensitive`).

| userConfig key → variable | kind | required | |
|---|---|---|---|
| `base_url` → `GITLAB_BASE_URL` | text | yes | e.g. `https://gitlab.example.com` (no `/api/v4`, no credentials) |
| `token` → `GITLAB_TOKEN` | secret | yes | access token with `api` scope, sent as `PRIVATE-TOKEN` |
| `default_project` → `GITLAB_DEFAULT_PROJECT` | text | no | `group/repo` used when a tool call gives no `project` |
| `english_only_writes` → `GITLAB_ENGLISH_ONLY_WRITES` | on/off | no | default **off**. When on, MR titles, descriptions and comments with Chinese/Japanese/Korean characters are rejected (only `true` turns it on) |

Missing or invalid config does not stop the server: every tool call returns an error that names the
missing variables (e.g. `not configured: missing GITLAB_TOKEN`) and never echoes a value. A
placeholder left unexpanded (`${user_config.token}`) counts as missing.
The plugin only talks to the configured `GITLAB_BASE_URL`.

## Tools

| tool | read/write | what it does |
|---|---|---|
| `get_project` | read | id, path, default branch, web URL |
| `list_branches` | read | branches, optional name search |
| `get_branch` | read | one branch: last commit, protected, merged |
| `create_merge_request` | write | open an MR: `sourceBranch`, `targetBranch` (required), `title`, `description` |
| `get_merge_request` | read | state, merged, source/target branch, draft, merge status, head pipeline, web URL |
| `list_merge_requests` | read | MRs of a project, most recently updated first: iid, title, state, author, source/target branch, updated, web URL, draft. Filters: `state` (`opened` default, `merged`, `closed`, `all`), `targetBranch`, `sourceBranch`, `createdByMe` (the token owner, looked up with `GET /user`), `limit` (default 20, max 100). No pipeline column: the list endpoint has none, use `get_merge_request` |
| `list_mr_pipelines` | read | pipelines of an MR |
| `list_pipeline_jobs` | read | jobs of a pipeline with status and failure reason |
| `comment_on_mr` | write | add a note to an MR |

Read tools set `annotations.readOnlyHint = true`; the host treats everything else as a write.

Since 0.2.0 every read tool also returns its result as MCP `structuredContent`: the same object as
the JSON text, scrubbed the same way, so a host can show it as a generic result card (a list such as
`branches`, `pipelines` or `jobs` becomes a table). The text result is unchanged, so models read it
as before. Write tools and errors return text only.

Since 0.3.0 every read tool also declares an MCP `outputSchema` whose only job is to give each field a
readable `title` (`Source branch`, `Updated`, ...), which a host's result card shows as labels and
column headers instead of the raw keys. The schema only names fields: every field is optional and of
any type and every object accepts other keys, because the MCP SDK checks each successful result's
`structuredContent` against it (and a client may too), and a stricter schema could turn a good read
into an error. It is written with `zod/v4` (`.meta({ title })`), the only zod that puts `title` into
the JSON Schema; the input schemas stay on zod 3. Errors are `isError` results, which are not checked.

## Hard rules (enforced in the plugin)

- **Never merged by a persona.** There is no merge / accept / approve tool and no
  parameter for `merge_when_pipeline_succeeds`, auto-merge or squash. `create_merge_request`
  sends exactly `source_branch`, `target_branch`, `title`, `description`; extra arguments are
  dropped by the input schema.
- **Target = recorded base.** `targetBranch` is a required argument: the plugin cannot know the
  base recorded when the branch was cut, so it never guesses one or falls back to the default
  branch. Source and target must differ.
- **English only (optional, off by default).** With `english_only_writes` on, MR titles, descriptions
  and MR comments are rejected if they contain Chinese, Japanese or Korean characters; the error tells
  the persona to rewrite in English. The check is deterministic, runs before any request, and never
  calls a model. With it off, text is written in whatever language it is in.
- **No credentials in output.** Errors are `GitLab HTTP <status>: <server message>` (or a
  network error code). Request URLs and headers never appear, and the token is scrubbed from
  every error and result.

## Skill

- `skills/merge-request/SKILL.md` (`gitlab:merge-request`) - how to open an MR correctly.
