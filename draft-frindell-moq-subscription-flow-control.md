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
  RELIABLE-RESET: I-D.ietf-quic-reliable-stream-reset

informative:
  WebTransport: I-D.ietf-webtrans-http3

--- abstract

This document defines an extension to Media over QUIC Transport (MOQT) that
lets a subscriber limit the number of subgroup streams and the total bytes a
publisher may send for an individual subscription. It defines a Setup Option
to negotiate the extension, message parameters for advertising these limits,
messages for granting credit and signaling flow control state, and a
session error code.

--- middle

# Introduction

Media over QUIC Transport (MOQT) {{MOQT}} delivers a subscription's Objects
across one or more subgroup streams, but provides no way for a subscriber to
bound how many streams or bytes a publisher sends for that subscription.
Transport-layer flow control operates per stream and per session, so it
cannot express a limit spanning a subscription's streams.

This extension lets a subscriber limit the total subgroup streams
(MAX_SUB_STREAMS) and total bytes (MAX_SUB_BYTES) for a subscription, and grant
further credit with SUB_FLOW_CONTROL_UPDATE. Publishers signal that they are
blocked with SUB_STREAMS_BLOCKED and SUB_BYTES_BLOCKED, and report the final
size of reset subgroup streams with SUBGROUP_RESET. Violations terminate the
session with FLOW_CONTROL_EXCEEDED.

Support for this extension is negotiated during session establishment using
the SUBSCRIPTION_FLOW_CONTROL Setup Option ({{negotiation}}).

# Conventions and Definitions

{::boilerplate bcp14-tagged}

This document uses the terminology and wire format notation of {{MOQT}}. The
subscriber sets the limits and the publisher honors them, regardless of which
endpoint is the client or server.

# Extension Negotiation {#negotiation}

## SUBSCRIPTION_FLOW_CONTROL Setup Option {#setup-option}

An endpoint indicates support for this extension by including the
SUBSCRIPTION_FLOW_CONTROL Setup Option (Option Type 0x0A) in its SETUP message.
Its value is a single varint that is reserved; senders SHOULD set it to 0 and
receivers MUST ignore it.

The extension is negotiated when an endpoint has both sent and received this
option, per the extension negotiation described in {{MOQT}}. It applies to both
directions of the session.

An endpoint that offers this extension MUST support RESET_STREAM_AT
({{RELIABLE-RESET}}). When an endpoint negotiates this extension with a peer
that does not support RESET_STREAM_AT, it MUST close the session with a
`PROTOCOL_VIOLATION`.

# Flow Control Model {#model}

A subscriber sets a subscription's initial limits by including the MAX_SUB_STREAMS
and/or MAX_SUB_BYTES Parameters in the control message that establishes it:
SUBSCRIBE or SUBSCRIBE_TRACKS.  Subscriptions initiated by a PUBLISH that are not
in response to a SUBSCRIBE_TRACKS start with no flow control credit.
The subscriber grants additional credit with SUB_FLOW_CONTROL_UPDATE
({{message-sub-flow-control-update}}). The two limits are independent; a
subscription MAY use either, both, or neither. Credit is not granted with
REQUEST_UPDATE because it solicits a response, which is unnecessary.

The limits apply only to subgroup streams; Objects sent as datagrams are not
counted. Limits and the messages defined here are scoped to a single session
and are not forwarded. A relay honors its downstream subscriber's limits and
independently sets its own limits toward its upstream publisher.

Limits are per subscription and cumulative over its lifetime: the stream count
is the total number of subgroup streams opened, and the byte count is the total
bytes sent across them ({{byte-accounting}}). A publisher MUST NOT exceed a
limit in effect for a subscription. An endpoint that detects a violation MUST
close the session with `FLOW_CONTROL_EXCEEDED` ({{errors}}).

## Stream Sequence {#stream-sequence}

When the extension is negotiated, every SUBGROUP_HEADER includes a Stream
Sequence field:

~~~
SUBGROUP_HEADER {
  Type Flags (vi64),
  Track Alias (vi64),
  Group ID (vi64),
  [Subgroup ID (vi64),]
  [Publisher Priority (8),]
  Stream Sequence (vi64),
}
~~~
{: #moq-sub-flow-control-subgroup-header title="SUBGROUP_HEADER with Stream Sequence"}

Stream Sequence uniquely identifies a subgroup stream within its subscription.
The publisher sets it to 0 on the first subgroup stream it opens for a
subscription and increments it by 1 for each subsequent stream.

When a subscriber receives a SUBGROUP_HEADER with a Stream Sequence greater than
or equal to the stream limit, it MUST close the session with a
`FLOW_CONTROL_EXCEEDED`. When a subscriber detects a Stream Sequence it has
already received for the subscription, it MUST close the session with a
`PROTOCOL_VIOLATION`.

## Publisher Behavior When Blocked {#blocked}

When sending would exceed a limit, the publisher MUST NOT open a subgroup stream
beyond the stream limit and MUST NOT send bytes beyond the byte limit, even if
this means stopping in the middle of a stream. It retains the blocked Objects
until it receives additional credit, and SHOULD send SUB_STREAMS_BLOCKED or
SUB_BYTES_BLOCKED, as applicable.

Being blocked does not extend an Object's lifetime; the delivery timeouts of
{{MOQT}} (SUBGROUP_DELIVERY_TIMEOUT and OBJECT_DELIVERY_TIMEOUT) continue to
apply. A subgroup stream reset because of an expired timeout is accounted for as
described in {{byte-accounting}}.

If retaining blocked Objects would exceed its resource limits, a publisher MAY
terminate the subscription with PUBLISH_DONE and `TOO_FAR_BEHIND` ({{MOQT}}).

## Byte Accounting {#byte-accounting}

The byte count includes all bytes serialized on the subscription's subgroup
streams: the SUBGROUP_HEADER (including the stream type) and each Object's
header fields and payload. It excludes QUIC and WebTransport framing.

Accounting is at byte granularity. A publisher MAY send part of an Object and
stop at the byte limit, leaving the stream open and resuming when credit
arrives, subject to the delivery timeout and ordering rules of {{MOQT}}.

Each subgroup stream is charged exactly once. For a stream closed with a FIN,
the subscriber charges the bytes it received. When a publisher resets a stream,
it reports the bytes sent on it in the Final Size field of SUBGROUP_RESET
({{message-subgroup-reset}}), and the subscriber charges that value.

A publisher MUST reset subgroup streams using RESET_STREAM_AT with a
reliable_size that includes the SUBGROUP_HEADER, so the subscriber always
learns the stream's Stream Sequence ({{stream-sequence}}) and can match it to
the corresponding SUBGROUP_RESET.

On native QUIC, this Final Size duplicates that of RESET_STREAM
({{Section 19.4 of QUIC}}), but WebTransport ({{WebTransport}})
implementations do not necessarily expose it. If the subscriber can
independently determine the bytes sent on a stream (from RESET_STREAM or a FIN)
and that differs from the value reported or otherwise charged, it MUST close the
session with `PROTOCOL_VIOLATION`.

For example, with 100 bytes of credit and a 200-byte Object:

~~~
1. Publisher sends 100 bytes (SUBGROUP_HEADER, Object header,
   partial payload), reaching the limit, and SHOULD send
   SUB_BYTES_BLOCKED.
2. The Object's delivery timeout expires.
3. Publisher resets the stream and sends SUBGROUP_RESET with
   Final Size = 100.
4. Subscriber charges 100 bytes: limit reached, not exceeded.
~~~

# Message Parameters {#parameters}

This extension defines two Message Parameters ({{MOQT}}). Each MAY appear in the
SUBSCRIBE or SUBSCRIBE_TRACKS, where it sets the initial limit for the
subscription, or in a SUB_FLOW_CONTROL_UPDATE, where it is added to the current
limit.

A cumulative limit MUST NOT exceed 2^64-1. If a parameter is absent when the
subscription is established, that limit does not apply and a later
SUB_FLOW_CONTROL_UPDATE MUST NOT include it. An endpoint that receives a
SUB_FLOW_CONTROL_UPDATE violating either rule MUST close the session with
`PROTOCOL_VIOLATION`.

## MAX_SUB_STREAMS Parameter {#max-sub-streams}

MAX_SUB_STREAMS (Parameter Type 0x33) is a varint limiting the total number of
subgroup streams the publisher can open for the subscription.

## MAX_SUB_BYTES Parameter {#max-sub-bytes}

MAX_SUB_BYTES (Parameter Type 0x36) is a varint limiting the total bytes the
publisher can send across the subscription's subgroup streams
({{byte-accounting}}).

# Messages {#messages}

Each message below is sent on the subscription's request stream using MOQT
control message framing with a 16-bit Length ({{MOQT}}). None consumes a
Request ID or solicits a response.

## SUB_FLOW_CONTROL_UPDATE {#message-sub-flow-control-update}

A subscriber sends `SUB_FLOW_CONTROL_UPDATE` to grant additional credit. It MAY
be sent any time after the subscription is established, whether or not the
publisher is blocked, so a subscriber that intends to grant credit MUST keep the
send direction of the request stream open. It has no effect if received after
the subscription ends.

~~~
SUB_FLOW_CONTROL_UPDATE Message {
  Type (vi64) = 0x14,
  Length (16),
  Number of Parameters (vi64),
  Parameters (..) ...,
}
~~~
{: #moq-transport-sub-flow-control-update-format title="MOQT SUB_FLOW_CONTROL_UPDATE Message"}

* Parameters: MAX_SUB_STREAMS and/or MAX_SUB_BYTES, each granting additional
  credit. A message carrying neither has no effect but is not an error.

## SUBGROUP_RESET {#message-subgroup-reset}

A publisher sends `SUBGROUP_RESET` to report the final byte count of a reset
subgroup stream ({{byte-accounting}}). Once the extension is negotiated, a
publisher MAY send it for any subscription, whether or not limits are in use.

~~~
SUBGROUP_RESET Message {
  Type (vi64) = 0x1F,
  Length (16),
  Stream Sequence (vi64),
  Final Size (vi64),
}
~~~
{: #moq-transport-subgroup-reset-format title="MOQT SUBGROUP_RESET Message"}

* Stream Sequence: The Stream Sequence of the reset subgroup stream
  ({{stream-sequence}}).

* Final Size: The bytes sent on the stream before it was reset.

## SUB_STREAMS_BLOCKED {#message-sub-streams-blocked}

A publisher sends `SUB_STREAMS_BLOCKED` when it wants to open a subgroup stream
but MAX_SUB_STREAMS prevents it.

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

A publisher sends `SUB_BYTES_BLOCKED` when it has data to send but MAX_SUB_BYTES
prevents it.

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

FLOW_CONTROL_EXCEEDED (0x1C):
: A session termination error code indicating that the peer violated a
  subscription flow control limit (MAX_SUB_STREAMS or MAX_SUB_BYTES).

# Security Considerations

These limits let a subscriber bound the resources a publisher consumes per
subscription, complementing transport flow control. A subscriber SHOULD set
limits consistent with the resources it will devote to a subscription.

SUB_STREAMS_BLOCKED and SUB_BYTES_BLOCKED are advisory. A subscriber MUST NOT
rely on receiving them and MUST enforce limits regardless. A subscriber that
grants credit only in response to them could stall a publisher that does not
send them; granting credit proactively avoids this.

A misbehaving publisher could under-report Final Size in SUBGROUP_RESET to
evade MAX_SUB_BYTES. Where the subscriber can independently determine the bytes
sent, a discrepancy is a `PROTOCOL_VIOLATION` ({{byte-accounting}}). An
endpoint MAY additionally use transport-level flow control.

# IANA Considerations

This document registers entries in registries established by {{MOQT}}. All
codepoints are provisional pending working group adoption and can be reassigned
by IANA to avoid collisions.

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
registry. None is the first message on a stream.

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
the base Media over QUIC Transport protocol. The initial conversion of that
proposal into this extension draft was drafted with the assistance of Claude
Code.
