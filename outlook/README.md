# outlook

Outlook mail for PM Bridge personas, **read only**. No code of our own: the plugin runs the
open-source [Softeria ms-365-mcp-server](https://github.com/Softeria/ms-365-mcp-server) (MIT) as a
stdio MCP server, pinned to one version, with `--org-mode --preset mail --read-only`.

```
.claude-plugin/plugin.json   userConfig: tenant_id (optional, default common); metadata.pmb.signIn (the Connect button)
.mcp.json                    npx -y @softeria/ms-365-mcp-server@0.158.0 --org-mode --preset mail --read-only
skills/outlook-mail/         skill outlook:outlook-mail (read only, least mail shown; not connected → ask the owner to press Connect)
test/                        manifest checks (node --test)
```

It ships with the app **disabled** (`defaultEnabled: false` in the marketplace). The first start
downloads the pinned package with `npx`, so it needs network access and `npx` on PATH.

## Version

`0.158.0` (published 2026-10-02) is pinned, not `latest`: `npx -y` would otherwise run whatever was
published last. When moving to a newer version, pick one that has been out for at least two weeks,
read its changelog, and redo the token-storage check below.

## Configuration

| userConfig key → variable | kind | required | |
|---|---|---|---|
| `tenant_id` → `MS365_MCP_TENANT_ID` | text | no (default `common`) | your organisation's tenant ID or domain; `common` lets any work account sign in |

## Sign-in (the owner, once): the Connect button

`plugin.json` names the server's own auth tools as the plugin's sign-in:
`metadata.pmb.signIn = { "login": "login", "status": "verify-login", "logout": "logout" }`.
PM Bridge then shows one **Connect** button on the plugin's card in Settings → Plugins:

1. Connect (the first time it shows the exact command line and that it downloads and runs code as
   you, then enables the plugin) calls `login`. Softeria answers at once with Microsoft's device-code
   sentence; PM Bridge shows the code big (already copied) and opens
   https://microsoft.com/devicelogin in your browser. The server keeps polling Microsoft in the
   background, so it must stay running until you finish.
2. You enter the code and sign in with your work account in your own browser. Nobody ever enters a
   password into PM Bridge.
3. PM Bridge asks `verify-login` every few seconds and shows "Connected: <name> (<address>)";
   Disconnect calls `logout`.

These three tools are the host's own: they are never offered to a persona (personas only get the
mail read tools, granted by default because they are annotated read-only). The device code is only
shown to you; it is not stored, logged or audited. From a terminal the same sign-in works with
`npx -y @softeria/ms-365-mcp-server@0.158.0 --org-mode --login`.

**Admin consent risk.** By default the server signs in through Softeria's own multi-tenant app.
Under Microsoft's default consent policy an ordinary user usually cannot consent to a third-party
app reading mail (`Mail.Read`), so the sign-in page may ask for administrator approval. Then either
request approval, or ask your organisation's admin to register a single-tenant public client app (device code allowed,
`Mail.Read` consented) and point the server at it with `MS365_MCP_CLIENT_ID` (add it to `.mcp.json`
`env` as a userConfig value). Do not borrow another mail client's client ID to get around consent.

If the sign-in page shows "Need admin approval" for "MS 365 MCP Server, Softeria AS (verified
publisher)", your organisation's admin may need to approve that app (or register your own app,
above) before this plugin works. Borrowing another client's ID (e.g. Outlook Web's) instead is not
an option: it would impersonate Microsoft's own app and bypass the tenant's consent policy.

## outlook or mac-mail?

- **mac-mail** (the sibling plugin) reads the mail already in Apple Mail on this Mac (or Legacy
  Outlook; New Outlook cannot be read by other apps). No sign-in and no tenant consent; Mail must be
  open. Use it while the tenant blocks third-party mail apps.
- **outlook** (this plugin) reads the Microsoft 365 mailbox online through Microsoft Graph, all of
  it, without any app running. Use it once your organisation's admin has approved the app (if approval is needed).

## Token storage check (do this after the first sign-in)

The server keeps the token cache in an encrypted file and only the 32-byte key in the macOS
keychain. If the keychain cannot be used it writes the key to a plain file instead, which makes the
tokens effectively plaintext. After signing in, check both:

1. The keychain has an item for service `ms-365-mcp-server`, account `cache-key`
   (`security find-generic-password -s ms-365-mcp-server -a cache-key` finds it; do not print the
   value).
2. There is **no** `.cache-key` file in `~/Library/Application Support/ms-365-mcp-server/`.

If `.cache-key` exists, **stop using the plugin**: disable it, delete that folder's cache and key
files, and sign out, until the keychain path works (PM Bridge never stores tokens in plain files).
