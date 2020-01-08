---
title: "Negotiating Acknowledgment Frequency in QUIC"
abbrev: QUIC Ack Frequency
docname: draft-iyengar-quic-ack-frequency
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

This document introduces a mechanism for negotiating the frequency at which a
receiver sends acknowledgments in a QUIC connection.


--- note_Note_to_Readers

Discussion of this draft takes place on the QUIC working group mailing list
(quic@ietf.org), which is archived at
\<https://mailarchive.ietf.org/arch/search/?email_list=quic\>.

Working Group information can be found at \<https://github.com/quicwg\>; source
code and issues list for this draft can be found at
\<https://github.com/quicwg/base-drafts/labels/-transport\>.

--- middle

# Introduction

A QUIC receiver acknowledges received packets in ACK frames.  Deciding
acknowledgment frequency requires careful consideration.

- A sender relies on receipt of ACK frames to determine the amount of data in
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
  BBR, can perform well with very few ACKs per RTT.

- Severely asymmetric link technologies, such as DOCSIS, LTE, and satellite
  links, connection throughput in the data direction becomes constrained when
  the reverse bandwidth is filled by acknowledgment packets. Today, when a link
  is filled with TCP acknowledgments, middleboxes often proactively drop a
  fraction of these packets, since later acknowledgements include information
  covered in earlier ones.

- CPU cost of too many acknowledgement packets. (Say more)

- New sender startup mechanisms, such as paced chirping, and congestion
  controllers will need a way for the sender to increase the frequency of
  acknowledgements when fine-grained feedback is required. (This is probably
  covered by timestamps.)

{{QUIC-TRANSPORT}} currently leaves the decision of how often to send
acknowledgements entirely to receiver implementations.  This document introduces
a mechanism for negotiating the frequency between both endpoints.


## Terms and Definitions

The keywords "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 {{!RFC2119}} {{!RFC8174}}
when, and only when, they appear in all capitals, as shown here.


# Mechanism


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

Thanks to all the people.
