---
name: create-plugin
description: How to make a new plugin or skill for the team by conversation - look for an existing MCP server or plugin first, ask the owner for the name and how secrets are handled, scaffold it in the standard format, validate it, and tell the owner how to add it.
license: Parts adapted from the skill-creator skill of github.com/anthropics/skills (Apache-2.0, Copyright 2026 Anthropic, PBC), see LICENSE.txt
metadata:
  tools: scaffold_plugin scaffold_skill validate_plugin ask_owner
---

# Create a plugin or a skill

A plugin is how a new connector, platform tool or skill reaches the team. It is a standard
Claude Code plugin: `.claude-plugin/plugin.json`, an optional `.mcp.json` with one MCP server, and
skills in `skills/<name>/SKILL.md` (Agent Skills). You only create files; the owner decides whether
the plugin is added, enabled and granted to anyone.

All three tools work inside one folder the owner sets ("Plugins folder" in Settings > Plugins >
creator, normally the pm-bridge-plugins repository). If a tool answers that no plugins folder is set,
tell the owner exactly that and where to set it; do not guess a folder.

## 1. Find out what is wanted

Before writing anything, settle these with the owner (`ask_owner`), using what the conversation
already shows first:

1. What should personas be able to do with it? Which system does it talk to?
2. When should a persona use it? (This becomes the skill's description.)
3. What does it need from the owner: a server URL, an account, a token, a folder?
4. What should the owner see on the plugin card? Always give `scaffold_plugin` a plain
   `displayName` (what it does in a few words, e.g. "YouTrack tickets", never just the technical
   name), a one-sentence plain `description`, and `zh` with the Chinese of both.

## 2. Reuse before you scaffold

Writing a server is the last resort. In this order:

1. **An existing plugin** in the plugins folder already covers it: use it, or add a skill to it
   with `scaffold_skill`. `validate_plugin` on a folder shows what it is and whether it loads.
2. **An existing MCP server** (the vendor's own, or a well-known open one): the owner can add it
   directly in Settings > Plugins > Add MCP server, or paste a Claude Desktop / Cursor / VS Code /
   Codex config into Settings > Plugins > Import. Suggest that first. If a skill should come with
   it, scaffold a plugin with `server: { kind: "command", command, args, env }` around that server
   plus the skill.
3. **Only knowledge, no new tools**: a skills-only plugin (`scaffold_plugin` with a `skill` and no
   `server`), or `scaffold_skill` into the owner's own skills plugin.
4. **Nothing exists**: `scaffold_plugin` with `server: { kind: "typescript" }`. It creates the same
   setup as the team's own plugins with one example read tool; a developer then writes the real
   tools. Do not promise working tools from the skeleton alone.

Never download or run a server to try it out; the owner enables plugins.

## 3. Ask about names and secrets

- **Name**: lowercase kebab-case, 2-32 characters (`tasks`, `outlook-mail`). It prefixes every tool
  (`tasks__search_tasks`) and skill (`tasks:triage`), so propose one and let the owner confirm it.
  Names starting with `claude-` or `anthropic-` are refused.
- **Settings** become `userConfig` items with snake_case keys (`base_url`, `token`), each with a
  title and a description. Ask which are required.
- **Secrets** (tokens, passwords, account books) are only ever `sensitive: true` userConfig items.
  The owner picks a stored key for them in Settings. Never put a secret value in any argument, file,
  README or message, and never ask the owner to tell you one; the tools refuse literal credentials.
  Map each environment variable the server reads to its key: `env: { "TASKS_TOKEN": "token" }`.

## 4. Write the skill

A skill is how a persona learns when and how to use the tools. Good skills:

- **Description** (one line, at most 1024 characters): say both what the skill does and when to use
  it, with the words a request would use. Models tend to under-use skills, so be a little insistent
  about when it applies.
- **Body**: the steps in order, in the imperative ("Call `get_task` and read every comment"), each
  naming the tool and what to check in its result. Explain why a rule matters instead of only
  shouting MUST; a model that knows the reason handles the case you did not list.
- Keep it short (well under 500 lines). The host gives personas only the SKILL.md body, so put
  everything a persona needs there.
- List every tool the skill uses in `tools` (they go to `metadata.tools`): the plugin's own tools by
  their plain names, core tools such as `ask_owner` or `read_file`, other plugins' tools as
  `<plugin>__<tool>`.
- A skill must not surprise the owner: no hidden actions, nothing that works around a permission.

## 5. Scaffold, validate, hand over

1. Call `scaffold_plugin` (or `scaffold_skill`). Both refuse to overwrite anything; if a folder or
   skill already exists, ask the owner instead of picking a new name on your own.
2. Read the `validation` in the result, or call `validate_plugin`. Fix every `error` before
   handing over (errors mean Claude Code or this host will not load it); mention the warnings.
3. Tell the owner, in plain words:
   - which folder was created and which files are in it;
   - for a TypeScript server: `npm install && npm run build && npm test` in that folder first;
   - to add it in **Settings > Plugins** (Add folder), fill in the settings (choosing a stored key
     for each secret), enable it, and grant its tools to the personas who need them.
