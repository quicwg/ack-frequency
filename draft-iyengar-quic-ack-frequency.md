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

normative:

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

  QUIC-TLS:
    title: "Using Transport Layer Security (TLS) to Secure QUIC"
    date: {DATE}
    seriesinfo:
      Internet-Draft: draft-ietf-quic-tls-latest
    author:
      -
        ins: M. Thomson
        name: Martin Thomson
        org: Mozilla
        role: editor
      -
        ins: S. Turner
        name: Sean Turner
        org: sn3rd
        role: editor

informative:

  QUIC-INVARIANTS:
    title: "Version-Independent Properties of QUIC"
    date: {DATE}
    seriesinfo:
      Internet-Draft: draft-ietf-quic-invariants-latest
    author:
      -
        ins: M. Thomson
        name: Martin Thomson
        org: Mozilla



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

This document introduces a mechanism for negotiating the frequency at which a
receiver sends acknowledgments in a QUIC connection.


## Terms and Definitions

The keywords "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 {{!RFC2119}} {{!RFC8174}}
when, and only when, they appear in all capitals, as shown here.

## Notational Conventions

Packet and frame diagrams in this document use the format described in Section
3.1 of {{?RFC2360}}, with the following additional conventions:

\[x\]:
: Indicates that x is optional

x (A):
: Indicates that x is A bits long

x (A/B/C) ...:
: Indicates that x is one of A, B, or C bits long

x (i) ...:
: Indicates that x uses the variable-length encoding defined in QUIC

x (*) ...:
: Indicates that x is variable-length


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
