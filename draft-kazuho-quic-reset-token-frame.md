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

TODO


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

Yet endpoints that issue zero-length CIDs are common. An endpoint that
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


# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
