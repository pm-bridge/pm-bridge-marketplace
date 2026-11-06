---
name: test-login
description: How to sign in to the app under test - pick the test account with pick_account and say why, get the emailed code (read_mail_otp) or authenticator code (totp) when the login asks for one. Never ask the owner for a password, never print credentials.
metadata:
  tools: list_accounts pick_account read_mail_otp totp
---

# Sign in to the app under test

1. **Pick the account, with a reason.** Call `pick_account`:
   - `loginType` when you know which login the screen needs (for example `normal`, `sso`,
     `national-id`, `biometric`), `environment` when you know the environment (`test`, `staging`, `production`);
   - `reason`: the ticket summary or the flow text, so a login type or environment named there
     decides between accounts (a "biometric login" ticket, a bug seen "on production").
   State the `why` it returns in your notes or report. If it starts with `WARNING` (several
   environments, none named), check the ticket for the environment and call it again with
   `environment` set; if the ticket does not say, ask the owner which environment, not for a password.
   If `id` is `null`, nothing was picked: report the `why` to the owner (an account is missing or
   has no password). Otherwise keep the `id`: it names this account in every plugin that uses the
   same account book. `list_accounts` shows every account in the book when you need the overview.
2. **The password never goes through you.** No tool returns it and you never need it. The login
   flow gets it from the owner's account book (for example pass the `id` from `pick_account` as
   `accountId` to the Maestro plugin's `run_flow`). Never ask the owner for a password, never type
   one into a message, a ticket, a file or a command.
3. **When the login asks for a code:**
   - emailed code: trigger the code in the app first, then call `read_mail_otp` right away
     (`subjectContains` or `sender` if you know them). It waits up to `timeoutSec` for the mail. On a
     timeout, call it again once; if the result says Automation is not allowed or Mail is not
     running, tell the owner exactly what it says (the owner allows Automation for Mail once, no
     administrator needed) and stop;
   - authenticator code: call `totp(accountId)` with the `id` `pick_account` returned. It never returns a
     code that is about to expire.
   Use the code immediately and only in the login. Never put it in a chat message, a ticket comment,
   a commit, a log or a file.
4. **Never print credentials.** Usernames and account ids are fine to mention; passwords,
   authenticator secrets and codes are not.
