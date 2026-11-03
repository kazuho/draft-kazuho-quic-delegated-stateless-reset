---
title: "Delegating QUIC Stateless Resets"
category: std
docname: draft-kazuho-quic-delegated-stateless-reset-latest
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
 -
    fullname: Stefano Duo
    org: Google
    email: stefanoduo@google.com
 -
    fullname: Joseph Beshay
    org: Meta
    email: jbeshay@meta.com
 -
    fullname: Matt Joras
    org: Meta
    email: mjoras@meta.com

normative:

informative:

...

--- abstract

This document specifies how a QUIC endpoint can delegate sending Stateless
Resets to a component that outlives its connection state. It also defines a
frame for clients using zero-length Connection IDs to advertise a Stateless
Reset Token and a transport parameter for servers to indicate support for
receiving the frame.


--- middle

# Introduction

Stateless reset ({{Section 10.3 of !RFC9000}}) allows an endpoint that has lost
connection state to signal to its peer that a connection can no longer be
continued. Nothing a Stateless Reset carries is protected by the connection's
keys or tied to its packet number spaces: the signal is authenticated solely by
the 128-bit Stateless Reset Token it ends with, which the receiver matches
against the tokens it retains.

QUIC uses that property to avoid retaining state. An endpoint
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

The ability to send a Stateless Reset is valuable precisely when connection
state has been lost involuntarily -- for example, when an application exits or
is terminated by the operating system. After the application is gone, the peer
may buffer data that will never be delivered and continue to send packets on
the connection until its idle timeout expires. It can also delay fallback
delivery, such as a push notification or a secondary connection. At scale,
abandoned connections consume memory and connection-table capacity. To cover
these scenarios, an endpoint can delegate the termination of the connection to
a component that outlives its own connection state, and that component can
reset the peer long after the endpoint is gone ({{delegation}}).

Delegation is optional. A delegate, such as an operating system, system service,
or application helper, needs to survive the loss of connection state and retain
access to the connection's network path.
A mobile operating system could withdraw an application's network access before
the application can send a CONNECTION_CLOSE frame. Once the connection has been
abandoned, a delegate that retains network access can reset the peer
({{proactive}}).
If a reset cannot be delivered, the
peer relies on the usual QUIC termination and timeout mechanisms.

A CONNECTION_CLOSE packet could be recorded for a delegate to send, but such a
packet is difficult to manage. It is protected under the
connection's packet-protection keys that endpoints periodically rotate, and it
carries a packet number that must fall within a dynamic range the peer will accept;
thus, the recorded packet would need constant refreshing to remain usable.
Moreover, because the packet is encrypted, a delegate that does not trust the
endpoint cannot verify that it only closes the connection.

In contrast, a Stateless Reset does not depend on packet-protection keys or
packet numbers, and its contents can be generated or verified by the delegate
({{division-of-labor}}). The delegate retains the token and the information
needed to match and route packets ({{delegation}}).

A delegated Stateless Reset can also be better formed than one sent by an
endpoint that has lost its state: the delegate can remember a Connection ID that
the peer has issued and place it where a 1-RTT packet carries the Destination
Connection ID, so that the Stateless Reset reaches the intended peer in a
load-balanced deployment ({{constructing}}).

In {{RFC9000}}, Stateless Reset Tokens are issued alongside Connection IDs.
An endpoint that uses zero-length Connection IDs cannot issue one in a
NEW_CONNECTION_ID frame ({{Section 5.1.1 of RFC9000}}). Only a server can provide a
token for its handshake Connection ID in the stateless_reset_token transport
parameter; clients cannot because client transport parameters are not
confidential.

Zero-length Connection IDs are common for clients. A client that identifies connections
solely by the 5-tuple has no need to issue Connection IDs and can reduce
per-packet overhead by using zero-length Connection IDs. To allow such clients
to issue Stateless Reset Tokens and delegate them, this document defines a
frame for advertising a token and a transport parameter through which servers
indicate support for receiving that frame ({{advertising}}).
Servers can advertise a token in the existing stateless_reset_token transport
parameter; the new frame is limited to clients that use zero-length Connection
IDs.

The mechanisms in this document apply to QUIC versions 1 and 2 ({{!RFC9369}}),
which use the same Stateless Reset format and short-header fields.


# Conventions and Definitions

{::boilerplate bcp14-tagged}


# Delegating a Stateless Reset {#delegation}

An endpoint can delegate the sending of a Stateless Reset to a component that
outlives its connection state. The delegate retains a registration containing:

* Information the endpoint uses to match incoming datagrams to the connection,
  such as endpoint-issued Connection IDs or routing information
  ({{Section 5.2 of RFC9000}}).
* A return path and a peer-issued Connection ID that the peer currently accepts
  ({{constructing}}).
* A Stateless Reset Token that the endpoint has advertised, or will advertise,
  to the peer ({{division-of-labor}}).

The delegate retains the token rather than recomputing it from a Connection ID.
It needs no packet-protection keys or packet numbers.

Both clients and servers can delegate resets, regardless of the length of their
Connection IDs. Tokens are advertised in the server-only stateless_reset_token
transport parameter or in NEW_CONNECTION_ID frames. A client using zero-length
Connection IDs can advertise a token for the whole connection in RESET_TOKEN
frames ({{advertising}}).

Registration can precede token advertisement, but the peer recognizes a reset
only after learning the token and associating it with the reset's source
address ({{Section 10.3.1 of RFC9000}}). For tokens associated with Connection
IDs, the peer also needs to have used and not retired the ID. A reset carrying
any other token is indistinguishable from a packet that cannot be decrypted and
is discarded. A delegate MAY track when the peer starts using a Connection ID,
and send a Stateless Reset carrying its token only once that happens.

The endpoint updates a registration when its matching information, peer-issued
Connection ID, or path changes. A reset sent with stale routing information
might be dropped.

Tokens associated with endpoint-issued Connection IDs remain bound to those
IDs when the path or peer-issued Connection ID changes. A token advertised
using RESET_TOKEN remains fixed for the connection's lifetime
({{reset-token-frame}}).

A connection can have registrations for multiple paths. Closing a socket or
losing a path does not authorize resetting a connection that remains active
on another path ({{proactive}}). A delegate MUST NOT use a canceled or expired
registration to send resets.

Once the endpoint has lost or abandoned connection state,
the delegate can send a Stateless Reset proactively ({{proactive}}), and
thereafter responds to datagrams matching the registration by sending it again,
but does so only if the Stateless Reset is smaller than the datagram that
triggered it, as required by {{Section 10.3.3 of RFC9000}}.

The delegate limits how long it retains a registration
after the endpoint loses or abandons connection state. A retained
registration applies only to the old connection. It MUST NOT prevent delivery
to a new connection that reuses the same routing information.

This document does not define an interface between an endpoint and a delegate.

Except as described in {{constructing}} and {{proactive}}, the requirements of
{{Section 10.3 of RFC9000}} and {{Section 10.3.3 of RFC9000}} governing the
construction and sending of a Stateless Reset apply unchanged.

## Token Generation and Packet Construction {#division-of-labor}

An endpoint and its delegate can divide the work of generating a Stateless
Reset Token and constructing a Stateless Reset according to their trust
relationship. When the delegate trusts the endpoint, such as when the delegate
is an application helper, the endpoint can generate the token itself and
provide it to the delegate as part of the registration ({{delegation}}).

When the endpoint and the delegate cross a privilege boundary, such as when the
delegate is an operating system that can withdraw the endpoint's network
access, the delegate can instead generate the token ({{fig-delegation-flow}}).
When the endpoint registers a new connection or a new Connection ID it issues,
the delegate forms and retains a Stateless Reset datagram ({{constructing}})
and returns the Stateless Reset Token to the endpoint to advertise to its peer.

~~~ aasvg
+-----------------------------------------------------------------+
|                                                                 |
|    Endpoint                Delegate                   Peer      |
|        |                      |                         |       |
|        |------ Register ----->|                         |       |
|        |     (Peer CID,       |                         |       |
|        |      Path, etc.)     |                         |       |
|        |                      | (Generates Reset Token  |       |
|        |                      |  & Reset Packet)        |       |
|        |<--- Reset Token -----|                         |       |
|        |                      |                         |       |
|        |--------------- Advertise Token --------------->|       |
|        |       (RESET_TOKEN or NEW_CONNECTION_ID)       |       |
|        |                      |                         |       |
|  +---------------------------------------------------+  |       |
|  |  Optional (if path or Peer CID changes):          |  |       |
|  |     |                      |                      |  |       |
|  |     |----- Update -------->|                      |  |       |
|  |     |  (Peer CID, Path)    | (Keeps Reset Token;  |  |       |
|  |     |                      |  Rebuilds Reset      |  |       |
|  |     |                      |  Packet)             |  |       |
|  +---------------------------------------------------+  |       |
|        |                      |                         |       |
|  [State Lost]                 |---- Proactive Reset --->|       |
|                               |   (Usually sufficient)  |       |
|                               |                         |       |
|                       +--------------------------------------+  |
|                       |  Optional (if more packets arrive):  |  |
|                       |       |                         |    |  |
|                       |       |<-- Incoming Datagram ---|    |  |
|                       |       |                         |    |  |
|                       |       |---- Stateless Reset --->|    |  |
|                       |       |   (If smaller than Rx)  |    |  |
|                       |       |                         |    |  |
|                       +--------------------------------------+  |
|                               |                         |       |
|                               | (Discards Reset         |       |
|                               |  & Registration)        |       |
|                                                                 |
+-----------------------------------------------------------------+
~~~
{: #fig-delegation-flow title="Delegating a Stateless Reset with a Delegate-Generated Token"}

In this division of labor ({{fig-division-of-labor}}), it is possible to deny
an endpoint the use of the Stateless Reset as a means of conveying
information of its own. The Stateless Reset Token and the other bytes that the
delegate generates ({{constructing}}) are outside the endpoint's control, and the
peer's Connection ID, which the endpoint supplies, is also the value that
it places in the Destination Connection ID field of the packets it sends. A
delegate can therefore monitor the packets that the endpoint sends
after the registration, and send the Stateless Reset only if those packets
carried that Connection ID. The Stateless Reset then carries no Connection ID
other than the one that the endpoint's own packets were already carrying to
the peer.

~~~ aasvg
+-------------------------------------------------------------------+
|                                                                   |
|   Stateless Reset Packet Layout           Source / Origin         |
|                                                                   |
|   +---+---+-----------------------+                               |
|   | 0 | 1 | Unpredictable (6 bits)| <---- Delegate (First Byte):  |
|   +---+---+-----------------------+         0: Short Header Form  |
|   |                               |         1: Fixed Bit          |
|   | Destination Connection ID     |         6 random bits         |
|   | (variable length)             |                               |
|   |                               | <---- Endpoint                |
|   |                               |       (Peer's active CID)     |
|   +-------------------------------+                               |
|   |                               |                               |
|   | Unpredictable Payload         | <---- Delegate (Random bytes) |
|   | (4 bytes)                     |                               |
|   |                               |                               |
|   +-------------------------------+                               |
|   |                               |                               |
|   | Stateless Reset Token         | <---- Delegate (Random token) |
|   | (16 bytes)                    |                               |
|   |                               |                               |
|   +-------------------------------+                               |
|                                                                   |
+-------------------------------------------------------------------+
~~~
{: #fig-division-of-labor title="Division of labor between endpoint and delegate"}


# Constructing a Routable Stateless Reset {#constructing}

When constructing a Stateless Reset, the endpoint or its delegate SHOULD place a
Connection ID that the peer currently accepts in the position in which a 1-RTT
packet carries the Destination Connection ID field.

If the peer uses a zero-length Connection ID, the Destination Connection ID
field is empty.

{{Section 10.3 of RFC9000}} calls for unpredictable bits, rather than the
Destination Connection ID, because an endpoint that has lost its state cannot
recover a Connection ID that its peer accepts, observing that the resulting packet
"could be incorrectly routed" where the Connection ID is critical for routing toward
the peer, as in load-balanced deployments. Because a delegate retains state, it can
send a Stateless Reset that reaches the peer and is less distinguishable from a valid
packet.

To maintain indistinguishability from a valid 1-RTT packet, the endpoint or
delegate MUST include at least 4 unpredictable bytes following the Destination
Connection ID. Doing so satisfies both the 21-byte minimum size requirement of
{{Section 10.3 of RFC9000}}, when zero-length Connection IDs are used, and provides
the 4-byte offset required by {{Section 5.4.2 of !RFC9001}}.

~~~
Stateless Reset {
  Fixed Bits (2) = 1,
  Unpredictable Bits (6),
  Destination Connection ID (0..160),
  Unpredictable Bits (32..),
  Stateless Reset Token (128),
}
~~~
{: #fig-reset-format title="Routable Stateless Reset Format"}

# Sending a Stateless Reset Proactively {#proactive}

An endpoint that retains the state necessary to send a CONNECTION_CLOSE frame
MUST use CONNECTION_CLOSE instead of a Stateless Reset, as required by
{{Section 11 of RFC9000}}.

Once the endpoint that delegated the Stateless Reset has lost or abandoned its
connection state, a delegate MAY send a single Stateless Reset proactively,
without waiting to receive a datagram from the peer. {{Section 10.3 of RFC9000}}
describes a Stateless Reset only as a response to a received packet because an
endpoint that has lost its state learns of the connection's existence only from
one; an endpoint or delegate that retains the token knows which connections it
may need to reset before any packet arrives.

A delegate MUST NOT send a Stateless Reset unless the endpoint has lost or
abandoned its connection state. Loss of one path does not by itself imply
abandonment. Once delegated teardown begins, the endpoint MUST NOT resume that
connection.


# Advertising a Token with Zero-Length Connection IDs {#advertising}

The RESET_TOKEN frame lets a client using zero-length Connection IDs advertise
a Stateless Reset Token. Servers indicate support with the reset_token
transport parameter. The token is bound to the connection and can be delegated
as described in {{delegation}}.


## The reset_token Transport Parameter {#reset-token-tp}

A server that is willing to accept RESET_TOKEN frames
({{reset-token-frame}}) from its client advertises the reset_token transport
parameter (0x-TBD).

The reset_token transport parameter has a zero-length value; its presence alone
conveys that willingness. A client MUST NOT send this transport parameter.
Receipt of this parameter from a client, or receipt of a non-zero value, MUST be
treated as a connection error of type TRANSPORT_PARAMETER_ERROR.


## The RESET_TOKEN Frame {#reset-token-frame}

The RESET_TOKEN frame (type 0x-TBD) carries a Stateless Reset Token that a
client associates with the connection rather than with any Connection ID.

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

A client MUST NOT send a RESET_TOKEN frame unless the server advertised the
reset_token transport parameter.

Only a client that uses a zero-length Connection ID may send a RESET_TOKEN
frame; the mechanism substitutes for the tokens that a Connection ID-issuing
client sends in NEW_CONNECTION_ID frames. A client that uses a non-zero-length
Connection ID MUST NOT send a RESET_TOKEN frame. A server that receives a
RESET_TOKEN frame from such a client MUST treat this as a connection error of
type PROTOCOL_VIOLATION.

An endpoint that has not advertised the reset_token transport parameter treats a
received RESET_TOKEN frame as a frame of unknown type, which is a connection
error of type FRAME_ENCODING_ERROR ({{Section 12.4 of RFC9000}}). A server MUST
NOT send a RESET_TOKEN frame. Because a client cannot advertise the transport
parameter, it treats a RESET_TOKEN frame from a server as a frame of unknown
type.

A server that receives a RESET_TOKEN frame recognizes the contained value as
a Stateless Reset Token for this connection and thereafter processes it exactly
as a token received in a NEW_CONNECTION_ID frame, following the requirements of
{{Section 10.3 of RFC9000}} for retaining the token and for detecting a
Stateless Reset that carries it, except that this token is immediately usable
and is not subject to the rule that excludes tokens for unused Connection IDs.

A client uses a single Stateless Reset Token for the lifetime of a
connection. It MAY send more than one RESET_TOKEN frame -- for example, to
retransmit a frame that was lost -- but every RESET_TOKEN frame it sends on a
connection MUST carry the same token value. A server that receives a
RESET_TOKEN frame carrying a token value different from one it received earlier
on the same connection MUST treat this as a connection error of type
PROTOCOL_VIOLATION. Receipt of the same value more than once is idempotent. The
token MUST NOT be used for another connection or for a Connection ID.


# Security Considerations

Knowledge of a Stateless Reset Token allows an entity to terminate the
connection. A token MUST be difficult to guess and MUST NOT be reused. A sender
can generate a token using a cryptographically secure random number generator.

Giving a delegate access to the token creates another opportunity for
disclosure, even if it can already terminate the connection. The endpoint needs
to consider this exposure when selecting a delegate.

The token and routing information MUST have confidentiality and integrity in
transit and at rest. The delegate MUST restrict registration creation, update,
cancellation, and access to the endpoint and entities authorized to act for it.

Removing a registration does not revoke a token retained by the delegate. A
token associated with a Connection ID is invalidated when the peer retires that
ID ({{Section 10.3 of RFC9000}}). A token advertised using RESET_TOKEN remains
valid for the connection's lifetime ({{reset-token-frame}}).

A delegate can retain multiple registrations for a connection, such as one for
each path. Implementations SHOULD limit the number of registrations to avoid
resource exhaustion.

A RESET_TOKEN frame is protected by 1-RTT packet protection. A Stateless Reset
reveals its token to an on-path observer. This does not create a new termination
capability because the peer enters the draining state after accepting the
reset. The token cannot be used for another connection.

A delegate sends using the source and destination addresses of a path on which
the endpoint recently received packets from the peer. This allows the peer to
apply the remote-address check in {{Section 10.3.1 of RFC9000}}.

# IANA Considerations

TBD


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
