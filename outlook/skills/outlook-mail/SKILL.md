---
name: outlook-mail
description: How to read the owner's Outlook mail with the outlook tools - read only, only the messages the task needs, and what to do when the account is not connected (the owner presses Connect; never a password).
metadata:
  tools: list-mail-messages get-mail-message ask_owner
---

# Read Outlook mail

The outlook plugin runs the open-source Softeria ms-365-mcp-server in **read-only** mode with the
mail tools only. It can list and read the signed-in account's mail; it cannot send, reply, move or
delete anything. If a task needs that, say so to the owner instead of looking for a way around it.

## 1. Signing in is the owner's, not yours

The owner connects the account themselves with the **Connect** button in Settings → Plugins (a
device code they enter in their own browser). You have no sign-in tool and never need one.

1. If a mail tool answers that the account is not signed in (or the token is missing or expired),
   stop and tell the owner, with `ask_owner`: "Outlook is not connected. Please press Connect on
   the outlook plugin in Settings → Plugins, then tell me to go on."
2. Never ask the owner for a password, a device code or a token, and never type one anywhere.
3. If the owner says the sign-in page asked for an administrator's approval, tell them that is the
   organisation's consent policy, not something you can fix.

## 2. Read only what the task needs

1. Narrow first: use `list-mail-messages` with a filter or search (sender, subject, date) rather
   than listing the whole mailbox.
2. Open a message with `get-mail-message` only when the task needs its body.
3. In your answer, quote or summarise only the part of a message the task needs. Do not paste
   whole mails, other people's addresses, signatures or unrelated threads into the chat, a ticket
   or memory.
4. Say which message you used (sender, subject, date), so the owner can check it.
5. If nothing matches, say so. Never guess what a mail said.
