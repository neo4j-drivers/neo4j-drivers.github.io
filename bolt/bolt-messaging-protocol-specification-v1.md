# Bolt Messaging Protocol Specification v1



## 1. Overview

This document describes version 1 of the Bolt messaging protocol.
The messaging protocol is used for the message exchanges that take place on a connection following a successful Bolt handshake.
For details of establishing a connection and performing a handshake, see the [Bolt Handshake Protocol](bolt-handshake-protocol-specification.md).

*NOTE: Byte values in this document are represented using hexadecimal notation unless otherwise specified.*



## 2. Message Exchange

Messages are exchanged in a request-response pattern between client and server.
Each request consists of exactly one message and each response consists of zero or more detail messages followed by exactly one summary message.
The presence or absence of detail messages in a response is directly related to the type of request message that has been sent.
In other words, some request message types elicit a response that may contain detail messages, others do not.

Messages may also be pipelined.
In other words, clients may send multiple requests eagerly without first waiting for responses.
When a failure occurs in this scenario, servers MUST ignore all subsequent requests until the client has explicitly acknowledged receipt of the failure.
This prevents inadvertent execution of statements that may not be valid.
More details of this process can be found in the sections below.


### 2.1. Serialization

Messages and their contents are serialized into network streams using [PackStream](../packstream/packstream-specification-v1.md).
Each message is represented as a PackStream structure with a fixed number of fields.
The message type is denoted by the structure signature, a single byte value.


### 2.2. Chunking

A layer of chunking is also applied to message transmissions as a way to more predictably manage packets of data.
The chunking process allows the message to be broken into one or more pieces, each of an arbitrary size, and for those pieces to be transmitted within separate chunks.

Each chunk consists of a two-byte header, detailing the chunk size in bytes followed by the chunk data itself.
Chunk headers are 16-bit unsigned integers, meaning that the maximum theoretical chunk size permitted is 65,535 bytes.

Each encoded message MUST be terminated with a chunk of zero size, i.e. `[00 00]`.
This is used to signal message boundaries to a receiving parties, allowing blocks of data to be fully received without requiring that the message is parsed immediately.
This also allows for unknown message types to be received and handled without breaking the messaging chain.


### 2.3. Request Messages

The table below describes the request messages defined by this specification.

| Name          | Signature | Fields                              | Description
|---------------|-----------|-------------------------------------|-----------------------
| `INIT`        | `01`      | user_agent::String, auth_token::Map | Initialize connection
| `ACK_FAILURE` | `0E`      |                                     | Acknowledge failure
| `RESET`       | `0F`      |                                     | Reset connection
| `RUN`         | `10`      | statement::String, parameters::Map  | Run statement
| `DISCARD_ALL` | `2F`      |                                     | Discard all results
| `PULL_ALL`    | `3F`      |                                     | Fetch all results

Request messages, conditions of usage and their permitted responses are described in more detail in the sections below.


### 2.4. Response Summary Messages

The table below describes the response summary messages defined by this specification.

| Name        | Signature | Fields        | Description
|-------------|-----------|---------------|---------------------
| `SUCCESS`   | `70`      | metadata::Map | Request succeeded
| `IGNORED`   | `7E`      |               | Request was ignored
| `FAILURE`   | `7F`      | metadata::Map | Request failed

### 2.5. Response Detail Messages

The table below describes the response detail messages defined by this specification.

| Name        | Signature | Fields        | Description
|-------------|-----------|---------------|-------------
| `RECORD`    | `71`      | data::List    | Data values

### 2.6. Protocol Errors

If a server or client receives a message type that is unexpected, according to the transitions described in this document, it must treat that as a protocol error.
Protocol errors are fatal and should immediately transition the state to `DEFUNCT`, closing any open connections.


## 3. Server States

Each connection maintained by a Bolt server will occupy one of several states throughout its lifetime.
This state is used to determine what actions may be undertaken by the client.


### 3.1. `DISCONNECTED`

No socket connection has yet been established.
This is the initial state and exists only in a logical sense prior to the socket being opened.

#### 3.1.1. Transitions from `DISCONNECTED`

- `<CONNECT>` to `CONNECTED` or `DEFUNCT`


### 3.2. `CONNECTED`

After a new connection has been established and handshake has been completed successfully, the server enters the `CONNECTED` state.
The connection has not yet been authenticated and permits only one transition, through successful initialization, into the `READY` state.

#### 3.2.1. Transitions from `CONNECTED`

- `INIT` to `READY` or `DEFUNCT`
- `<DISCONNECT>` to `DEFUNCT`


### 3.3. `READY`

The `READY` state indicates that the connection is ready to accept a new `RUN` request.
This is the "normal" state for a connection and becomes available after successful initialization and when not executing another statement.

The server will generally transition from here to `STREAMING` but in some exceptional circumstances will enter either `FAILED` or `INTERRUPTED` states.

#### 3.3.1. Transitions from `READY`

- `RUN` to `STREAMING` or `FAILED`
- `<INTERRUPT>` to `INTERRUPTED`
- `<DISCONNECT>` to `DEFUNCT`


### 3.4. `STREAMING`

When `STREAMING`, a result is available for streaming from server to client.
This result must be fully consumed or discarded by a client before the server can re-enter the `READY` state and allow any further statements to be executed.

#### 3.4.1. Transitions from `STREAMING`

- `PULL_ALL` to `READY` or `FAILED`
- `DISCARD_ALL` to `READY` or `FAILED`
- `<INTERRUPT>` to `INTERRUPTED`
- `<DISCONNECT>` to `DEFUNCT`


### 3.5. `FAILED`

When `FAILED`, a connection is in a temporarily unusable state.
This is generally as the result of encountering a recoverable error.
No more work will be processed until the failure has been acknowledged by `ACK_FAILURE` or until the connection has been `RESET`.
This mode ensures that only one failure can exist at a time, preventing cascading issues from batches of work.

#### 3.5.1. Transitions from `FAILED`

- `ACK_FAILURE` to `READY` or `DEFUNCT`
- `<INTERRUPT>` to `INTERRUPTED`
- `<DISCONNECT>` to `DEFUNCT`


### 3.6. `INTERRUPTED`

This state occurs between the server receiving the jump-ahead `<INTERRUPT>` and the queued `RESET` message.
Most incoming messages are ignored when `INTERRUPTED`, with the exception of the `RESET` that allows transition back to `READY`.

#### 3.6.1. Transitions from `INTERRUPTED`

- `RUN` to `INTERRUPTED`
- `DISCARD_ALL` to `INTERRUPTED`
- `PULL_ALL` to `INTERRUPTED`
- `RESET` to `READY` or `DEFUNCT`
- `<INTERRUPT>` to `INTERRUPTED`
- `<DISCONNECT>` to `DEFUNCT`


### 3.7. `DEFUNCT`

This is not strictly a connection state, but is instead a logical state that exists after a connection has been closed.
When `DEFUNCT`, a connection is permanently usable.
This may arise due to a graceful shutdown or can occur after an unrecoverable error or protocol violation.
Clients and servers should clear up any resources associated with a connection on entering this state, including closing any open sockets.
This is a terminal state on which no further transitions may be carried out.



## 4. Requests


### 4.1. `INIT`

`INIT` is a request for the connection to be authorized for use with the remote database. 

The `INIT` message uses the structure signature `01` and passes two fields: *user agent* (string) and *auth token* (map).
No detail messages should be returned.
Valid summary messages are `SUCCESS` and `FAILURE`.

The server MUST be in the `CONNECTED` state to be able to process an `INIT` request.
For any other states, receipt of an `INIT` request MUST be considered a protocol violation and lead to connection closure.

Clients should send `INIT` requests to the network immediately after connection and process the corresponding response before using that connection in any other way.

A receiving server may choose to register or otherwise log the user agent but may also ignore it if preferred.
User agents should use the form `"name/version"`, for example `"my-client/1.2.3"`.

The auth token should be used by the server to determine whether the client is permitted to exchange further messages.
If this authentication fails, the server MUST respond with a `FAILURE` message and immediately close the connection.
Clients wishing to retry initialization should establish a new connection.

#### 4.1.1. `INIT` Synopsis

```
INIT "user_agent" {metadata}
```

The following fields are defined for inclusion in the auth token.

- `scheme`
- `principal`
- `credentials`

#### 4.1.2. `INIT` State Transitions

| Initial State | Final State | Response     |
|---------------|-------------|--------------|
| `CONNECTED`   | `READY`     | `SUCCESS {}` |
| `CONNECTED`   | `DEFUNCT`   | `FAILURE {}` |

#### 4.1.3. `INIT` Response `SUCCESS`

A success response indicates that the client is permitted to exchange further messages.
Servers can include metadata that describes details of the server environment and/or the connection.

- server: "Neo4j/3.4.0"

#### 4.1.4. `INIT` Response `FAILURE`

A failure response indicates that the client is not permitted to exchange further messages.
Servers may choose to include metadata describing the nature of the failure but MUST immediately close the connection after the failure has been sent.


### 4.2. `RUN`

`RUN` submits a new statement for execution, the result of which will be consumed by a subsequent message, such as `PULL_ALL`.

The `RUN` message uses the structure signature `10` and passes two fields: *statement* (string) and *parameters* (map).
No detail messages should be returned.
Valid summary messages are `SUCCESS`, `FAILURE` and `IGNORED`.

The server MUST be in a `READY` state to be able to successfully process a `RUN` request.
If the server is in a `FAILED` or `INTERRUPTED` state, the request will be `IGNORED`.
For any other states, receipt of a `RUN` request will be considered a protocol violation and will lead to connection closure.

The statement generally contains a database query or remote procedure call.
This document does not specify how the server should handle an incoming statement.

The parameters generally contain variable fields for a database query or remote procedure call.
This document does not specify how the server should handle incoming parameters.

#### 4.2.1. `RUN` Synopsis

```
RUN "statement" {parameters}
```

#### 4.2.2. `RUN` State Transitions

| Initial State | Final State   | Response     |
|---------------|---------------|--------------|
| `READY`       | `STREAMING`   | `SUCCESS {}` |
| `READY`       | `FAILED`      | `FAILURE {}` |
| `INTERRUPTED` | `INTERRUPTED` | `IGNORED`    |

#### 4.2.3. `RUN` Response `SUCCESS`

If a `RUN` request has been successfully received and is considered valid, the server should respond with a `SUCCESS` message and enter the *STREAMING* state.
The server may attach metadata to the message to provide header detail for the results that follow.

Clients should not consider a `SUCCESS` response to indicate completion of the execution of that statement, merely acceptance of it.

#### 4.2.4. `RUN` Response `FAILURE`

If a `RUN` request cannot be processed successfully or is invalid, the server should respond with a `FAILURE` message and enter the *FAILED* state.
The server may attach metadata to the message to provide more detail on the nature of the failure.

#### 4.2.5. `RUN` Response `IGNORED`

TODO


### 4.3. `DISCARD_ALL`

`DISCARD_ALL` issues a request to discard the outstanding result and return to a `READY` state.
A receiving server should not abort the request but continue to process it without streaming any detail back to the client.

A `DISCARD_ALL` message uses the signature `2F` and passes no fields.
No detail messages should be returned.
Valid summary messages are `SUCCESS`, `FAILURE` and `IGNORED`.

The server MUST be in a `STREAMING` state to be able to successfully process a `DISCARD_ALL` request.
If the server is in a `FAILED` or `INTERRUPTED` state, the request will be `IGNORED`.
For any other states, receipt of a `DISCARD_ALL` request will be considered a protocol violation and will lead to connection closure.

#### 4.3.1. `DISCARD_ALL` Synopsis

```
DISCARD_ALL
```

#### 4.3.2. `DISCARD_ALL` State Transitions

| Initial State | Final State   | Response     |
|---------------|---------------|--------------|
| `STREAMING`   | `READY`       | `SUCCESS {}` |
| `STREAMING`   | `FAILED`      | `FAILURE {}` |
| `INTERRUPTED` | `INTERRUPTED` | `IGNORED`    |

#### 4.3.3. `DISCARD_ALL` Response `SUCCESS`

If a `DISCARD_ALL` request has been successfully received, the server should respond with a `SUCCESS` message and enter the *READY* state.
The server may attach metadata to the message to provide footer detail for the discarded results.

#### 4.3.4. `DISCARD_ALL` Response `FAILURE`

If a `DISCARD_ALL` request cannot be processed successfully, the server should respond with a `FAILURE` message and enter the *FAILED* state.
The server may attach metadata to the message to provide more detail on the nature of the failure.

#### 4.3.5. `DISCARD_ALL` Response `IGNORED`

A server that receives a `DISCARD_ALL` request while `FAILED` or `INTERRUPTED` should respond with an `IGNORED` message and discard the request without processing it.
No state change should occur.


### 4.4. `PULL_ALL`

`PULL_ALL` issues a request to stream the outstanding result back to the client, before returning to a `READY` state.
Result detail consists of zero or more detail messages being sent before the summary message.
This version of the protocol defines one such detail message, namely `RECORD`.

A `PULL_ALL` message uses the signature `3F` and passes no fields.
Zero or more detail messages may be returned.
Valid summary messages are `SUCCESS`, `FAILURE` and `IGNORED`.

The server MUST be in a `STREAMING` state to be able to successfully process a `PULL_ALL` request.
If the server is in a `FAILED` or `INTERRUPTED` state, the request will be IGNORED.
For any other states, receipt of a `PULL_ALL` request will be considered a protocol violation and will lead to connection closure.

#### 4.4.1. `PULL_ALL` Synopsis

```
PULL_ALL
```

#### 4.4.2. `PULL_ALL` State Transitions

| Initial State | Final State   | Response                      |
|---------------|---------------|-------------------------------|
| `STREAMING`   | `READY`       | \[`RECORD` ...\] `SUCCESS {}` |
| `STREAMING`   | `FAILED`      | \[`RECORD` ...\] `FAILURE {}` |
| `INTERRUPTED` | `INTERRUPTED` | `IGNORED`                     |

#### 4.4.3. `PULL_ALL` Response `SUCCESS`

Following any relevant detail messages, and assuming the `PULL_ALL` request has been successfully processed, the server should respond with a `SUCCESS` message and enter the `READY` state.
The server may attach metadata to the message to provide footer detail for the results.

#### 4.4.4. `PULL_ALL` Response `FAILURE`

If a `PULL_ALL` request cannot be processed successfully, the server should respond with a `FAILURE` message and enter the `FAILED` state.
The server may attach metadata to the message to provide more detail on the nature of the failure.

Failure may occur at any time during result streaming which may lead to a number of detail messages preceding the `FAILURE` message.
In this case, that result detail should be considered invalid.

#### 4.4.5. `PULL_ALL` Response `IGNORED`

A server that receives a `PULL_ALL` request while `FAILED` or `INTERRUPTED` should respond with an `IGNORED` message and discard the request without processing it.
No state change should occur.

#### 4.4.6. `PULL_ALL` Response `RECORD`

Zero or more `RECORD` messages may be returned in response to a `PULL_ALL` prior to the trailing summary message.
Each record carries with it a list of values which form the data content of the record.
The order of the values within the list should be meaningful to the client, perhaps based on a requested ordering for that result, but no guarantees should be made around the order of records within the result.

A record should only be considered valid if followed by a `SUCCESS` summary message.
Until this summary has been received, the record's validity should be considered tentative.


### 4.5. `ACK_FAILURE`

`ACK_FAILURE` signals to the server that the client has acknowledged a previous failure and should return to a `READY` state.

The `ACK_FAILURE` message has the signature `0E` and passes no fields.
No detail messages should be returned.
Valid summary messages are `SUCCESS` and `FAILURE`.

The server MUST be in a `FAILED` state to be able to successfully process an `ACK_FAILURE` request.
For any other states, receipt of an `ACK_FAILURE` request will be considered a protocol violation and will lead to connection closure.

#### 4.5.1. `ACK_FAILURE` Synopsis

```
ACK_FAILURE
```

#### 4.5.2. `ACK_FAILURE` State Transitions

| Initial State | Final State   | Response     |
|---------------|---------------|--------------|
| `FAILED`      | `READY`       | `SUCCESS {}` |
| `FAILED`      | `DEFUNCT`     | `FAILURE {}` |

#### 4.5.3. `ACK_FAILURE` Response `SUCCESS`

If an `ACK_FAILURE` request has been successfully received, the server should respond with a `SUCCESS` message and enter the `READY` state.

#### 4.5.4. `ACK_FAILURE` Response `FAILURE`

If an `ACK_FAILURE` request is received while not in the `FAILED` state, the server should respond with a `FAILURE` message and immediately close the connection.
The server may attach metadata to the message to provide more detail on the nature of the failure.


### 4.6. `RESET`

`RESET` requests that the connection should be set back to its initial `READY` state, as if an `INIT` had just successfully completed.
`RESET` is unique in that, on arrival at the server, it splits into two separate signals.
Firstly, an `<INTERRUPT>` jumps ahead in the message queue, stopping any unit of work that happens to be executing, and putting the state machine into an `INTERRUPTED` state.
Secondly, the `RESET` queues along with all other incoming messages and is used to put the state machine back to `READY` when its turn for processing arrives.

This essentially means that the `INTERRUPTED` state exists only transitionally between the arrival of a `RESET` in the message queue and the later processing of that `RESET` in its proper position.
`INTERRUPTED` is therefore the only state to automatically resolve without any further input from the client and whose entry does not generate a response message.


#### 4.6.1. `RESET` Synopsis

```
RESET
```

#### 4.6.2. `<INTERRUPT>` State Transitions

| Initial State | Final State   | Response |
|---------------|---------------|----------|
| `READY`       | `INTERRUPTED` | *n/a*    |
| `STREAMING`   | `INTERRUPTED` | *n/a*    |
| `FAILED`      | `INTERRUPTED` | *n/a*    |
| `INTERRUPTED` | `INTERRUPTED` | *n/a*    |

#### 4.6.3. `RESET` State Transitions

| Initial State | Final State   | Response     |
|---------------|---------------|--------------|
| `INTERRUPTED` | `READY`       | `SUCCESS {}` |
| `INTERRUPTED` | `DEFUNCT`     | `FAILURE {}` |

#### 4.6.3. `RESET` Response `SUCCESS`

If a `RESET` request has been successfully received, the server should respond with a `SUCCESS` message and enter the `READY` state.

#### 4.6.4. `RESET` Response `FAILURE`

If `RESET` is received before the server enters a `READY` state, it should trigger a `FAILURE` followed by immediate closure of the connection.
Clients receiving a `FAILURE` in response to `RESET` should treat that connection as `DEFUNCT` and dispose of it.
