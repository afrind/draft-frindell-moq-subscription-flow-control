---
title: "Subscription Flow Control Extension for Media over QUIC Transport"
abbrev: "moq-sub-flow-control"
category: std

docname: draft-frindell-moq-subscription-flow-control-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
ipr: trust200902
area: "Web and Internet Transport"
workgroup: "Media Over QUIC"
keyword:
 - media over quic
 - flow control
venue:
  group: "Media Over QUIC"
  type: "Working Group"
  mail: "moq@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/moq/"
  github: "afrind/draft-frindell-moq-subscription-flow-control"
  latest: "https://afrind.github.io/draft-frindell-moq-subscription-flow-control/draft-frindell-moq-subscription-flow-control.html"

stand_alone: yes
smart_quotes: no
pi: [toc, sortrefs, symrefs, docmapping]

author:
  -
    ins: A. Frindell
    name: Alan Frindell
    organization: Meta
    email: afrind@meta.com
    role: editor

  -
    ins: I. Swett
    name: Ian Swett
    organization: Google
    email: ianswett@google.com
    role: editor

normative:
  MOQT: I-D.ietf-moq-transport
  QUIC: RFC9000

informative:
  WebTransport: I-D.ietf-webtrans-http3

--- abstract

This document defines an extension to Media over QUIC Transport (MOQT) that
allows a subscriber to limit the number of subgroup streams and the total
number of bytes a publisher may send for an individual subscription.
Support for the extension is negotiated using a Setup Option.

--- middle

# Introduction

Media over QUIC Transport (MOQT) {{MOQT}} delivers the Objects for a
subscription in subgroup streams and/or datagrams. The base protocol does not
provide a way for a subscriber to bound the number of subgroup streams a
publisher opens for a subscription, or the total number of bytes a publisher
sends for it. A subscriber that wishes to protect its resources must rely on
transport-layer flow control, which operates per stream and per session
rather than per subscription, and which cannot express a limit that
spans the multiple streams belonging to a single subscription.

This document defines the Subscription Flow Control extension. It lets a
subscriber advertise, at subscription time and thereafter, a limit on:

* the total number of subgroup streams a publisher may open for the
  subscription (MAX_SUB_STREAMS), and

* the total number of bytes a publisher may send across all subgroup streams
  of the subscription (MAX_SUB_BYTES).

The subscriber grants additional credit after the subscription is established
using the SUB_FLOW_CONTROL_UPDATE message, a unidirectional notification sent
on the subscription's control stream that does not consume a Request ID or
solicit a response, unlike REQUEST_UPDATE.

The extension also defines messages a publisher uses to signal that it has
reached a limit (SUB_STREAMS_BLOCKED and SUB_BYTES_BLOCKED) and a message a
publisher uses to report the final byte count of a subgroup stream it reset
(SUBGROUP_RESET), so that byte accounting remains accurate. A violation of a
negotiated limit terminates the session with the FLOW_CONTROL_EXCEEDED error
code.

Support for this extension is negotiated during session establishment using
the SUBSCRIPTION_FLOW_CONTROL Setup Option ({{negotiation}}), following the
extension negotiation procedure of {{MOQT}}.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

This document uses the terminology of {{MOQT}}, including Object, Subgroup,
subgroup stream, subscription, publisher, subscriber, request stream, Setup
Option, and Message Parameter.

In this document, "publisher" refers to the endpoint that sends Objects for a
subscription, and "subscriber" refers to the endpoint that receives them, as
defined in {{MOQT}}. The subscriber sets the flow control limits and the
publisher honors them, regardless of which endpoint is the client or the
server.

All wire formats use the notation defined in {{MOQT}}. Setup Options are
encoded as Key-Value-Pairs and Message Parameters use the Message Parameter
encoding, both as defined in {{MOQT}}.

# Extension Negotiation {#negotiation}

An endpoint indicates support for this extension by including the
SUBSCRIPTION_FLOW_CONTROL Setup Option in its SETUP message ({{MOQT}}).

## SUBSCRIPTION_FLOW_CONTROL Setup Option {#setup-option}

The SUBSCRIPTION_FLOW_CONTROL option (Option Type 0x0A) indicates that the
endpoint supports this extension. The Option Type is even, so its value is a
single varint. The value is reserved; senders SHOULD set it to 0 and receivers
MUST ignore it.

The extension is negotiated when an endpoint has both sent and received a
SUBSCRIPTION_FLOW_CONTROL option, as described in the extension negotiation
procedure of {{MOQT}}. It applies to both directions of the session.

# Flow Control Model {#model}

A subscriber sets a subscription's initial flow control limits by including the
MAX_SUB_STREAMS and/or MAX_SUB_BYTES parameters in the message that establishes
the subscription (SUBSCRIBE or PUBLISH_OK), and grants additional credit by
sending SUB_FLOW_CONTROL_UPDATE ({{message-sub-flow-control-update}}). The two
limits are independent; a subscription MAY use either, both, or neither.

Credit is granted with SUB_FLOW_CONTROL_UPDATE rather than REQUEST_UPDATE
because REQUEST_UPDATE solicits a REQUEST_OK or REQUEST_ERROR response, which is
unsuitable for frequent, fire-and-forget flow control grants ({{messages}}).

These limits apply only to Objects delivered on subgroup streams. Objects
delivered as datagrams are not counted and are not subject to MAX_SUB_STREAMS or
MAX_SUB_BYTES; an endpoint that needs to bound datagram resource use relies on
transport-level flow control.

Flow control limits, and all of the messages defined by this extension, are
scoped to a single session and are not forwarded end-to-end. When a relay is
present, it honors the limits set by its downstream subscriber (acting as the
publisher on that request stream) and independently sets its own limits toward
its upstream publisher (acting as the subscriber); the two are unrelated.

Each limit is scoped to a single subscription. A publisher MUST NOT exceed a
limit that is in effect for a subscription. Counting is cumulative over the
lifetime of the subscription:

* The stream count is the total number of subgroup streams the publisher has
  opened for the subscription.

* The byte count is the total number of bytes the publisher has sent across
  all subgroup streams for the subscription, as defined in
  {{byte-accounting}}.

An endpoint that detects a violation of a limit MUST close the session with
`FLOW_CONTROL_EXCEEDED` ({{errors}}). Publisher behavior when a limit is reached,
including signaling with SUB_STREAMS_BLOCKED and SUB_BYTES_BLOCKED, is described
in {{blocked}}.

## Publisher Behavior When Blocked {#blocked}

When a publisher has data for a subscription but sending it would exceed the
MAX_SUB_STREAMS or MAX_SUB_BYTES limit, it MUST NOT open a subgroup stream
beyond the stream limit and MUST NOT send bytes beyond the byte limit, including
stopping in the middle of a subgroup stream when the byte limit is reached. The
publisher retains the
affected Objects, deferring the opening of new subgroup streams and the
transmission of buffered bytes until the subscriber grants additional credit
with SUB_FLOW_CONTROL_UPDATE ({{message-sub-flow-control-update}}). The
publisher SHOULD send SUB_STREAMS_BLOCKED or SUB_BYTES_BLOCKED, as applicable,
when it becomes blocked.

Objects retained while blocked remain subject to the base protocol's delivery
guarantees; being blocked by flow control does not extend an Object's lifetime;
the delivery timeout mechanisms (SUBGROUP_DELIVERY_TIMEOUT and
OBJECT_DELIVERY_TIMEOUT) defined in {{MOQT}} continue to apply. An Object or
subgroup whose delivery timeout expires while the publisher is blocked is
handled as specified in {{MOQT}} even though the flow control limit prevented it
from being sent. A subgroup stream reset for this reason is accounted for as
described in {{byte-accounting}}.

A publisher is not required to buffer blocked data indefinitely. If retaining
blocked Objects would cause the publisher to exceed its resource limits, the
publisher MAY terminate the subscription using PUBLISH_DONE with
`TOO_FAR_BEHIND`, as defined in {{MOQT}}, rather than continuing to wait for
additional credit. This is the same mechanism the base protocol provides for a
subscriber that fails to consume Objects at a sufficient rate.

## Byte Accounting {#byte-accounting}

The byte count charged against MAX_SUB_BYTES includes all the bytes serialized
on subgroup streams for the subscription, comprising the SUBGROUP_HEADER (which
begins with the stream type) and each Object (including its header fields and
payload). It does not include QUIC or WebTransport transport framing.

Accounting is at byte granularity, not Object granularity. A publisher MAY send
part of an Object and stop when it reaches the byte limit; it is not required to
withhold an entire Object because the Object would not fit within the remaining
credit. A publisher that stops in the middle of an Object leaves the subgroup
stream open and resumes sending on it once additional credit arrives, subject to
the delivery timeout and ordering rules of {{MOQT}}.

For a subgroup stream that the publisher closes normally (with a FIN), the
subscriber charges the number of bytes it received on that stream. When a
publisher resets a subgroup stream before sending all of its data, it reports
the total number of bytes it sent on that stream in the Final Size field of the
SUBGROUP_RESET message ({{message-subgroup-reset}}), and the subscriber charges
Final Size for that stream. Accounting is cumulative: each SUBGROUP_RESET
accounts for one reset of a subgroup stream, and a stream is charged exactly
once, whether it ends with a FIN or a reset.

For native QUIC, the reported Final Size is redundant with the Final Size
carried in the QUIC RESET_STREAM frame ({{Section 19.4 of QUIC}}), which the
transport delivers to the receiver regardless of loss. SUBGROUP_RESET is needed
because a WebTransport ({{WebTransport}}) implementation does not necessarily
surface the reset stream's final size to the application. When the subscriber
can independently determine the number of bytes the publisher sent on the
stream (for example, from the QUIC RESET_STREAM Final Size, or from the bytes
it received on a stream closed with a FIN), and that number differs from the
value the publisher reports or the subscriber would otherwise charge, the
subscriber MUST close the session with `PROTOCOL_VIOLATION`.

Because a publisher does not maintain more than one open subgroup stream with
the same Group ID and Subgroup ID at a time ({{MOQT}}), the subscriber can
correlate each SUBGROUP_RESET with the subgroup stream it reset.

For example, with 100 bytes of remaining credit and a 200-byte Object to send:

~~~
Remaining credit: 100 bytes      Object to send: 200 bytes

1. Publisher sends 100 bytes on the subgroup stream (SUBGROUP_HEADER
   + Object header + as much payload as fits), reaching the limit.
2. Publisher SHOULD send SUB_BYTES_BLOCKED.
3. The Object's delivery timeout expires.
4. Publisher resets the subgroup stream and sends SUBGROUP_RESET
   with Final Size = 100.
5. Subscriber charges 100 bytes; the byte count now equals the
   limit -- reached but never exceeded.
~~~

# Message Parameters {#parameters}

This extension defines two message parameters. They follow the Message
Parameter rules of {{MOQT}} and use the Message Parameter encoding
(Type Delta followed by Value) defined there. Each MAY appear in a SUBSCRIBE or
PUBLISH_OK message that establishes a subscription, or in a
SUB_FLOW_CONTROL_UPDATE message ({{message-sub-flow-control-update}}) for that
subscription.

For both parameters, the value in SUBSCRIBE or PUBLISH_OK is the initial limit,
and the value in SUB_FLOW_CONTROL_UPDATE is additive, granting credit beyond the
current limit. The cumulative limit MUST NOT exceed 2^64-1; an endpoint that
receives a SUB_FLOW_CONTROL_UPDATE that would raise a limit above 2^64-1 MUST
close the session with `PROTOCOL_VIOLATION`. If a parameter is omitted from the
SUBSCRIBE or PUBLISH_OK that establishes a subscription, that limit does not
apply to the subscription, and a later SUB_FLOW_CONTROL_UPDATE MUST NOT include
it; an endpoint that receives a SUB_FLOW_CONTROL_UPDATE carrying a parameter
that was absent at establishment MUST close the session with
`PROTOCOL_VIOLATION`.

## MAX_SUB_STREAMS Parameter {#max-sub-streams}

The MAX_SUB_STREAMS parameter (Parameter Type 0x33) is a varint. It sets or
updates the limit on the total number of subgroup streams the publisher can open
for the subscription. Publisher behavior when this limit is reached is described
in {{blocked}}.

## MAX_SUB_BYTES Parameter {#max-sub-bytes}

The MAX_SUB_BYTES parameter (Parameter Type 0x36) is a varint. It sets or
updates the limit on the total number of bytes the publisher can send across all
subgroup streams for the subscription, counted as described in
{{byte-accounting}}. Publisher behavior when this limit is reached is described
in {{blocked}}.

# Messages {#messages}

This extension defines four messages. Each is sent on the subscription's
request stream and uses the MOQT control message framing with a 16-bit Length
field, as defined for control messages in {{MOQT}}. None of these messages
consumes a Request ID or solicits a response.

## SUB_FLOW_CONTROL_UPDATE {#message-sub-flow-control-update}

A subscriber sends a `SUB_FLOW_CONTROL_UPDATE` message on the subscription's
request stream to grant the publisher additional flow control credit. It
carries the MAX_SUB_STREAMS and/or MAX_SUB_BYTES parameters ({{parameters}}).
A subscriber MAY send it at any time after the subscription is established,
whether or not the publisher has signaled that it is blocked. A subscriber that
intends to grant credit therefore MUST keep the send direction of the request
stream open ({{MOQT}}). A SUB_FLOW_CONTROL_UPDATE received after the
subscription has ended has no effect.

~~~
SUB_FLOW_CONTROL_UPDATE Message {
  Type (vi64) = 0x14,
  Length (16),
  Number of Parameters (vi64),
  Parameters (..) ...,
}
~~~
{: #moq-transport-sub-flow-control-update-format title="MOQT SUB_FLOW_CONTROL_UPDATE Message"}

* Number of Parameters: The number of parameters that follow.

* Parameters: MAX_SUB_STREAMS ({{max-sub-streams}}) and/or MAX_SUB_BYTES
  ({{max-sub-bytes}}), each granting additional credit for the corresponding
  limit. A SUB_FLOW_CONTROL_UPDATE that carries neither parameter has no effect
  but is not an error.

## SUBGROUP_RESET {#message-subgroup-reset}

A publisher sends a `SUBGROUP_RESET` message on the subscription's request
stream to report the final byte count of a subgroup stream that was reset.
This allows the subscriber to accurately account for bytes consumed against the
MAX_SUB_BYTES limit ({{max-sub-bytes}}).

A publisher MAY send SUBGROUP_RESET for any subscription at its discretion,
regardless of whether flow control limits are in use, provided the extension
has been negotiated.

~~~
SUBGROUP_RESET Message {
  Type (vi64) = 0x1F,
  Length (16),
  Group ID (vi64),
  Subgroup ID (vi64),
  Final Size (vi64),
}
~~~
{: #moq-transport-subgroup-reset-format title="MOQT SUBGROUP_RESET Message"}

* Group ID: The Group ID of the reset subgroup stream.

* Subgroup ID: The Subgroup ID of the reset subgroup stream.

* Final Size: The total number of bytes sent on the subgroup stream before it
  was reset, counted as described in {{byte-accounting}}.

## SUB_STREAMS_BLOCKED {#message-sub-streams-blocked}

A publisher sends a `SUB_STREAMS_BLOCKED` message on the subscription's request
stream to signal that it wants to open a subgroup stream but is prevented from
doing so by the MAX_SUB_STREAMS limit ({{max-sub-streams}}).

~~~
SUB_STREAMS_BLOCKED Message {
  Type (vi64) = 0x12,
  Length (16),
  Maximum Streams (vi64),
}
~~~
{: #moq-transport-sub-streams-blocked-format title="MOQT SUB_STREAMS_BLOCKED Message"}

* Maximum Streams: The stream limit that was reached.

## SUB_BYTES_BLOCKED {#message-sub-bytes-blocked}

A publisher sends a `SUB_BYTES_BLOCKED` message on the subscription's request
stream to signal that it has data it wants to send but is prevented from
sending it by the MAX_SUB_BYTES limit ({{max-sub-bytes}}).

~~~
SUB_BYTES_BLOCKED Message {
  Type (vi64) = 0x13,
  Length (16),
  Maximum Bytes (vi64),
}
~~~
{: #moq-transport-sub-bytes-blocked-format title="MOQT SUB_BYTES_BLOCKED Message"}

* Maximum Bytes: The byte limit that was reached.

# Error Handling {#errors}

This extension defines the FLOW_CONTROL_EXCEEDED session termination error
code. An endpoint uses it to close the session when the peer violates a
subscription flow control limit, as described in {{model}}.

FLOW_CONTROL_EXCEEDED (0x1C):
: The peer violated a subscription flow control limit (MAX_SUB_STREAMS or
  MAX_SUB_BYTES).

# Security Considerations

The limits defined by this extension let a subscriber bound the resources a
publisher can consume on its behalf per subscription, which complements the
connection- and stream-level flow control provided by the underlying
transport. A subscriber SHOULD set limits consistent with the resources it is
willing to devote to a subscription.

The SUB_STREAMS_BLOCKED and SUB_BYTES_BLOCKED messages are advisory. A
subscriber MUST NOT rely on receiving them before a limit is reached, and MUST
be prepared to enforce a limit even if the publisher does not signal that it is
blocked. Conversely, a subscriber that grants additional credit only in
response to these messages could stall a publisher that does not send them;
implementations that wish to avoid this can grant credit proactively.

Because SUBGROUP_RESET reports a byte count that the subscriber charges against
MAX_SUB_BYTES, a misbehaving publisher could report fewer bytes than it actually
sent in order to evade the limit. To prevent this, whenever the subscriber can
independently determine the number of bytes sent on a stream (for example, from
the QUIC RESET_STREAM Final Size), any discrepancy from the charged value is a
`PROTOCOL_VIOLATION` ({{byte-accounting}}). An endpoint MAY additionally bound
resource use with transport-level flow control.

# IANA Considerations

This document registers entries in registries established by {{MOQT}}. All
requested codepoints are provisional pending working group adoption and can be
reassigned by IANA to avoid collisions with the base protocol or other
extensions.

## Setup Option {#iana-setup-option}

IANA is requested to add the following entry to the "MOQT Setup Options"
registry:

| Type | Name | Specification |
|-----:|:-----|:--------------|
| 0x0A | SUBSCRIPTION_FLOW_CONTROL | {{setup-option}} |

## Message Parameters {#iana-parameters}

IANA is requested to add the following entries to the "MOQT Message Parameters"
registry:

| Parameter Type | Parameter Name | Specification |
|---------------:|:---------------|:--------------|
| 0x33 | MAX_SUB_STREAMS | {{max-sub-streams}} |
| 0x36 | MAX_SUB_BYTES | {{max-sub-bytes}} |

## Message Types {#iana-message-types}

IANA is requested to add the following entries to the "MOQT Message Types"
registry. All four messages are sent on the request stream and none is the
first message on a stream.

| ID   | Messages | Stream |
|-----:|:---------|:-------|
| 0x12 | SUB_STREAMS_BLOCKED ({{message-sub-streams-blocked}}) | Request |
| 0x13 | SUB_BYTES_BLOCKED ({{message-sub-bytes-blocked}}) | Request |
| 0x14 | SUB_FLOW_CONTROL_UPDATE ({{message-sub-flow-control-update}}) | Request |
| 0x1F | SUBGROUP_RESET ({{message-subgroup-reset}}) | Request |

## Session Termination Error Code {#iana-error-code}

IANA is requested to add the following entry to the "MOQT Session Termination
Error Codes" registry:

| Name | Code | Specification |
|:-----|:----:|:--------------|
| FLOW_CONTROL_EXCEEDED | 0x1C | {{errors}} |

--- back

# Acknowledgments
{:numbered="false"}

This extension is derived from a proposal to add subscription flow control to
the base Media over QUIC Transport protocol.

The initial conversion of that proposal into this extension draft was drafted
with the assistance of Claude Code.
