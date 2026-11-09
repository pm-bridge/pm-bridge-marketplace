# PM Bridge plugin marketplace

English | [中文](README.zh-CN.md)

The official PM Bridge plugins, each released and updated on its own. Only the built plugins are here, not their sources.

## How to install

PM Bridge has this marketplace already: open **Settings → Integrations → Marketplace**, find the plugin you want and press **Install**; once you confirm, enable it under **Tools & skills**.

Any app that supports Claude Code plugin marketplaces can also add `https://github.com/pm-bridge/pm-bridge-marketplace`.

## Plugins

| Plugin | Name | Version | What it does |
|---|---|---|---|
| Plugin and skill creator | `creator` | 0.1.0 | Creates new plugins and skills in the standard format by chat, and checks them. |
| Dev ticket workflow (skills) | `dev-loop` | 0.1.0 | Step-by-step instructions for taking a dev ticket to a merge request and closing it only after the merge; it runs no program. |
| Feishu (Lark) IM bridge | `feishu` | 0.2.0 | Brings Feishu groups and chats into PM Bridge groups: their messages come in, and you can send to them from the app, images and files too. |
| GitLab merge requests | `gitlab` | 0.4.0 | Work with GitLab projects, branches, merge requests and pipelines; it can never merge. |
| Knowledge base search | `knowledge` | 0.1.0 | Searches YouTrack articles and your local folders together as one knowledge base. |
| Mac mail (read-only) | `mac-mail` | 0.1.0 | Reads the mail already on this Mac in Apple Mail (and in Legacy Outlook) and can never send, move, delete or mark anything. |
| Phone and emulator testing (Maestro) | `maestro` | 0.1.0 | Tests the Android app on a phone or emulator with Maestro and reports the result with screenshots from that run. |
| Outlook mail (read-only) | `outlook` | 0.1.0 | Reads your Outlook mail and cannot send, move or delete anything; you connect it with one button and a code in your browser. |
| One-off jobs (skills) | `quick-tools` | 0.1.0 | How a persona does a one-off job you need now - sign or edit a PDF, convert documents and images, pull out a table - with a throwaway script on this Mac; it runs no program itself. |
| Test accounts and login codes | `test-accounts` | 0.1.0 | Picks a test account for logging into the app under test and reads its emailed or authenticator code; it never shows a password. |
| Code workspace per task (git worktree) | `worktree` | 0.2.0 | Gives each dev task its own copy of the code on its own branch, then commits and pushes; it never merges. |
| YouTrack tickets | `youtrack` | 0.6.0 | Find, read, comment on, update, assign and create YouTrack tickets, and read knowledge base articles. |
