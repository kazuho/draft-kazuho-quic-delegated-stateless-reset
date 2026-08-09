---
title: "Delegating QUIC Stateless Resets"
category: std
docname: draft-kazuho-quic-reset-token-frame-latest
wg: quic
ipr: trust200902
keyword: internet-draft
stand_alone: yes
pi: [toc, sortfefs, symrefs]
author:
 -
    fullname:
      :: 奥 一穂
      ascii: Kazuho Oku
    org: Fastly
    email: kazuhooku@gmail.com

normative:

informative:

...

--- abstract

This document specifies how a QUIC endpoint may delegate the emission of
Stateless Resets to a component that outlives the QUIC stack -- for example, an
operating system that resets the peers of an application that has exited. As such
delegation is most useful on the client side, where endpoints often use
zero-length Connection IDs, this document also defines a transport parameter and
a frame that allow those endpoints to advertise a Stateless Reset Token.


--- middle

# Introduction

Stateless reset ({{Section 10.3 of !RFC9000}}) allows an endpoint that has lost
connection state to signal to its peer that a connection can no longer be
continued. Nothing a Stateless Reset carries is protected by the connection's
keys or tied to its packet number spaces: the signal is authenticated solely by
the 128-bit Stateless Reset Token it ends with, which the receiver matches
against the tokens it retains.

QUIC version 1 uses that property to avoid retaining state. An endpoint
associates a Stateless Reset Token with each Connection ID it issues, delivering
the token to the peer either in the stateless_reset_token transport parameter or
in a NEW_CONNECTION_ID frame. When such an endpoint later receives a packet that
it cannot associate with any active connection, it recomputes the token from the
Destination Connection ID carried in that packet -- typically by applying a
static key to the Connection ID ({{Section 10.3.2 of RFC9000}}) -- and emits a
Stateless Reset carrying that token. Because the token is derived from the
Connection ID, the resetting endpoint need not retain any per-connection state.

The same property permits a different arrangement, and it is the subject of
this document: the entity that sends a Stateless Reset need not be the endpoint
that ran the connection.

The ability to send a Stateless Reset is valuable precisely when connection state
has been lost involuntarily -- for example, when an application exits or is
terminated by the operating system. After the application is gone, the peer may
continue to send packets on the connection until its idle timeout expires. To cover
these scenarios, an endpoint can delegate the termination of the connection to a
component that outlives its own connection state, and that component can reset
the peer long after the endpoint is gone ({{delegation}}).

A CONNECTION_CLOSE packet could be recorded and used for this scenario, but
such a packet is difficult to manage for a delegate. It is protected under the
connection's packet-protection keys that endpoints periodically rotate, and it
carries a packet number that must fall within a dynamic range the peer will accept;
thus, the recorded packet would need constant refreshing to remain usable.

In contrast, sending a Stateless Reset requires only the static Stateless Reset
Token and the 5-tuple on which to send it. These remain valid until the endpoint
retires the Connection ID with which the token is associated.

A delegated Stateless Reset can also be better formed than one sent by an
endpoint that has lost its state. The delegate holds its token deliberately, and
can hold alongside it a Connection ID that the peer accepts, placing that value
where a 1-RTT packet carries the Destination Connection ID. An endpoint that has
lost its state has no such value available and must use unpredictable bits
instead, at the cost that the Stateless Reset may be misrouted ({{Section 10.3
of RFC9000}}).

The operating system could instead retain a CONNECTION_CLOSE packet recorded by
the application and replay it, but such a packet is difficult to manage. It is
protected under the connection's packet-protection keys that endpoints rotate
and carries a packet number that must fall within the range the peer will
accept, so the recorded packet must be refreshed periodically to remain usable.
A Stateless Reset Token is tied to neither, and so does not decay on its own; it
remains valid until the endpoint retires the Connection ID with which the token
is associated. The operating system need only remember the token.

With QUIC version 1, Stateless Reset Tokens are issued alongside Connection IDs;
an endpoint that uses zero-length Connection IDs cannot issue one
({{Section 5.1.1 of RFC9000}}). Yet they are common. A client that
identifies connections solely by the 5-tuple has no need to issue Connection IDs
and can reduce per-packet overhead by using zero-length Connection IDs.
To allow such endpoints to issue Stateless Reset Tokens and delegate them, this
document defines a transport parameter and a frame ({{advertising}}).


# Conventions and Definitions

{::boilerplate bcp14-tagged}


# Delegating a Stateless Reset {#delegation}

An endpoint can delegate the sending of a Stateless Reset to a component that
outlives its connection state. The delegate retains the Stateless Reset Token,
rather than recomputing it from a Connection ID as a resetting endpoint would,
together with whatever the endpoint uses to match a received packet to the
connection -- a Connection ID, or the 5-tuple when a client uses a zero-length
Connection ID ({{Section 5.2 of RFC9000}}). To construct a reset that
can be routed to the peer, it also needs a Connection ID that the peer issued
({{constructing}}). It needs no packet-protection keys and no packet number.

Any endpoint that has issued a Stateless Reset Token can delegate it, whatever
the length of the Connection IDs it issues. An endpoint that issues Connection
IDs delegates one of the tokens it has already distributed, in the
stateless_reset_token transport parameter or in a NEW_CONNECTION_ID frame. A
client that uses a zero-length Connection ID cannot issue a token under QUIC
version 1; {{advertising}} defines how it does so.

Except as described in {{constructing}} and {{proactive}}, the requirements of
{{Section 10.3 of RFC9000}} and {{Section 10.3.3 of RFC9000}} governing the
construction and sending of a Stateless Reset apply unchanged.

## Delegating a Stateless Reset to the Operating System {#os-offload}

In some environments, the operating system sends a Stateless Reset on behalf of
an application that has exited or has been terminated. The application and the
operating system can divide the work as follows.

The operating system exposes an interface through which an application
registers, for one of its connections, a Connection ID that the peer has issued
and currently accepts, together with what identifies that connection in the
datagrams the peer sends -- the socket, and any Connection IDs the application
itself has issued. From that registration it forms a datagram containing, in
order:

* a first byte of its own making, in which the two most significant bits are 01
  and the remainder is unpredictable;

* the peer's Connection ID, in the position in which a 1-RTT packet carries the
  Destination Connection ID field ({{constructing}}); and

* 20 bytes that it generates at random, of which the leading 4 separate the
  token from the header and the trailing 16 are the Stateless Reset Token.

The datagram is the smallest that can be mistaken for a 1-RTT packet carrying the
peer's Connection ID. The operating system retains it against that registration
and returns the token to the application, which then makes it known to its peer,
either as the token associated with one of its own
Connection IDs or, if it uses zero-length Connection IDs, in a
RESET_TOKEN frame ({{reset-token-frame}}).

Once the application is no longer able to process packets for the connection,
the operating system responds to a datagram matching that registration by
sending the retained datagram, but does so only if the retained datagram is
smaller than the one that triggered it, as required by
{{Section 10.3.3 of RFC9000}}.

Eventually, the operating system discards the stateless reset and the
registration, as the peer will nevertheless abandon the connection due to not
making progress.

In this division of labor, it is possible to deny an application the use of the
Stateless Reset as a means of conveying information of its own. The Stateless
Reset Token and the other bytes that the operating system generates are outside
the application's control, and the peer's Connection ID, which the application
supplies, is also the value that it places in the Destination Connection ID
field of the packets it sends. An operating system can therefore monitor the
packets that the application sends after the registration, and send the
Stateless Reset only if those packets carried that Connection ID. The
Stateless Reset then carries no Connection ID other than the one that the
application's own packets were already carrying to the peer.

The operating system can also send the retained datagram as soon as the
application becomes unable to process packets, without waiting for the peer to
send anything ({{proactive}}).


# Constructing a Routable Stateless Reset {#constructing}

When constructing a Stateless Reset, the endpoint or its delegate SHOULD place a
Connection ID issued by the peer in the position in which a 1-RTT packet carries
the Destination Connection ID field, rather than unpredictable bits. A peer
receiving the datagram is unable to decrypt it and therefore compares its
trailing 16 bytes against the Stateless Reset Tokens it retains
({{Section 10.3.1 of RFC9000}}).

{{Section 10.3 of RFC9000}} calls for unpredictable bits in that position
because an endpoint that has lost its state cannot recover a Connection ID that
its peer accepts, and it observes that the resulting packet "could be
incorrectly routed" where the Connection ID is critical for routing toward the
peer, as it is in load-balanced deployments. An endpoint or delegate that has
retained such a Connection ID sends a Stateless Reset that reaches the peer and
is less distinguishable from a valid packet.


# Sending a Stateless Reset Proactively {#proactive}

A delegate MAY send a single Stateless Reset proactively,
without waiting to receive a datagram from the peer. {{Section 10.3 of RFC9000}}
describes a Stateless Reset only as a response to a received packet because an
endpoint that has lost its state learns of the connection's existence only from
one; an endpoint or delegate that retains the token knows which connections it
may need to reset before any packet arrives.


# Advertising a Token with Zero-Length Connection IDs {#advertising}

An endpoint that uses zero-length Connection IDs cannot advertise a Stateless
Reset Token by any means available in QUIC version 1. This section defines a
transport parameter and a frame through which it can do so. The token is bound
to the connection and can be delegated as described in {{delegation}}.


## The reset_token Transport Parameter

An endpoint that is willing to accept RESET_TOKEN frames
({{reset-token-frame}}) from its peer advertises the reset_token transport
parameter (0x-TBD).

The reset_token transport parameter has a zero-length value; its presence alone
conveys that willingness. An endpoint that receives a reset_token transport
parameter with a non-zero length MUST treat it as a connection error of type
TRANSPORT_PARAMETER_ERROR.

Advertising the parameter carries no implication that the advertising endpoint
will itself send RESET_TOKEN frames.


## The RESET_TOKEN Frame {#reset-token-frame}

The RESET_TOKEN frame (type 0x-TBD) carries a Stateless Reset Token that the
sender associates with the connection rather than with any Connection ID.

~~~
RESET_TOKEN Frame {
  Type (i) = 0x-TBD,
  Stateless Reset Token (128),
}
~~~
{: #reset-token-format title="RESET_TOKEN Frame Format"}

The frame contains the following field:

Stateless Reset Token:
: A 128-bit value that the sender will use to construct a Stateless Reset
  ({{Section 10.3 of RFC9000}}) for this connection. The token MUST be
  generated so that it is hard to guess, as required by {{Section 10.3 of
  RFC9000}}.

RESET_TOKEN frames are ack-eliciting; a sender that detects the loss of a
RESET_TOKEN frame retransmits the token it carried. A RESET_TOKEN frame MUST
only be sent in a 1-RTT packet.

Only an endpoint that uses a zero-length Connection ID may send a RESET_TOKEN
frame; the mechanism substitutes for the tokens that a Connection ID-issuing
endpoint issues using NEW_CONNECTION_ID frames. An endpoint
that uses a non-zero-length Connection ID MUST NOT send a RESET_TOKEN frame. An
endpoint that receives a RESET_TOKEN frame from a peer that uses a
non-zero-length Connection ID MUST treat this as a connection error of type
PROTOCOL_VIOLATION.

An endpoint that has not advertised the reset_token transport parameter treats a
received RESET_TOKEN frame as a frame of unknown type, which is a connection
error of type FRAME_ENCODING_ERROR ({{Section 12.4 of RFC9000}}).

An endpoint that receives a RESET_TOKEN frame recognizes the contained value as
a Stateless Reset Token for this connection and thereafter processes it exactly
as a token received in a NEW_CONNECTION_ID frame, following the requirements of
{{Section 10.3 of RFC9000}} for retaining the token and for detecting a
Stateless Reset that carries it.

An endpoint uses a single Stateless Reset Token for the lifetime of a
connection. It MAY send more than one RESET_TOKEN frame -- for example, to
retransmit a frame that was lost -- but every RESET_TOKEN frame it sends on a
connection MUST carry the same token value. An endpoint that receives a
RESET_TOKEN frame carrying a token value different from one it received earlier
on the same connection MUST treat this as a connection error of type
PROTOCOL_VIOLATION.


# Security Considerations

TODO Security


# IANA Considerations

TBD


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
