# apollo-outlook-signature-fix

After linking Apollo, your email signature needs to be fixed.

## The problem

You connect a Microsoft 365 (Outlook) mailbox to Apollo and send a test from a sequence. The signature that arrives is not the one you set up in Outlook:

- Outlook adds your signature in the compose window. Apollo sends through the mailbox API and never opens that window, so your Outlook signature is never added.
- Apollo has its own signature field, but its editor strips tables, borders and most fonts and colours when it saves. A two-column signature (logo on the left, a divider, text on the right) comes out with the logo stuck on the same line as your name.

## The fix

Let Microsoft 365 add the signature on the way out. An Exchange mail flow rule with a "disclaimer" action appends your signature HTML to every email the mailbox sends, whether it was written in Outlook, sent by Apollo, or sent by any other tool. This is how many companies run central signatures.

Then remove every signature on the Apollo side, so each email gets exactly one: yours, in your exact layout, maintained in one place.

## What you need

- A Microsoft 365 mailbox (Exchange Online) connected to Apollo
- Admin access to the Exchange admin centre for your domain
- Your logo or icon hosted at a public `https://` address (the steps show how)
- About 20 minutes, plus up to 30 minutes for the rule to start working

Google Workspace mailboxes are not covered here.

## Use it with Claude Code

Clone this repo into your Claude Code skills folder:

```
git clone https://github.com/MrPresident-cms/apollo-outlook-signature-fix.git ~/.claude/skills/apollo-outlook-signature-fix
```

Then tell Claude: "fix my email signature for Apollo". Claude reads `SKILL.md`, builds your signature HTML from the template, and walks you through the Exchange rule, the Apollo settings and the test. You do the sign-ins yourself; Claude never asks for your password.

## Do it by hand

Everything Claude does is written out in [`SKILL.md`](SKILL.md), step by step. The signature template is in [`templates/signature.html`](templates/signature.html).

## Things that will catch you out

- New rules are created **switched off**. Turn the rule on, or nothing happens.
- Give the rule up to 30 minutes after switching it on before you test.
- The copy in your Sent Items will not show the signature. Check the email that arrives.
- A logo pasted into an Outlook signature is usually embedded in the email itself. That doesn't work in a rule and shows as broken in Gmail. It needs a public `https://` address.
- Any signature left in Apollo (the mailbox signature, a step's signature setting, or one typed into a step) contains your marker text, so the rule skips that email and the broken Apollo version goes out instead. Clear all three.
- After clearing Apollo's signatures, reload every open Apollo tab (Ctrl+Shift+R) before sending a test. A page loaded earlier still sends the old signature.
- The rule adds the signature at the very bottom of the email. In a reply, that is under the quoted conversation.
