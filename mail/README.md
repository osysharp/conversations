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
what `ReceiveInbound` did. [Osysharp.Mail.Receiving](https://osyrin.com/templates/kits/mail-receiving/) watches the
app's mailboxes (Gmail and Microsoft 365 over OAuth, IMAP), and `app.Mail.Receiver = new MailIntoConversations();` files
each message it accepts through it; an app's own webhook can call it too. Called directly, it runs as the caller's
account, which the app's `IsConversationStaff` must admit.

What it reads:

- **who** — the sender's address (lower-cased) and the name they gave;
- **which conversation** — `In-Reply-To` and `References` against the Message-IDs the conversation has sent and received;
  a thread id admits only someone who is in that thread, so a forwarded mail cannot join a stranger to it;
- **where it came to** — the first of your `Endpoints` it was addressed to; the reply goes from there;
- **who else** — every other `To` and `Cc` address, which is how a copied colleague's later reply is let in;
- **automatic or not** — `Auto-Submitted` other than `no`, `Precedence: bulk/junk/list/auto_reply`, the `X-Autoreply`
  family, or a bounce (`mailer-daemon@`, `postmaster@`, `X-Failed-Recipients`).

## Examples

Taken from this package's own tests, which compile and run on every release.

```osy
using Osysharp.Identity;
using Osysharp.Mail;
using Osysharp.Conversations;
using Osysharp.Conversations.Mail;
using ConversationsMailTests;

[Test(Desk)]
void a_mail_received_becomes_a_message_from_whoever_wrote_it() {
  runas (Inbox) {
    var mail = Received("Ana Lima <Ana@Example.test>", "<m1@example.test>", "", "billing@desk.test");
    mail.Text = null;
    mail.Html = "<p>My invoice is <b>wrong</b> &amp; late.</p>";
    mail.Cc = ["Boss <boss@example.test>", "support@desk.test"];
    var r = ReceiveMail(mail);
    UnitOfWork.Commit();
    Assert.True(r.Started);
  }
  runas (Ada) {
    var c = Conversation.Single();
    Assert.Equal("Invoice", c.Subject);
    Assert.Equal("email", c.ReplyChannel);
    // The address it came TO is the one we answer from.
    Assert.Equal("billing@desk.test", c.ReplyEndpoint);
    var m = Message.Single();
    Assert.Equal("My invoice is wrong & late.", m.Text);
    Assert.Equal("Ana Lima", m.ByName);
    // Everyone else it was written to — not the sender, not our own addresses.
    Assert.Equal("boss@example.test", m.AlsoTo);
    Assert.Equal(1, ContactPoint.Count(p => p.Value == "ana@example.test"));
  }
}
```

From [`tests/mail.test.osy`](tests/mail.test.osy).

```osy
using Osysharp.Identity;
using Osysharp.Mail;
using Osysharp.Mail.Receiving;
using Osysharp.Conversations;
using Osysharp.Conversations.Mail;
using Osysharp.Storage;
using ConversationsMailTests;

[Test(Mailbox)]
void a_mail_from_the_watched_mailbox_lands_with_only_what_was_new_and_keeps_the_rest() {
  runas (Ada) { Watch(); }
  runas (Ada) {
    Arrive(1, GmailReplyWithFiles());
    UnitOfWork.Commit();
    TestClock.Advance(TimeSpan.FromMinutes(1));
    Poll();
  }
  runas (Ada) {
    var m = Message.Single(x => x.ExternalId == "<bo-2@example.test>");
    // What Bo wrote THIS time is the message…
    Assert.Equal("Still broken - screenshot and the file attached.", m.Text);
    // …what Gmail quoted under it is kept beside it, not thrown away…
    Assert.Contains("Can you send a screenshot?", m.QuotedText ?? "");
    // …and so is the whole original, and both files, each read for what it really is.
    Assert.True(m.Original != null);
    var files = MessageAttachment.Where(a => a.Message == m).OrderBy(a => a.FileName).ToList();
    Assert.Equal(2, files.Count);
    Assert.Equal("Export.cs", files[0].FileName);
    Assert.Equal(AttachmentKind.Code, files[0].Kind);
    Assert.Equal("csharp", files[0].Language);
    Assert.Equal(AttachmentKind.Image, files[1].Kind);
    // The receiver kept its own record, handed over once.
    Assert.Equal(ReceivedMailStatus.Handled, ReceivedMail.Single().Status);
  }
}
```

From [`tests/receiving.test.osy`](tests/receiving.test.osy).

```osy
using Osysharp.Identity;
using Osysharp.Mail;
using Osysharp.Conversations;
using Osysharp.Conversations.Mail;
using ConversationsMailTests;

[Test(Desk)]
void a_reply_goes_from_the_address_it_came_to_and_threads_in_the_customers_mail() {
  runas (Inbox) { ReceiveMail(Received("ana@example.test", "<m1@example.test>", "Wrong total.", "billing@desk.test")); UnitOfWork.Commit(); }
  runas (Ada) {
    Post(Conversation.Single(), "Corrected — a new invoice is on its way.");
    UnitOfWork.Commit();
    DeliverAll();
  }
  runas (Ada) {
    var sent = Carried.Single();
    Assert.Equal("billing@desk.test", sent.From);
    Assert.Equal("ana@example.test", sent.To);
    Assert.Equal("Re: Invoice", sent.Subject);
    Assert.Equal("Corrected — a new invoice is on its way.", sent.Text);
    Assert.Equal("<m1@example.test>", sent.InReplyTo);
    Assert.Contains("<m1@example.test>", sent.References);
    var delivery = MessageDelivery.Single();
    Assert.Equal(DeliveryState.Sent, delivery.State);
    // The delivery's id IS the Message-ID the carrier sent — what the customer's answer will quote.
    Assert.Equal(sent.MessageId, delivery.ExternalId);
  }
}
```

From [`tests/mail.test.osy`](tests/mail.test.osy).

## Learn more

- [Osysharp.Conversations.Mail in the package catalogue](https://osysharp.com/packages/#osysharp-conversations-mail)
- [The Osy# reference](https://osysharp.com/reference/) — the language, the runtime and every kit
- [`llms-full.txt`](https://osysharp.com/llms-full.txt) — the whole reference and specification as one Markdown file, for a coding agent

## Licence

MIT — see [`LICENSE`](LICENSE).
