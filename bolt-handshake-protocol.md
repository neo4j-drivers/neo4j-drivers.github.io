# Bolt Handshake Protocol v1

Bolt is a client-server protocol designed primarily for executing queries against a database server.
Communication occurs through request-response exchanges, in much the same way as HTTP.
Unlike HTTP, however, Bolt connections are stateful.

A Bolt connection always begins with a fixed handshake wherein the client identifies itself as a Bolt client and initiates a version negotiation.
The outcome of this negotiation determines the version of messaging protocol that follows.
Messaging protocols are described elsewhere.

NOTE: Byte values in this document are represented using hexadecimal notation unless otherwise specified.


## Endianness

Bolt requires that all values that can vary by endianness should be transmitted using network byte order, also known as big-endian byte order.


## Connection & Disconnection

Bolt communication is intended to take place over a TCP connection.
The default port is TCP 7687 but any port can be used.

There is no formal shutdown procedure for a Bolt connection.
Either peer may close the connection at TCP level at any time.
Both client and server should be prepared for that to occur and should handle it appropriately.


## Handshake

Immediately following a successful connection, the client MUST initiate a handshake.
This handshake is a fixed exchange used to determine the version of messaging protocol that follows.


### Bolt Identification

The first part of the handshake is used to identify to the server that this is a Bolt connection.
It should be sent by a client immediately following the establishment of a successful connection and does not require a server response.

The identification consists of the following four bytes:

```
C: 60 60 B0 17
```

### Version Negotiation

After identification, a small client-server exchange occurs to determine which version of the messaging protocol to use.
In this, the client submits up to four protocol versions encoded as big-endian 32-bit unsigned integers.
Should a match be found for a version supported by the server, the response will contain that version encoded as a single 32-bit integer.
If no match is found, a zero value is returned followed by immediate closure of the connection by the server.

Within this exchange, a zero value (four zero bytes) always represents "no protocol version".
For the client, this can be used as a filler if fewer than four protocol versions are known.
For the server, this indicates no version match has been found.

A server should assume that the versions contained within a client's request have been sent in order of preference.
Therefore, if a match occurs for more than one version, the first match should be selected.

An example exchange wherein the client and server both understand only protocol version 1 would consist of the following:

```
C: 00 00 00 01 00 00 00 00 00 00 00 00 00 00 00 00
S: 00 00 00 01
```

Where both parties are aware of two protocol versions, the exchange below may be seen instead:

```
C: 00 00 00 02 00 00 00 01 00 00 00 00 00 00 00 00
S: 00 00 00 02
```

## Messaging

A connection may be used for general application-specific messaging following a successful handshake.
Bolt messaging protocols are versioned and described elsewhere.
