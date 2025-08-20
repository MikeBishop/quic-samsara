---
title: "Exchanging Congestion Control Data in QUIC"
abbrev: "Congestion Data Frame"
category: std

docname: draft-yuan-quic-congestion-data-latest
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
    fullname: 袁靖昊
    asciiFullname: Junghao Yuan
    organization: Bytedance
    email: yuanjinghao@bytedance.com
 -
    fullname: 肖磊
    asciiFullname: Lei Xiao
    organization: Bytedance
    email: xiaolei.shawn@bytedance.com
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

One optimization approach is to use historical congestion information to
initialize the next congestion model, providing the congestion algorithm with
reliable input to help it exit the slow start phase. The most direct and
reliable information can be samples collected during the congestion algorithm,
such as the congestion window size and probe bandwidth.

While clients often connect to a manageable number of servers and can retain
such state, servers typically service orders of magnitude more clients and
cannot feasibly retain such information. Further, servers are often deployed
with many instances and attempting to coordinate the sharing of this information
between them may prove impractical. Thus, for a server to implement Careful
Resume, some external means of recalling its previous state is useful.

This document specifies a mechanism which allows a QUIC {{!QUIC}} endpoint to
periodically export its congestion control state, optionally in an
integrity-protected manner. This exported state is sent to the peer in a
CONGESTION_DATA frame.

When establishing a subsequent connection, an endpoint with persistent storage
can include this data in a CONGESTION_DATA_RECALL frame in its 0-RTT or 1-RTT
packets, assisting its peer to recall the information necessary to perform
Careful Resume.

This data may also be useful for application-layer purposes, such as
adaptive-bit-rate video. The exported information is readable by the peer and
can be exposed to the application through local interfaces.

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

This document also uses terminology defined in {{!QUIC}} and
{{!QUIC-TLS=RFC9001}}, in particular the frame layout notation from {{Section
1.3 of QUIC}}.

# Transport Parameter

Desire and willingness to receive the frames defined in this specification is
indicated by means of a QUIC transport parameter
(name=support_congestion_data, value=TBD). The support_congestion_data
transport parameter is an integer value (represented as a variable-length
integer) that encodes three one-bit flags:

CONSUME (0x01):

: This indicates that the sender is interested in receiving CONGESTION_DATA
frames for its own uses during the current connection, independent of its peer's
ability to reuse the data in the future.


CACHE (0x02):

: This indicates that the sender is willing to receive CONGESTION_DATA
frames and potentially return the contents in a CONGESTION_DATA_RECALL
frame on a subsequent connection.


CONSIDER (0x04):

: This indicates that the sender is willing to have values it may have
provided on a previous connection returned to it in a CONGESTION_DATA_RECALL
frame.


All other bits MUST be set to zero when sending and MUST be ignored on receipt.

The default for this parameter is 0, which indicates that the endpoint does not
support CONGESTION_DATA or CONGESTION_DATA_RECALL frames. A value other
than 0 indicates that the endpoint supports the indicated frame types and is
willing to receive such frames on this connection.

An endpoint MUST NOT send CONGESTION_DATA or CONGESTION_DATA_RECALL frames
until it has received the support_congestion_data transport parameter with a
non-zero value during the handshake (or during a previous handshake if 0-RTT is
used).

An endpoint MUST NOT send CONGESTION_DATA frames to a peer which did not set
the CONSUME or CACHE flags. An endpoint MUST NOT send CONGESTION_DATA_RECALL
frames to a peer which did not set the CONSIDER flag. An endpoint that receives
a frame for which it has not indicated support via the transport parameter MUST
terminate the connection with an error of type PROTOCOL_VIOLATION.

# Network Statistics

Each network statistic is structured as a TLV:

~~~
Network Statistic structure {
  Type (i),
  Length (i),
  Value (..),
}
~~~

This structure includes the following fields:

Type:

: Indicates the statistic being offered, encoded as a variable-length integer.

Length:

: The length of the Value field in bytes, encoded as a variable-length integer.

Value:

: A Type-specific value carrying the payload of the indicated statistic.

This specification defines a number of initial statistics, but additional
statistics can be added by registering a value in the appropriate registry (see
{{iana-considerations}}). An implementation MUST ignore any statistics it cannot
understand, but MAY decline to return protected statistics to a peer if it
cannot verify that it is willing to share the contained information.

## Timestamp

The Timestamp statistic (0xc8) indicates the time at which the sender generated
this frame. This can be used on future connections to determine whether the
recalled statistics are recent enough to be useful.


## Network Type

The Network Type statistic (0xcc) indicate's the sender's understanding of its
network access medium, encoded as a single byte value. Note that this is purely
advisory, since applications will only be aware of the local network at best.

The defined values are:

+------+---------------------+
| 0x00 | Reserved            |
| 0x01 | Wire/Ethernet       |
| 0x02 | Reserved            |
| 0x03 | WLAN                |
| 0x04 | 2G Mobile           |
| 0x05 | 3G Mobile           |
| 0x06 | 4G Mobile           |
| 0x07 | 5G Mobile           |
+------+---------------------+

## Path Tuple

The Path Tuple statistic (0xca) encodes an identifier of the path on which these
statistics were generated. Knowing the connection addresses as seen from the
peer's perspective can be useful for a number of scenarios (e.g.
{{?STUN=RFC5389}}), and the similarity of the previous endpoints to those of a
new connection will be a consideration in deciding whether returned statistics
might be applicable to a new connection.

The structure of the value is:

~~~
Path Endpoint structure {
  Type (i) = 0x202,
  Length (i) = 12/36,
  Local IP (4/16),
  Local Port (2),
  Remote IP (4/16),
  Remote Port (2),
}
~~~

It contains the following fields:

Local IP:

: The sender's IP address on the associated transport path, encoded as either 4
or 16 bytes depending on IP version.

Local Port:

: The sender's port number on the associate transport path, encoded as a
two-byte integer.

Remote IP:

: The receiver's IP address as observed by the sender on the associated
transport path, encoded as either 4 or 16 bytes depending on IP version.

Remote Port:

: The receiver's port number as observed by the sender on the associated
transport path

The IP version being used can be inferred from the length of the payload.

## Slow Start Status

The Slow Start Status (0xcb) statistic indicates whether the sender's congestion
controller is in the Slow Start phase. The value is a single byte, set to 0x00
if the sender is not in Slow Start and 0x01 if the sender is in Slow Start.

## Maximum Congestion Window

The Maximum Congestion Window statistic (0xcd) indicates the maximum congestion
window (CWD) sampled within the observation period. This value is measured in
bytes and is encoded as a variable-length integer. The congestion window is a key
metric in congestion control algorithms, as it represents the amount of
unacknowledged data that a sender can have in flight on the network. A larger
congestion window generally allows for a higher sending rate.

## Maximum In-Flight Data

The Maximum In-Flight Data statistic (0xce) indicates the maximum in-flight data
sampled within the observation period. It is encoded as a variable-length
integer, measured in bytes, and represents the highest number of bytes sent by
the sender that have not yet been acknowledged by the receiver during the
measurement period. This metric provides insight into the actual amount of data
in transit at any given time, which can be useful for diagnosing network
performance issues.

## Smoothed RTT

The Smoothed RTT statistic (0xcf) indicates the most recent smoothed Round-Trip
Time (RTT) within the observation period. The value is encoded as a
variable-length integer measured in milliseconds. This value is calculated as
defined in {{!RFC9002}}. RTT is a key metric for congestion control, estimating
the time it takes for a packet to travel from the sender to the receiver and
back. The smoothed RTT calculation accounts for both the latest RTT and a
historical average, helping to dampen the effect of short-term network
fluctuations.

## Minimum RTT

The Minimum RTT statistic (0xd0) indicates the minimum RTT sampled within the
observation period. It is encoded as a variable-length integer, measured in
milliseconds. This metric provides a baseline for the best-case network latency
observed during the measurement period. A low minimum RTT can indicate a stable
and efficient network path, while a high one might suggest persistent latency
issues.

## RTT Variance

The RTT Variance statistic (0xd1) indicates the most recent RTT deviation within
the observation period. It is encoded as a variable-length integer, measured in
milliseconds, and is calculated as defined in {{!RFC9002}}. This metric
quantifies the variability of the RTT, providing insight into network jitter and
stability. A low RTT variance suggests a consistent network path, while a high
value indicates significant fluctuations in network latency.

## Latest Bandwidth

The Latest Bandwidth statistic (0xd2) indicates the current throughput of the
connection. This value is encoded as a variable-length integer measured in
kilobits per second (kbps). This metric represents the instantaneous sending
capacity as perceived by the sender and is a crucial input for congestion
control algorithms.

## Maximum Bandwidth

The Maximum Bandwidth statistic (0xd3) indicates the maximum bandwidth sampled
within the observation frame period. It is encoded as a variable-length integer
measured in kilobits per second (kbps). This metric provides a view of the peak
network capacity observed during the measurement period, which can be useful for
understanding the best possible performance on the current network path.

## Throughput

The Throughput statistic (0xd4) indicates the valid throughput for data
(excluding retransmissions) within the observation period. It is encoded as a
variable-length integer measured in kilobits per second (kbps). This metric is a
measure of the effective data rate delivered to the receiver's application
layer.

This calculation isolates the useful data rate, providing a more accurate
measure of application-level performance than the raw sending rate, which
includes retransmitted data.

## Send Rate

The Send Rate statistic (0xd5) indicates the sending rate for all data,
including retransmissions, within the observation period. It is encoded as a
variable-length integer, measured in kilobits per second (kbps). This metric
provides a measure of the total data rate at which the sender is transmitting
data.

This metric is useful for understanding the sender's total load on the network.

## Receive Rate

The Receive Rate statistic (0xd6) indicates the receiving rate within the
observation period. It is encoded as a variable-length integer measured in
kilobits per second (kbps). This metric measures the total rate at which the
receiver is acknowledging data, including both new data and retransmissions.

This metric is useful for understanding the receiver's perspective on the data
flow and can be used to compare against the sender's rate to identify potential
bottlenecks.

## Input Rate

The Input Rate statistic (0xd7) indicates the input bitrate from the application
layer within the observation period. It is encoded as a variable-length integer
measured in kilobits per second (kbps). This metric represents the rate at which
data is being provided to the receiving application.

This metric gives an end-to-end view of the application data flow.

## Loss Rate

The Loss Rate statistic (0xd8) indicates the arithmetic mean of the packet loss
rate samples within the observation period. The value is expressed as a
percentage at 0.1% resolution, encoded as a variable-length integer between 1
and 1000. This metric provides a clear measure of the quality of the network
path, as it quantifies the proportion of packets that are sent but not received.
A high loss rate often indicates network congestion or instability. 

## Buffer Length

The Buffer Length statistic (0xd9) indicates the current amount of data cached
by the QUIC connection layer buffer when the observation frame is generated. It
is encoded as a variable-length integer measured in bytes. This metric reflects
the amount of data the sender is holding in its buffer before transmission,
which can be an important indicator of the sender's ability to keep up with the
application's sending rate and can also be a sign of network congestion.


# Frame Types

## CONGESTION_DATA Frames

CONGESTION_DATA frames (types 0xTBD1) provide a list of Network Statistics
values which the sender chooses to share about the state of the network
connection from its viewpoint.

~~~
CONGESTION_DATA Frame {
  Type (i) = 0xTBD1,
  Protected Count (i),
  Protected Network Statistics (..) ...,
  [Integrity Tag (TODO)],
  Unprotected Count (i),
  Unprotected Network Statistics (..) ...,
}
~~~

CONGESTION_DATA frames contain the following fields:

Protected Count:

: A variable-length integer representing the number of Network Statistics in the
Protected Network Statistics field.

Protected Network Statistics:

: A sequence of Network Statistics objects whose length is given by Protected
Count.

Integrity Tag:

: A 32-byte integrity tag, calculated as described in {{integrity-tag}}. This
field is absent if Protected Count is zero.

Unprotected Count:

: A variable-length integer representing the number of Network Statistics in the
Unprotected Network Statistics field.

Unprotected Network Statistics:

: A sequence of Network Statistics objects whose length is given by Unprotected
Count.

CONGESTION_DATA frames are not retransmittable, though a loss event might
trigger the generation of a new CONGESTION_DATA frame; see
{{sending-network-statistics}}.

CONGESTION_DATA frames can be sent at any point in the connection after 0-RTT
or 1-RTT keys have been established, though useful data will likely not be
available until at least one round-trip has occurred. If a CONGESTION_DATA
frame is received in an Initial or Handshake packet, it MUST be treated as a
connection error of type PROTOCOL_VIOLATION.


### Sending Network Statistics

If an endpoint wishes to receive CONGESTION_DATA_RECALL frames on
future connections with the peer and the peer has set the CACHE flag, the
endpoint MAY send CONGESTION_DATA frames containing the values it
wishes to recall in future connections in the Protected Network Statistics
field. It MAY send additional CONGESTION_DATA frames when these values have
changed significantly and it wishes to update the stored values, or when a
previous CONGESTION_DATA frame is declared lost.

If the peer has set the CONSUME flag, an endpoint SHOULD send
CONGESTION_DATA frames periodically throughout the connection's lifetime.
However, an implementation SHOULD NOT send additional CONGESTION_DATA frames
if the connection has been idle since the last such frame was sent.

In addition to any values the endpoint placed in Protected Network Statistics,
the endpoint includes such other values as it is willing to provide in the
Unprotected Network Statistics field.

If an endpoint sends multiple CONGESTION_DATA frames, it is not required to
include the same set of network statistics in each frame. For example, some
statistics are more useful sent at a regular frequency, while others only need
to be sent if they have changed significantly from the last value known to have
been received. However, as the server does not control which CONGESTION_DATA
will be cached, it SHOULD include the same Protected Network Statistics fields
in each frame.

### Integrity Tag

The integrity tag is calculated over the Protected Count and Protected Network
Statistics field by TODO Magic Crypto Fairy Dust.

It incorporates a key known only to the sender. This enables the sender to
verify that the values it provided have not been tampered with when they are
returned in a CONGESTION_DATA_RECALL frame, and that the values are recent
enough that it is still willing to use them.

## CONGESTION_DATA_RECALL Frames

CONGESTION_DATA_RECALL frames (type 0xTBD2) contain a list of Network
Statistics values which the sender received from the recipient during a previous
connection.

This frame SHOULD be sent as early as possible in the connection once 0-RTT or
1-RTT keys are available. While the frame MAY be sent at any point in the
connection, if it arrives after the recipient has exited slow-start the values
it contains will likely not be useful.

~~~
CONGESTION_DATA_RECALL Frame {
  Type (i) = 0xTBD2,
  Protected Count (i),
  Protected Network Statistics (..) ...,
  Integrity Tag (TODO),
}
~~~

CONGESTION_DATA_RECALL frames contain the following fields:

Protected Count:

: A variable-length integer representing the number of Network Statistics in the
Protected Network Statistics field, received in a CONGESTION_DATA frame from
the peer on a previous connection.

Protected Network Statistics:

: A sequence of Network Statistics objects whose length is given by Protected
Count, received in a CONGESTION_DATA frame from the peer on a previous
connection.

Integrity Tag:

: A 32-byte integrity tag, received in a CONGESTION_DATA frame from
the peer on a previous connection.

If a CONGESTION_DATA_RECALL frame is received in an Initial or Handshake
packet, it MUST be treated as a connection error of type PROTOCOL_VIOLATION.

### Recalling Network Statistics

Upon receipt of a CONGESTION_DATA_RECALL frame, an endpoint computes the
expected Integrity Tag value as in {{integrity-tag}}. If the Integrity Tag value
does not match, the frame is ignored.

If the tag is acceptable, the endpoint takes the network statistics contained in
the frame and incorporates them into its congestion control strategy. For
example, it might exit the Reconnaissance Phase of Careful Resume
{{?CAREFUL-RESUME}}. The specifics of how this is done are outside the scope of
this extension.

Endpoints MUST NOT process more than one CONGESTION_DATA_RECALL frame on a
connection. Subsequent CONGESTION_DATA_RECALL frames MUST be ignored without
processing, regardless of whether the first frame was valid.

# Security Considerations

## Privacy

Clients choosing to return network statistics to a server provide a potential
tracking mechanism. However, this tracking mechanism provides no additional
capabilities to a server beyond those already enabled by the address validation
tokens defined in {{Section 8.1.3 of QUIC}}. While address validation tokens are
opaque and can contain any data the server might wish to recall, the statistics
being transported by this mechanism are visible to the clients. Clients can
inspect the values to ensure that nothing objectionable is being saved;
implementations MAY choose not to send CONGESTION_DATA_RECALL packets which
contain statistics they cannot interpret.

Clients SHOULD NOT send CONGESTION_DATA_RECALL packets on connections where they
would not have sent an Address Validation token if one were available. Clients
SHOULD discard stored network statistics when other potential tracking
mechanisms (e.g. HTTP Cookies) are cleared by the user. 


# IANA Considerations

This document has IANA actions which need to be written down. There will
probably be a registry of possible values to send.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
