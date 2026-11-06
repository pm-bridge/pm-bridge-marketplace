# test-accounts plugin

A PM Bridge plugin for logging into the app under test: an account book of test accounts, picking
one by login type and environment (with the reason), reading an emailed login code from macOS
Mail.app, and computing authenticator (TOTP) codes. It is a standard Claude Code plugin: one MCP
stdio server plus one skill. **No tool returns a password or an authenticator secret.**

```
.claude-plugin/plugin.json   name, version, description, author, userConfig (+ metadata.pmb: account book)
.mcp.json                    server "test-accounts": node ${CLAUDE_PLUGIN_ROOT}/dist/server.js, env from ${user_config.KEY}
skills/test-login/SKILL.md   the skill test-accounts:test-login
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

## Configuration

Read once at startup from the process environment. `.mcp.json` maps each variable to a
`${user_config.KEY}` placeholder that the host (or Claude Code) fills from the owner's settings;
`plugin.json` `userConfig` declares each KEY (title, description, `required`, `sensitive`),
and `metadata.pmb.variables.accounts.kind = "accountBook"` makes `accounts` an account book (host only). The test accounts are one **account book** the owner keeps
in Settings > Keys and chooses here:

| userConfig key → variable | kind | required | |
|---|---|---|---|
| `accounts` → `TEST_ACCOUNTS` | accountBook | yes | arrives as a **JSON array string** `[{ id, username, password, loginType, environment, purpose, totpSecret? }]` in the book's order; `id` is the host's `acct-<16 hex>`; login type (`normal`, `sso`, `national-id`, `biometric`, ...), environment (`test`, `staging`, `production`, ...) and purpose are free text, `""` when not filled in; `totpSecret` (base32 or an `otpauth://totp/` URI) is there only when one is stored |
| `otp_senders` → `OTP_SENDERS` | text | needed for `read_mail_otp` | allowed senders, comma-separated: full addresses or `@domain`. Empty: mail is never read |
| `otp_subject_hint` → `OTP_SUBJECT_HINT` | text | no | only mail whose subject contains this |
| `otp_keywords` → `OTP_KEYWORDS` | text | no | phrases next to the code, comma-separated; default `authentication token code, email verification code` |
| `otp_code_pattern` → `OTP_CODE_PATTERN` | text | no | regular expression of a code; default six digits `(?<!\d)\d{6}(?!\d)` |

Missing or invalid config does not stop the server: every tool call returns an error. A missing
`TEST_ACCOUNTS`, one that is not valid JSON or not an array, or an account without an id or
username is an error naming the variable, the account's position or id, and never a value (the
JSON parser's own message is dropped, since it can quote part of the book). A placeholder left
unexpanded (`${user_config.accounts}`) counts as missing. Tools refer to an account by its `id`.

`read_mail_otp` runs `osascript` as a child process. It gets this process's environment without
the plugin's own variables, so the account book never reaches it.

## Tools

All four are reads (`annotations.readOnlyHint = true`).

| tool | arguments | returns |
|---|---|---|
| `list_accounts` | - | `{ count, accounts: [{ id, username, loginType, environment, purpose, hasTotp }] }` |
| `pick_account` | `loginType?`, `environment?`, `reason?` | `{ id, username, loginType, environment, purpose, why }`, or `{ id: null, why }` when nothing is picked |
| `read_mail_otp` | `sender?`, `subjectContains?`, `sinceSeconds?` (default 300, max 3600), `timeoutSec?` (default 45, max 55) | `{ code, source: "mail" }` |
| `totp` | `accountId` | `{ accountId, code, secondsRemaining, digits }` |

### Picking an account

- Only accounts with a username and a password are usable.
- Each account scores 2 for its login type and 2 for its environment when `reason` names them.
  Latin terms need a word boundary (`kiosk` does not match `kiosks`), Chinese terms match as a
  substring, a 1-character term never matches. Highest score wins; ties go to the account first in the book.
- Nothing matched: the first usable account; when those accounts span several environments the
  `why` starts with `WARNING` and names them (so a wrong-environment pick is never silent).
- New here: `loginType` and `environment` arguments are **filters**. When no account has them,
  nothing is picked and `why` lists what is configured, rather than falling back to another account.

### Reading the emailed code

- Mail is read through **Apple Events**: `osascript` asks Mail.app for the newest 20 inbox messages
  received since `sinceSeconds` ago, polling every 3 s until `timeoutSec`. This needs the
  user-level Automation consent only (an "OK" dialog the first time; no administrator). Reading
  `~/Library/Mail` directly would need Full Disk Access, which a standard account on a managed
  Mac often cannot grant, so it is not done.
- Only mail from `OTP_SENDERS` counts; `sender` can only narrow to one of them. The code is the
  pattern match nearest a keyword (MessAuto's approach); digits shown one per box are re-joined;
  two different codes in the window is an error, never a guess; the same code resent is fine.
- Automation denied (-1743): the error tells the owner to allow it once in System Settings >
  Privacy & Security > Automation, under the app that runs PM Bridge, for Mail. Mail not running
  (-600), script failure, timeout (with whether an allowed sender / the keywords were seen), and
  not-macOS each have their own message.
- `timeoutSec` stays under the host's 60 s plugin-call timeout, so the persona gets a reason, not a
  host timeout.
- The code is returned plainly: the login needs it. It is short-lived and is not one of the host's
  known values (the host only matches known values of 6+ characters, and a code is not
  registered anyway), so the host cannot mask it by value. The host's *pattern* redaction does
  mask a 4-8 digit number right after words like "OTP" or "verification code"; the result is
  `{ "code": ... }` with no such word, so it reaches the persona intact (a test guards this).

### Authenticator codes

RFC 6238 over HOTP (RFC 4226) with node's crypto: base32 (case, spaces, hyphens, padding
tolerated) or `otpauth://totp/` URIs with their digits, period and algorithm (SHA1/256/512). A code
with less than 8 s left is never returned: the tool waits for the next one (never more than one
period).

## How a password reaches a login flow without reaching the model

No tool here returns it, and none ever will (a `credential_for_flow` tool would put it in the
model's context). When this was part of the app, it was done host-side: the main process read the keychain and put
`ACCOUNT` (username) and `PASSWORD` into the Maestro child's environment, scrubbed both (and the
code, `PMB_TEST_OTP`) out of every output, and ran an emailed-code login as two flows
(`login-otp-request`, then `login-otp-confirm` with `${PMB_TEST_OTP}`) around the wait for the mail.

The hand-off to the maestro plugin follows that design:

- The maestro plugin declares its **own** account-book variable (kind `accountBook`). The owner chooses
  the same account book there as here: several plugins can choose one book, so nothing is typed
  twice and each value lives once in the keychain.
- The persona passes the maestro plugin the **`id`** that `pick_account` returned (`accountId` of
  `run_flow`). Both plugins receive the same book, so the id names the same account in both. The
  maestro plugin finds that account in its own config, hands `ACCOUNT` / `PASSWORD` /
  `LOGIN_TYPE` to the flow, never into a file or its result, and scrubs them from its output
  (see the maestro README for how they reach Maestro).
- Codes: for a two-phase login the maestro plugin should obtain the code itself between the two
  flows (copying this plugin's mail and TOTP modules, since plugins share no code), so the code
  never passes through the model either. A persona *can* pass a code from `read_mail_otp` / `totp`
  as a tool argument (the host redacts arguments value by value, and a bare number is not masked),
  but one call that requests, waits and confirms is what proved to work in the app.

## Tests

Selection rules with their original tests, the RFC 6238 vectors (SHA1, SHA256, SHA512), the mail code
extraction and polling with a stubbed `osascript` runner (no test touches Mail.app), the
permission-denied and Mail-not-running paths, every tool over an in-memory MCP connection with a
check after each call that no fake password or authenticator secret is in the result, config from
the environment (missing, unexpanded placeholders, a book that is not JSON or not an array, without
echoing it), a real `osascript` run (macOS only, Mail.app not involved) checking that the child gets
none of the plugin's variables, and a stdio end-to-end test that spawns the built server from
`.mcp.json` with the placeholders filled, checks that the `${user_config.*}` keys `.mcp.json` uses equal
`userConfig` (with `accounts` a sensitive account book in `metadata.pmb`), that no password crosses the transport, and that unset variables are errors.
