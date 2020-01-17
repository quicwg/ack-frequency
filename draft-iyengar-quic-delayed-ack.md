---
title: "A Generalized Mechanism for Delaying Acknowledgements in QUIC"
abbrev: Delaying QUIC Acknowledgements
docname: draft-iyengar-quic-delayed-ack
date: {DATE}
category: std
ipr: trust200902
area: Transport
workgroup: QUIC

stand_alone: yes
pi: [toc, sortrefs, symrefs, docmapping]

author:
  -
    ins: J. Iyengar
    name: Jana Iyengar
    org: Fastly
    email: jri.ietf@gmail.com

  -
    ins: I. Swett
    name: Ian Swett
    org: Google
    email: ian.swett@google.com

normative:

  QUIC-TRANSPORT:
    title: "QUIC: A UDP-Based Multiplexed and Secure Transport"
    date: {DATE}
    seriesinfo:
      Internet-Draft: draft-ietf-quic-transport-latest
    author:
      -
        ins: J. Iyengar
        name: Jana Iyengar
        org: Fastly
        role: editor
      -
        ins: M. Thomson
        name: Martin Thomson
        org: Mozilla
        role: editor

  QUIC-RECOVERY:
    title: "QUIC Loss Detection and Congestion Control"
    date: {DATE}
    seriesinfo:
      Internet-Draft: draft-ietf-quic-recovery-latest
    author:
      -
        ins: J. Iyengar
        name: Jana Iyengar
        org: Fastly
        role: editor
      -
        ins: I. Swett
        name: Ian Swett
        org: Google
        role: editor


--- abstract

This document describes a QUIC extension for an endpoint to influence its peer's
delaying of acknowledgements.

--- note_Note_to_Readers

Discussion of this draft takes place on the QUIC working group mailing list
(quic@ietf.org), which is archived at
\<https://mailarchive.ietf.org/arch/search/?email_list=quic\>.

Working Group information can be found at \<https://github.com/quicwg\>; source
code and issues list for this draft can be found at
\<https://github.com/quicwg/base-drafts/labels/-transport\>.

--- middle

# Introduction

This document describes a QUIC extension for an endpoint to influence its peer's
delaying of acknowledgements.

## Terms and Definitions

The keywords "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 {{!RFC2119}} {{!RFC8174}}
when, and only when, they appear in all capitals, as shown here.

In the rest of this document, "sender" refers to a QUIC data sender (and
acknowledgement receiver). Similarly, "receiver" refers to a QUIC data receiver
(and acknowledgement sender).

An "acknowledgement packet" refers to a QUIC packet that contains only an ACK
frame.

This document uses terms, definitions, and notational conventions described in
Section XX of {{QUIC-TRANSPORT}}. Frame diagrams in this document use the format
described in Section XX of {{QUIC-TRANSPORT}}.

# Motivation

A receiver acknowledges received packets, but it can delay sending these
acknowledgements. The delaying of acknowledgements can impact connection
throughput, loss detection and congestion controller performance at a data
sender, and CPU utilization at both a data sender and a data receiver.

Reducing the frequency of acknowledgement packets can improve connection and
endpoint performance in the following ways:

- Sending UDP packets can be noticeably CPU intensive on some
  platforms. Reducing the number of packets that only contain acknowledgements
  can therefore reduce the amount of CPU consumed at a data receiver. Experience
  shows that this cost reduction can be significant for high bandwidth
  connections.

- Similarly, receiving UDP packets can also be CPU intensive, and reducing
  acknowledgement frequency reduces this cost at a data sender.

- Severely asymmetric link technologies, such as DOCSIS, LTE, and satellite
  links, connection throughput in the data direction becomes constrained when
  the reverse bandwidth is filled by acknowledgment packets. When traversing
  such links, reducing the number of acknowledgments allows connection
  throughput to scale much further.

Unfortunately, there are undesirable consequences to simply reducing the
acknowledgement frequency, especially to an arbitrary fixed value, as follows:

- A sender relies on receipt of acknowledgements to determine the amount of data in
  flight and to detect losses, see {{QUIC-RECOVERY}}. Consequently, how often a
  receiver sends acknowledgments dictates how long it takes for losses to be
  detected at the sender.

- Starting a connection up quickly without inducing much queue is important for
  latency reduction, for both short and long flows. The sender often needs
  frequent acknowledgments during this phase; see slow start and hystart.

- Congestion controllers that are purely window based and strictly adherent to
  packet conservation, such as the one defined in {{QUIC-RECOVERY}}, rely on
  receipt of acknowledgments to move the congestion window forward and release
  additional data.  Such controllers suffer performance penalties when
  acknowledgements are not sent frequently enough.  On the other hand, for
  long-running flows, congestion controllers that are not window-based, such as
  BBR, can perform well with very few acknowledgements per RTT.

- New sender startup mechanisms, such as paced chirping, and congestion
  controllers will need a way for the sender to increase the frequency of
  acknowledgements when fine-grained feedback is required.

{{QUIC-TRANSPORT}} currently specifies a simple delayed acknowledgement
mechanism that a receiver can use: send an acknowledgement for every other
packet, and for every packet (for a short while) when reordering is
observed. This simple mechanism does not allow a sender to signal its
constraints, which in turn limits what a receiver can do to delay
acknowledgements and reduce acknowledgement frequency. This extension provides a
mechanism to solve this problem.

# Negotiating Extension Use

Endpoints advertise their support of the extension described in this document by
sending the following transport parameter:

min_ack_delay (0xXXXX):

: A variable-length integer representing the minimum amount of time in
  microseconds by which the endpoint can delay an acknowledgement. Values of
  2^14 or greater are invalid.

An endpoint's min_ack_delay MUST NOT be greater than the its max_ack_delay.
Endpoints that support this extension MUST treat receipt of a min_ack_delay that
is greater than the received max_ack_delay as a connection error of type
PROTOCOL_VIOLATION. Note that while the endpoint's max_ack_delay transport
parameter is in milliseconds (Section 18.2 in {{QUIC-TRANSPORT}}), min_ack_delay
is specified in microseconds.


# ACK-FREQUENCY Frame

Delaying acknowledgements as much as possible reduces both work done by the
endpoints and network load. An endpoint's loss detection and congestion control
mechanisms however need to be tolerant of this delay at the peer. An endpoint
signals its tolerance to its peer using an ACK-FREQUENCY frame, shown below:

~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                      Packet Tolerance (i)                   ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Update Max Ack Delay (i)                 ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~

ACK-FREQUENCY frames have a type of 0xXX, and contain the following fields:

Packet Tolerance:

: A variable-length integer representing the maximum number of ack-eliciting
  packets after which the receiver sends an acknowledgement.

Update Max Ack Delay:

: A variable-length integer representing an update to the peer's `max_ack_delay`
  transport parameter (Section 18.2 in {{QUIC-TRANSPORT}}). The value of this
  field is in microseconds. Any value smaller than the min_ack_delay advertised
  by this endpoint is invalid, and MUST be treated as a connection error of type
  PROTOCOL_VIOLATION.

ACK-FREQUENCY frames are ack-eliciting. However, their loss does not require
retransmission.

An endpoint MUST NOT send an empty ACK-FREQUENCY frame. Endpoints MUST treat
receipt of an empty ACK-FREQUENCY frame as a connection error of type
PROTOCOL_VIOLATION.

An endpoint MAY send ACK-FREQUENCY frames multiple times during a connection and
with different values.

An endpoint will have committed a max_ack_delay value to the peer, which
specifies the maximum amount of time by which the endpoint will delay sending
acknowledgments. When the endpoint receives an ACK-FREQUENCY frame, it MUST
update this maximum time to the value proposed by the peer in the Update Max
Ack Delay field.

# Sending Acknowledgments

On receiving an ACK-FREQUENCY frame, an endpoint will have an updated maximum
delay for sending an acknowledgement, and a packet threshold as well. The
endpoint MUST send an acknowledgement when either of the two thresholds are
met. {{loss}} and {{batch}} section describes exceptions to this strategy.

An endpoint is expected to bundle acknowledgements when possible. Every time an
acknowledgement is sent, bundled or otherwise, all counters and timers related
to delaying of acknowledgments are reset.

## Expediting Loss and Congestion Signals {#loss}

To expedite loss detection, endpoints SHOULD send an acknowledgement immediately
on receiving an ack-eliciting packet that is out of order. Endpoints MAY
continue sending acknowledgements immediately on each subsequently received
packet, but they SHOULD return to using delay thresholds as specified above
within a period of 1/8 x RTT, unless more ack-eliciting packets are received out
of order.

Similarly, packets marked with the ECN Congestion Experienced (CE) codepoint in
the IP header SHOULD be acknowledged immediately, to reduce the peer's response
time to congestion events.

## Batch Processing of Packets {#batch}

For performance reasons, an endpoint can receive incoming packets from the
underlying platform in a batch of multiple packets. This batch can contain
enough packets to cause multiple acknowledgements to be sent.

To avoid sending multiple acknowledgements in rapid succession, an endpoint MAY
process all packets in a batch before determining whether a threshold has been
met and an acknowledgement is to be sent in response.


# Multiple ACK-FREQUENCY Frames

An endpoint can send multiple ACK-FREQUENCY frames, and each one of them can
have different values.

If a received ACK-FREQUENCY frame is the first one in this connection, the
endpoint MUST immediately record any values from the frame and start using
them. The endpoint MUST also record the packet number of the enclosing packet.

If a received ACK-FREQUENCY frame is not the first one in this connection, the
endpoint MUST check if this is a more recent ACK-FREQUENCY frame than any
previous ones, as follows:

- If the enclosing packet number is greater than the recorded one, the endpoint
  MUST immediately replace old recorded state with values received in this
  frame. The endpoint MUST also replace the value of the recorded packet number
  with that of the enclosing packet.

- If the enclosing packet number is not greater than the recorded one, the
  endpoint MUST ignore this ACK-FREQUENCY frame.


# Security Considerations
TBD.

# IANA Considerations {#iana}
TBD.

--- back

# Change Log

> **RFC Editor's Note:** Please remove this section prior to publication of a
> final version of this document.


# Acknowledgments
{:numbered="false"}

The following people directly contributed key ideas that shaped this draft:
Bob Briscoe, Kazuho Oku, Marten Seemann.
