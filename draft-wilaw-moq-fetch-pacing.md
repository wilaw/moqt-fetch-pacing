---
title: "MOQT FETCH Pacing"
category: std

docname: draft-wilaw-moq-fetch-pacing-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: "Web and Internet Transport"
workgroup: "Media Over QUIC"
keyword:
 - moqt
venue:
  group: "Media Over QUIC"
  type: "Working Group"
  mail: "moq@ietf.org"

author:
 -
    fullname: Will Law
    organization: Akamai
    email: wilaw@akamai.com
 -
    fullname: Suhas Nandakumar
    organization: Cisco
    email: snandaku@cisco.com
 -
    fullname: Gwendal Simon
    organization: Synamedia
    email: gsimon@synamedia.com

normative:
  MOQT: I-D.draft-ietf-moq-transport-19
  SCONE: I-D.draft-ietf-scone-protocol-04

informative:
  SAMMY:
    title: "Sammy: Smoothing video traffic to be a friendly internet neighbor"
    author:
      - ins: "B. Spang"
        name: "Bruce Spang"
      - ins: "S. Kunamalla"
        name: "Shravya Kunamalla"
      - ins: "R. Teixeira"
        name: "Renata Teixeira"
      - ins: "T. Huang"
        name: "Te-Yuan Huang"
      - ins: "G. Armitage"
        name: "Grenville Armitage"
      - ins: "R. Johari"
        name: "Ramesh Johari"
      - ins: "N. McKeown"
        name: "Nick McKeown"
    date: "2023-09"
    seriesInfo:
      - name: "Proceedings of the ACM SIGCOMM 2023 Conference"
        value: "pp. 754-768"
      - name: "DOI"
        value: "10.1145/3603269.3604839"
...

--- abstract

This draft defines an extension to MOQT to enable the paced delivery of
content returned in response to a FETCH message.


--- middle

# Introduction

Standard congestion control algorithms attempt to download data as quickly
as possible. Because throughput between a relay and client is typically much
higher than the encoded rate of the content, data retrieved via a FETCH message
in MOQT can arrive in bursts far exceeding what is needed for playback. This
burstiness contributes to buffer bloat and packet loss on the intermediary network.
A better goal for a delivery system is not to download as fast as possible, but to
download as fast as needed. In the context of Adaptive Bitrate Switching (ABR),
"as needed" means a rate that avoids triggering unnecessary downward bitrate switches
while still delivering enough data to allow the client to switch up when appropriate.
Delivering data at this pace, rather than as fast as possible, has been shown to
improve throughput and reduce retransmissions and Round-Trip-Time (RTT) ({{SAMMY}}).

This draft defines a mechanism for a client to negotiate support for a pacing extension
during SETUP, and to activate that behavior via a new parameter accompanying a FETCH message.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# FETCH PACING SETUP Extension {#fetch-pacing-setup-extension}
This draft defines a new MOQT extension, FETCH PACING, negotiated per {{MOQT}}
Section 3.2. Support for the extension is advertised via a new Boolean Setup Option, FETCH_PACING_SUPPORTED,
included in the SETUP message. An endpoint that implements this specification MUST include
FETCH_PACING_SUPPORTED in its SETUP message; an endpoint that omits it MUST be treated as not
supporting the extension. Once both peers have sent and received SETUP, each endpoint determines
whether FETCH Pacing is part of the negotiated feature set: the extension is considered active for
the session only if both the client and the peer serving the FETCH included FETCH_PACING_SUPPORTED.

# PACING_RATE Message Parameter {#pacing-rate-message-parameter}
Once negotiated, a client activates pacing on a FETCH message by including a new Message Parameter,
PACING_RATE. The value of PACING_RATE is a varint as defined by {{MOQT}} Sect 1.4.1. Variable-Length Integers.
This rate signal is an integer in the range (0-126). A value outside of that range MUST result in
the publisher rejecting the FETCH with a REQUEST_ERROR message.

The rate signal is interpreted as defined in SCONE ({{SCONE}} Section 5.1). The rate follows a logarithmic
scale defined as:

Base rate (b_min) = 100 Kbps
Bitrate at value n = b_min * 10^(n/20)

where n is an integer between 0 and 126 represented by the value of the PACING_RATE parameter.

An endpoint that receives a FETCH message containing PACING_RATE from a client with which the extension
was negotiated MUST pace delivery of the requested Objects at approximately the indicated rate. The timebase
of this pacing SHOULD be at least the Group duration of the track being paced. Different relays will
offer slightly different behavior and clients SHOULD NOT depend on precise pacing for their application
success.

If the PACING_RATE parameter is absent, the endpoint MUST deliver Objects using its default,
unpaced behavior. A client MUST NOT send PACING_RATE unless FETCH Pacing was successfully negotiated in SETUP;
per Section 3.3 of {{MOQT}}, sending an unnegotiated Message Parameter risks session termination.

The PACING_RATE parameter does not persist across FETCH instances and MUST be repeated with each new FETCH message
in which pacing is required. Since FETCH provides no update mechanism, the pacing rate cannot be modified or removed
once set. A client wishing to modify the pacing rate MUST cancel their FETCH request and issue a new FETCH message
with the updated pacing value.

A relay MAY propagate a pacing parameter upstream only if it is assured that its downstream clients will FETCH
that track at that rate or lower.

## Bitrate examples

Table 1 lists some example values for rate signals and the corresponding bitrate for each.

| n  | Bitrate (Mbps) |
|----|----------------|
| 0  | 0.10           |
| 1  | 0.11           |
| 3  | 0.14           |
| 10 | 0.32           |
| 20 | 1.00           |
| 25 | 1.78           |
| 30 | 3.16           |
| 35 | 5.62           |
| 40 | 10.00          |
| 45 | 17.78          |
| 50 | 31.62          |
| 55 | 56.23          |
| 60 | 100.00         |


# Security Considerations

Pacing signals are client provided and therefore might be abused by malicious clients. These clients
can only send values in the range 0-126 as values outside this range will result in a REQUEST_ERROR,
so range-based attacks are limited.

If a client sets a pacing rate far below the encoded bitrate of the track, for example setting a 100kbps
limit on an hour of content encoded at 10Mbps, then there will be two consequences:
* An edge relay will pull the content to the edge with an upstream fetch, cache it and then release it
  slowly to the client. This can result in increased cache space pressure on the edge relay.
* The relay will spend more time delivering content than it otherwise would have, consuming additional
  resources.

These scenarios are similar to a client SUBSCRIBING to content with forward=0 and can be mitigated
by general protective measures. For example, a relay could terminate a FETCH that is taking too
long to drain with a REQUEST_ERROR.

Additionally, if the mechanism by which the relay paces the content is inefficient, then a client can
cause additional work for the relay by activating pacing on FETCH requests. Relays that cannot
offer efficient pacing might choose to not offer pacing during SETUP.

# IANA Considerations

This document requests that IANA add two new entries to registries defined
in {{MOQT}}.

## FETCH_PACING_SUPPORTED Setup Option

IANA is requested to add the following entry to the "Setup Options"
registry (Section 15.4 of {{MOQT}}):

| Type | Name                   | Specification  |
|------|------------------------|----------------|
| TBD1 | FETCH_PACING_SUPPORTED | This document  |

FETCH_PACING_SUPPORTED is a boolean Setup Option (see {{fetch-pacing-setup-extension}})
that an endpoint includes in its SETUP message to indicate support for the
FETCH Pacing extension defined in this document.

## PACING_RATE Message Parameter

IANA is requested to add the following entry to the "Message Parameters"
registry (Section 15.7 of {{MOQT}}):

| Parameter Type | Parameter Name | Specification  |
|----------------|----------------|----------------|
| TBD2           | PACING_RATE    | This document  |

PACING_RATE is a Message Parameter (see {{pacing-rate-message-parameter}}) that a
client includes in a FETCH message to activate pacing for that request,
once FETCH_PACING_SUPPORTED has been successfully negotiated per Section 3.2 of
{{MOQT}}.


--- back


