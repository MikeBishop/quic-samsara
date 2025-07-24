---
title: "Samsara: Transport State Sharing and Recovery"
abbrev: "Samsara"
category: std

docname: draft-bishop-quic-samsara-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: "Web and Internet Transport"
workgroup: "QUIC"
keyword:
 - QUIC
 - congestion control
 - careful resume
venue:
  group: "QUIC"
  type: "Working Group"
  mail: "quic@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/quic/"
  github: "MikeBishop/quic-samsara"
  latest:

author:
 -
    fullname: Mike Bishop
    organization: Akamai Technologies
    email: mbishop@evequefou.be

normative:
  QUIC:
    RFC9000:

informative:

...

--- abstract

This draft defines a new transport frame which enables consenting endpoints
to share congestion control state about the network connection for various
purposes. It also allows an endpoint's own congestion control state to be echoed
back to it by a peer for consideration at the beginning of a future connection.

--- middle

# Introduction

All Internet transports are required to either use a Congestion Control (CC)
algorithm, or to constrain their rate of transmission {{!RFC8085}}. Most
congestion control algorithms take time to ramp up the sending rate, called the
"Slow-Start phase". This defines a time in which a sender intentionally uses
less capacity than might be available so as to avoid or limit overshoot of the
available capacity for the path. This is because any overshoot can have
detrimental effects both for the current flow and for any other flows with which
it shares network bottlenecks.

At the same time, using less capacity than is available necessarily limits
performance early in the connection. Careful Resume
{{?CAREFUL-RESUME=I-D.ietf-tsvwg-careful-resume}} is a mechanism whereby
remembered congestion control parameters can be validated as potentially
applicable to a new connection, probed, and ultimately used to grow the
congestion window more rapidly than slow-start would otherwise permit.

While clients often connect to a manageable number of servers and can retain
such state, servers typically service orders of magnitude more clients and
cannot feasibly retain such information. Further, servers are often deployed
with many instances and attempting to coordinate the sharing of this information
between them may prove impractical. Thus, for a server to implement Careful
Resume, some external means of recalling its previous state is useful.

This document specifies a mechanism which allows a QUIC {{!QUIC}} endpoint to
periodically export its congestion control state, optionally in an
integrity-protected manner. This exported state is sent to the peer in a
NETWORK_STATISTICS frame.

When establishing a subsequent connection, an endpoint with persistent storage
can include this data in a NETWORK_STATISTICS_RECALL frame in its 0-RTT or 1-RTT
packets, assisting its peer to recall the information necessary to perform
Careful Resume.

## Peer Visibility

The peer's viewpoint on a connection can be useful for debugging and as
additional information to be considered by congestion controllers and
application-layer protocols. Therefore, this extension deliberately does not
encrypt the data reported to the peer. Instead, the data is provided in
cleartext with an optional integrity tag.

If a server wishes to recall information about past connections without sharing
that data with the client, this information can already be encoded in address
validation tokens without requiring the cooperation of the client; see {{Section
8.1.3 of QUIC}}.

## Conventions and Definitions

{::boilerplate bcp14-tagged}

# Transport Parameter

Desire and willingness to receive the frames defined in this specification is
indicated by means of a QUIC transport parameter
(name=support_network_statistics, value=TBD). The support_network_statistics
transport parameter is an integer value (represented as a variable-length
integer) that encodes three one-bit flags:

CONSUME (0x01):

: This indicates that the sender is interested in receiving NETWORK_STATISTICS
frames for its own uses during the current connection, independent of its peer's
ability to reuse the data in the future.


CACHE (0x02):

: This indicates that the sender is willing to receive NETWORK_STATISTICS
frames and potentially return the contents in a NETWORK_STATISTICS_RECALL
frame on a subsequent connection.


CONSIDER (0x04):

: This indicates that the sender is willing to have values it may have
provided on a previous connection returned to it in a NETWORK_STATISTICS_RECALL
frame.


All other bits MUST be set to zero when sending and MUST be ignored on receipt.

The default for this parameter is 0, which indicates that the endpoint does not
support NETWORK_STATISTICS or NETWORK_STATISTICS_RECALL frames. A value other
than 0 indicates that the endpoint supports the indicated frame types and is
willing to receive such frames on this connection.

An endpoint MUST NOT send NETWORK_STATISTICS or NETWORK_STATISTICS_RECALL frames
until it has received the support_network_statistics transport parameter with a
non-zero value during the handshake (or during a previous handshake if 0-RTT is
used).

An endpoint MUST NOT send NETWORK_STATISTICS frames to a peer which did not set
the CONSUME or CACHE flags. An endpoint MUST NOT send NETWORK_STATISTICS_RECALL
frames to a peer which did not set the CONSIDER flag. An endpoint that receives
a frame for which it has not indicated support via the transport parameter MUST
terminate the connection with an error of type PROTOCOL_VIOLATION.

# Network Statistics

TODO - Populate with the list of values from CCTK spec

# Frame Types

## NETWORK_STATISTICS Frames

NETWORK_STATISTICS frames (types 0xTBD1) provide a list of Network Statistics
values which the sender chooses to share about the state of the network
connection from its viewpoint.

~~~
NETWORK_STATISTICS Frame {
  Type (i) = 0xTBD1,
  Protected Count (i),
  Protected Network Statistics (..) ...,
  [Integrity Tag (TODO)],
  Unprotected Count (i),
  Unprotected Network Statistics (..) ...,
}
~~~

NETWORK_STATISTICS frames contain the following fields:

Protected Count:

: A variable-length integer representing the number of Network Statistics in the
Protected Network Statistics field.

Protected Network Statistics:

: A sequence of Network Statistics objects whose length is given by Protected
Count.

Integrity Tag:

: A TODO-byte integrity tag, calculated as described in {{integrity-tag}}. This
field is absent if Protected Count is zero.

Unprotected Count:

: A variable-length integer representing the number of Network Statistics in the
Unprotected Network Statistics field.

Unprotected Network Statistics:

: A sequence of Network Statistics objects whose length is given by Unprotected
Count.

NETWORK_STATISTICS frames are not retransmittable, though a loss event might
trigger the generation of a new NETWORK_STATISTICS frame; see
{{sending-network-statistics}}.

NETWORK_STATISTICS frames can be sent at any point in the connection after 0-RTT
or 1-RTT keys have been established, though useful data will likely not be
available until at least one round-trip has occurred. If a NETWORK_STATISTICS
frame is received in an Initial or Handshake packet, it MUST be treated as a
connection error of type PROTOCOL_VIOLATION.


### Sending Network Statistics

If an endpoint wishes to receive NETWORK_STATISTICS_RECALL frames on
future connections with the peer and the peer has set the CACHE flag, the
endpoint MAY send NETWORK_STATISTICS frames containing the values it
wishes to recall in future connections in the Protected Network Statistics
field. It MAY send additional NETWORK_STATISTICS frames when these values have
changed significantly and it wishes to update the stored values, or when a
previous NETWORK_STATISTICS frame is declared lost.

If the peer has set the CONSUME flag, an endpoint SHOULD send
NETWORK_STATISTICS frames periodically throughout the connection's lifetime.
However, an implementation SHOULD NOT send additional NETWORK_STATISTICS frames
if the connection has been idle since the last such frame was sent.

In addition to any values the endpoint placed in Protected Network Statistics,
the endpoint includes such other values as it is willing to provide in the
Unprotected Network Statistics field.

If an endpoint sends multiple NETWORK_STATISTICS frames, it is not required to
include the same set of network statistics in each frame. For example, some
statistics are more useful sent at a regular frequency, while others only need
to be sent if they have changed significantly from the last value known to have
been received.

### Integrity Tag

The integrity tag is calculated over the Protected Count and Protected Network
Statistics field by TODO Magic Crypto Fairy Dust.

It incorporates a key known only to the sender. This enables the sender to
verify that the values it provided have not been tampered with when they are
returned in a NETWORK_STATISTICS_RECALL frame, and that the values are recent
enough that it is still willing to use them.

## NETWORK_STATISTICS_RECALL Frames

NETWORK_STATISTICS_RECALL frames (type 0xTBD2) contain a list of Network
Statistics values which the sender received from the recipient during a previous
connection.

This frame SHOULD be sent as early as possible in the connection once 0-RTT or
1-RTT keys are available. While the frame MAY be sent at any point in the
connection, if it arrives after the recipient has exited slow-start the values
it contains will likely not be useful.

~~~
NETWORK_STATISTICS_RECALL Frame {
  Type (i) = 0xTBD2,
  Protected Count (i),
  Protected Network Statistics (..) ...,
  Integrity Tag (TODO),
}
~~~

NETWORK_STATISTICS_RECALL frames contain the following fields:

Protected Count:

: A variable-length integer representing the number of Network Statistics in the
Protected Network Statistics field, received in a NETWORK_STATISTICS frame from
the peer on a previous connection.

Protected Network Statistics:

: A sequence of Network Statistics objects whose length is given by Protected
Count, received in a NETWORK_STATISTICS frame from the peer on a previous
connection.

Integrity Tag:

: A TODO-byte integrity tag, received in a NETWORK_STATISTICS frame from
the peer on a previous connection.

If a NETWORK_STATISTICS_RECALL frame is received in an Initial or Handshake
packet, it MUST be treated as a connection error of type PROTOCOL_VIOLATION.

### Recalling Network Statistics

Upon receipt of a NETWORK_STATISTICS_RECALL frame, an endpoint computes the
expected Integrity Tag value as in {{integrity-tag}}. If the Integrity Tag value
does not match, the frame is ignored.

If the tag is acceptable, the endpoint takes the network statistics contained in
the frame and incorporates them into its congestion control strategy. For
example, it might exit the Reconnaissance Phase of Careful Resume
{{?CAREFUL-RESUME}}. The specifics of how this is done are outside the scope of
this extension.

Endpoints MUST NOT process more than one NETWORK_STATISTICS_RECALL frame on a
connection. Subsequent NETWORK_STATISTICS_RECALL frames MUST be ignored without
processing, regardless of whether the first frame was valid.

# Security Considerations

TODO Security


# IANA Considerations

This document has IANA actions which need to be written down.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
