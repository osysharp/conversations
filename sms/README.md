# Osysharp.Conversations.Sms

Text messages as a channel for [Osysharp.Conversations](https://osyrin.com/templates/kits/conversations/). A reply the desk writes on a
conversation the customer started by text goes out through [Osysharp.Sms](https://osyrin.com/templates/kits/sms/)'s outbox — from the app's
own number, the one the customer texted — and a text received becomes a message from that number, continuing the
number's latest conversation if it moved within the window (seven days by default), or starting a new one. A carrier's
delivery report moves the delivery it reports on. A reply too long for one text is refused when it is written.

## Use it

```osy
// app.osy
use Osysharp.Identity@0;
use Osysharp.Sms@0;
use Osysharp.Sms.FortySixElks@0;              // or another carrier's sender
use Osysharp.Conversations@0;
// Osysharp.Conversations.Sms is not written: it arrives by itself, because the app uses what it completes (`osy lock` says so)
use Osysharp.Workflow;
use Osysharp.Ui;
```

```osy
using Osysharp.Sms;
using Osysharp.Conversations;
using Osysharp.Conversations.Sms;

app.Sms = new SmsSetup { Sender = new ElksSmsSender { Username = …, Password = … } };
app.Conversations = new ConversationsSetup { Channels = [new SmsConversationChannel { Number = "+46766861004" }] };

// The app's own endpoint for the carrier's inbound webhook and delivery report calls:
//   ReceiveSms(from, to, message, id);  SmsDeliveryReport(id, delivered, problem);
```

## What it will not do

Text from a NAME. `app.Sms.From = "Acme"` is right for a one-time code; a conversation needs a number to be answered
on, so the channel is not ready until it has one — and says so.

## Source and tests

`model/SmsChannel.osy`; `tests/sms.test.osy` asserts what the carrier was handed, with a recording sender — no test texts
anybody.

## Examples

Taken from this package's own tests, which compile and run on every release.

```osy
using Osysharp.Identity;
using Osysharp.Sms;
using Osysharp.Conversations;
using Osysharp.Conversations.Sms;
using ConversationsSmsTests;

[Test(Desk)]
void a_text_received_becomes_a_message_from_that_number() {
  runas (Hook) {
    var r = ReceiveSms("+46 70-111 22 33", "+46766861004", "Is my order on its way?", "in-1");
    UnitOfWork.Commit();
    Assert.True(r.Started);
  }
  runas (Ada) {
    var c = Conversation.Single();
    Assert.Equal("sms", c.ReplyChannel);
    Assert.Equal("+46766861004", c.ReplyEndpoint);
    Assert.Equal("Is my order on its way?", Message.Single().Text);
    // The number, as every comparison spells it.
    Assert.Equal(1, ContactPoint.Count(p => p.Kind == ContactPointKind.Phone && p.Value == "+46701112233"));
  }
}
```

From [`tests/sms.test.osy`](tests/sms.test.osy).

```osy
using Osysharp.Identity;
using Osysharp.Sms;
using Osysharp.Conversations;
using Osysharp.Conversations.Sms;
using ConversationsSmsTests;

[Test(Desk)]
void a_reply_is_texted_from_the_desks_number_and_its_report_moves_the_delivery() {
  runas (Hook) { ReceiveSms("+46701112233", "+46766861004", "Is my order on its way?", "in-1"); UnitOfWork.Commit(); }
  runas (Ada) {
    Post(Conversation.Single(), "Yes — it arrives tomorrow.");
    UnitOfWork.Commit();
    DeliverAll();
  }
  var carrierId = "";
  runas (Ada) {
    var sent = Carried.Single();
    Assert.Equal("+46701112233", sent.To);
    // From the NUMBER customers text — never the app's alphanumeric default (`app.Sms.From = "Desk"`).
    Assert.Equal("+46766861004", sent.From);
    Assert.Equal("Yes — it arrives tomorrow.", sent.Text);
    Assert.Equal(DeliveryState.Sent, MessageDelivery.Single().State);
    carrierId = sent.CarrierId;
  }
  runas (Hook) { SmsDeliveryReport(carrierId, true); UnitOfWork.Commit(); }
  runas (Ada) { Assert.Equal(DeliveryOutcome.Delivered, MessageDelivery.Single().Outcome); }
}
```

From [`tests/sms.test.osy`](tests/sms.test.osy).

```osy
using Osysharp.Identity;
using Osysharp.Sms;
using Osysharp.Conversations;
using Osysharp.Conversations.Sms;
using ConversationsSmsTests;

[Test(Desk)]
void a_text_within_the_window_continues_the_conversation_and_one_after_it_starts_a_new_one() {
  runas (Hook) { ReceiveSms("+46701112233", "+46766861004", "First question.", "in-1"); UnitOfWork.Commit(); }
  TestClock.Advance(TimeSpan.FromDays(2));
  runas (Hook) {
    var r = ReceiveSms("+46701112233", "+46766861004", "And another thing.", "in-2");
    UnitOfWork.Commit();
    Assert.True(!r.Started);
  }
  TestClock.Advance(TimeSpan.FromDays(8));
  runas (Hook) {
    var r = ReceiveSms("+46701112233", "+46766861004", "Weeks later — a new question.", "in-3");
    UnitOfWork.Commit();
    Assert.True(r.Started);
  }
  runas (Ada) { Assert.Equal(2, Conversation.Count()); }
}
```

From [`tests/sms.test.osy`](tests/sms.test.osy).

## Learn more

- [The Osysharp.Conversations.Sms reference](https://osysharp.com/reference/ui/conversations-kit/)
- [Osysharp.Conversations.Sms in the package catalogue](https://osysharp.com/packages/#osysharp-conversations-sms)
- [The Osy# reference](https://osysharp.com/reference/) — the language, the runtime and every kit
- [`llms-full.txt`](https://osysharp.com/llms-full.txt) — the whole reference and specification as one Markdown file, for a coding agent

## Licence

MIT — see [`LICENSE`](LICENSE).
