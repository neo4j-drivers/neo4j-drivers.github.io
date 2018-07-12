# Bolt Messaging Protocol v1

This document describes version 1 of the Bolt messaging protocol.
The messaging protocol is used for the message exchanges that take place following a successful Bolt handshake.
For details of establishing a connection and performing a handshake, see the [Bolt Handshake Protocol](bolt-handshake-protocol-specification.md).

NOTE: Byte values in this document are represented using hexadecimal notation unless otherwise specified.


## Message Exchange

Messages are exchanged in a request-response pattern.
Each request consists of exactly one message and each response consists of zero or more detail messages followed by exactly one summary message.
The presence or absence of detail messages in a response is directly related to the type of request message that has been sent.
In other words, some request message types elicit a response that may contain detail messages, others do not.

### Serialization

Messages and their contents are serialized into network streams using [PackStream](packstream-specification-v1.md).
Each message is represented as a PackStream structure with a fixed number of fields.
The message type is denoted by the structure signature, a single byte value.

### Request Messages

The table below describes the request messages defined by this specification.

| Name | Signature | Fields                              | Description
|------|-----------|-------------------------------------|-----------------------
| INIT        | 01 | user_agent::String, auth_token::Map | Initialize connection
| ACK_FAILURE | 0E |                                     | Acknowledge failure
| RESET       | 0F |                                     | Reset connection
| RUN         | 10 | statement::String, parameters::Map  | Run statement
| DISCARD_ALL | 2F |                                     | Discard all results
| PULL_ALL    | 3F |                                     | Fetch all results

Request messages, conditions of usage and their permitted responses are described in more detail later in this document.

### Response Summary Messages

The table below describes the response summary messages defined by this specification.

| Name | Signature | Fields        | Description
|------|-----------|---------------|---------------------
| SUCCESS     | 70 | metadata::Map | Request succeeded
| IGNORED     | 7E | metadata::Map | Request was ignored
| FAILURE     | 7F | metadata::Map | Request failed

### Response Detail Messages
The table below describes the response detail messages defined by this specification.

| Name | Signature | Fields        | Description
|------|-----------|---------------|-------------
| RECORD      | 71 | data::List    | Data values

### Chunking

To simplify the process of reading messages from and writing messages to a network stream, a layer of chunking is also applied to message transmissions.
This chunking process allows the message to be broken into one or more pieces, each of an arbitrary size, and for those pieces to be transmitted within separate chunks.

Each chunk consists of a two-byte header, detailing the chunk size in bytes followed by the chunk data itself.
Chunk headers are 16-bit unsigned integers, meaning that the maximum theoretical chunk size permitted is 65,535 bytes.

Each encoded message MUST be terminated with a chunk of zero size, i.e. `[00 00]`.
This is used to signal message boundaries to a receiving parties, allowing blocks of data to be fully received without requiring that the message is parsed immediately.
This also allows for unknown message types to be received and handled without breaking the messaging chain.


## Server State

During the lifetime of a connection, the server will maintain a state for that connection.
This state is used to determine what actions may be undertaken by the client.

### CONNECTED

After a new connection has been established and handshake has been completed successfully, the server should enter the *CONNECTED* state.
This state permits only one transition, through successful initialization, into the *READY* state.

### READY

The *READY* state indicates that the connection is ready to accept a new RUN request.
This is the "normal" state for a connection and becomes available after successful initialization and when not executing another statement.

The server will generally transition from here to *STREAMING* but in some exceptional circumstances will enter either *FAILED* or *INTERRUPTED* states.

### STREAMING

When *STREAMING*, a result is available for streaming.
This result must be fully consumed or discarded by a client before the server can re-enter the *READY* state and allow any further statements to be executed.

### FAILED

_TODO_

### INTERRUPTED

_TODO_


## Initialization

Before a connection can be used for query exchange, it MUST go through initialization.
This process requires the client to supply a user agent and an auth token.

Before initialization, the server should be in the *CONNECTED* state.
After successful initialization, this will transition to *READY*.

### INIT

An `INIT` request supplies client and authentication details to the server.

An `INIT` request message has the signature 01 and passes two fields: user agent and auth token.
No detail messages should be returned. Valid summary messages are `SUCCESS` and `FAILURE`.

The server MUST be in the *CONNECTED* state to be able to process an `INIT` request.
For any other states, receipt of an `INIT` request MUST be considered a protocol violation and lead to connection closure.

Clients should send `INIT` requests to the network immediately after connection and process the corresponding response before using that connection in any other way.

#### User Agent
A receiving server may choose to register or otherwise log the user agent but may also ignore it if preferred.
User agents should use the form name/version, for example my-client/1.2.3.

#### Auth Token
The auth token should be used by the server to determine whether the client is permitted to exchange further messages.
If this authentication fails, the server MUST respond with a `FAILURE` message and immediately close the connection.
Clients wishing to retry initialization should establish a new connection.

The contents of the auth token are implementation-specific and will be defined in other documents.

#### SUCCESS
A success response indicates that the client is permitted to exchange further messages.
Servers may choose to include metadata that describes details of the server environment and/or the connection.

#### FAILURE
A failure response indicates that the client is not permitted to exchange further messages.
Servers may choose to include metadata describing the nature of the failure but MUST immediately close the connection after the failure has been sent.


## Statement Exchange and Result Streaming

The main purpose of a Bolt connection is to exchange statements and results.
This is achieved by exchanging pairs of `RUN`/`DISCARD_ALL` or `RUN`/`PULL_ALL` requests with their corresponding responses.

During this phase of the protocol, servers will generally alternate between two logical states: *READY* and *STREAMING*.

Statement exchanges may be pipelined.
In other words, clients may send multiple requests eagerly without first waiting for responses.
When a failure occurs in this scenario, servers MUST ignore all subsequent requests until the client has explicitly acknowledged receipt of the failure.
This prevents inadvertent execution of statements that may not be valid.
More details of this process can be found in the sections below.

### RUN

A `RUN` request submits a new statement for execution.

A `RUN` request message has the signature `10` and passes two fields: statement and parameters.
No detail messages should be returned.
Valid summary messages are *SUCCESS*, *FAILURE* and *IGNORED*.

The server MUST be in a *READY* state to be able to successfully process a `RUN` request.
If the server is in a *FAILED* or *INTERRUPTED* state, the request will be *IGNORED*.
For any other states, receipt of a `RUN` request will be considered a protocol violation and will lead to connection closure.

#### Statement
The statement generally contains a database query or remote procedure call.
This document does not specify how the server should handle an incoming statement.

#### Parameters
The parameters generally contain variable fields for a database query or remote procedure call.
This document does not specify how the server should handle incoming parameters.

#### SUCCESS
If a `RUN` request has been successfully received and is considered valid, the server should respond with a `SUCCESS` message and enter the *STREAMING* state.
The server may attach metadata to the message to provide header detail for the results that follow.

Clients should not consider a `SUCCESS` response to indicate completion of the execution of that statement, merely acceptance of it.

#### IGNORED
A server that receives a `RUN` request while *FAILED* or *INTERRUPTED* should respond with an `IGNORED` message and discard the request without processing it.
No state change should occur.

#### FAILURE
If a `RUN` request cannot be processed successfully or is invalid, the server should respond with a `FAILURE` message and enter the *FAILED* state.
The server may attach metadata to the message to provide more detail on the nature of the failure.

### DISCARD_ALL

A `DISCARD_ALL` request instructs the server to discard any remaining result detail and return to a READY state.
A receiving server should not abort the request but continue to process it without streaming any detail back to the client.

A `DISCARD_ALL` request message has the signature `2F` and passes no fields. No detail messages should be returned.
Valid summary messages are *SUCCESS*, *FAILURE* and *IGNORED*.

The server MUST be in a *STREAMING* state to be able to successfully process a `DISCARD_ALL` request.
If the server is in a *FAILED* or *INTERRUPTED* state, the request will be `IGNORED`.
For any other states, receipt of a `DISCARD_ALL` request will be considered a protocol violation and will lead to connection closure.

#### SUCCESS
If a `DISCARD_ALL` request has been successfully received, the server should respond with a `SUCCESS` message and enter the *READY* state.
The server may attach metadata to the message to provide footer detail for the discarded results.

#### IGNORED
A server that receives a `DISCARD_ALL` request while *FAILED* or *INTERRUPTED* should respond with an `IGNORED` message and discard the request without processing it.
No state change should occur.

#### FAILURE
If a `DISCARD_ALL` request cannot be processed successfully, the server should respond with a `FAILURE` message and enter the *FAILED* state.
The server may attach metadata to the message to provide more detail on the nature of the failure.

### PULL_ALL

A `PULL_ALL` request instructs the server to stream all result detail back to the client before returning to a *READY* state.
Result detail consists of zero or more detail messages being sent before the summary message.
This version of the protocol defines only one such detail message, namely `RECORD`.

A `PULL_ALL` request message has the signature `3F` and passes no fields.
Zero or more detail messages may be returned. Valid summary messages are `SUCCESS`, `FAILURE` and `IGNORED`.

The server MUST be in a *STREAMING* state to be able to successfully process a `PULL_ALL` request.
If the server is in a *FAILED* or *INTERRUPTED* state, the request will be IGNORED.
For any other states, receipt of a `PULL_ALL` request will be considered a protocol violation and will lead to connection closure.

#### RECORD
Zero or more `RECORD` messages may be returned in response to a `PULL_ALL` prior to the trailing summary message.
Each record carries with it a list of values which form the data content of the record.
The order of the values within the list should be meaningful to the client, perhaps based on a requested ordering for that result, but no guarantees should be made around the order of records within the result.

A record should only be considered valid if followed by a `SUCCESS` summary message.
Until this summary has been received, the record's validity should be considered tentative.

#### SUCCESS
Following any relevant detail messages, and assuming the `PULL_ALL` request has been successfully processed, the server should respond with a `SUCCESS` message and enter the *READY* state.
The server may attach metadata to the message to provide footer detail for the results.

#### IGNORED
A server that receives a `PULL_ALL` request while *FAILED* or *INTERRUPTED* should respond with an `IGNORED` message and discard the request without processing it.
No state change should occur.

#### FAILURE
If a `PULL_ALL` request cannot be processed successfully, the server should respond with a `FAILURE` message and enter the *FAILED* state.
The server may attach metadata to the message to provide more detail on the nature of the failure.

Failure may occur at any time during result streaming which may lead to a number of detail messages preceding the `FAILURE` message.
In this case, that result detail should be considered invalid.

### ACK_FAILURE
An `ACK_FAILURE` request signals to the server that the client has acknowledged a previous failure and should return to a *READY* state.

An `ACK_FAILURE` request message has the signature `0E` and passes no fields.
No detail messages should be returned.
Valid summary messages are `SUCCESS` and `FAILURE`.

The server MUST be in a *FAILED* state to be able to successfully process an `ACK_FAILURE` request.
For any other states, receipt of an `ACK_FAILURE` request will be considered a protocol violation and will lead to connection closure.

#### SUCCESS
If an `ACK_FAILURE` request has been successfully received, the server should respond with a `SUCCESS` message and enter the *READY* state.

#### FAILURE
If an `ACK_FAILURE` request is received while not in the *READY* state, the server should respond with a `FAILURE` message and immediately close the connection.
The server may attach metadata to the message to provide more detail on the nature of the failure.

### RESET

_TODO_

### SUCCESS

_TODO_

### FAILURE

_TODO_
