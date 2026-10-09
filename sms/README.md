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
use Osysharp.Sms.Elks@0;              // or another carrier's sender
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
