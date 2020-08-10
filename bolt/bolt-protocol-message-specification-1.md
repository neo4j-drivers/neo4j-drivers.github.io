# Bolt Protocol Message Specification - Version 1


* [**Version 1**](#version-1)


**NOTE:** Byte values are represented using hexadecimal notation unless otherwise specified.


# Version 1

## Overview

This section describes version 1 of the Bolt messaging protocol.

The message specification describes the message exchanges that take place on a connection following a successful Bolt handshake.

For details of establishing a connection and performing a handshake, see [Bolt Protocol Handshake Specification](bolt-protocol-handshake-specification.md).


## Bolt Protocol Server State Specification

For the server, each connection using the Bolt Protocol will occupy one of several states throughout its lifetime.

This state is used to determine what actions may be undertaken by the client.

See, [Bolt Protocol Server State Specification Version 1](bolt-protocol-server-state-specification-1.md)


### Server Signals

Jump ahead is that the signal will imediatly be available before any messages are processed in the message queue.

| Server Signal   | Jump Ahead | Description            |
|:----------------|:----------:|------------------------|
| `<INTERRUPT>`   | x          | a interrupt signal     |
| `<DISCONNECT>`  |            | a disconnect signal    |


### Protocol Errors

If a server or client receives a message type that is unexpected, according to the transitions described in this document, it must treat that as a protocol error.
Protocol errors are fatal and should immediately transition the server state to `DEFUNCT`, closing any open connections.


## Message Exchange

Messages are exchanged in a request-response pattern between client and server.
Each request consists of exactly one message and each response consists of zero or more detail messages followed by exactly one summary message.
The presence or absence of detail messages in a response is directly related to the type of request message that has been sent.
In other words, some request message types elicit a response that may contain detail messages, others do not.

Messages may also be pipelined.
In other words, clients may send multiple requests eagerly without first waiting for responses.
When a failure occurs in this scenario, servers **must** ignore all subsequent requests until the client has explicitly acknowledged receipt of the failure.
This prevents inadvertent execution of queries that may not be valid.
More details of this process can be found in the sections below.


### Serialization

Messages and their contents are serialized into network streams using [PackStream Specification Version 1](../packstream/packstream-specification-1.md).

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


## Messages

* **Request Message**, the client sends a message.
* **Summary Message**, the server will always respond with one summary message if the connection is still open.
* **Detail Message**, the server will always repond with zero or more detail messages before sending a summary message.


| Message                                         | Signature | Request Message | Summary Message | Detail Message | Fields                                                                                                          | Description                                                      |
|-------------------------------------------------|:---------:|:---------------:|:---------------:|:--------------:|-----------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------|
| [`INIT`](#request-message---init)               | `01`      | x               |                 |                | `user_agent::String`, `auth_token::Dictionary(scheme::String, principal::String, credentials::String)`          | initialize connection                                            |
| [`ACK_FAILURE`](#request-message---ack_failure) | `0E`      | x               |                 |                |                                                                                                                 | acknowledge a failure response                                   |
| [`RESET`](#request-message---reset)             | `0F`      | x               |                 |                |                                                                                                                 | reset the connection, triggers a `<INTERRUPT>` signal            |
| [`RUN`](#request-message---run)                 | `10`      | x               |                 |                | `query::String`, `parameters::Dictionary`                                                                       | execute a query                                                  |
| [`DISCARD_ALL`](#request-message---discard_all) | `2F`      | x               |                 |                |                                                                                                                 | discard all records                                              |
| [`PULL_ALL`](#request-message---pull_all)       | `3F`      | x               |                 |                |                                                                                                                 | fetch all records                                                |
| [`SUCCESS`](#summary-message---success)         | `70`      |                 | x               |                | `metadata::Dictionary`                                                                                          | request succeeded                                                |
| [`IGNORED`](#summary-message---ignored)         | `7E`      |                 | x               |                |                                                                                                                 | request was ignored                                              |
| [`FAILURE`](#summary-message---failure)         | `7F`      |                 | x               |                | `metadata::Dictionary(code::String, message::String)`                                                           | request failed                                                   |
| [`RECORD`](#detail-message---record)            | `71`      |                 |                 | x              | `data::List`                                                                                                    | data values                                                      |


### Request Message - `INIT`

The `INIT` message is a request for the connection to be authorized for use with the remote database.

The `INIT` message uses the structure signature `01` and passes two fields: `user agent` (String) and `auth_token` (Dictionary).

The server **must** be in the `CONNECTED` state to be able to process an `INIT` request.
For any other states, receipt of an `INIT` request **must** be considered a protocol violation and lead to connection closure.

Clients should send `INIT` requests to the network immediately after connection and process the corresponding response before using that connection in any other way.

A receiving server may choose to register or otherwise log the user agent but may also ignore it if preferred.

The auth token should be used by the server to determine whether the client is permitted to exchange further messages.
If this authentication fails, the server **must** respond with a `FAILURE` message and immediately close the connection.
Clients wishing to retry initialization should establish a new connection.

**Signature:** `01`

**Fields:**

```
user_agent::String,
auth_token::Dictionary(
  scheme::String,
  principal::String,
  credentials::String,
)
```

  - The `user_agent` should conform to `"Name/Version"` for example `"Example/1.0.0"`. (see, [developer.mozilla.org/en-US/docs/Web/HTTP/Headers/User-Agent](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/User-Agent))
  - The `scheme` is the authentication scheme. Predefined schemes are `“none”`, `“basic”`.


**Detail Messages:**

No detail messages should be returned.

**Valid Summary Messages:**

* `SUCCESS`
* `FAILURE`


#### Synopsis

```
INIT "user_agent" {auth_token}
```

Example:

```
INIT "Example/1.0.0" {"scheme": "basic", "principal": "neo4j", "credentials": "password"}
```


#### Server Response `SUCCESS`

A `SUCCESS` message response indicates that the client is permitted to exchange further messages.
Servers can include metadata that describes details of the server environment and/or the connection.

The following fields are defined for inclusion in the `SUCCESS` metadata.

  - `server` (e.g. `"Neo4j/3.4.0"`)

Example:

```
SUCCESS {"server": "Neo4j/3.4.0"}
```


#### Server Response `FAILURE`

A `FAILURE` message response indicates that the client is not permitted to exchange further messages.

Servers may choose to include metadata describing the nature of the failure but **must** immediately close the connection after the failure has been sent.

Example:

```
FAILURE {"message": "example failure", "code": "Example.Failure.Code"}
```


### Request Message - `ACK_FAILURE`

`ACK_FAILURE` signals to the server that the client has acknowledged a previous failure and should return to a `READY` state.

**Signature:** `0E`

**Fields:**

No fields.

**Detail Messages:**

No detail messages should be returned.

**Valid Summary Messages:**

* `SUCCESS`
* `FAILURE`

The server **must** be in a `FAILED` state to be able to successfully process an `ACK_FAILURE` request.
For any other states, receipt of an `ACK_FAILURE` request will be considered a protocol violation and will lead to connection closure.

#### Synopsis

```
ACK_FAILURE
```

Example:

```
ACK_FAILURE
```

#### Server Response `SUCCESS`

If an `ACK_FAILURE` request has been successfully received, the server should respond with a `SUCCESS` message and enter the `READY` state.

The server may attach metadata to the `SUCCESS` message.

Example:

```
SUCCESS {}
```

#### Server Response `FAILURE`

If an `ACK_FAILURE` request is received while not in the `FAILED` state, the server should respond with a `FAILURE` message and immediately close the connection.

The server may attach metadata to the message to provide more detail on the nature of the failure.

Example:

```
FAILURE {"message": "example failure", "code": "Example.Failure.Code"}
```


### Request Message - `RESET`

The `RESET` message requests that the connection should be set back to its initial `READY` state, as if an `INIT` had just successfully completed.

The `RESET` message is unique in that, on arrival at the server, it splits into two separate signals.

Firstly, an `<INTERRUPT>` **signal jumps ahead in the message queue**, stopping any unit of work that happens to be executing, and putting the state machine into an `INTERRUPTED` state.

Secondly, the `RESET` queues along with all other incoming messages and is used to put the state machine back to `READY` when its turn for processing arrives.

This essentially means that the `INTERRUPTED` state exists only transitionally between the arrival of a `RESET` in the message queue and the later processing of that `RESET` in its proper position.

The `INTERRUPTED` state is therefore the only state to automatically resolve without any further input from the client and whose entry does not generate a response message.

**Signature:** `0F`

**Fields:**

No fields.

**Detail Messages:**

No detail messages should be returned.

**Valid Summary Messages:**

* `SUCCESS`
* `FAILURE`


#### Synopsis

```
RESET
```

Example:

```
RESET
```

#### Server Response `SUCCESS`

If a `RESET` message request has been successfully received, the server should respond with a `SUCCESS` message and enter the `READY` state.


Example:

```
SUCCESS {}
```

#### Server Response `FAILURE`

If `RESET` message is received before the server enters a `READY` state, it should trigger a `FAILURE` followed by immediate closure of the connection.
Clients receiving a `FAILURE` in response to `RESET` should treat that connection as `DEFUNCT` and dispose of it.

Example:

```
FAILURE {"message": "example failure", "code": "Example.Failure.Code"}
```


### Request Message - `RUN`

A `RUN` message submits a new query for execution, the result of which will be consumed by a subsequent message, such as `PULL_ALL`.

**PackStream Structure Signature:** `10`

**Fields:**

```
query::String,
parameters::Dictionary
```

**Detail Messages:**

No detail messages should be returned.

**Valid Summary Messages:**

* `SUCCESS`
* `IGNORED`
* `FAILURE`

The server **must** be in a `READY` state to be able to successfully process a `RUN` request.
If the server is in a `FAILED` or `INTERRUPTED` state, the request will be `IGNORED`.
For any other states, receipt of a `RUN` request will be considered a protocol violation and will lead to connection closure.

The query generally contains a [Cypher](https://neo4j.com/docs/cypher-manual/current/) query or a remote procedure call.
This document does not specify how the server should handle an incoming query.

The parameters generally contain variable fields for a database query or remote procedure call.
This document does not specify how the server should handle incoming parameters.

#### Synopsis

```
RUN "query" {parameters}
```

Example:

```
RUN "RETURN $a AS x" {"a": 1}
```

#### Server Response `SUCCESS`

If a `RUN` request has been successfully received and is considered valid by the server, the server should respond with a `SUCCESS` message and enter the *STREAMING* state.
The server may attach metadata to the message to provide header detail for the results that follow.

Clients should not consider a `SUCCESS` response to indicate completion of the execution of that query, merely acceptance of it.

The following fields are defined for inclusion in the metadata.

- `fields` (e.g. `["name", "age"]`)
- `result_available_after` (e.g. `123`)

Example:

```
SUCCESS {"fields": ["x"], "result_available_after": 123}
```

#### Server Response `IGNORED`

A server that receives a `RUN` request while `FAILED` state or `INTERRUPTED` state should respond with an `IGNORED` message and discard the request without processing it.
No state change should occur.

Example:

```
IGNORED
```

#### Server Response `FAILURE`

If a `RUN` message cannot be processed successfully or is invalid, the server should respond with a `FAILURE` message and enter the `FAILED` state.
The server may attach metadata to the message to provide more detail on the nature of the failure.

Example:

```
FAILURE {"message": "example failure", "code": "Example.Failure.Code"}
```

### Request Message - `DISCARD_ALL`

The `DISCARD_ALL` message issues a request to discard the outstanding result and return to a `READY` state.

A receiving server should not abort the request but continue to process it without streaming any detail messages back to the client.

**Signature:** `2F`

**Fields:**

No fields

**Detail Messages:**

No detail messages should be returned.

**Valid Summary Messages:**

* `SUCCESS`
* `IGNORED`
* `FAILURE`

The server **must** be in a `STREAMING` state to be able to successfully process a `DISCARD_ALL` request.
If the server is in a `FAILED` state or `INTERRUPTED` state, the request will be `IGNORED`.
For any other states, receipt of a `DISCARD_ALL` request will be considered a protocol violation and will lead to connection closure.

#### Synopsis

```
DISCARD_ALL
```


Example:

```
DISCARD_ALL
```

#### Server Response `SUCCESS`

If a `DISCARD_ALL` request has been successfully received, the server should respond with a `SUCCESS` message and enter the `READY` state.
The server may attach metadata to the message to provide footer detail for the discarded results.

The following fields are defined for inclusion in the metadata.

- `bookmark` (e.g. `"bookmark:1234"`)
- `result_consumed_after` (e.g. `123`)

Example:

```
SUCCESS {"bookmark": "example_bookmark_identifier", "result_consumed_after": 123}
```

#### Server Response `IGNORED`

A server that receives a `DISCARD_ALL` request while in `FAILED` state or `INTERRUPTED` state, should respond with an `IGNORED` message and discard the request without processing it.

No state change should occur.

Example:

```
IGNORED
```

#### Server Response `FAILURE`

If a `DISCARD_ALL` message request cannot be processed successfully, the server should respond with a `FAILURE` message and enter the `FAILED` state.
The server may attach metadata to the message to provide more detail on the nature of the failure.

Example:

```
FAILURE {"message": "example failure", "code": "Example.Failure.Code"}
```


### Request Message - `PULL_ALL`

The `PULL_ALL` message issues a request to stream the outstanding result back to the client, before returning to a `READY` state.

Result detail consists of zero or more detail messages being sent before the summary message.

This version of the protocol defines one such detail message, namely `RECORD`.

A `PULL_ALL` message uses the signature `3F` and passes no fields.
Zero or more detail messages may be returned.

**Signature:** `3F`

**Fields:**

No fields

**Detail Messages:**

zero or more `RECORD`

**Valid Summary Messages:**

* `SUCCESS`
* `IGNORED`
* `FAILURE`

The server **must** be in a `STREAMING` state to be able to successfully process a `PULL_ALL` request.
If the server is in a `FAILED` or `INTERRUPTED` state, the request will be IGNORED.
For any other states, receipt of a `PULL_ALL` request will be considered a protocol violation and will lead to connection closure.

#### Synopsis

```
PULL_ALL
```

Example:

```
PULL_ALL
```

#### Server Response `RECORD`

Zero or more `RECORD` messages may be returned in response to a `PULL_ALL` prior to the trailing summary message.

Each record carries with it **a list of values** which form the data content of the record.

The order of the values within the list should be meaningful to the client, perhaps based on a requested ordering for that result, but no guarantees should be made around the order of records within the result.

A record should only be considered valid if followed by a `SUCCESS` summary message.

Until this summary has been received, the record's validity should be considered tentative.

Example:

```
RECORD [1, 2, 3]
```


#### Server Response `SUCCESS`

Following any relevant detail messages, and assuming the `PULL_ALL` request has been successfully processed, the server should respond with a `SUCCESS` message and enter the `READY` state.

The server may attach metadata to the message to provide footer detail for the results.

The following fields are defined for inclusion in the metadata.

- `bookmark` (e.g. `"bookmark:1234"`)
- `result_consumed_after` (e.g. `123`)

Example:

```
SUCCESS {"bookmark": "example_bookmark_identifier", "result_consumed_after": 123}
```

#### Server Response `IGNORED`

A server that receives a `PULL_ALL` request while `FAILED` or `INTERRUPTED` should respond with an `IGNORED` message and discard the request without processing it.
No state change should occur.

Example:

```
IGNORED
```

#### Server Response `FAILURE`

If a `PULL_ALL` request cannot be processed successfully, the server should respond with a `FAILURE` message and enter the `FAILED` state.
The server may attach metadata to the message to provide more detail on the nature of the failure.

Failure may occur at any time during result streaming which may lead to a number of detail messages preceding the `FAILURE` message.
In this case, that result detail should be considered invalid.

Example:

```
FAILURE {"message": "example failure", "code": "Example.Failure.Code"}
```


### Summary Message - `SUCCESS`

The `SUCCESS` message indicates that the corresponding request has succeeded as intended.

It may contain metadata relating to the outcome.

Metadata keys are described in the section of this document relating to the message that began the exchange.

**Signature:** `70`

**Fields:**

```
metadata::Dictionary
```

#### Synopsis

```
SUCCESS {metadata}
```

Example:

```
SUCCESS {"example": "see specific message for server response metadata"}
```


### Summary Message - `IGNORED`

The `IGNORED` message indicates that the corresponding request has not been carried out.

**Signature:** `7E`

**Fields:**

No fields

#### Synopsis

```
IGNORED
```

Example:

```
IGNORED
```

### Summary Message - `FAILURE`


**Signature:** `7F`

**Fields:**

```
metadata::Dictionary(
  code::String,
  message::String,
)
```

- The `code` is a textual code that uniquely identifies the type of failure. 
- The `message` is a textual description of the failure, intended for human consumption.


#### Synopsis

```
FAILURE {metadata}
```

Example:

```
FAILURE {"message": "example failure", "code": "Example.Failure.Code"}
```


### Detail Message - `RECORD`

A `RECORD` message carries a sequence of values corresponding to a single entry in a result.

**Signature:** `71`

These messages are currently only ever received in response to a `PULL_ALL` message and will always be followed by a **summary message**.

**Fields:**

```
data::List
```

#### Synopsis

```
RECORD [data]
```

Example 1:

```
RECORD ["1", "2", "3"]
```

Example 2:

```
RECORD [{"point": [1, 2]}, "example_data", 123]
```


# Appendix - Message Exchange Examples

* The `C:` stands for client.
* The `S:` stands for server.


## Example 1

```
C: 60 60 B0 17
C: 00 00 00 01 00 00 00 00 00 00 00 00 00 00 00 00
S: 00 00 00 01
C: INIT "user_agent": "Example/1.0.0" {"scheme": "basic", "principal": "user", "credentials": "password"}
S: SUCCESS {"server": "Neo4j/3.4.0"}
```

## Example 2

```
C: 60 60 B0 17
C: 00 00 00 01 00 00 00 00 00 00 00 00 00 00 00 00
S: 00 00 00 01
C: INIT "user_agent": "Example/1.0.0" {"scheme": "basic", "principal": "user", "credentials": "password"}
S: SUCCESS {"server": "Neo4j/3.4.0"}
C: RUN "RETURN $x AS example" {"x": 123}
S: SUCCESS {"fields": ["example"]}
C: PULL_ALL
S: RECORD [123]
S: SUCCESS {"bookmark": "example-bookmark:1", "t_last": 300}
```
