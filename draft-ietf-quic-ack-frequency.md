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
  -
    ins: M. Kühlewind
    name: Mirja Kühlewind
    org: Ericsson
    email: mirja.kuehlewind@ericsson.com

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
acknowledgements. Delaying acknowledgements can impact connection
throughput, loss detection and congestion controller performance at a data
sender, and CPU utilization at both a data sender and a data receiver.

Reducing the frequency of acknowledgements can improve connection and
endpoint performance in the following ways:

- Sending UDP packets can be very CPU intensive on some platforms. Reducing
  the number of packets that only contain acknowledgements reduces the CPU
  consumed at a data receiver. Experience shows that this reduction can be
  critical for high bandwidth connections.

- Similarly, receiving and processing UDP packets can also be CPU intensive, and
  reducing acknowledgement frequency reduces this cost at a data sender.

- For asymmetric link technologies, such as DOCSIS, LTE, and satellite,
  connection throughput in the forward path can become constrained
  when the reverse path is filled by acknowledgment packets {{?RFC3449}}.
  When traversing such links, reducing the number of acknowledgments can achieve
  higher connection throughput, lower the impact on other flows or optimise the
  overall use of transmission resources {{Cus22}}.

- The rate of acknowledgment packets can reduce link efficiency, including
  transmission opportunities or battery life, as well as transmission
  opportunities available to other flows sharing the same link.


As discussed in {{implementation}} however, there can be undesirable consequences
to congestion control and loss recovery if a receiver unilitaerally reduces the
acknowledgment frequency. A sender's constraints on the acknowledgement
frequency need to be taken into account to maximize congestion controller and
loss recovery performance.

{{QUIC-TRANSPORT}} specifies a simple delayed acknowledgement mechanism that a
receiver can use: send an acknowledgement for every other packet, and for every
packet that is received out of order (Section 13.2.1 of {{QUIC-TRANSPORT}}).
This does not allow a sender to signal its preferences or constraints. This
extension provides a mechanism to solve this problem.

# Negotiating Extension Use {#nego}

Endpoints advertise their support of the extension described in this document by
sending the following transport parameter ({{Section 7.2 of QUIC-TRANSPORT}}):

min_ack_delay (0xff04de1a):

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
  connection error of type TRANSPORT_PARAMETER_ERROR. On receiving a valid value in
  this field, the endpoint MUST update its `max_ack_delay` to the value provided
  by the peer.

Reordering Threshold:

: A variable-length integer that indicates the maximum packet
  reordering before eliciting an immediate ACK, as specified in {#out-of-order}.
  If no ACK_FREQUENCY frames have been received, the endpoint immediately
  acknowledges any subsequent packets that are received out-of-order, as specified
  in {{Section 13.2 of QUIC-TRANSPORT}}, corresponding to a default value of 1.
  A value of 0 indicates out-of-order packets do not elicit an immediate ACK.

ACK_FREQUENCY frames are ack-eliciting. When an ACK_FREQUENCY frame is lost,
the sender is encouraged to send another ACK_FREQUENCY frame, unless an
ACK_FREQUENCY frame with a larger Sequence Number value has already been sent.
However, it is not forbidden to retransmit the lost frame (see Section 13.3 of
{{QUIC-TRANSPORT}}), because the receiver will ignore duplicate or out-of-order
ACK_FREQUENCY frames based on the Sequence Number.

An endpoint can send multiple ACK_FREQUENCY frames with different values within a
connection. A sending endpoint MUST send monotonically increasing values in the
Sequence Number field, since this field allows ACK_FREQUENCY frames to be processed
out of order. A receiving endpoint MUST ignore a received ACK_FREQUENCY frame if the
Sequence Number value in the frame is smaller than the largest currently processed value.

# IMMEDIATE_ACK Frame {#immediate-ack-frame}

A sender can use an ACK_FREQUENCY frame to reduce the number of acknowledgements
sent by a receiver, but doing so increases the likelihood that time-sensitive
feedback is delayed as well. For example, as described in {{loss}}, delaying
acknowledgements can increase the time it takes for a sender to detect packet
loss. Sending an IMMEDIATE_ACK frame can help mitigate this problem.

An IMMEDIATE_ACK frame can be useful in other situations as well. For example,
if a sender wants an immediate RTT measurement or if a sender wants to establish
receiver liveness as quickly as possible. PING frames
({{Section 19.2 of QUIC-TRANSPORT}}) are ack-eliciting, but if a PING frame is
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

## Response to long idle periods

It is important to receive timely feedback after long idle periods,
e.g. update stale RTT measurements. When no acknowledgement has been sent in
over one smoothed round trip time, receivers are encouraged to send an
acknowledgement soon after receiving an ack-eliciting packet. This is not an
issue specific to this document, but the mechanisms specified herein could
create excessive delays.

## Response to Out-of-Order Packets {#out-of-order}

As specified in {{Section 13.2.1 of QUIC-TRANSPORT}}, endpoints are expected to
send an acknowledgement immediately on receiving a reordered ack-eliciting
packet. This extension modifies that behavior when an ACK_FREQUENCY frame with
a Reordering Threshold value other than 1 has been received.

If the most recent ACK_FREQUENCY frame received from the peer has a `Reordering
Threshold` value of 0, the endpoint SHOULD NOT send an immediate
acknowledgement in response to packets received out of order, and instead
rely on the peer's `Ack-Eliciting Threshold` and `max_ack_delay` thresholds
for sending acknowledgements.

If the most recent ACK_FREQUENCY frame received from the peer has a `Reordering
Threshold` value larger than 1, the endpoint tests the amount of reordering
before deciding to send an acknowledgement. The specification uses the following
definitions:

Largest Unacked:
: The largest packet number among all received ack-eliciting packets.

Largest Acked:
: The Largest Acknowledged value sent in an ACK frame.

Largest Reported:
: The largest packet number that could be declared lost with the specified
  Reordering Threshold, which is Largest Acked - Reordering Threshold + 1.

Unreported Missing:
: Packets with packet numbers between the Largest Unacked and Largest Reported
  that have not yet been received.

An endpoint that receives an ACK_FREQUENCY frame with a non-zero Reordering
Threshold value SHOULD send an immediate ACK when the gap
between the smallest Unreported Missing packet and the Largest Unacked is greater
than or equal to the Reordering Threshold value. Sending this additional ACK will
reset the `max_ack_delay` timer and `Ack-Eliciting Threshold` counter (as any ACK
would do).

See {{examples}} for examples explaining this behavior. See {{set-threshold}}
for guidance on how to choose the reordering threshold value when sending
ACK_FREQUENCY frames.

### Examples {#examples}

When the reordering threshold is 1, any time a packet is received
and there is a missing packet, an immediate ACK is sent.

If the reordering theshold is 3 and ACKs are only sent due to reordering:

~~~
  Receive 1
  Receive 3 -> 2 Missing
  Receive 4 -> 2 Missing
  Receive 5 -> Send ACK because of 2
  Receive 8 -> 6,7 Missing
  Receive 9 -> Send ACK because of 6, 7 Missing
  Receive 10 -> Send ACK because of 7
~~~

If the reordering threshold is 5 and ACKs are only sent due to reordering:

~~~
  Receive 1
  Receive 3 -> 2 Missing
  Receive 5 -> 2 Missing, 4 Missing
  Receive 6 -> 2 Missing, 4 Missing
  Receive 7 -> Send ACK because of 2, 4 Missing
  Receive 8 -> 4 Missing
  Receive 9 -> Send ACK because of 4
~~~

## Setting the Reordering Threshold value {#set-threshold}

To ensure timely loss detection, it is optimal to send a Reordering
Threshold value of 1 less than the packet threshold used by the data sender for
loss detection. If the threshold is smaller, an ACK_FRAME is sent before the
packet can be declared lost based on the packet threshold. If the value is
larger, it causes unnecessary delays. ({{Section 18.2 of QUIC-RECOVERY}})
recommends a default packet threshold for loss detection of 3, equivalent to
a Reordering Threshold of 2.

## Expediting Explicit Congestion Notification (ECN) Signals {#congestion}

An endpoint SHOULD send an immediate acknowledgement when a packet marked
with the ECN Congestion Experienced (CE) {{?RFC3168}} codepoint in the IP
header is received and the previously received packet was not marked CE.

Doing this maintains the peer's response time to congestion events, while also
reducing the ACK rate compared to {{Section 13.2.1 of QUIC-TRANSPORT}} during
extreme congestion or when peers are using DCTCP {{?RFC8257}} or other
congestion controllers (e.g. {{?I-D.ietf-tsvwg-aqm-dualq-coupled}}) that mark
more frequently than classic ECN {{?RFC3168}}.

## Batch Processing of Packets {#batch}

To avoid sending multiple acknowledgements in rapid succession, an endpoint can
process all packets in a batch before determining whether to send an ACK frame
in response, as stated in {{Section 13.2.2 of QUIC-TRANSPORT}}.

# Computation of Probe Timeout Period

On sending an update to the peer's `max_ack_delay`, an endpoint can use this new
value in later computations of its Probe Timeout (PTO) period; see {{Section 5.2.1
of QUIC-RECOVERY}}.

Until the packet carrying this frame is acknowledged, the endpoint MUST use the
greater of the current `max_ack_delay` and the value that is in flight when
computing the PTO period. Doing so avoids spurious PTOs that can be caused by an
update that decreases the peer's `max_ack_delay`.

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
a packet loss or an ECN CE marking. Decreasing the acknowledgement frequency
can delay a sender's response to network congestion or cause it to underutilize
the available bandwidth.

To limit the consequences of reduced acknowledgement frequency, a sender
can cause a receiver to send an acknowledgement at least once per round trip
when there are ack-eliciting packets in flight.

A sender can accomplish this by setting the Request Max Ack
Delay value to no more than the estimated round trip time.
The sender can also improve feedback and robustness to
variation in the path RTT by setting the Ack-Eliciting Threshold
to a value no larger than the current congestion window. Alternatively,
a sender can accomplish this by sending an IMMEDIATE_ACK frame once each
round trip time, although if the packet containing an IMMEDIATE_ACK is lost,
detection of that loss will be delayed by the reordering threshold or requested
max ack delay.

When setting the Request Max Ack Delay as a function of RTT, it is usually
better to use the Smoothed RTT ({{Section 5.3 of QUIC-RECOVERY}}) or another
estimate of the typical round trip time, but not Min RTT. This avoids eliciting an
unnecessarily high number of acknowledgements when min_rtt is much smaller than
smoothed_rtt.

Note that the congestion window and the RTT change over the lifetime of a
connection and therefore might require sending frequent ACK_FREQUENCY frames to
ensure optimal performance.

It is possible that the RTT is smaller than the receiver's timer granularity,
as communicated via the 'min_ack_delay' transport parameter, preventing the
receiver from sending an acknowledgment every RTT in time unless packets are
acknowledged immediately.  In these cases, Reordering Threshold values other than 1
can can delay loss detection more than an RTT.

### Application-Limited Connections

A congestion controller that is limited by the congestion window relies upon receiving
acknowledgements to send additional data into the network.  An increase in
acknowledgement delay increases the delay in sending data, which can reduce the
achieved throughput.  Congestion window growth can also depend upon receiving
acknowledgements. This can be particularly significant in slow start
({{Section 7.3.1 of QUIC-RECOVERY}}), when delaying acknowledgements can delay
the increase in congestion window and can create larger packet bursts.

If the sender is application-limited, acknowledgements can be delayed
unnecessarily when entering idle periods. Therefore, if no further data is
buffered to be sent, a sender can send an IMMEDIATE_ACK frame with the last data
packet before an idle period to avoid waiting for the ack delay.

If there are no inflight packets, no acknowledgements will be received for at least
a round trip when sending resumes. The Max Ack Delay and Ack-Eliciting Threshold
values used by the receiver can further delay acknowledgements.  In this case, the
sender can include an IMMEDIATE_ACK or ACK_FREQUENCY frame in the first
Ack-Eliciting packet to avoid waiting for substantially more than a round trip
for an acknowledgement.

## Burst Mitigation

Receiving an acknowledgement can allow a sender to release new packets into the
network. If a sender is designed to rely on the timing of peer acknowledgments
("ACK clock"), delaying acknowledgments can cause undesirable bursts of data
into the network. In keeping with Section 7.7 of {{QUIC-RECOVERY}}, a sender
can either employ pacing or limit bursts to the initial congestion window.

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

A sender might use timers to detect loss of PMTU probe packets
({{Section 14 of QUIC-TRANSPORT}}). A sender MAY
bundle an IMMEDIATE_ACK frame with any PMTU probes to avoid triggering such
timers.

## Connection Migration {#migration}

To avoid additional delays to connection migration confirmation when using this
extension, a client can bundle an IMMEDIATE_ACK frame with the first non-probing
frame ({{Section 9.2 of QUIC-TRANSPORT}}) it sends or it can send only an
IMMEDIATE_ACK frame, which is a non-probing frame.

An endpoint's congestion controller and RTT estimator are reset upon
confirmation of migration (Section 9.4 of [QUIC-TRANSPORT]);
this changes the pattern of acknowledgements received after migration.

Therefore, an endpoint that has sent an ACK_FREQUENCY frame earlier in the
connection ought to send a new ACK_FREQUENCY frame upon confirmation of
connection migration with updated information, e.g. to consider the new RTT estimate.

# Security Considerations

An improperly configured or malicious data sender could cause a
data receiver to acknowledge more frequently than its available resources
permit. However, there are two limits that make such an attack largely
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
0xff04de1a                   | min_ack_delay     | {{nego}}
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
