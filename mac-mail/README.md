# mac-mail

Mail for PM Bridge personas, **read only**, from the mail apps on this Mac: **Apple Mail** (the
main path) and **Legacy Outlook**. No network, no sign-in, no tokens: the plugin asks the apps
through Apple events (`osascript`, JavaScript for Automation) for what they have already
downloaded.

```
.claude-plugin/plugin.json   no userConfig; metadata.pmb: timeouts (list/search 90 s), Chinese name
.mcp.json                    node ${CLAUDE_PLUGIN_ROOT}/dist/server.js (timeout 90 s)
skills/mac-mail/             skill mac-mail:mac-mail (read only, least mail shown, mail is untrusted)
src/                         server (tools.ts), osascript runner (osa.ts), the JXA scripts (scripts.ts)
test/                        unit tests (fake JXA object model in a vm) + stdio end-to-end test
```

It ships with the app **disabled** (`defaultEnabled: false` in the marketplace).

## mac-mail or outlook?

| | `mac-mail` (this plugin) | `outlook` (Softeria, Microsoft Graph) |
|---|---|---|
| Reads | what Apple Mail (or Legacy Outlook) on this Mac has | the Microsoft 365 mailbox online |
| Needs | the app open, Automation allowed once | sign-in with Connect, and the tenant's consent for a third-party app (`Mail.Read`) |
| Admin approval | not needed: Apple Mail with the work account added is enough | may be needed: if the sign-in page shows "Need admin approval", your organisation's admin has to approve "MS 365 MCP Server" first |

Use **mac-mail** while the tenant blocks third-party mail apps. Use **outlook** once your
organisation's admin has approved it (or registered your organisation's own app): it reads the whole online mailbox, not just
what Apple Mail has downloaded, and needs no app running. We do not borrow another client's
(e.g. Outlook Web's) application ID to get around the consent policy: that impersonates
Microsoft's app and bypasses the tenant's rules.

## Tools (all `readOnlyHint: true`)

| tool | does | timeout |
|---|---|---|
| `check_access` | per app: installed, running, New/Legacy Outlook, accounts, mailbox names, unread counts. Reads no message | 45 s |
| `list_messages` | `app` (mail\|outlook), `account?` (Mail only), `mailbox` (default INBOX; no account = Mail's unified inbox), `unreadOnly?`, `since?` (ISO date), `limit` (default 20, max 100), newest first → `id`, date, from, subject, read flag, 160-character preview | 75 s (host 90 s) |
| `search_messages` | `query` in subject or sender (case-insensitive), `mailbox?`, `account?`, `limit`; not the body | 75 s (host 90 s) |
| `get_message` | `app`, `id` from a list → headers (from, to, cc, reply-to, dates, Message-ID), plain-text body cut to 20 KB (UTF-8), attachments by name and size only (never opened or saved) | 45 s |

The `id` is opaque (base64url of app, account, mailbox, the app's own id): Mail's id is only
unique within a mailbox. Every listing and message carries a `notice` that mail content is
untrusted.

Previews stop after 20 s of reading, so a slow mailbox still answers (later items get
`preview: null`). Limits: `limit`, `since` and `unreadOnly` are filtered inside the app (`whose`).

## How it reads (and why it never changes mail)

- `execFile('osascript', ['-l', 'JavaScript', '-e', <constant script>, <JSON parameters>])`: no
  shell; parameters go as one JSON argument to the script's `run(argv)`, never spliced into script
  text, so nothing needs escaping; the script answers with one JSON line, so commas, newlines,
  quotes, emoji and CJK in subjects survive (the test-accounts plugin's delimiter approach is not
  used). Timeout per call; nothing is shared with other plugins at runtime (the runner approach is
  copied from test-accounts).
- The scripts only read properties. None sets `readStatus` / `isRead`, flags, moves, deletes,
  sends, or opens a window. Reading `content` (Mail) or `plainTextContent` (Outlook) through Apple
  events is not the same as displaying the message, and the apps mark a message read when it is
  displayed. Apple and Microsoft do not document this either way and it is **not verified**; so `get_message` reads the read flag before and after and returns a
  `warning` if the app changed it (the plugin never sets it back: that would be a write).
- An app that is not running is reported, never started (`Application(...).running()` asks the
  system, not the app).
- Errors: `-1743` → "allow it in System Settings > Privacy & Security > Automation"; `-600` /
  not running → "open the app"; a killed call or `-1712` → timeout (narrow the request); unknown
  account / mailbox → the names that exist.

**Automation consent.** The first call that talks to Mail (or Outlook) makes macOS ask whether the
app that launched the plugin (PM Bridge, or the terminal / Claude Code in development) may control
Mail. That is an ordinary per-user dialog; no administrator is needed. If it was declined, turn it
on in System Settings > Privacy & Security > Automation, under that app.

## New Outlook cannot be read

Current Microsoft Outlook for Mac runs **New Outlook** by default (`defaults read com.microsoft.Outlook
IsRunningNewOutlook` = 1). New Outlook has no working AppleScript object model for mail: Microsoft
365 roadmap item 88537 ("Outlook: Support for AppleScript in the New Outlook for Mac") was planned
for December 2025 and then cancelled: Microsoft wrote that scripted workflows depend on Legacy
Outlook behaviour and "full compatibility isn't feasible". The plugin reads that preference first
(a preferences read, no Apple event) and, for New Outlook, every Outlook call answers:

> New Outlook does not allow other apps to read mail: Microsoft dropped AppleScript support for New
> Outlook for Mac. Use app "mail" (Apple Mail) instead; ...

Legacy Outlook still has its old dictionary, so the Outlook scripts target it. But Microsoft says
Legacy Outlook **stops working against Exchange Online mailboxes from October 2026** (EWS retired
2026-10-01), so for a Microsoft 365 mailbox it is at best a cache. In practice: use Apple Mail.

## Build and test

```sh
npm install
npm test        # tsc -> dist/, then unit tests and the stdio end-to-end test
```

The tests never call the real `osascript`, Mail or Outlook (that would read private mail and pop
permission dialogs). The unit tests run the **real script texts** in a Node `vm` against a fake JXA
object model (`test/fakeJxa.ts`, `test/fixtures.ts`): callable specifiers, batch property reads,
`whose` with `_and` / `_or` / `_contains` / `_greaterThan`, `-1743` errors, a stopped app. That
checks the scripts' logic and the JSON transport, not how the real apps answer.

## Not verified (to be tested in the app)

- The scripts against the real Apple Mail and Legacy Outlook: property names (`dateReceived`,
  `readStatus`, `mailAttachments`, `timeReceived`, `plainTextContent`, ...), `whose` with dates,
  message order, Mail's unified inbox and `messages.whose({id})`, Outlook's `messages.byId`.
- That reading a body never marks a message read (the `warning` would show it).
- Speed on a large mailbox (each listing reads the dates of every matching message in one event).
- The Automation dialog wording and which app it names when PM Bridge starts the plugin.
