---
title: "QUIC Acknowledgement Frequency"
docname: draft-ietf-quic-ack-frequency-latest
date: {DATE}
category: std
consensus: true
ipr: trust200902
area: Transport
workgroup: QUIC

stand_alone: yes

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
    email: ianswett@google.com

normative:

  QUIC-TRANSPORT:
    title: "QUIC: A UDP-Based Multiplexed and Secure Transport"
    date: 2021-05
    seriesinfo:
      RFC: 9000
      DOI: 10.17487/RFC9000
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
    date: 2021-05
    seriesinfo:
      RFC: 9002
      DOI: 10.17487/RFC9002
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

informative:

  Cus22:
    title: "Reducing the acknowledgement frequency in IETF QUIC"
    date: 2022-10
    seriesinfo:
      DOI: 10.1002/sat.1466
      name: IJSCN
    author:
      -
        name: A. Custura
        org: University of Aberdeen
      -
        name: R. Secchi
        org: University of Aberdeen
      -
        name: G. Fairhurst
        org: University of Aberdeen

--- abstract

This document describes a QUIC extension for an endpoint to control its peer's
delaying of acknowledgements.

--- note_Note_to_Readers

Discussion of this draft takes place on the QUIC working group mailing list
(quic@ietf.org), which is archived at
[](https://mailarchive.ietf.org/arch/search/?email_list=quic). Source
code and issues list for this draft can be found at
[](https://github.com/quicwg/ack-frequency).

Working Group information can be found at [](https://github.com/quicwg).

--- middle

# Introduction

This document describes a QUIC extension for an endpoint to control its peer's
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
{{Section 1.2 and Section 1.3 of QUIC-TRANSPORT}}.

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

- Similarly, receiving and processing UDP packets can also be CPU intensive, and
  reducing acknowledgement frequency reduces this cost at a data sender.

- For asymmetric link technologies, such as DOCSIS, LTE, and satellite,
  connection throughput in the forward path can become constrained
  when the reverse path is filled by acknowledgment packets {{?RFC3449}}.
  When traversing such links, reducing the number of acknowledgments can achieve
  higher connection throughput, lower the impact on other flows or optimise the
  overall use of transmission resources {{Cus22}}.

- The rate of acknowledgment packets can impact link efficiency, including
  transmission opportunities or battery life, as well as transmission
  opportunities available to other flows sharing the same link.


As discussed in {{implementation}} however, there can be undesirable consequences
to congestion control and loss recovery if a receiver uniltaerally reduces the
acknowledgment frequency. A sender's constraints on the acknowledgement
frequency need to be taken into account to maximize congestion controller and
loss recovery performance.

{{QUIC-TRANSPORT}} currently specifies a simple delayed acknowledgement
mechanism that a receiver can use: send an acknowledgement for every other
packet, and for every packet that is received out of order (Section
13.2.1 of {{QUIC-TRANSPORT}}). This
simple mechanism does not allow a sender to signal its constraints. This
extension provides a mechanism to solve this problem.

# Negotiating Extension Use {#nego}

Endpoints advertise their support of the extension described in this document by
sending the following transport parameter ({{Section 7.2 of QUIC-TRANSPORT}}):

min_ack_delay (0xff03de1a):

: A variable-length integer representing the minimum amount of time in
  microseconds by which the endpoint that is sending this value is able to
  delay an acknowledgement. This limit could be based on the receiver's clock
  or timer granularity. 'min_ack_delay' is used by the peer to avoid requesting
  too small a value in the Request Max Ack Delay field of the ACK_FREQUENCY
  frame.

An endpoint's min_ack_delay MUST NOT be greater than its max_ack_delay.
Endpoints that support this extension MUST treat receipt of a min_ack_delay that
is greater than the received max_ack_delay as a connection error of type
TRANSPORT_PARAMETER_ERROR. Note that while the endpoint's max_ack_delay
transport parameter is in milliseconds ({{Section 18.2 of QUIC-TRANSPORT}}),
min_ack_delay is specified in microseconds.

The min_ack_delay transport parameter is a unilateral indication of support for
receiving ACK_FREQUENCY frames.  If an endpoint sends the transport parameter,
the peer is allowed to send ACK_FREQUENCY frames independent of whether it also
sends the min_ack_delay transport parameter or not.

Receiving a min_ack_delay transport parameter indicates that the peer might send
ACK_FREQUENCY frames in the future. Until an ACK_FREQUENCY frame is received,
receiving this transport parameter does not cause the endpoint to
change its acknowledgement behavior.

Endpoints MUST NOT remember the value of the min_ack_delay transport parameter
they received for use in a subsequent connection. Consequently, ACK_FREQUENCY
frames cannot be sent in 0-RTT packets, as per {{Section 7.4.1 of QUIC-TRANSPORT}}.

This Transport Parameter is encoded as per {{Section 18 of QUIC-TRANSPORT}}.

# ACK_FREQUENCY Frame {#ack-frequency-frame}

Delaying acknowledgements as much as possible reduces both work done by the
endpoints and network load. An endpoint's loss detection and congestion control
mechanisms however need to be tolerant of this delay at the peer. An endpoint
signals the frequency it wants to receive ACK frames to its peer using an
ACK_FREQUENCY frame, shown below:

~~~
ACK_FREQUENCY Frame {
  Type (i) = 0xaf,
  Sequence Number (i),
  Ack-Eliciting Threshold (i),
  Request Max Ack Delay (i),
  Reordering Threshold (i)
}
~~~

Following the common frame format described in {{Section 12.4 of
QUIC-TRANSPORT}}, ACK_FREQUENCY frames have a type of 0xaf, and contain the
following fields:

Sequence Number:

: A variable-length integer representing the sequence number assigned to the
  ACK_FREQUENCY frame by the sender to allow receivers to ignore obsolete
  frames.

Ack-Eliciting Threshold:

: A variable-length integer representing the maximum number of ack-eliciting
  packets the recipient of this frame receives before sending an acknowledgment.
  A receiving endpoint SHOULD send at least one ACK frame when more than this
  number of ack-eliciting packets have been received. A value of 0 results in
  a receiver immediately acknowledging every ack-eliciting packet. By default, an
  endpoint sends an ACK frame for every other ack-eliciting packet, as specified in
  {{Section 13.2.2 of QUIC-TRANSPORT}}, which corresponds to a value of 1.

Request Max Ack Delay:

: A variable-length integer representing the value to which the endpoint
  requests the peer update its `max_ack_delay`
  ({{Section 18.2 of QUIC-TRANSPORT}}). The value of this field is in
  microseconds, unlike the 'max_ack_delay' transport parameter, which is in
  milliseconds. Sending a value smaller than the `min_ack_delay` advertised
  by the peer is invalid. Receipt of an invalid value MUST be treated as a
  connection error of type PROTOCOL_VIOLATION. On receiving a valid value in
  this field, the endpoint MUST update its `max_ack_delay` to the value provided
  by the peer.

Reordering Threshold:

: A variable-length integer that indicates how many
  out of order packets can arrive before eliciting an immediate ACK. If no
  ACK_FREQUENCY frames have been received, the endpoint immediately acknowledges
  any subsequent packets that are received out of order, as specified in
  {{Section 13.2 of QUIC-TRANSPORT}}, as such the default value is 1.
  A value of 0 indicates out-of-order packets do not elicit an immediate ACKs.

ACK_FREQUENCY frames are ack-eliciting. When an ACK_FREQUENCY frame is lost,
is encouraged to send an ACK_FREQUENCY frame, unless an ACK_FREQUENCY frame
with a larger Sequence Number value has already been sent. However, it is not
forbidden to retransmit the lost frame (see Section 13.3 of {{QUIC-TRANSPORT}),
as the receiver will ignore duplicate or out-of-order ACK_FREQUENCY frames
based on the Sequence Number.

An endpoint can send multiple ACK_FREQUENCY frames with different values within a
connection. A sending endpoint MUST send monotonically increasing values in the
Sequence Number field, since this field allows ACK_FREQUENCY frames to be processed
out of order. A receiving endpoint MUST ignore a received ACK_FREQUENCY frame if the
Sequence Number value in the frame is not greater than the largest processed thus far.

# IMMEDIATE_ACK Frame {#immediate-ack-frame}

A sender can use an ACK_FREQUENCY frame to reduce the number of acknowledgements
sent by a receiver, but doing so increases the chances that time-sensitive
feedback is delayed as well. For example, as described in {{loss}}, delaying
acknowledgements can increase the time it takes for a sender to detect packet
loss. The IMMEDIATE_ACK frame helps mitigate this problem.

An IMMEDIATE_ACK frame can be useful in other situations as well. For example,
if a sender wants an immediate RTT measurement or if a sender wants to establish
receiver liveness as quickly as possible. PING frames
({{Section 19.2 of QUIC-TRANSPORT}}) are ack-eliciting but if a PING frame is
sent without an IMMEDIATE_ACK frame, the receiver might not immediately send
an ACK based on its local ACK strategy.

By definition IMMEDIATE_ACK frames are ack-eliciting.
An endpoint SHOULD send a packet containing an ACK frame immediately upon
receiving an IMMEDIATE_ACK frame. An endpoint MAY delay sending an ACK frame
despite receiving an IMMEDIATE_ACK frame. For example, an endpoint might do this
if a large number of received packets contain an IMMEDIATE_ACK or if the
endpoint is under heavy load.

~~~
IMMEDIATE_ACK Frame {
  Type (i) = 0xac,
}
~~~

IMMEDIATE_ACK frames do not need to be retransmitted.

# Sending Acknowledgments {#sending}

Prior to receiving an ACK_FREQUENCY frame, endpoints send acknowledgements as
specified in {{Section 13.2.1 of QUIC-TRANSPORT}}.

On receiving an ACK_FREQUENCY frame and updating its `max_ack_delay`
and `Ack-Eliciting Threshold` values ({{ack-frequency-frame}}), the endpoint sends an
acknowledgement when one of the following conditions are met:

- Since the last acknowledgement was sent, the number of received ack-eliciting
  packets is greater than the `Ack-Eliciting Threshold`.

- Since the last acknowledgement was sent, `max_ack_delay` amount of time has
  passed.

{{out-of-order}}, {{congestion}}, and {{batch}} describe exceptions to this
strategy.

## Response to Out-of-Order Packets {#out-of-order}

As specified in {{Section 13.2.1 of QUIC-TRANSPORT}}, endpoints are expected to
send an acknowledgement immediately on receiving a reordered ack-eliciting
packet. This extension modifies this behavior.

In order to ensure timely loss detection, the Reordering Threshold provided
in the ACK_FREQUENCY frame SHOULD NOT be larger than the re-ordering threshold
used by the data sender for loss detection. ({{Section 18.2 of QUIC-RECOVERY}})
recommends a default packet threshold for loss detection of 3.

An endpoint, that receives an ACK_FREQUENCY frame with a Reordering
Threshold value other than 0x00, SHOULD immediately send an ACK frame
when the packet number of largest unacknowledged packet since
the last detected reordering event exceeds the Reordering Threshold.

If the most recent ACK_FREQUENCY frame received from the peer has a `Reordering
Threshold` value of 0x00, the endpoint does not make this exception. That
is, the endpoint SHOULD NOT send an immediate acknowledgement in response to
packets received out of order, and instead continues to use the peer's
`Ack-Eliciting Threshold` and `max_ack_delay` thresholds for sending
acknowledgements.

## Expediting Congestion Signals {#congestion}

An endpoint SHOULD send an immediate acknowledgement when a packet marked
with the ECN Congestion Experienced (CE) {{?RFC3168}} codepoint in the IP
header is received and the previously received packet was not marked CE.

Doing this maintains the peer's response time to congestion events, while also
reducing the ACK rate compared to {{Section 13.2.1 of QUIC-TRANSPORT}} during
extreme congestion or when peers are using DCTCP {{?RFC8257}} or other
congestion controllers (e.g. {{?I-D.ietf-tsvwg-aqm-dualq-coupled}}) that mark
more frequently than classic ECN {{?RFC3168}}.


## Batch Processing of Packets {#batch}

For performance reasons, an endpoint can receive incoming packets from the
underlying platform in a batch of multiple packets. This batch can contain
enough packets to cause multiple acknowledgements to be sent.

To avoid sending multiple acknowledgements in rapid succession, an endpoint can
process all packets in a batch before determining whether to send an ACK frame
in response, as stated in {{Section 13.2.2 of QUIC-TRANSPORT}}.

# Computation of Probe Timeout Period

On sending an update to the peer's `max_ack_delay`, an endpoint can use this new
value in later computations of its Probe Timeout (PTO) period; see {{Section 5.2.1
of QUIC-RECOVERY}}. The endpoint MUST however wait until the ACK_FREQUENCY
frame that carries this new value is acknowledged by the peer.

Until the frame is acknowledged, the endpoint MUST use the greater of the
current `max_ack_delay` and the value that is in flight when computing the PTO
period. Doing so avoids spurious PTOs that can be caused by an update that
increases the peer's `max_ack_delay`.

While it is expected that endpoints will have only one ACK_FREQUENCY frame in
flight at any given time, this extension does not prohibit having more than one
in flight. When using `max_ack_delay` for PTO computations, endpoints MUST use
the maximum of the current value and all those in flight.

When the number of in-flight ack-eliciting packets is larger than the
ACK-Eliciting Threshold, an endpoint can expect that the peer will not need to
wait for its `max_ack_delay` period before sending an acknowledgement. In such
cases, the endpoint MAY therefore exclude the peer's 'max_ack_delay' from its PTO
calculation.  When Ignore Order is enabled and loss causes the peer to not
receive enough packets to trigger an immediate acknowledgement, the receiver
will wait 'max_ack_delay', increasing the chances of a premature PTO.
Therefore, if Ignore Order is enabled, the PTO MUST be larger than the peer's
'max_ack_delay'.


# Determining Acknowledgement Frequency {#implementation}

This section provides some guidance on a sender's choice of acknowledgment
frequency and discusses some additional considerations. Implementers can select
an appropriate strategy to meet the needs of their applications and congestion
controllers.

## Congestion Control

A sender needs to be responsive to notifications of congestion, such as
a packet loss or an ECN CE marking. Also, window-based congestion controllers
that strictly adhere to packet conservation, such as the one defined in
{{QUIC-RECOVERY}}, rely on receipt of acknowledgments to send additional data into
the network, and will suffer degraded performance if acknowledgments are delayed
excessively.

To enable a sender to respond to potential network congestion, a sender SHOULD
cause a receiver to send an acknowledgement at least once per RTT if there are
unacknowledged ack-eliciting packets in flight. A sender can accomplish this by
sending an IMMEDIATE_ACK frame once per round-trip time (RTT), or it can set the
Ack-Eliciting Threshold and Request Max Ack Delay values to be no more than a
congestion window and an estimated RTT, respectively. If the sender is
application-limited, a large ack delay can delay acknowledgement information
unnecessarily when entering idle periods. Therefore, if no further data is buffered to be
sent, a sender can send an IMMEDIATE_ACK frame with the last data
packet before an idle period to avoid waiting for the ack delay.

## Burst Mitigation

Acknowledging more packets in a single ACK frame can cause larger bursts
of packets into the network. Senders MUST continue to follow Section 7.7
of {{QUIC-RECOVERY}} and either employ pacing or limit bursts to the
initial congestion window.

## Loss Detection and Timers {#loss}

Acknowledgements are fundamental to reliability in QUIC. Consequently,
delaying or reducing the frequency of acknowledgments can cause loss detection
at the sender to be delayed.

A QUIC sender detects loss using packet thresholds on receiving an
acknowledgement (Section 6.1.1 of {{QUIC-RECOVERY}}); delaying the
acknowledgement therefore delays this method of detecting losses.

Reducing acknowledgement frequency reduces the number of RTT samples that a
sender receives (Section 5 of {{QUIC-RECOVERY}}), making a sender's RTT estimate
less responsive to changes in the path's RTT. As a result, any mechanisms that
rely on an accurate RTT estimate, such as time-threshold loss detection (Section
6.1.2 of {{QUIC-RECOVERY}}) or Probe Timeout (Section 6.2 of {{QUIC-RECOVERY}}),
will be less responsive to changes in the path's RTT, resulting in either
delayed or unnecessary packet transmissions.

To limit these consequences of reduced acknowledgement frequency, a sender
SHOULD cause a receiver to send an acknowledgement at least once per RTT if
there are unacknowledged ack-eliciting packets in flight. A sender can
accomplish this by sending an IMMEDIATE_ACK frame once per round-trip time
(RTT), or it can set the Ack-Eliciting Threshold and Request Max Ack Delay
values to be no more than a congestion window and an estimated RTT,
respectively.

A sender might use timers to detect loss of PMTU probe packets
({{Section 14 of QUIC-TRANSPORT}}). A sender SHOULD
bundle an IMMEDIATE_ACK frame with any PTMU probes to avoid triggering such
timers.

## Connection Migration {#migration}

To avoid additional delays to connection migration confirmation when using this
extension, a client can bundle an IMMEDIATE_ACK frame with the first non-probing
frame ({{Section 9.2 of QUIC-TRANSPORT}}) it sends or it can send only an
IMMEDIATE_ACK frame, which is a non-probing frame.

An endpoint's congestion controller and RTT estimator are reset upon
confirmation of migration ({{Section 9.4 of QUIC-TRANSPORT}}), which can impact
the number of acknowledgements received after migration. An endpoint that has
sent an ACK_FREQUENCY frame earlier in the connection SHOULD update and send a
new ACK_FREQUENCY frame immediately upon confirmation of connection migration.


# Security Considerations

An improperly configured or malicious data sender could cause a
data receiver to acknowledge more frequently than its available resources
permits. However, there are two limits that make such an attack largely
inconsequential. First, the acknowledgement rate is bounded by the rate at which
data is received. Second, ACK_FREQUENCY and IMMEDIATE_ACK frames can only request
an increase in the acknowledgment rate, but cannot force it.

In general, with this extension, a sender cannot force a receiver to acknowledge
more frequently than the receiver considers safe based on its resource constraints.

# IANA Considerations {#iana}

This document defines a new transport parameter to advertise support of the
extension described in this document and two new frame types to registered
by IANQ in the respective "QUIC Protocol" registries under
[https://www.iana.org/assignments/quic/quic.xhtml](https://www.iana.org/assignments/quic/quic.xhtml).

The following entry in {{transport-parameters}} should be added to
the "QUIC Transport Parameters" registry under the "QUIC Protocol" heading.

Value                        | Parameter Name.   | Specification
-----------------------------|-------------------|-----------------
0xff03de1a                   | min_ack_delay.    | {{nego}}
{: #transport-parameters title="Addition to QUIC Transport Parameters Entries"}


The following frame types should be added to the "QUIC Frame Types"
registry under the "QUIC Protocol" heading.


Value      | Frame Name          | Specification
-----------|---------------------|-----------------
0xaf       | ACK_FREQUENCY       | {{ack-frequency-frame}}
0xac       | IMMEDIATE_ACK       | {{immediate-ack-frame}}
{: #frame-types title="Addition to QUIC Frame Types Entries"}

--- back

# Change Log

> **RFC Editor's Note:** Please remove this section prior to publication of a
> final version of this document.


# Acknowledgments
{:numbered="false"}

The following people directly contributed key ideas that shaped this draft:
Bob Briscoe, Kazuho Oku, Marten Seemann.
