# Bolt Protocol Message Specification - Version 3

* [**Version 3**](#version-3)


**NOTE:** Byte values are represented using hexadecimal notation unless otherwise specified.


# Version 3

Added BEGIN, COMMIT, ROLLBACK
INIT became HELLO, added GOODBYE
Removed ACK_FAILURE (using RESET instead)
Added extra metadata field to RUN and BEGIN

## Deltas

The changes compared to Bolt protocol version 2 are listed below:

* The `INIT` message has been replaced with `HELLO` message.
* The `ACK_FAILUER` message has been removed. Use `RESET` message instead.
* Added `extra::Map` field to `RUN` message.
* Added `extra::Map` field to `BEGIN` message.
* New `HELLO` message.
* New `GOODBYE` message.
* New `BEGIN` message.
* New `COMMIT` message.
* New `ROLLBACK` message.
* New `RESET` message.

## Overview of Major Version Changes

Several new messages have been introduced to support new features.
The `INIT` message have been replaced with the `HELLO` message that have a more generic structure.

The new request message `GOODBYE` have been introduced to be able to tell the server that the connection have been closed. 

In Bolt protocol version 3 the concept of **Auto-commit Transaction** and **Explicit Transaction** have been introduced.
**Auto-commit Transaction** is the concept of being in the `READY` server state and transition to the `STREAMING` server state.
A new **Explicit Transaction** is created with the request message `BEGIN` and closed with the request message `COMMIT`.
See, [**Bolt Protocol Server State Specification Version 3**](bolt-protocol-server-state-specification-3.md#appendix---bolt-message-state-transitions)

The request message `ACK_FAILURE` has been removed and to reset the connection the new request message `RESET` should be used.
The `RESET` message will trigger an `<INTERRUPT>` signal on the server.


## Bolt Protocol Server State Specification

For the server, each connection using the Bolt Protocol will occupy one of several states throughout its lifetime.

This state is used to determine what actions may be undertaken by the client.

See, [**Bolt Protocol Server State Specification Version 3**](bolt-protocol-server-state-specification-3.md)


### Server Signals

Jump ahead is that the signal will imediatly be available before any messages are processed in the message queue.

| Server Signal   | Jump Ahead | Description            |
|:----------------|:----------:|------------------------|
| `<INTERRUPT>`   | x          | an interrupt signal    |
| `<DISCONNECT>`  |            | a disconnect signal    |


### Protocol Errors

If a server or client receives a message type that is unexpected, according to the transitions described in this document, it must treat that as a protocol error.
Protocol errors are fatal and should immediately transition the server state to `DEFUNCT`, closing any open connections.

## Message Exchange

Messages are exchanged in a request-response pattern between client and server.

Each request consists of exactly one message and each response consists of zero or more detail messages followed by exactly one summary message.
The presence or absence of detail messages in a response is directly related to the type of request message that has been sent.
In other words, some request message types elicit a response that may contain detail messages, others do not.

Messages may also be pipelined. In other words, clients may send multiple requests eagerly without first waiting for responses.

When a failure occurs in this scenario, servers **must** ignore all subsequent requests until the client has explicitly acknowledged receipt of the failure.
This prevents inadvertent execution of queries that may not be valid.

More details of this process can be found in the sections below.


### Serialization

Messages and their contents are serialized into network streams using [**PackStream Specification Version 1**](../packstream/packstream-specification-1.md).

**Each message is represented as a PackStream structure**, that contains a fixed number of fields.

The message type is denoted by the PackStream structure **tag byte** and each message is defined in the Bolt protocol.

**Serialization is specified with PackStream Version 1.**


### Chunking

A layer of chunking is also applied to message transmissions as a way to more predictably manage packets of data.

The chunking process allows the message to be broken into one or more pieces, each of an arbitrary size, and for those pieces to be transmitted within separate chunks.

Each chunk consists of a **two-byte header**, detailing the chunk size in bytes followed by the chunk data itself.
Chunk headers are **16-bit unsigned integers**, meaning that the maximum theoretical chunk size permitted is 65,535 bytes.

Each encoded message **must** be terminated with a chunk of zero size, i.e.

```
00 00
```

This is used to signal message boundaries to a receiving parties, allowing blocks of data to be fully received without requiring that the message is parsed immediately.
This also allows for unknown message types to be received and handled without breaking the messaging chain.

The Bolt protocol encodes each message using a chunked transfer encoding.

* Each message is transferred as one or more chunks of data.
* Each chunk starts with a two-byte header, an unsigned big-endian 16-bit integer, representing the size of the chunk not including the header.
* A message can be divided across multiple chunks, allowing client and server alike to transfer large messages without having to determine the length of the entire message in advance.
* Chunking applies on each message individually.
* One chunk cannot contain more than one message.
* Each message ends with two bytes with the value `00 00`, these are not counted towards the chunk size. (Or you can think they are individual chunks of size 0)


This section provides some examples to illustrate how Bolt chunks messages.

Example: **A message in one chunk**

Message data containing 16 bytes,

```
00 01 02 03 04 05 06 07 08 09 0A 0B 0C 0D 0E 0F
```

The chunk,

```
00 10 00 01 02 03 04 05 06 07 08 09 0A 0B 0C 0D 0E 0F 00 00
```

chunk header: `00 10`

end marker: `00 00`


Example: **A message split in two chunks**

Message data containig 20 bytes:

```
00 01 02 03 04 05 06 07 08 09 0A 0B 0C 0D 0E 0F 01 02 03 04
```

chunk 1,
chunk 2,

```
00 10 00 01 02 03 04 05 06 07 08 09 0A 0B 0C 0D 0E 0F
00 04 01 02 03 04 00 00
```

chunk 1 header: `00 10`

chunk 1 end marker: no end marker, still message data

chunk 2 header: `00 04`

chunk 2 end marker: `00 00`


Example: **Two messages**


Message 1 data containing 16 bytes:

```
00 01 02 03 04 05 06 07 08 09 0A 0B 0C 0D 0E 0F
```

Message 2 data containing 8 bytes:

```
0F 0E 0D 0C 0B 0A 09 08
```

The two messages encoded with chunking,

```
00 10 00 01 02 03 04 05 06 07 08 09 0A 0B 0C 0D 0E 0F 00 00
00 08 0F 0E 0D 0C 0B 0A 09 08 00 00
```


### Pipelining

The client may send multiple requests eagerly without first waiting for responses.


### Transaction

A [transaction](https://en.wikipedia.org/wiki/Database_transaction) is the concept of [atomic](https://en.wikipedia.org/wiki/Atomicity_(database_systems)) units of work.

An **Auto-commit Transaction** is a basic but limited form of transaction.

The concept of **Auto-commit Transaction** is when the server is in the `READY` state and the transaction is opened with the request message `RUN` and the response of a summary message `SUCCESS`.

The **Auto-commit Transaction** is successfully closed with the summary message `SUCCESS` for the request message `PULL_ALL` or the request message `DISCARD_ALL`.
Thus the **Auto-commit Transaction** can only contain one `RUN` request message.

Example 1:

```
...
C: HELLO ...
S: SUCCESS ...  // Server is in READY state

C: RUN ...      // Open a new Auto-commit Transaction
S: SUCCESS ...  // Server is in STREAMING state

C: PULL_ALL ...
S: RECORD ...
   ...
S: RECORD ...
S: SUCCESS ... // Server is in READY state and this implies that the Auto-commit Transaction is closed.
```

See, [**Bolt Protocol Server State Specification Version 3**](bolt-protocol-server-state-specification-3.md#appendix---bolt-message-state-transitions)

An **Explicit Transaction** is a more generic transaction that can contain several `RUN` request messages.

The concept of **Explicit Transaction** is when the server is in the `READY` state and the transaction is opened with the request message `BEGIN` and the response of a summary message `SUCCESS` (thus transition into the `TX_READY` server state).

The **Explicit Transaction** is successfully closed with the request message `COMMIT` and the response of a summary message `SUCCESS`.
The result stream (detail messages) must be fully consumed or discarded by a client before the server can transition to the `TX_READY` state and thus be able to close the transaction with a `COMMIT` request message.

The **Explicit Transaction** can be gracefully discarded and set to the initial server state of `READY` with the request message `ROLLBACK`.


Example 2:

```
...
C: HELLO ...
S: SUCCESS ...  // Server is in READY state

C: BEGIN ...    // Open a new Explicit Transaction
S: SUCCESS ...  // Server is in TX_READY state

C: RUN ...
S: SUCCESS ... // Server is in TX_STREAMING state, first stream is open

C: PULL_ALL ...
S: RECORD ...
   ...
S: RECORD ...
S: SUCCESS ... // Server is in TX_READY state, first stream has been fully consumed

C: RUN ...
S: SUCCESS ... // Server is in TX_STREAMING state, second stream is open

C: PULL_ALL ...
S: RECORD ...
   ...
S: RECORD ...
S: SUCCESS ... // Server is in TX_READY state, second stream has been fully consumed

C: COMMIT   // Close the Explicit Transaction
S: SUCCESS  // Server is in READY state
```

See, [**Bolt Protocol Server State Specification Version 3**](bolt-protocol-server-state-specification-3.md#appendix---bolt-message-state-transitions)


## Messages

| Message       | Signature | Request Message | Summary Message | Detail Message | Fields                                                                                                            | Description                                             |
|---------------|:---------:|:---------------:|:---------------:|:--------------:|-------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------|
| `HELLO`       | `01`      | x               |                 |                | `extra::Dictionary(user_agent::String, scheme::String, principal::String, credentials::String)`                   | initialize connection                                   |
| `GOODBYE`     | `02`      | x               |                 |                |                                                                                                                   | close the connection, triggers a `<DISCONNECT>` signal  |
| `RESET`       | `0F`      | x               |                 |                |                                                                                                                   | reset the connection, triggers a `<INTERRUPT>` signal   |
| `RUN`         | `10`      | x               |                 |                | `query::String`, `parameters::Dictionary`, `extra::Dictionary`                                                    | execute a query                                         |
| `DISCARD_ALL` | `2F`      | x               |                 |                |                                                                                                                   | discard all records                                     |
| `PULL_ALL`    | `3F`      | x               |                 |                |                                                                                                                   | fetch all records                                       |
| `BEGIN`       | `11`      | x               |                 |                | `extra::Dictionary(bookmarks::List<String>, tx_timeout::Integer, tx_metadata::Dictionary, mode::String)`          | begin a new transaction                                 |
| `COMMIT`      | `12`      | x               |                 |                |                                                                                                                   | commit a transaction                                    |
| `ROLLBACK`    | `13`      | x               |                 |                |                                                                                                                   | rollback a transaction                                  |
| `SUCCESS`     | `70`      |                 | x               |                | `metadata::Dictionary`                                                                                            | request succeeded                                       |
| `IGNORED`     | `7E`      |                 | x               |                |                                                                                                                   | request was ignored                                     |
| `FAILURE`     | `7F`      |                 | x               |                | `metadata::Dictionary(code::String, message::String)`                                                             | request failed                                          |
| `RECORD`      | `71`      |                 |                 | x              | `data::List`                                                                                                      | data values                                             |



# Appendix - Message Exchange Examples

* The `C:` stands for client.
* The `S:` stands for server.


## Example 1

```
C: 60 60 B0 17
C: 00 00 00 03 00 00 00 00 00 00 00 00 00 00 00 00
S: 00 00 00 03
C: HELLO {"user_agent": "Example/3.0.0", "scheme": "basic", "principal": "user", "credentials": "password"}
S: SUCCESS {"server": "Neo4j/3.5.0", "connection_id": "example-connection-id:1"}
C: GOODBYE
```

## Example 2

```
C: 60 60 B0 17
C: 00 00 00 03 00 00 00 00 00 00 00 00 00 00 00 00
S: 00 00 00 03
C: HELLO {"user_agent": "Example/3.0.0", "scheme": "basic", "principal": "user", "credentials": "password"}
S: SUCCESS {"server": "Neo4j/3.5.0", "connection_id": "example-connection-id:1"}
C: RUN "RETURN $x AS example" {"x": 123} {"mode": "r"}
S: SUCCESS {"fields": ["example"]}
C: PULL_ALL
S: RECORD [123]
S: SUCCESS {"bookmark": "example-bookmark:1", "t_last": 300, "type": "r"}
C: GOODBYE
```
