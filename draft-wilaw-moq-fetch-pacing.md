---
title: "MOQT FETCH Pacing"
category: info

docname: draft-wilaw-moq-fetch-pacing-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
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
  MoQTransport: I-D.draft-ietf-moq-transport-19

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
download as fast as needed. In the context of ABR, "as needed" means a rate that
avoids triggering unnecessary downward bitrate switches while still delivering
enough data to allow the client to switch up when appropriate. Delivering data at
this pace, rather than as fast as possible, has been shown to improve throughput and
reduce retransmissions and RTT ({{SAMMY}}). This draft defines a mechanism for a
client to negotiate support for a pacing extension during SETUP, and to activate
that behavior via a new parameter accompanying a FETCH message.


# Conventions and Definitions

{::boilerplate bcp14-tagged}


# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
