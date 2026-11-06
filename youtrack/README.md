# youtrack plugin

A PM Bridge plugin that gives personas YouTrack issues: search, read, comment, change state,
close, assign, create, log time, tag, read attachments (screenshots as images) and read
knowledge base articles. It is a standard Claude Code plugin: one MCP stdio server plus one
skill.

```
.claude-plugin/plugin.json   name, version, description, author, userConfig
.mcp.json                    server "youtrack": node ${CLAUDE_PLUGIN_ROOT}/dist/server.js, env from ${user_config.KEY}
skills/triage/SKILL.md       the skill youtrack:triage
src/ test/                   the server and its tests
```

## Build and run

```sh
npm install
npm run build   # tsc -> dist/server.js (not committed)
npm test        # builds, then runs unit tests and the stdio end-to-end test
```

The host runs the server from `.mcp.json`. `dist/` is not committed: run
`npm install && npm run build` once after checking out.

## Issue card (MCP Apps)

In a host that renders MCP Apps cards (it offers the `io.modelcontextprotocol/ui` extension with
`text/html;profile=mcp-app` in its initialize), `get_issue` is listed with
`_meta.ui.resourceUri = "ui://youtrack/issue-card.html"` and its result carries `structuredContent`
for the card: `id, url, summary, project, state, stateField, resolved, assignee, assigneeLogin,
priority, updated` (`url` is `<base_url>/issue/<id>`; `priority` is `null` without a Priority field;
no description, comments or attachments). The text result is unchanged, so models and hosts without
cards see the full issue as before; a client without the extension sees no `_meta.ui` at all.

The card shows the id (a link), summary, state (green when resolved), assignee, priority and when it
was updated, with "Open in browser" and "Refresh" (`get_issue` again through the host). Its words follow
the host's locale (English or Chinese). `ui/issue-card.ts` + `ui/issue-card.html` are built by
`scripts/build-card.mjs` (part of `npm run build`) into one HTML file with everything inline (it fetches
nothing), embedded in the server as `src/generated/issue-card-html.ts` (not committed).

Since 0.3.0 the card also has "Change state", "Assign" and "Comment". The menus come from
`card_issue_options` (listed only to a client that renders cards, with `_meta.ui.visibility = ["app"]`,
so a host never offers it to a model): the issue's project states (`name`, `isResolved`, `archived`)
and the users its assignee field can take (`login`, `fullName`; banned users left out, at most 500),
read through the issue like `list_project_states` reads states, so a regular token is enough:
`customFields(projectCustomField(bundle(values(name,isResolved,archived),aggregatedUsers(login,fullName,banned))))`
(YouTrack REST: `UserProjectCustomField.bundle` is a `UserBundle`, whose `aggregatedUsers` are all its
users, individuals and group members; https://www.jetbrains.com/help/youtrack/devportal/api-entity-UserBundle.html).
A picked state whose `isResolved` is true goes through `close_issue` (with it as `preferredState`),
any other through `change_state` (`ui/actions.ts`); assigning calls `assign_issue`, a comment
`add_comment`. When English-only writes is on, a comment with Chinese/Japanese/Korean characters is
not sent (the card says so; `add_comment` refuses it too); when it is off (the default) any language is sent. Every change is a `tools/call` through the host,
which should ask the user first; afterwards the card reads the issue again.

## Result data (structuredContent)

Since 0.4.0 the read tools without a card of their own (`search_issues`, `list_project_states`,
`list_attachments`, `search_articles`, `get_article`) also return their result as MCP
`structuredContent`: the same object as the JSON text, scrubbed the same way (no token, no signed
attachment URLs), so a host can show it as a generic result card (a list such as `issues` becomes a
table). The text result is unchanged, so models read it as before. `get_issue` and
`card_issue_options` keep their card data; write tools and errors return text only.

Since 0.5.0 those five tools also declare an MCP `outputSchema` whose only job is to give each field
a readable `title` (`Issue`, `Assignee`, `Updated`, ...), which a host's result card shows as labels
and column headers instead of the raw keys. The schema only names fields: every field is optional and
of any type and every object accepts other keys, because the MCP SDK checks each successful result's
`structuredContent` against it (and a client may too), and a stricter schema could turn a good read
into an error. It is written with `zod/v4` (`.meta({ title })`), the only zod that puts `title` into
the JSON Schema; the input schemas stay on zod 3. `get_issue` and `card_issue_options` declare none.

## Configuration

Read once at startup from the process environment. `.mcp.json` maps each variable to a
`${user_config.KEY}` placeholder that the host (or Claude Code) fills from the owner's settings;
`plugin.json` `userConfig` declares each KEY (title, description, `required`, `sensitive`).

| userConfig key → variable | kind | required | |
|---|---|---|---|
| `base_url` → `YOUTRACK_BASE_URL` | text | yes | e.g. `https://youtrack.example.com` (no credentials in the URL) |
| `token` → `YOUTRACK_TOKEN` | secret | yes | YouTrack permanent token, sent as `Authorization: Bearer` |
| `english_only_writes` → `YOUTRACK_ENGLISH_ONLY_WRITES` | on/off | no | default **off**. When on, comments, issue summaries and descriptions, work item descriptions and tags with Chinese/Japanese/Korean characters are rejected (only `true` turns it on) |

Missing or invalid config does not stop the server: every tool call returns an error that names the
missing variables (e.g. `not configured: missing YOUTRACK_TOKEN`) and never echoes a value. A
placeholder left unexpanded (`${user_config.token}`) counts as missing.
The plugin only talks to the configured `baseUrl`.

## Tools

| tool | read/write | what it does |
|---|---|---|
| `search_issues` | read | YouTrack query -> id, summary, project, state, resolved, assignee |
| `get_issue` | read | full issue: description, comments, custom fields, tags, state, resolved; an issue card in hosts that render MCP Apps |
| `list_project_states` | read | the project's state bundle with each state's `isResolved` flag |
| `card_issue_options` | read | for the issue card only (hosts that render MCP Apps): the project's states and the assignable users |
| `add_comment` | write | add a comment |
| `change_state` | write | move to a state of the project bundle (validated against the bundle) |
| `close_issue` | write | move to a state whose `isResolved` is true; optional comment |
| `assign_issue` | write | set the assignee field by login |
| `list_attachments` | read | an issue's attachments: id, name, mimeType, size, author, readable |
| `get_attachment` | read | one attachment: PNG/JPEG/GIF/WebP as MCP image content, text files as text; others refused |
| `create_issue` | write | create an issue in a project (optional Type); assigned to the token owner unless `assignToMe: false` |
| `add_work_item` | write | log minutes of spent time, optional description and date |
| `add_tag` | write | add a tag (the only de-dup marker) |
| `search_articles` | read | knowledge base articles matching a query, optionally in one project |
| `get_article` | read | one article's full content as text, plus attachment names |

Endpoints (the same ones the old app used): `GET issues/{id}/attachments`,
`GET issues/{id}/attachments/{attachmentId}` then the returned file `url`; `GET admin/projects`
(short name to id) and `POST issues`, then the `Assignee me` command; `POST
issues/{id}/timeTracking/workItems`; the `tag <name>` command; `GET articles`, `GET articles/{id}`.
There is no remove-tag tool: the old app never removed tags. Zip / web-bundle previews are not
supported.

Read tools set `annotations.readOnlyHint = true`; the host treats everything else as a write.

## Hard rules (enforced in the plugin)

- **English only (optional, off by default).** With `english_only_writes` on, comments (`add_comment`,
  `close_issue`), issue summaries and descriptions (`create_issue`), work item descriptions (`add_work_item`)
  and tags (`add_tag`) are rejected if they contain Chinese, Japanese or Korean characters; the error tells
  the persona to rewrite in English, and the write tools' descriptions ask for English. The check is
  deterministic (a character-range test), runs before any request, and never calls a model. With it off,
  text is written in whatever language it is in.
- **close = close.** `close_issue` picks a state whose `isResolved` flag is true from the issue's
  own project bundle. Archived states, negative dispositions (Rejected, Won't fix, Duplicate, ...)
  and needs-info-like names are excluded; names that read as a positive completion (Fixed, Done,
  Closed, ...) are only a tie-breaker. If the workflow refuses one state, the next is tried; the
  result is confirmed by re-reading `issue.resolved`. If the project has no usable resolved state
  the issue is left unchanged and an error is returned - it never falls back to another state.
- **States are admin-configurable.** Resolution is decided by `isResolved` / `issue.resolved`,
  never by a hardcoded state name; the state field is found by type (`StateIssueCustomField`) or
  by the name State/Status.
- **De-duplication never uses state.** Tags (`add_tag`, e.g. `pmb-delivered`) are the only marker
  for "already handled"; a state is never used as one (states are admin-configurable and close = close).
- **Created issues go to the token owner.** `create_issue` assigns the new issue with YouTrack's
  `me` (the old app's rule: app-created issues are assigned to the current user). If that fails the
  issue is still reported as created, with `assigned: false`, so a retry never makes a duplicate.
- **Attachments are size-capped and URL-free.** At most 5 MB (`maxBytes` can only lower it): a
  larger image is refused, a text file is cut and marked `truncated`. The file is fetched only from
  the configured server's origin; YouTrack's signed download URL is never returned. Binary files
  that are neither images nor text (zip, video, pdf, ...) are refused with their type and size.
- **No credentials in output.** Errors are `YouTrack HTTP <status>: <server message>` (or a network
  error code). Request URLs and headers never appear, and the token is scrubbed from every error
  and result.
- **No command injection.** State values are only ever taken from the project bundle, logins and
  tags must be a single token, and an issue type is sent as a field value (never as command text),
  so an argument cannot append a second YouTrack command.

## Skill

- `skills/triage/SKILL.md` (`youtrack:triage`) - how to read and triage a ticket with these tools.
