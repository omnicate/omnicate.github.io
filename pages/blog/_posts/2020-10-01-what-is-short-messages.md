---
layout: blogpost
permalink: /blog/what-is-a-short-message
title: What the heck is a short message?
date: 2020-10-01
tags: telco MAP TCAP SS7 FORWARDSM SMS
author: <a href="https://www.linkedin.com/in/sebastian-weddmark-olsson/">Sebastian Weddmark Olsson</a>, Telco newb
---

I will try as best as I can to give an explanation of what happens
when you send an SMS from your phone.

Disclaimer: Telco stuff is hard.

Also disclaimer: this blog post will also contain alot of ackronyms,
after all, it is telco.

_Aaand down the rabbit hole we go..._

# Where to even start

In the *SS7* (telco/telecom/telecommunications) network there are many
different nodes (servers), with different kinds of tasks.

The group of protocols that is used to send signals over *IP* between
these nodes is called *SIGTRAN* (derived from "signaling transport").
Older networks that have not switched to *IP* do not use *SIGTRAN*.

*SIGTRAN* protocols are the lower layer protocols used for signaling,
they range from *SCTP* (Stream Control Transmission Protocol) to
*M2PA* (Message Transfer Part 2 User Peer-to-Peer Adaptation Layer)
and *M3UA* (Message Transfer Part 3 User Adaptation Layer).

*SCTP* is like a mix between *UDP* and *TCP*. It is supposed to be
quicker than *TCP*, but more reliable than *UDP*.

Both *M2PA* and *M3UA* support *SCTP* management, and the reporting of
status changes of those, as well as providing transfer of *MTP3*
(Message Transfer Part 3) messages.

On top of *SIGTRAN* are the *SS7* protocols.

What I'm going to talk about are the protocols on the very top of *SS7*,
specifically the *MAP* (Mobile Application Part) as well as the *TCAP*
(Transaction Capabilities Application Part). There are other protocols
inbetween, for instance *SCCP* (Signalling Connection Control Part)
which handles some handshaking, routing, and resilience.

The *MAP* layer is used when talking to some of the telco nodes such
as *HLR* (Home location registry), *VLR* (Visitor location registry),
*MSC* (Mobile switching centre), *SGSN* (Serving *GPRS* [ackronym in
ackronyms; go telco!] support node) and the *SMSC* (Short message
service centre).

# MAP versions and TCAP dialogues

There are some iterations of *MAP*, v1, v2, v3, and v4, and all
messages almost always comes in pair, an acknowledgement
(`ReturnResult` or `ReturnError`) for each sent message (`Invoke`).

To determine which version to use between two nodes, the sending node
tries to start the transaction (called a dialogue) by sending a *TCAP*
`Begin` message with the *MAP* message and it's highest compatible
version. If the receiving node cannot talk that version, it sends a
*TCAP* `Abort` message with it's highest compatible version. For v1 there
might not even be a reason, but the sending node might try to send the
*MAP* message as v1 anyway.

In my head it goes like this:

```
Node 1: "Hi, I want to talk version 3 to you about this"
Node 2: "No I don't understand you, but we can talk version 2 about it instead"
Node 1: "Ok, then I want to talk version 2 to you about this instead"
Node 2: "Aah, now I see..."
```

Or maybe

```
Node 1: "Hi, I want to talk version 3 to you about this"
Node 2: "No"
Node 1: "Ok, then I want to talk to you about this in version 1 instead"
Node 2: "Maybe I will talk to you, maybe I will not"
```

For *TCAP* dialogues there are (mainly) four message types.  `Begin`,
`Continue`, `End`, `Abort`. Each of the types have an ID (or two, as I
said, telco is complicated), a component and a dialogue part. The
component contains the *MAP* messages, and the dialogue part contains
the version and application to use (that is *MAP* Application;
i.e. which type of message it contains), but it is only used in the
first message from both nodes for the version negotiation.

_I think this covers most of it, let's get back to the fun part._

# How does SMS work?

*SMS* was initally implemented because of the wish to send text
messages to pagers using the phoneline when it was not in use for
phonecalls. It was decided at a meeting in Oslo to be released to the
public when some French and German company understood it's
value. (Don't quote me on any of this).

When you send an *SMS*, the *SMS* is transfered to the *MSC* or the
*SGSN* in your current (serving) network. The *MSC* or *SGSN* then
sends an packet called `MO-Forward-SM` towards the *SMSC* in your
current network. It stands for "Mobile Originating Forward Short
Message" meaning it started from your (mobile-)phone.

The *SMSC* then sends another packet, this time a `MT-Forward-SM`,
towards the *MSC* in the recipients network. In this case *MT* stands
for Mobile Terminated, meaning it goes towards the recipients phone.

Dia have amazing icons:
<div class="post-images center">
    <img src="/img/blog/sms/forward-sm.svg" alt="You calling your mom" />
</div>

The similarities in *MO* and *MT* requests are that they both contain
a origin and destination address as well as the user data (your actual
text message), and a possibly a correlation id which is basically a
mapping between your SIM-card id and a temporary id and was originally
used for making sure that the sending network payed for *SMS*s towards
the receiving network.

For *MO* the origin address is your *MSISDN* (read telephone number),
and the destination is the *GT* address (Global title; a way to route
stuff) of the *SMSC*. For *MT* messages the origin address is the *GT*
of the *SMSC* and the destination address is either the recipients
*IMSI* (read SIM-card) or the recipients correlation id. It could also
be a *LMSI* which is a 4-byte network location identifier if the
recipient is also within the same network as the sender.

Ever wondered why there is a limit to the size of the text message you
are sending?

<div class="post-images center">
    <img src="/img/blog/sms/160_chars.png" width="50%" alt="Characters left: 2/160" />
</div>

If you (god forbid) you would break the protocol and send a text
message greater than 140 bytes, which translates to 160, 140, or 70
characters depending on locale [1], then your phone would break up the
message into multiple text messages. This arbitrary size of 140 bytes
is not really arbitrary at all. It was chosen because it would
precisely fit into a single *MTP3* *SIF* (Signalling Information
Field) when routing label, *SCCP*, *TCAP* and *MAP* layers were taken
into account.

[1] There is something called *GSM7* bit-packing. Instead of using 1
byte (8 bits) per character, *GSM7* uses 7 bits. This means that
instead of 140 characters, you can get up to 160 characters per
*SMS*. The drawback is that you will have a smaller subset of
characters to use, only the most common is supported. If you include
any non-*GSM7* characters in your *SMS* then the *SMS* will
automatically be converted to use *USC-2* instead. *USC-2* uses
2-bytes, or 16 bits, instead of *GSM7*s 7 bits. That leaves you with
70 characters per *SMS*. *USC-2* is similar for the basic multilingual
plane to *UTF-16*. In fact *UTF-16* is an extension of *UCS-2*. The
main difference is that *USC-2* is fixed width, while *UTF-16* is
variable width of one or two 16-bits code points. Most phones will
however see *USC-2* text and think it is *UTF-16* and thus decode it
wrongly.

Using emojis will convert the encoding:
<div class="post-images">
    <img src="/img/blog/sms/67_chars.png" width="50%" alt="Characters left: 45/67 (3)" />
</div>

However when *MAP* v2 started to use *TCAP* dialogues there was more
information to put into the packet and 140 bytes might not be left for
the *SMS*. The *SMSC* would then need to break up the message into
chunks, and start the transaction an empty *TCAP* `Begin` message and
set a flag in the *MT* request called `moreMessagesToSend`. It would
then send the actual text inside `Continue` messages. In the end
(_hehe_) the `End` message is transmitted as a response and the
transaction is finished.

The response back to a *MO* request is, as previous stated, an
acknowledgement if the *SMS* have been successfully submitted to the
*SMSC* or not (again either `returnResult` or `returnError`). For *MT*
requests the acknowledgement is if the *SMS* is successfully delivered
or not.

If the *MT* request is not successful, the *SMSC* could ask the *HLR*
(the Home Location Registry is basically a database containing user
subscriptions and knowledge of which nodes the mobile talked to
latest) to be notified when the user comes back online. A bunch of
other *MAP* messages are then involved, such as
- `reportSMDeliveryStatus`,
- `informServiceCentre`,
- `alterServiceCentre`, and
- `readyForSM`.

_At least this is main idea I think..._

# Differences in MAP versions for SMS

There are three *MAP* versions defined for *SMS*.

In version 1, the dialogue portion was not invented and all chunks are
sent in new *TCAP* dialogues. The size of the user data could then
be 140 bytes.

In version 1 and version 2 there is no difference between *MT* and
*MO*. Everything is sent as another type of message `Forward-SM`,
which does not include any privacy correlation ids, and there are no
fancy responses with delivery status. There is still an
acknowledgement, but is in a form of an empty message.

Only way to see difference between an MO and a MT in version 1 is to
look at the addresses and see if they are either coming from an SMSC
or going to an SMSC.

The `moreMessagesToSend` flag was implemented in version 2, so it exist
only for version 2 and version 3.

Ok, to recap, what do we have now

- `Begin`, `Continue`, `End`, `Abort` messages.
- Dialogue handshake in the first request/response messages sent.
- `MT-Forward-SM`, `MO-Forward-SM`, `Forward-SM`
- Involved parties: Mobile phones, *MSC* and *SMSC*

_Wait we are missing something. I've only covered 2G,3G.._

# What about 4G/LTE and beyond (5G)?

_Ouch._

*LTE* networks does not use any of the *M3UA*, *SCCP*, *TCAP*, *MAP*
protocols. In *LTE* networks the main message type is *Diameter* which
doesn't contain fragmentation and can contain larger
messages. Everything is sent in one request and every request is
answered with a response. *Diameter* could use either *TCP* or *SCTP* as
transport layer.

To make *SMSes* work on *LTE* networks a new interface *SGs* was
invented which translates *SS7* messages to *Diameter* messages.  This
interface is in most cases used by the *MSC* to translate the messages
to *Diameter* and forward it to the *MME* (Mobility Management Entity,
similar to *SGSN* but in the *LTE* network). The *MME* then forwards
it to the *UE* (user equipment, same as mobile subscriber or *MS* in
*GSM*/*GPRS* networks).

For 5G the *SMSC* is called *SMSF*; The Centre becomes a Function. The
signalling will be based on *HTTP2*/*JSON* ontop of *TCP*. The *SMSF*
will still need to support both *MAP* and *Diameter*.

Relevant xkcd:
<div class="post-images">
    <a href="https://xkcd.com/2365/"><img src="https://imgs.xkcd.com/comics/messaging_systems.png" /></a>
</div>

# Headache

Hopefully you did not get a (too severe) headache by reading this
post.

I've spared you with a lot of details on the lower level of
protocols. There are loads of implementation details that must match
the specifications, otherwise you will get all kinds of Aborts and
possibly even dropped traffic.
For instance we learned that we accidentally sent dialogue portions in
more than the first response back, which seemed to work at first
glance; at closer inspection we found out that some messages were
dropped because the length of the packet became larger in size than an
allowed value. We could still send them, but the other side was not
able to receive them.

Remember: Telco is old and complex. However, it should still function
with different setups and on different hardware, vendors and with
environment.

Fun-fact: Sometimes a boolean value is not just encoded as a 1
or 0. To save bandwith telco decided that you could also just define
it as a `NULL OPTIONAL` meaning that if it is defined (but lacks a
value), then it is considered true.  if it is not defined then it is
considered false. This is the case for the `moreMessagesToSend` flag.

Hope you enjoy the reading as much as I enjoy digging into these
protocols!