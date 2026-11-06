# creator plugin

A PM Bridge plugin that lets a persona (or the owner, through a persona) make new plugins and
skills in the standard format by conversation, so a new connector does not start from hand-written
boilerplate. It is a standard Claude Code plugin: one MCP stdio server plus one skill.

```
.claude-plugin/plugin.json   name, version, description, author, userConfig
.mcp.json                    server "creator": node ${CLAUDE_PLUGIN_ROOT}/dist/server.js, env from ${user_config.KEY}
skills/create-plugin/        the skill creator:create-plugin (SKILL.md, LICENSE.txt for the adapted parts)
src/ test/                   the server and its tests (test/fixtures: one plugin folder per validation rule)
```

## Build and run

```sh
npm install
npm run build   # tsc -> dist/server.js (not committed)
npm test        # builds, then runs unit tests and the stdio end-to-end test
```

## Configuration

| userConfig key → variable | kind | required | |
|---|---|---|---|
| `plugins_root` → `CREATOR_PLUGINS_ROOT` | text (directory) | no (per machine, no default) | the only folder the creator writes in, normally the pm-bridge-plugins repository; new plugins are its subfolders |

Read once at startup. The creator ships with the app enabled, so its skill works in chat
before anything is set; while the folder is unset, every tool call returns an error asking the owner
to set "Plugins folder" in Settings > Plugins > creator. A relative or non-existent folder (or an
unexpanded placeholder) is an error too; errors name `CREATOR_PLUGINS_ROOT` and never echo the value.

## Tools

| tool | read/write | what it does |
|---|---|---|
| `scaffold_plugin` | write | create `<root>/<name>/`: `.claude-plugin/plugin.json` (name, version, description, author, optional userConfig), optional `.mcp.json` (a TypeScript stdio server skeleton, or an existing command such as `npx <package>`; `env` maps variables to `${user_config.KEY}`), optional first skill, README; returns the created files and a validation |
| `scaffold_skill` | write | add `skills/<name>/SKILL.md` (Agent Skills front matter: name, description, `metadata.tools`) to a plugin folder under the root |
| `validate_plugin` | read | check a plugin folder under the root against the official rules and the host's rules; returns `ok` and each finding (level, source, file, message) |

Write tools are annotated `readOnlyHint: false` (and `destructiveHint: false`: they only create
files), so the host treats them as writes: not granted by default, and when granted, a
confirmation card by default.

## Rules

- **Writes stay in the root.** Plugin and skill names are validated (plugin: lowercase kebab-case,
  2-32 characters; skill: Agent Skills name), file paths are built from them, every folder on the
  way must be a real folder (symbolic links are refused, never followed), and the real path must stay
  under the root's real path.
- **Never overwrite.** `scaffold_plugin` refuses an existing folder; every file is created with
  O_EXCL, and all targets are checked before the first write, so a refusal leaves nothing behind.
- **No secrets in files.** Secrets are only `sensitive` userConfig items. `env` values name a
  userConfig key (or `CLAUDE_PLUGIN_DATA` / `CLAUDE_PLUGIN_ROOT`), never a literal; a command line
  with a literal credential (`--api-key abc...`, `TOKEN=...`) or an undeclared `${user_config.KEY}`
  is refused. Reserved names (`claude-*`, `anthropic-*`, ...) are refused.
- **Generated TypeScript servers** use this repository's setup exactly (package.json scripts and
  dependency ranges, tsconfig, `src/server.ts` + `src/tools.ts`, a vitest test over an in-memory
  MCP connection) with one example read tool `about` that reports which settings are filled in,
  never their values.

## What validate_plugin checks

Official (Claude Code [plugin manifest reference](https://code.claude.com/docs/en/plugins-reference),
[Agent Skills](https://agentskills.io/specification)): manifest JSON, `name` required, reserved and
non-kebab names, missing version / description / author, unknown top-level fields (stripped by
Claude Code), strict userConfig items (unknown field, type, title, description), `${user_config.KEY}`
used but not declared, paths that do not start with `./`, leave the folder or do not exist, http://
to a non-loopback host, literal credentials in env / headers, a root `CLAUDE.md`.

Host (mirrors `packages/core/src/plugin-manifest.ts` and `skills.ts` in the PM Bridge app, copied, not
imported): the name `[a-z][a-z0-9-]{1,31}`, the old `pmb-plugin.json` format, server entries as
`parseServer` reads them, `metadata.pmb.variables`, `metadata.pmb.timeouts`, `metadata.pmb.signIn` and `metadata.pmb.i18n`, `displayName`, a top-level `pmb`
(not read), declared-but-unused userConfig items, plain `${NAME}` variables, skill front matter (name
rules, name = folder, description 1-1024), and the components the host lists as unsupported.

An `error` means Claude Code or the host refuses the plugin (or skips that skill); a `warning` means
it loads but something is ignored or looks wrong. `test/fixtures/expected.json` records the host's
own outcome for each fixture folder; the tests check the validator agrees.

## Credits

The writing guidance in `skills/create-plugin/SKILL.md` (what to ask first, how to write a
description and a body) is adapted from the `skill-creator` skill in
[anthropics/skills](https://github.com/anthropics/skills/tree/main/skills/skill-creator),
Apache License 2.0; the licence text is in `skills/create-plugin/LICENSE.txt`. Everything else is
our own.
