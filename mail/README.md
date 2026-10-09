# Osysharp.Conversations.Mail

Email as a channel for [Osysharp.Conversations](https://osyrin.com/templates/kits/conversations/). A reply the desk writes on a conversation
the customer started by mail goes out through [Osysharp.Mail](https://osyrin.com/templates/kits/mail/)'s outbox — from the address the
customer wrote to, with `In-Reply-To` and `References` so it lands in their thread — and the Message-ID it went out
under is kept on the conversation, so their answer comes back to it. A mail received becomes a message from whoever
wrote it, continuing the conversation it answers. An out-of-office is recorded and never answered; a bounce marks the
delivery it answers failed, and the mail system that sent it is never listed as a customer.

## Use it

```osy
// app.osy
use Osysharp.Identity@0;
use Osysharp.Mail@0;
use Osysharp.Conversations@0;
// Osysharp.Conversations.Mail is not written: it arrives by itself, because the app uses what it completes (`osy lock` says so)
use Osysharp.Workflow;
use Osysharp.Ui;
```

```osy
using Osysharp.Mail;
using Osysharp.Conversations;
using Osysharp.Conversations.Mail;

app.Mail = new MailSetup { Sender = new ResendSender { … }, From = "Acme Support <support@acme.com>" };
app.Conversations = new ConversationsSetup {
  Channels = [new MailConversationChannel { Endpoints = ["support@acme.com", "billing@acme.com"] }],
};
```

When the support address is something staff set — a settings row rather than a literal — override `Addresses()`. It
is asked when mail goes or comes, so the row is read then, and not every time the app's conversations setup is read:

```osy
public class DeskMailChannel : MailConversationChannel {
  public override List<string> Addresses() { return [SupportSettings.Single().Address]; }
}
app.Conversations = new ConversationsSetup { Channels = [new DeskMailChannel()] };
```

The desk's staff send the replies, so the app grants them the outbox:

```osy
partial entity OutgoingMail { security { allow create, read when IsConversationStaff; } }
```

Until the mail sender is ready (its key set), a reply is **held** on the conversation with the sender's own reason —
the desk reads what to fix.

### When a mail client drops the thread

Some clients answer without `In-Reply-To` or `References`. Set `PlusAddressReplies = true` and each reply carries a
`Reply-To` naming its conversation — `support+c-<id>@acme.com` — so the answer finds its way back by address alone. It
needs your mailbox to accept plus addresses (Gmail and Microsoft 365 do; otherwise the customer's answer bounces), so it
is off by default. The address only points: whoever writes to it must be in that conversation's thread to continue it,
exactly as for a quoted Message-ID — anyone else starts a conversation of their own.

## Receiving

`ReceiveMail(MailMessage)` is the one entry point: it takes a mail as the mailbox receiver hands it over and answers
what `ReceiveInbound` did. The connected-mailbox receiver (Gmail and Microsoft 365 over OAuth, IMAP) calls it for each
message; until that ships, an app's own webhook can. It runs as the receiver's account, which the app's
`IsConversationStaff` must admit.

What it reads:

- **who** — the sender's address (lower-cased) and the name they gave;
- **which conversation** — `In-Reply-To` and `References` against the Message-IDs the conversation has sent and received;
  a thread id admits only someone who is in that thread, so a forwarded mail cannot join a stranger to it;
- **where it came to** — the first of your `Endpoints` it was addressed to; the reply goes from there;
- **who else** — every other `To` and `Cc` address, which is how a copied colleague's later reply is let in;
- **automatic or not** — `Auto-Submitted` other than `no`, `Precedence: bulk/junk/list/auto_reply`, the `X-Autoreply`
  family, or a bounce (`mailer-daemon@`, `postmaster@`, `X-Failed-Recipients`).
