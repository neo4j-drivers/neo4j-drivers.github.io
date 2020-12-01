---
layout: default
---
# Bolt Handshake Protocol Specification

Bolt is a client-server protocol designed primarily for executing queries against a database server.
Communication occurs through request-response exchanges, in much the same way as HTTP.
Unlike HTTP, however, Bolt connections are stateful.

A Bolt connection always begins with a fixed handshake wherein the client identifies itself as a Bolt client and initiates a version negotiation.
The outcome of this negotiation determines the version of messaging protocol that follows.
Messaging protocols are described elsewhere.

**Note:** *Byte values are represented using hexadecimal notation unless otherwise specified.*


## Endianness

Bolt requires that all values that can vary by endianness should be transmitted using network byte order, also known as [big\-endian](https://en.wikipedia.org/wiki/Endianness#Etymology) byte order.


## Connection & Disconnection

Bolt communication is intended to take place over a TCP connection.
The default port is **TCP 7687** but any port can be used.

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
In this, the client submits exactly **four protocol versions**, each encoded as a **big-endian 32-bit unsigned integer** for a total of **128 bits**.
Protocol version zero can be used as a placeholder if fewer than four versions are required in the exchange.
Should a match be found for a version supported by the server, the response will contain that version encoded as a **single 32-bit integer**.
If no match is found, a zero value is returned followed by immediate closure of the connection by the server.

Within this exchange, a zero value (four zero bytes) always represents "no protocol version".
For the client, this can be used as a filler if fewer than four protocol versions are known.
For the server, this indicates no version match has been found.

A server should assume that the versions contained within a client's request have been sent in order of preference.
Therefore, if a match occurs for more than one version, the first match should be selected.


Example 1: The client are aware of the Bolt protocol versions 1. The server responds with version 1.

```
C: 60 60 B0 17
C: 00 00 00 01 00 00 00 00 00 00 00 00 00 00 00 00
S: 00 00 00 01
```

Example 2: The client are aware of the two Bolt protocol versions 1 and 2. The server responds with version 2. 

```
C: 60 60 B0 17
C: 00 00 00 02 00 00 00 01 00 00 00 00 00 00 00 00
S: 00 00 00 02
```

Example 3: The client are aware of the three Bolt protocol versions 1, 2 and 3. The server responds with version 2. 

```
C: 60 60 B0 17
C: 00 00 00 03 00 00 00 02 00 00 00 01 00 00 00 00
S: 00 00 00 02
```

Example 4: The client are aware of the Bolt protocol versions 3. The server responds with no version, the server do not support communication with the client.

```
C: 60 60 B0 17
C: 00 00 00 03 00 00 00 00 00 00 00 00 00 00 00 00
S: 00 00 00 00
```


With Bolt version 4.0 the version scheme now supports Major and Minor versioning number. The first 16 bits are reserved. 8 bits represents the Minor version. 8 bits represents the Major version.

Example 5: Version 4.1.

```
00 00 01 04
```


Example 6: The client are aware of the three Bolt protocols 3, 4.0, and 4.1. The server responds with version 4.1.

```
C: 60 60 B0 17
C: 00 00 01 04 00 00 00 04 00 00 00 03 00 00 00 00
S: 00 00 01 04
```


## Bolt Message Specification

A connection may be used for general application-specific messaging following a successful handshake.
The Bolt protocol communicates with specific messages that are versioned, see the message specification.
