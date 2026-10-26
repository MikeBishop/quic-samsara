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
    asciiFullname: Jinghao Yuan
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
 -
    fullname: Rich Salz
    organization: Akamai Technologiers
    email: rsalz@akamai.com

normative:
  QUIC: RFC9000

informative:
  HTTP-FLV-ABR:
    title: "Adaptive Bitrate Live Streaming over HTTP-FLV: A Practical System Perspective"
    target: https://doi.org/10.1145/3789240.3829169
    author:
      - { ins: L. Zhang, name: Le Zhang }
      - { ins: T. Meng, name: Tong Meng }
      - { ins: B. Lu, name: Bingcong Lu }
      - { ins: J. Yuan, name: Jinghao Yuan }
      - { ins: L. Chen, name: Lu Chen }
      - { ins: H. Liu, name: Huanting Liu }
      - { ins: L. Xiao, name: Lei Xiao }
      - { ins: N. Wu, name: Nailiang Wu }
      - { ins: Z. Sha, name: Zhou Sha }
      - { ins: C. Yan, name: Changqing Yan }
      - { ins: J. Zhang, name: Jianrong Zhang }
      - { ins: J. Kuang, name: Jianxin Kuang }
      - { ins: L. Song, name: Li Song }
    date: 2026-08
    seriesinfo:
      DOI: 10.1145/3789240.3829169
    ann: ACM SIGCOMM 2026 Conference.

--- abstract

This draft defines a new transport frame which enables consenting endpoints
to share congestion control state about the network connection for various
purposes, including client-side adaptive bitrate selection during an ongoing
connection. It also allows an endpoint's own congestion control state to be echoed
back to it by a peer for consideration at the beginning of a future connection.

--- middle

# Introduction

All Internet transports are required to either use a congestion control
algorithm, or to constrain their rate of transmission {{!RFC8085}}. Most
congestion control algorithms take time to ramp up the sending rate, called the
"Slow-Start phase." This defines a time in which a sender intentionally uses
less capacity than might be available so as to avoid or limit overshoot of the
available capacity for the path. This is because any overshoot can have
detrimental effects both for the current flow and for any other flows with which
it shares network bottlenecks.

At the same time, using less capacity than is available necessarily limits
performance early in the connection. Careful Resume
{{?CAREFUL-RESUME=RFC9959}} is a mechanism whereby
remembered congestion control parameters can be validated as potentially
applicable to a new connection, probed, and ultimately used to grow the
congestion window more rapidly than slow-start would otherwise permit.

One optimization approach is to use historical congestion information to
provide the congestion algorithm with
reliable input to help it exit the slow start phase. The most direct and
reliable information can be samples collected during the congestion algorithm,
such as the congestion window size and probe bandwidth.

While clients often connect to a manageable number of servers and can retain
such state, servers typically service orders of magnitude more clients and
cannot feasibly retain such information. Further, servers are often deployed
with many instances and attempting to coordinate the sharing of this information
between them may prove impractical. Thus, for a server to implement Careful
Resume, some external means of recalling its previous state is useful.

This document specifies a mechanism which allows a QUIC {{QUIC}} endpoint to
periodically export its congestion control state, optionally in an
integrity-protected manner. This exported state is sent to the peer in a
CONGESTION_DATA frame.
When establishing a subsequent connection, an endpoint with persistent storage
can include this data in a CONGESTION_DATA_RECALL frame in its 0-RTT or 1-RTT
packets, assisting its peer to recall the information necessary to perform
Careful Resume.

This mechanism is comparable to HTTP cookies, {{?COOKIES=RFC6265}}, but
for transport information.
This data may also be useful for application-layer purposes, such as
adaptive-bit-rate video. The exported information is readable by the peer and
can be exposed to the application through local interfaces. {{client-abr}}
describes how a client can consume server-side statistics for bitrate
selection during the current connection, independently of state recall.

## Peer Visibility

The peer's viewpoint of a connection can be useful for debugging and as
additional information to be considered by on-path entities such
as congestion controllers and
application-layer protocols. Therefore, this extension deliberately does not
encrypt the data reported to the peer. Instead, the data is provided in
cleartext with an optional integrity tag.

If a server wishes to recall information about past connections without sharing
that data with the client, this information can already be encoded in address
validation tokens without requiring the cooperation of the client; see {{Section
8.1.3 of QUIC}}.

## Conventions and Definitions

{::boilerplate bcp14-tagged}

This document also uses terminology defined in {{QUIC}} and
{{!QUIC-TLS=RFC9001}}, in particular the frame layout notation from {{Section
1.3 of QUIC}}.

# Transport Parameter

Desire and willingness to receive the frames defined in this specification is
indicated by means of the following QUIC transport parameter:

~~~
support_congestion_data(i)
~~~

The support_congestion_data value is a variable-length integer that
encodes these three one-bit flags:

CONSUME (0x01):

: This indicates that the sender is interested in receiving CONGESTION_DATA
frames for its own uses during the current connection, independent of the
receiver's ability to reuse the data in the future.


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

# Network Statistics {#net-stats}

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

A receiver may get a message with multiple occurrences of a particular TLV
value. If the values are identical, the reciever SHOULD ignore them.
If they differ, and one of the values is protected by an integrity tag,
the receiver SHOULD treat this as an attack and close the connection.
If none of the instances are integrity-protected, the receiver MAY
ignore them, use only one of the instances, or close the connection
as it determines to be most appropriate.

In the sub-section below, only the names are used; the numeric value that
appears in the protocol is defined in {{net-stats-registry}}.

The Integrity Tag {{integrity-tag}} uses this statistics structure
to make scanning easier.

## Timestamp

The Timestamp statistic indicates the time at which the sender generated
this frame. This can be used on future connections to determine whether the
recalled statistics are recent enough to be useful.

    *Note* Format of the timestamp is TBD.


## Path Tuple

The Path Tuple statistic encodes an identifier of the path on which these
statistics were generated. Knowing the connection addresses as seen from the
peer's perspective can be useful for a number of scenarios (e.g.,
{{?STUN=RFC8489}}). The reciever MAY use this to to compare similarity of the
previous endpoints to those of a
new connection will when deciding if returned statistics
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

    *NOTE* I am confused about this structure.  Is the Type and Length
    from the TLV?  If so, should "0x202" be the assigned number "0xca" ?

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

The Slow Start Status statistic indicates whether the sender's congestion
controller is in the Slow Start phase. The value is a single byte, set to 0x00
if the sender is not in Slow Start and 0x01 if the sender is in Slow Start.

## Network Type

The Network Type statistic indicates the sender's understanding of its
network access medium, encoded as a single byte value. Note that this is purely
advisory, since applications will only be aware of the local network at best.

The defined values are at {{net-type-registry}}.

## Maximum Congestion Window

The Maximum Congestion Window statistic indicates the maximum congestion
window (CWD) sampled within the observation period, measured in bytes.
It is encoded as a variable-lenth integer.
CWD is a
key metric in congestion control algorithms, as it represents the amount of
unacknowledged data that a sender can have in flight on the network. A larger
CWD generally allows for a higher sending rate.

## Maximum In-Flight Data

The Maximum In-Flight Data statistic indicates the maximum in-flight data
sampled within the observation period, measured in bytes.
It is encoded as a variable-length integer.
This represents the highest number of bytes sent by
the sender that have not yet been acknowledged by the receiver during the
measurement period. This metric provides insight into the actual amount of data
in transit at any given time, which can be useful for diagnosing network
performance issues.

## Smoothed RTT

The Smoothed RTT statistic indicates the most recent smoothed Round-Trip
Time (RTT) within the observation period, measured in milliseconds.
It is encoded as a variable-length integer. It is calculated as
defined in {{!RFC9002}}. RTT is a key metric for congestion control, estimating
the time it takes for a packet to travel from the sender to the receiver and
back. The smoothed RTT calculation accounts for both the latest RTT and a
historical average, helping to dampen the effect of short-term network
fluctuations.

## Minimum RTT

The Minimum RTT statistic indicates the minimum RTT sampled within the
observation period, measured in milliseconds.
It is encoded as a variable-length integer,
This metric provides a baseline for the best-case network latency
observed during the measurement period. A low minimum RTT can indicate a stable
and efficient network path, while a high one might suggest persistent latency
issues.

## RTT Variance

The RTT Variance statistic indicates the most recent RTT deviation within
the observation period, measured in milliseconds.
It is encoded as a variable-length integer.
It is calculated as defined in {{!RFC9002}}. This metric
quantifies the variability of the RTT, providing insight into network jitter and
stability. A low RTT variance suggests a consistent network path, while a high
value indicates significant fluctuations in network latency.

## Latest Bandwidth

The Latest Bandwidth statistic indicates the current raw throughput of
the connection, measured in kilobits per second (kbps).
It is encoded as a variable-length integer.
This metric represents the instantaneous sending
capacity as perceived by the sender and is a crucial input for congestion
control algorithms.

## Maximum Bandwidth

The Maximum Bandwidth statistic indicates the maximum raw throughput
sampled within the observation period, measured in kbps.
It is encoded as a variable-length integer.
This metric provides a view of
the peak network capacity observed during the measurement period, which can be
useful for understanding the best possible performance on the current network
path.

## Throughput

The Throughput statistic indicates the useful throughput for data,
excluding retransmissions, within the observation period, measured in kbps.
It is encoded as a variable-length integer. This metric is a
measure of the effective data rate delivered to the receiver's application
layer.
It isolates the useful data rate, providing a more accurate
measure of application-level performance than the raw sending rate, which
includes retransmitted data.

## Send Rate

The Send Rate statistic indicates the sending rate for all data,
including retransmissions, within the observation period, measured in kbps.
It is encoded as a variable-length integer. This metric
provides a measure of the total data rate at which the sender is transmitting
data.
It is useful for understanding the sender's total load on the network.

## Receive Rate

The Receive Rate statistic indicates the receiving rate within the
observation period in kbps.
It is encoded as a variable-length integer.
This metric measures the total rate at which the
receiver is acknowledging data, including both new data and retransmissions.
It is useful for understanding the receiver's perspective on the data
flow and can be used to compare against the sender's rate to identify potential
bottlenecks.

## Input Rate

The Input Rate statistic indicates the input bitrate from the application
layer within the observation period in kbps.
It is encoded as a variable-length integer.
This metric represents the rate at which
data is being provided to the receiving application.
It gives an end-to-end view of the application data flow.

## Loss Rate

The Loss Rate statistic indicates the arithmetic mean of the packet loss
rate samples within the observation period. The value is expressed as a
percentage at 0.1% resolution within a range of 0 to 1000 inclusive.
It is encoded as a variable-length integer.
This metric provides a clear measure of the quality of the network
path, as it quantifies the proportion of packets that are sent but not received.
A high loss rate often indicates network congestion or instability.

## Buffer Length

The Buffer Length statistic indicates the current amount of data cached
by the QUIC connection layer buffer when the observation frame is generated
in bytes.
It is encoded as a variable-length integer. This metric reflects
the amount of data the sender is holding in its buffer before transmission,
which can be an important indicator of the sender's ability to keep up with the
application's sending rate and can also be a sign of network congestion.


# Frame Types

## CONGESTION_DATA Frames

CONGESTION_DATA frames (type TBD1) provide a list of Network Statistics
values which the sender chooses to share about the state of the network
connection from its viewpoint.

~~~
CONGESTION_DATA Frame {
  Type (i) = TBD1,
  Protected Count (i),
  Protected Network Statistics (..) ...,
  [Integrity Tag (1) ...],
  Unprotected Count (i),
  Unprotected Network Statistics (..) ...,
}
~~~

CONGESTION_DATA frames contain the following fields:

Protected Count:

: A variable-length integer representing the number of Network Statistics in the
Protected Network Statistics field.

Protected Network Statistics:

: A sequence of Network Statistics objects whose length is given by the
Protected Count.

Integrity Tag:

: A message integrity check,  as described in {{integrity-tag}}. This
field is absent if Protected Count is zero.
While this document provides some examples, the format of the check
MUST be treated as opaque by the receiver.

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

### Integrity Tag {#integrity-tag}

The integrity tag is calculated over the Protected Count and Protected Network
Statistics field by the sender.

This field is identical to the Network Statistics structure, and as such
the value is a
variable-length set of bytes, whose format is known only
to the sender. The purpose of this field is to provide suitable assurance
to the sender that, when the statistics are later sent back to it through
the CONGESTION_DATA_RECALL frame, that they
have not been modified. This is often called a "message authentication
code" (MAC). To emphasize that only a portion of a message is protected,
this document does not use that term.

The algorithm for generating and verifying an integrity tag MAY
depend on the ordering of the Protected fields although some implementations
may perform a simple canonicalization by sorting the statistics by
type identifier. Because of this, receivers SHOULD NOT modify the
content or ordering of any of the Protected statistics in any way,
unless they have out-of-band knowledge that it is safe to do so.

If the server has a nonce or other private material, it can hash that
with the incoming Protected fields and use that as the outgoing Integrity
tag. This can be either a simple hash of both parts, or an HMAC keyed
hash {{?RFC2104}} can be used.

Being able to change algorithms without large-scale protocol modifications
is important. Servers may wish to use a fixed-number of leading bytes to
indicate the algorithm they are using. It is also a best practice to
generate new private data periodically, while still allowing old messages
to be validated. To handle this, it is a good idea to use a fixed
number of secondary bytes to act as a key or nonce identifier.

If or when an Intgrity Tag algorithm is to be widely used, it can be documented
and assigned a number in the Network Statistic Registry {{net-stats-registry}}.

A sample implementation is provided in {{integrity-impl}}.

## CONGESTION_DATA_RECALL Frames

CONGESTION_DATA_RECALL frames (type TBD2) contain a list of Network
Statistics values which the sender received from the recipient during a previous
connection.

This frame SHOULD be sent as early as possible in the connection once 0-RTT or
1-RTT keys are available. While the frame MAY be sent at any point in the
connection, if it arrives after the recipient has exited slow-start the values
it contains will likely not be useful.

~~~
CONGESTION_DATA_RECALL Frame {
  Type (i) = TBD2,
  Protected Count (i),
  Protected Network Statistics (..) ...,
  Integrity Tag (1..),
}
~~~

    *NOTE* Do we want to allow unprotected statistics here also, with the
    caveat that the receiver may reject them, or even the whole message?

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

: The integrity tag, received in a CONGESTION_DATA frame from
the peer on a previous connection. See {{integrity-tag}}.

If a CONGESTION_DATA_RECALL frame is received in an Initial or Handshake
packet, it MUST be treated as a connection error of type PROTOCOL_VIOLATION.

### Recalling Network Statistics

Upon receipt of a CONGESTION_DATA_RECALL frame, an endpoint computes the
expected Integrity Tag value as in {{integrity-tag}}. If the Integrity Tag value
does not match, the frame is ignored.

If the tag is acceptable, the endpoint can consider the network statistics as
inputs to its congestion control strategy, subject to freshness, path, and
reuse checks as illustrated in {{server-state-recall}}. Successful integrity
verification does not by itself establish that the current path supports the
recalled capacity. The specifics of how a congestion controller consumes
accepted statistics, for example in Careful Resume {{?CAREFUL-RESUME}}, are
outside the scope of this extension.

Endpoints MUST NOT process more than one CONGESTION_DATA_RECALL frame on a
connection. Subsequent CONGESTION_DATA_RECALL frames MUST be ignored without
processing, regardless of whether the first frame was valid.

# Application Use Cases

The following examples illustrate the use of exported statistics for
server-side state recovery and client-side application decisions. They do not
prescribe a congestion control algorithm, an adaptation algorithm, or an
application API.

## Server-Side Congestion State Recovery {#server-state-recall}

A server can use CONGESTION_DATA_RECALL to recover its own observations from
an earlier connection, providing historical inputs for mechanisms such as
Careful Resume {{?CAREFUL-RESUME}}. The client acts as a storage and return
mechanism for server-generated information. The objective is to return a
self-contained, integrity-protected record that the server can assess without
retaining the full measurement history for every client.

During the earlier connection, the client advertises CACHE. The server selects
a set of observations and sends them together in the Protected Network
Statistics of a CONGESTION_DATA frame, with an Integrity Tag covering the
Protected Count and the complete protected sequence. The client saves that
sequence and tag together. On a subsequent connection, after determining that
the server permits recall through CONSIDER, the client returns the saved
record in a CONGESTION_DATA_RECALL frame. With remembered permission this can
occur in 0-RTT; otherwise the client waits for permission on the new connection.

The client associates each cached record with the authenticated identity of
the server that issued it and the server endpoint on that connection. When
selecting a record for recall, the client MUST select one issued by the same
server; records from another server MUST NOT be returned, even if that server
also advertises CONSIDER. For example, the cache can be scoped by the
application service identity (such as an HTTPS origin) and the server IP
address and port. An IP address alone is not an authenticated server identity.
If the client cannot establish the required association, it omits RECALL.
For 0-RTT, selection uses the server identity associated with the resumption
context and the remembered permission, rather than a new handshake result.

~~~
Client A                                    Server A
   |                                            |
   |          Connection 1                      |
   |-- CACHE ---------------------------------->|
   |<------------------------------- CONSIDER --|
   |<-- CONGESTION_DATA ------------------------|
   |    protected statistics + integrity tag    |
   |                                            |
   | Save the complete record, scoped to A      |
   |                                            |
   |          Connection 2 to A                 |
   |                                            |
   | Select A's record; check permission        |
   |-- CONGESTION_DATA_RECALL ----------------->|
   |    same protected statistics + tag         |
   |                                            | Verify record,
   |                                            | age, path, reuse
   |                                            | before recovery

Client A                                    Server B
   |                                            |
   |          New connection to B               |
   |<------------------------------- CONSIDER --|
   |                                            |
   | Cached record belongs to Server A         |
   | No cached record for Server B             |
   | Cannot construct RECALL from A's record   |
   |                                            |
   |-- CONGESTION_DATA_RECALL --X                |
   |    Not sent: server identity mismatch      |
   |                                            |
   |<---------- Normal data transfer ---------->|
~~~

Here, the same server means the authenticated service at the selected server
endpoint, not a QUIC Connection ID or a particular worker process. Behind a
load balancer, the client cannot identify which process will receive the new
connection. Such a deployment is responsible for consistent record validation
and reuse control across accepting instances, or for declining recovery when
the receiving instance cannot validate the record. Matching the server on the
client does not replace the server's checks on the current network path.

The record provides three categories of information: previously observed
capacity, the context in which it was observed, and its age. The following
statistics can supply these inputs:

| Information | Network Statistics | Purpose |
|:------------|:-------------------|:--------|
| Capacity observations | Maximum Congestion Window, Maximum In-Flight Data, Maximum Bandwidth, or Throughput | Provide historical window, flight-size, or rate measurements appropriate to the sender's controller |
| Timing context | Minimum RTT | Retain the minimum RTT associated with the capacity observation |
| Path context | Path Tuple | Identify the server-side view of the endpoints associated with the observation |
| Freshness | Timestamp | Bound the age of the exported record under the server's lifetime policy |
| Optional context | Slow Start Status, Loss Rate | Help interpret observations that may have been affected by startup or congestion |

The server selects an observation period and a combination of statistics that
are sufficient for its intended recovery method. Capacity and timing values
refer to the same path and a consistent observation context. A maximum
congestion window or maximum flight size is not automatically a validated
capacity estimate: startup overshoot and application-limited transmission need
to be accounted for when selecting and interpreting the record. In particular,
these maxima are not defined as Careful Resume's saved_cwnd. If a controller
requires an additional derived value or context that the existing statistics
cannot express, it can be specified as an additional Network Statistic without
changing the RECALL frame structure.

Timestamp currently describes frame generation time, and its encoding remains
to be specified. For a complete recovery record, the server also needs to
bound the age of the underlying observations. For example, it can issue the
record when collecting the observations, or enforce a known maximum delay
between observation and issuance. Reissuing old measurements with a new
Timestamp does not make those measurements fresh. The server applies its
lifetime policy and rejects expired records. These rules allow the record to
provide the necessary historical inputs; they do not assert that any arbitrary
selection of statistics is sufficient for recovery.

All values used to make a recovery decision, including the path and time
information, are carried in the protected sequence. For this use case, the
server uses a cryptographic authentication mechanism such as HMAC with a
server-held secret, as discussed in {{integrity-tag}}. Verifying the tag then
establishes that the returned record was issued within the server's trusted
key domain and has not been altered by the client. It prevents a client from
inflating the recorded capacity, replacing the path or timestamp, or combining
fields from different authenticated records. The client returns the protected
sequence and its tag intact; unprotected statistics are not promoted into
authenticated recovery inputs.

Authenticity, freshness, and applicability are separate checks. The server
compares the protected path context with the new connection using its path
matching policy and any local indication of a path change. A valid tag does
not prove that the network conditions are unchanged. Missing required values,
unsupported encodings, a failed tag check, an expired record, or incompatible
path context cause the server to decline recovery and use normal congestion
control. QUIC packet protection secures the exchange on the current connection;
the Integrity Tag additionally binds the historical record across connections.
Neither mechanism guarantees confidentiality of the statistics from the client.

A valid record can also be replayed without being modified. Limiting RECALL
processing to one frame per connection does not prevent the same record from
being returned on several connections, including through 0-RTT replay. A server
using a recovery method that disallows concurrent reuse therefore maintains
appropriate usage and invalidation state, coordinated across instances that
accept the same records. Record identity can be derived from the authenticated
record, with related records grouped under the server's endpoint policy.
Client-side storage reduces retained measurement state, but does not remove
these reuse-control requirements.

Once these checks succeed, the recovered record supplies authenticated,
contextualized historical inputs to the server's congestion controller. The
controller remains responsible for validating their use on the current path.
This separation allows RECALL to support different recovery algorithms while
preserving QUIC's address-validation, anti-amplification, and flow-control
requirements.

## Client-Side Adaptive Bitrate Streaming {#client-abr}

In adaptive bitrate (ABR) streaming, a client selects among media variants
according to network conditions and playback state. Client-side observations
of media download speed provide one input to that decision. The server's
congestion controller has additional observations, including bandwidth
samples, RTT, and loss rate, which can be shared with the client through
CONGESTION_DATA frames.

Section 5.1 of {{HTTP-FLV-ABR}} describes a deployed example in TikTok Live,
where a QUIC frame named CCTK (Congestion Control ToKen) periodically carries
server-side congestion control statistics to the client. The paper reports
that this feedback is the default bandwidth-estimation input to its ABR
algorithm and is supported by its major CDN providers. That deployment
motivates the use case here; CCTK is not a wire-format definition of the
CONGESTION_DATA frame specified in this document.

Using this extension, the client sets the CONSUME flag in its
support_congestion_data transport parameter. After receiving this indication,
the server sends CONGESTION_DATA frames during active media delivery, as
described in {{sending-network-statistics}}. The client exposes the received
statistics to its media application through a local interface. This use case
does not require CACHE or CONSIDER, or a CONGESTION_DATA_RECALL exchange.
For feedback used only on the current connection, the server can place the
statistics in Unprotected Network Statistics with Protected Count set to
zero. These statistics remain protected by QUIC packet protection; they
simply lack the additional integrity tag used for later recall.

~~~
Client / media application                  Media server
          |                                      |
          |-- support_congestion_data: CONSUME ->|
          |                                      |
          |<----------- Media data --------------|
          |<--------- CONGESTION_DATA -----------|
          |     bandwidth, RTT, loss, ...        |
          |                                      |
     Select bitrate using                        |
     feedback and playback state                 |
          |                                      |
          |-- Application request for variant -->|
          |<----------- Media data --------------|
~~~

The application can combine Latest Bandwidth or Throughput with Smoothed RTT
and Loss Rate, together with its local playback buffer and available bitrate
ladder, to select a media variant. Recent feedback on an existing connection
can also inform initial bitrate selection for a newly requested media stream
on that connection. Without suitable feedback, the application falls back to
local observations and a conservative initial selection.

Connection-level feedback provides a common input when several media streams
share a QUIC connection, reducing the need to infer network conditions
independently from each viewing session. It does not specify a per-stream
bandwidth allocation: the application still accounts for competing streams
and their combined demand. The choice of a variant and the mechanism for
switching to it remain application-layer decisions, independent of whether
the client or server executes the media switch.

Feedback requires interpretation rather than direct conversion into a target
media bitrate. Section 6.3 of {{HTTP-FLV-ABR}} reports that using an aggressive
initial sending rate as a bandwidth sample led clients to select high-bitrate
variants more often and increased rebuffering. A configured sending rate is
not necessarily a measurement of sustainable delivery capacity. Applications
can use Slow Start Status and local delivery observations to qualify early
samples, and account for missing or stale feedback and path changes.
CONGESTION_DATA frames are not retransmitted, so adaptation cannot depend on
receiving every report.

Section 7.2 of {{HTTP-FLV-ABR}} also discusses an extension beyond its deployed
ABR inputs: using send-buffer information to help distinguish limited media
supply from last-mile congestion. Buffer Length can provide such additional
context here, although an empty server buffer alone does not establish the
cause of reduced throughput. This is a possible application of the statistics,
not an evaluated ABR benefit in that paper.

The CCTK deployment described in the paper allows a client to request a
feedback interval. This extension defines the CONSUME indication but does not
negotiate an interval. The sender chooses its reporting schedule under
{{sending-network-statistics}}, balancing freshness and overhead. No new
frame type or statistic is required for the basic use case described here.

# Security Considerations

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
would not have sent an Address Validation token if one were available. A client
MAY also decide not to send the packet if the length of the integrity tag does
not correspond to a digest length and a few additional bytes. This is
admittedly inelegant. and could be avoided if the format of the tag were
publicly defined, and an IANA registry for tag algorithms defined.

Clients SHOULD discard stored network statistics when other potential tracking
mechanisms (e.g. HTTP Cookies) are cleared by the user.

# IANA Considerations

IANA is requested to take the following actions, replacing
"ThisRFC" with the RFC number when assigned.

## New QUIC Transport Parameters Entry

Add a new entry to the "QUIC Transport Parameters" registry with the
following values:

| Field Name        | Value                   |
|:------------------|:------------------------|
| Value             | TBD                     |
| Parameter Name    | support_congestion_data |
| Status            | permanent               |
| Specification     | ThisRFC                 |
| Date              | TBD                     |
| Change Controller | IETF                    |
| Contact           | quic@ietf.org           |
| Notes             | None                    |

## New QUIC Network Type Registry {#net-type-registry}

A new registry "QUIC Network Type" is created with the following fields:

Value:

: Numeric value

Meaning:

: Brief textual description

Reference:

: A pointer to the defining document

The registration policy for this registry is "Specification Required" as
described in {{!RFC8126, Section 4.6}}.

The initial value of the registry is as follows:

| Value | Meaning        | Reference |
|:------|:---------------|:----------|
|  0x00 | Reserved       | ThisRFC   |
|  0x01 | Wire/Ethernet  | ThisRFC   |
|  0x02 | Reserved       | ThisRFC   |
|  0x03 | WLAN           | ThisRFC   |
|  0x04 | 2G Mobile      | ThisRFC   |
|  0x05 | 3G Mobile      | ThisRFC   |
|  0x06 | 4G Mobile      | ThisRFC   |
|  0x07 | 5G Mobile      | ThisRFC   |

## New QUIC Network Statistics Registry {#net-stats-registry}

A new "QUIC Network Statistics" registry is created.
It follows
the registration policies defined in {{!RFC9000, Section 22.1}}.
In addition to the fields described in that section, permanent registrations
MUST include the following fields:

Type:

: The type of statistic, as described in ThisRFC, Section {{net-stats}}.

Name:

: A short name for the field.

The initial value of the table is:

| Type | Name                      |
|:-----|:--------------------------|
| 0xc8 | Timestamp                 |
| 0xca | Path Tuple                |
| 0xcb | Slow Start Status         |
| 0xcc | Network Type              |
| 0xcd | Maximum Congestion Window |
| 0xce | Maximum In-Flight Data    |
| 0xcf | Smoothed RTT              |
| 0xd0 | Minimum RTT               |
| 0xd1 | RTT Variance              |
| 0xd2 | Latest Bandwidth          |
| 0xd3 | Maximum Bandwidth         |
| 0xd4 | Throughput                |
| 0xd5 | Send Rate                 |
| 0xd6 | Receive Rate              |
| 0xd7 | Input Rate                |
| 0xd8 | Loss Rate                 |
| 0xd9 | Buffer Length             |
| 0xda | Integrity Tag             |

These fields are permanent, and therefore all have the following values
for the common fields:

| Field Name        | Value                   |
|:------------------|:------------------------|
| Status            | permanent               |
| Specification     | ThisRFC                 |
| Date              | TBD                     |
| Change Controller | IETF                    |
| Contact           | quic@ietf.org           |
| Notes             | None                    |

--- back

# Sample Integrity Tag implementation {#integrity-impl}

This section is not normative.

We define an Integrity Tag format that consists of a one-byte algorithm
identifier, two
bytes of private nonce, and a digest. Based on the choice of
algorithm, this is a 32-byte SHA256 digest. The entire tag is therefore
35 bytes long.

    *NOTE* Do we need ASCII art for that layout?

The digest will be computed over the nonce, five bytes of 0x01, and the
wire-format of the protected fields.

In this example, we will use our third nonce, which is the
ASCII values of "quic-new-frame":

    algid = 1
    keyid = 3
    nonces[] = [
        [ None ], [ None ],
        [113, 117, 105, 99, 45, 110, 101, 119, 45, 102, 114, 97, 109, 101]
        ]
    padding = [ 0x01, 0x01, 0x01, 0x01, 0x01 ]

The Network Statistics values are:

    *NOTE* TBD; need a sample network statistics

Which have the following wire representation:

    *NOTE* Calculate them

The `value` for the Integrity tag is represented by the
following psuedo-code:

    digest = sha56.new()
    digest.add(14, nonce[2])
    digest.add(5, padding)
    digest.add(??, network_statistics)
    value = digest.finish()

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
