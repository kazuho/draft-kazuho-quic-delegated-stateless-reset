---
title: "QUIC Extension for Advertising Stateless Reset Tokens"
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

QUIC endpoints that identify connections solely by the 5-tuple can use
zero-length connection IDs, but QUIC version 1 offers such endpoints no way to
advertise a stateless reset token, and so no way to benefit from stateless
reset. This document defines a transport parameter and a frame that let an
endpoint using zero-length connection IDs supply a stateless reset token to its
peer, extending stateless reset to those endpoints.


--- middle

# Introduction

Stateless reset ({{Section 10.3 of !RFC9000}}) allows an endpoint that has lost
connection state to signal to its peer that a connection can no longer be
continued. An endpoint associates a Stateless Reset Token with each Connection
ID it issues, delivering the token to the peer either in the
stateless_reset_token transport parameter or in a NEW_CONNECTION_ID frame. When
such an endpoint later receives a packet that it cannot associate with any
active connection, it recomputes the token from the Destination Connection ID
carried in that packet -- typically by applying a static key to the Connection
ID ({{Section 10.3.2 of RFC9000}}) -- and emits a Stateless Reset carrying that
token. Because the token is derived from the Connection ID, the resetting
endpoint need not retain any per-connection state; because the token is hard to
guess, an off-path attacker cannot forge a reset.

This mechanism is unavailable to endpoints that use zero-length Connection IDs.
Such an endpoint has no Connection ID from which a token could be derived, and
QUIC version 1 provides no way for it to advertise a token by other means: the
stateless_reset_token transport parameter cannot be sent by a client, and
NEW_CONNECTION_ID frames cannot be sent by an endpoint that issues zero-length
Connection IDs ({{Section 5.1.1 of RFC9000}}).

Yet endpoints that issue zero-length Connection IDs are common. An endpoint that
identifies connections solely by the 5-tuple has no need to issue Connection IDs
and can reduce per-packet overhead by using zero-length Connection IDs. This is
typical of client deployments; a representative example is a mobile operating
system on which a client application maintains QUIC connections.

For such deployments, the ability to send a Stateless Reset is valuable
precisely when connection state has been lost involuntarily -- for example, when
the application exits or is terminated by the operating system. After the
application is gone, the peer may continue to send packets on the connection
until its idle timeout expires. If the operating system could send a Stateless
Reset in response to those packets, the peer would learn immediately that the
connection is dead.

The operating system could instead retain a CONNECTION_CLOSE packet recorded by
the application and replay it, but such a packet is difficult to manage. It is
protected under the connection's packet-protection keys and carries a packet
number that must fall within the range the peer will accept; every packet the
application sends advances that state, so the recorded packet must be refreshed
continually to remain usable. A Stateless Reset Token, by contrast, is a fixed
value that remains valid for the lifetime of the connection. The operating
system need only remember the token together with the 5-tuple, and can then emit
resets indefinitely without any further coordination with the application.

This document defines a frame that allows a QUIC endpoint to advertise a
Stateless Reset Token that is not associated with any Connection ID, thereby
extending stateless reset to endpoints that do not issue Connection IDs. Because
the advertised token is bound to the connection rather than to a Connection ID,
the advertising endpoint retains the token instead of recomputing it from a
received Connection ID.


# Conventions and Definitions

{::boilerplate bcp14-tagged}


# The reset_token Transport Parameter

An endpoint that supports this extension and is willing to accept RESET_TOKEN
frames ({{reset-token-frame}}) from its peer advertises the reset_token
transport parameter (0x-TBD).

The reset_token transport parameter has a zero-length value; its presence alone
signals support. An endpoint that receives a reset_token transport parameter
with a non-zero length MUST treat it as a connection error of type
TRANSPORT_PARAMETER_ERROR.

Advertising this transport parameter is a permission for the peer to send
RESET_TOKEN frames; it carries no implication that the advertising endpoint will
itself send them.


# The RESET_TOKEN Frame {#reset-token-frame}

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
endpoint would otherwise derive from the Connection IDs it issues. An endpoint
that uses a non-zero-length Connection ID MUST NOT send a RESET_TOKEN frame. An
endpoint that receives a RESET_TOKEN frame from a peer that uses a
non-zero-length Connection ID MUST treat this as a connection error of type
PROTOCOL_VIOLATION.

An endpoint that does not support this extension treats a received RESET_TOKEN
frame as a frame of unknown type, which is a connection error of type
FRAME_ENCODING_ERROR ({{Section 12.4 of RFC9000}}).

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


# Sending a Stateless Reset {#sending}

{{Section 10.3 of RFC9000}} contemplates an endpoint that has lost the state of
a connection and is reacting to a packet it can no longer associate with one.
Such an endpoint cannot recover the Connection ID that its peer expects, and so
{{Section 10.3 of RFC9000}} observes that the Destination Connection ID of a
Stateless Reset is necessarily a random value, and that the resulting packet
"could be incorrectly routed" where the Connection ID is critical for routing
toward the peer. That is the common case for load-balanced deployments.

An endpoint using this extension is not in that position. It retains the
Stateless Reset Token deliberately, and can retain alongside it a Connection ID
that the peer currently accepts. When constructing a Stateless Reset, such an
endpoint therefore SHOULD use a Connection ID issued by the peer in the position
in which a 1-RTT packet carries the Destination Connection ID field, rather than
unpredictable bits. Doing so allows the Stateless Reset to be routed to the peer
and makes it less distinguishable from a valid packet. A peer that receives such
a datagram is unable to decrypt it and therefore compares its trailing 16 bytes
against the Stateless Reset Tokens it retains, as required by {{Section 10.3.1
of RFC9000}}.

Aside from the choice of the Destination Connection ID, the requirements of
{{Section 10.3 of RFC9000}} and {{Section 10.3.3 of RFC9000}} governing the
construction and sending of a Stateless Reset apply unchanged.


# Security Considerations

TODO Security


# IANA Considerations

TBD


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
