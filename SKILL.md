---
name: apollo-outlook-signature-fix
description: Fix an email signature that comes out wrong when a Microsoft 365 / Outlook mailbox sends through Apollo (or any tool that sends via the mailbox API). Builds the signature HTML, has the user create an Exchange mail flow "disclaimer" rule that appends it to every outgoing email, then removes every signature on the Apollo side and tests. Use when the user says their signature is missing, different or broken in Apollo emails, the logo sits on the wrong line, or they ask to fix their signature after linking Apollo.
---

# Apollo + Outlook signature fix

Outlook adds a signature in the compose window. Apollo sends through the mailbox API, so the Outlook signature never gets added, and Apollo's own signature editor strips tables, borders and most styling when it saves. The fix is to let Exchange Online append the signature to every outgoing email with a mail flow rule, and to remove every signature on the Apollo side.

Before starting, define done with the user: **an email sent from an Apollo sequence step arrives at an outside inbox (for example Gmail) with the signature in the right layout, exactly once.** Nothing less counts as fixed.

## Ground rules

- The user signs in to everything themselves. Never ask for, type or store a password or a one-time code.
- The user sends the test emails. Do not send email on their behalf.
- Creating a mail flow rule changes how the whole mailbox sends. Explain what the rule does and get the user's clear yes before helping create it.
- Back up anything you overwrite: write the current Apollo signature down before clearing it.
- Never claim a step worked off a success message. Re-read the setting or the received email.

## Step 1. Check this applies

Confirm all three, and stop if one fails:

1. The mailbox is Microsoft 365 / Exchange Online (Outlook). Google Workspace is not covered by this skill.
2. The mailbox is linked to Apollo.
3. The user can open the Exchange admin centre (https://admin.exchange.microsoft.com) for their domain. Some resellers, GoDaddy among them, run Microsoft 365 for the customer; it can still work, but the user may have to sign in again more than once.

## Step 2. Collect the signature

Get the details from the user's current Outlook signature (Outlook on the web: Settings → Accounts → Signatures, or a screenshot):

| Placeholder | What goes in it |
|---|---|
| `{{FULL_NAME}}` | Name as shown |
| `{{JOB_TITLE}}`, `{{COMPANY}}` | Title line |
| `{{WEBSITE_URL}}`, `{{WEBSITE_LABEL}}` | Link target (`https://...`) and the text shown |
| `{{PHONE}}` | Phone number as shown |
| `{{LOGO_URL}}` | Public `https://` address of the logo (see below) |
| `{{LOGO_WIDTH}}`, `{{LOGO_HEIGHT}}` | Display size in pixels |
| `{{FONT_STACK}}` | e.g. `Arial, Helvetica, sans-serif` |
| `{{NAME_COLOR}}`, `{{TEXT_COLOR}}`, `{{DIVIDER_COLOR}}` | Hex colours, e.g. `#17140f`, `#6b655c`, `#dad5cc` |

Fill a field only from what the user's signature or the user says. If a value is unknown, ask; do not guess. If the user has no phone line or no logo, remove that part of the template rather than leaving a placeholder.

**The logo needs a public `https://` address.** A logo pasted into an Outlook signature is usually embedded inside the email (a `data:` image). That cannot be used in a rule and often shows as broken in Gmail. Options:

- The user's own website, if they can upload a file there.
- Apollo's image upload: in Apollo, Settings → Domains & Mailboxes → Mailboxes only → click the mailbox → Overview tab → scroll to **Edit Signature** → image button. Apollo stores the file and gives it a public address. Read the address from the editor's HTML view (the `<>` button), then close without saving, or clear the signature in Step 5 anyway.

Check the address loads without being signed in, for example `curl -sL -o /dev/null -w "%{http_code} %{size_download}\n" "<url>"` should print `200` and the file size.

**Pick a marker.** The rule must skip emails that already carry the signature (emails written in Outlook, where Outlook adds its own). Use a piece of text that only appears in the signature. The phone number works well. Write it down for Step 3.

Fill `templates/signature.html` with the values and save the result as a file the user can open and copy from. Do not change the table structure: Outlook and Gmail both render it.

## Step 3. Create the Exchange rule (the user clicks, you guide)

The user goes to https://admin.exchange.microsoft.com → **Mail flow** → **Rules**:

1. **+ Add a rule** → **Apply disclaimers**.
2. Name: `Signature - <name> outbound`.
3. **Apply this rule if:** *The sender* → *is this person* → pick the mailbox owner → Save. (The rule summary later shows this as "Is received from '<address>'". That is the same condition.)
4. **Do the following:** *Apply a disclaimer to the message* → *Append a disclaimer*.
   - **Enter text**: paste the full HTML from Step 2 → Save.
   - **Select one** (fallback action): **Ignore** → Save. Do not pick "Wrap": it turns an email the rule can't edit into an attachment.
5. **Except if:** *The subject or body* → *subject or body includes any of these words* → type the marker from Step 2 → **Add** → Save.
6. **Next** → leave Rule mode on **Enforce** → **Next** → **Finish** → **Done**.
7. **The rule is created switched off.** Click the rule in the list, turn on **Enable or disable rule**, close the panel, click **Refresh**, and check the Status column says **Enabled**.

## Step 4. Wait, then test the rule on its own

Allow 15 to 30 minutes after enabling. Then the user:

1. Writes a new email in Outlook, **deletes the signature Outlook adds**, and sends it to an outside address (for example their Gmail).
2. Opens it in that inbox, not in Sent Items. The rule adds the signature on the way out, so the Sent Items copy never shows it.

The signature should sit at the bottom with the logo on the left and the text beside it. If it is missing, in this order: the rule is still Disabled; it has not been long enough; the email contains the marker text; the condition says recipient instead of sender.

## Step 5. Remove every signature in Apollo

Any signature Apollo adds contains the marker, so the rule skips that email and Apollo's broken version goes out. Clear all of them:

1. **The mailbox signature:** Settings → Domains & Mailboxes → Mailboxes only → click the mailbox → Overview → **Edit Signature**. Write down the current content as a backup, clear it, click **Save**.
2. **Every email step of every sequence:** switch off the step's signature setting ("include signature"). With the Apollo API or MCP this is `include_signature: false` on each touch.
3. **Any signature typed into a step body:** remove it, so the body ends at the sign-off ("Kind regards,").

Re-read each one afterwards (the API's mailbox list returns `signature_html`, which should be empty; a sequence read shows `include_signature` per touch).

## Step 6. Test through Apollo

First have the user close every Apollo tab that was open before Step 5, or reload it with Ctrl+Shift+R. A sequence page loaded earlier still holds the old signature, and its test button sends it even though the saved settings are clean. The rule then sees the marker and skips the email, so it looks as if nothing changed.

The user sends a test of the first step of a sequence to an outside address. Check the received email:

- the signature appears once, under the sign-off, in the right layout;
- the logo loads.

Then test a follow-up step that replies in the same thread. Apollo may quote the earlier email in a follow-up. If the quoted part contains the marker, the rule skips the follow-up and it goes out without a signature. Decide with the user whether that is acceptable, or choose a marker that is not in the signature itself.

Only call it fixed when the received emails look right.

## Things to tell the user

- The rule covers every email from that mailbox: Outlook, Apollo and any other tool. Their Outlook signature can stay on; the marker keeps it from being doubled.
- The signature goes at the very bottom of the email. In replies, that is below the quoted conversation.
- To change the signature later, edit the rule's disclaimer text in the Exchange admin centre. There is one place to update.
- If the logo is hosted in Apollo and that upload is ever deleted, the logo breaks in every email. Their own website is the safer home.
