# Bolt Protocol Message Specification - Version 4


* [**Version 4.0**](#version-40)

* [**Version 4.1**](#version-41)

* [**Appendix - Bolt Message Exchange Examples**](#appendix---bolt-message-exchange-examples)


**Note:** *Byte values are represented using hexadecimal notation unless otherwise specified.*


# Version 4.0

## Deltas

The changes compared to Bolt protocol version 3 are listed below:

* `DISCARD_ALL` message renamed to `DISCARD` and introduced new fields.
* `PULL_ALL` message renamed to `PULL` and introduced new fields.
* The `BEGIN` message now have a field `db` to specify a database name.
* The `RUN` message now have a field `db` to specify a database name.
* **Explicit transaction** (`BEGIN`+`RUN`) can now get a server response with a `SUCCESS` and metadata key `qid` (query identification).
* The `DISCARD` message can now discard an arbitrary number of records. New fields `n` and `qid`.
* The `DISCARD` message can now get a server response with a `SUCCESS` and metadata key `has_more`. 
* The `PULL` message can now fetch an arbitrary number of records. New fields `n` and `qid`.
* The `PULL` message can now get a server response with a `SUCCESS` and metadata key `has_more`. 



## Overview of Major Version Changes

The handshake have been re-specified to support Major and Minor versions.

Queries within an **explicit transaction** can be consumed out of order with the new behaviour of `PULL` and `DISCARD`.

```
PULL {"n": ?, "qid": ?}
```

```
DISCARD {"n": ?, "qid": ?}
```

The `qid` is included in the `SUCCESS` response of a `RUN` message within an **explicit transaction** (`BEGIN`+`RUN`).
See [Appendix - Example 4](#example-4).


## Bolt Protocol Server State Specification

For the server, each connection using the Bolt Protocol will occupy one of several states throughout its lifetime.

This state is used to determine what actions may be undertaken by the client.

See, [Bolt Protocol Server State Specification Version 4](bolt-protocol-server-state-specification-4.md)


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


## Messages - Version 4.0

* **Request Message**, the client sends a message.
* **Summary Message**, the server will always respond with one summary message.
* **Detail Message**, the server will always repond with zero or more detail messages before sending a summary message.


| Message       | Signature | Request Message | Summary Message | Detail Message | Fields                                                                                                                                                         | Description                                             |
|---------------|:---------:|:---------------:|:---------------:|:--------------:|----------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------|
| `HELLO`       | `01`      | x               |                 |                | `extra::Dictionary(user_agent::String, scheme::String, principal::String, credentials::String)`                                                                | initialize connection                                   |
| `GOODBYE`     | `02`      | x               |                 |                |                                                                                                                                                                | close the connection, triggers a `<DISCONNECT>` signal  |
| `RESET`       | `0F`      | x               |                 |                |                                                                                                                                                                | reset the connection, triggers a `<INTERRUPT>` signal   |
| `RUN`         | `10`      | x               |                 |                | `query::String`, `parameters::Dictionary`, `extra::Dictionary(bookmarks::List<String>, tx_timeout::Integer, tx_metadata::Dictionary, mode::String, db:String)` | execute a query                                         |
| `DISCARD`     | `2F`      | x               |                 |                | `extra::Dictionary(n::Integer`, `qid::Integer)`                                                                                                                | discard records                                         |
| `PULL`        | `3F`      | x               |                 |                | `extra::Dictionary(n::Integer`, `qid::Integer)`                                                                                                                | fetch records                                           |
| `BEGIN`       | `11`      | x               |                 |                | `extra::Dictionary(bookmarks::List<String>, tx_timeout::Integer, tx_metadata::Dictionary, mode::String, db::String)`                                           | begin a new transaction                                 |
| `COMMIT`      | `12`      | x               |                 |                |                                                                                                                                                                | commit a transaction                                    |
| `ROLLBACK`    | `13`      | x               |                 |                |                                                                                                                                                                | rollback a transaction                                  |
| `SUCCESS`     | `70`      |                 | x               |                | `metadata::Dictionary`                                                                                                                                         | request succeeded                                       |
| `IGNORED`     | `7E`      |                 | x               |                |                                                                                                                                                                | request was ignored                                     |
| `FAILURE`     | `7F`      |                 | x               |                | `metadata::Dictionary(code::String, message::String)`                                                                                                          | request failed                                          |
| `RECORD`      | `71`      |                 |                 | x              | `data::List`                                                                                                                                                   | data values                                             |



### `HELLO` - Message

The `HELLO` message request the connection to be authorized for use with the remote database.

The server **must** be in the `CONNECTED` state to be able to process a `HELLO` message.
For any other states, receipt of an `HELLO` request **must** be considered a protocol violation and lead to connection closure.

Clients should send `HELLO` message to the server immediately after connection and process the corresponding response before using that connection in any other way.

If authentication fails, the server **must** respond with a `FAILURE` message and immediately close the connection.

Clients wishing to retry initialization should establish a new connection.

**Signature:** `01`

**Fields:**

```
extra::Dictionary(
  user_agent::String,
  scheme::String,
  principal::String,
  credentials::String,
)
```

  - The `user_agent` should conform to `"Name/Version"` for example `"Example/4.0.0"`. (see, [developer.mozilla.org/en-US/docs/Web/HTTP/Headers/User-Agent](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/User-Agent))
  - The `scheme` is the authentication scheme. Predefined schemes are `“none”`, `“basic”`, `“kerberos”`.

**Detail Messages:**

No detail messages should be returned.

**Valid Summary Messages:**

* `SUCCESS`
* `FAILURE`


#### Synopsis

```
HELLO {user_agent::String, scheme::String, principal::String, credentials::String)
```

Example:

```
HELLO {"user_agent": "Example/4.0.0", "scheme": "basic", "principal": "user-id-1", "credentials": "user-password"}
```


#### Server Response `SUCCESS`

A `SUCCESS` message response indicates that the client is permitted to exchange further messages.
Servers can include metadata that describes details of the server environment and/or the connection.

The following fields are defined for inclusion in the `SUCCESS` metadata.

  - `server::String` (server agent string, example `"Neo4j/4.0.0"`)
  - `connection_id::String` (unique identifier of the bolt connection used on the server side, example: `"bolt-61"`)

Example:

```
SUCCESS {"server": "Neo4j/4.0.0"}
```


#### Server Response `FAILURE`

A `FAILURE` message response indicates that the client is not permitted to exchange further messages.
Servers may choose to include metadata describing the nature of the failure but **must** immediately close the connection after the failure has been sent.

Example:

```
FAILURE {"message": "example failure", "code": "Example.Failure.Code"}
```


### `GOODBYE` - Message

The `GOODBYE` message notifies the server that the connection is terminating gracefully.

On receipt of this message, the server should immediately shut down the socket on its side without sending a response.

A client may shut down the socket at any time after sending the `GOODBYE` message.

This message interrupts the server current work if there is any.

**Signature:** `02`

**Fields:**

No fields.

**Detail Messages:**

No detail messages should be returned.

**Valid Summary Messages:**

No summary message should be returned.

#### Synopsis

```
GOODBYE
```

Example:

```
GOODBYE
```

### `RESET` - Message

The `RESET` message requests that the connection should be set back to its initial state, as if a `HELLO` message had just been successfully completed.

The `RESET` message is unique in that it on arrival at the server, it jumps ahead in the message queue, stopping any unit of work that happens to be executing.

All the queued messages originally in front of the `RESET` message will then be `IGNORED` until the `RESET` position is reached.
Then from this point, the server state is reset to a state that is ready for a new session.

**Signature:** `0F`

**Fields:**

No fields.

**Detail Messages:**

No detail messages should be returned.

**Valid Summary Messages:**

No summary message should be returned.

#### Synopsis

```
RESET
```

Example:

```
RESET
```

### `RUN` - Message

The `RUN` message requests that a Cypher query is executed with a set of parameters and additional extra data.

This message could both be used for running an explicit transaction or an autocommit transaction. The transaction type is implied by the order of message sequence.

**Signature:** `10`

**Fields:**

```
query::String
paramaters::Dictionary
extra::Dictionary(
  bookmarks::List<String>,
  tx_timeout::Integer,
  tx_metadata::Dictionary,
  mode::String,
  db:String,
)
```

  - The `query` can be a Cypher syntax or a procedure call.
  - The `parameters` is a dictionary of parameters to be used in the `query` string.

  An **explicit transaction** (`BEGIN`+`RUN`) does not carry any data in the extra `extra` field.

  For **autocommit transaction** (`RUN`) the `extra` field carries:

  - The `bookmarks` is a list of strings containg some kind of bookmark identification e.g [“neo4j-bookmark-transaction:1”, “neo4j-bookmark-transaction:2”]
  - The `tx_timeout` is an integer in that specifies a transaction timeout in ms.
  - The `tx_metadata` is a dictionary that can contain some metadata information, mainly used for logging.
  - The `mode` specifies what kind of server the `RUN` message is targeting. For write access use `"w"` and for read access use `"r"`. Defaults to write access if no mode is sent.
  - The `db` specifies the database name for multi-database to select where the transaction takes place. If no `db` is sent or empty string it implies that it is the default database.

**Detail Messages:**

No detail messages should be returned.

**Valid Summary Messages:**

* `SUCCESS`
* `IGNORED`
* `FAILURE`

#### Synopsis

```
RUN "query" {parameters} {extra}
```

Example 1:

```
RUN "RETURN $x AS x" {"x": 1} {bookmarks: [], "tx_timeout": 123, "tx_metadata": {"log": "example_message"}, mode: "r"}
```

Example 2:

```
RUN "RETURN $x AS x" {"x": 1} {}
```

Example 3:

```
RUN "CALL dbms.procedures()" {} {}
```

#### Server Response `SUCCESS`

A `SUCCESS` message response indicates that the client is permitted to exchange further messages.

The following fields are defined for inclusion in the `SUCCESS` metadata.

  - `fields::List<String>`, the fields of the return result. e.g. [“name”, “age”, ...]
  - `t_first::Integer`, the time, specified in ms, which the first record in the result stream is available after.
  
  For **explicit transaction** (`BEGIN`+`RUN`):

  - `qid::Integer`, specifies the server assigned statement id to reference the server side resultset with commencing `BEGIN`+`RUN`+`PULL` and `BEGIN`+`RUN`+`DISCARD` messages.

Example 1:

```
SUCCESS {"fields": ["x"], "t_first": 123}
```

Example 2:

```
SUCCESS {"fields": ["x"], "t_first": 123, "qid": 7000}
```

#### Server Response `IGNORED`

Example:

```
IGNORED
```

#### Server Response `FAILURE`

TODO: A `FAILURE` message response indicates that the client is not permitted to exchange further messages before the server is in a 

Example:

```
FAILURE {"message": "example failure", "code": "Example.Failure.Code"}
```

### `DISCARD` - Message

The `DISCARD` message requests that the remainder of the result stream should be thrown away.

**Signature:** `2F`

**Fields:**

```
extra::Dictionary{
  n::Integer,
  qid::Integer,
}
```

  - The `n` specifies how many records to throw away. `n=-1` will throw away all records.
  - The `qid` (query identification) specifies the result of which statement the operation should be carried out. (Explicit transaction only). `qid=-1` can be used to denote the last executed statement and if no ``.

**Detail Messages:**

No detail messages should be returned.

**Valid Summary Messages:**

* `SUCCESS`
* `IGNORED`
* `FAILURE`


#### Synopsis

```
DISCARD {extra}
```

Example 1:

```
DISCARD {"n": -1, "qid": -1}
```

Example 2:

```
DISCARD {"n": 1000}
```

#### Server Response `SUCCESS`

The following fields are defined for inclusion in the `SUCCESS` metadata.

  - `has_more::Boolean`, True if there are more records to stream

  or

  - `bookmark::String`, the bookmark after committing this transaction. (Autocommit transaction only).
  - `db::String`, the database name where the query was executed

Example 1:

```
SUCCESS {"has_more": True}
```

Example 2:

```
SUCCESS {"bookmark": "example-bookmark:1", "db": "example_database"}
```

#### Server Response `IGNORED`

Example:

```
IGNORED
```

#### Server Response `FAILURE`

TODO: A `FAILURE` message response indicates that the client is not permitted to exchange further messages before the server is in a 

Example:

```
FAILURE {"message": "example failure", "code": "Example.Failure.Code"}
```


### `PULL` - Message

The `PULL` message requests that the remainder of the result stream should be thrown away.

**Signature:** `3F`

**Fields:**

```
extra::Dictionary{
  n::Integer,
  qid::Integer,
}
```

  - The `n` specifies how many records to fetch. `n=-1` will fetch all records.
  - The `qid` (query identification) specifies the result of which statement the operation should be carried out. (Explicit transaction only). `qid=-1` can be used to denote the last executed statement and if no ``.

**Detail Messages:**

Zero or more `RECORD`

**Valid Summary Messages:**

* `SUCCESS`
* `IGNORED`
* `FAILURE`

#### Synopsis

```
PULL {extra}
```

Example 1:

```
PULL {"n": -1, "qid": -1}
```

Example 2:

```
PULL {"n": 1000}
```

#### Server Response `SUCCESS`

The following fields are defined for inclusion in the `SUCCESS` metadata.

  - `has_more::Boolean`, True if there are more records to stream

  or

  - `bookmark::String`, the bookmark after committing this transaction. **autocommit transaction only**.
  - `t_last::Integer`, the time, specified in ms, which the last record in the result stream is consumed after.
  - `type::String`, the type of the statement, e.g. “r” for read-only statement, “w” for write-only statement, “rw” for read-and-write, and “s” for schema only.
  - `stats::Dictionary`, counter information, such as db-hits etc.
  - `plan::Dictionary`, plan result.
  - `profile::Dictionary`, profile result.
  - `notifications::Dictionary`, any notification generated during execution of this statement.
  - `db::String`, the database name where the query was executed.

Example 1:

```
SUCCESS {"has_more": True}
```

Example 2:

```
SUCCESS {"bookmark": "example-bookmark:1", "db": "example_database", "t_last": 123}
```

#### Server Response `IGNORED`

Example:

```
IGNORED
```

#### Server Response `FAILURE`

TODO: A `FAILURE` message response indicates that the client is not permitted to exchange further messages before the server is in a 

Example:

```
FAILURE {"message": "example failure", "code": "Example.Failure.Code"}
```


### `BEGIN` - Message

The `BEGIN` message request the creation of a new **explicit transaction**.

This message should then be followed by a `RUN` message.

The **explicit transaction** is closed with either the `COMMIT` message or `ROLLBACK` message.


**Signature:** `11`

**Fields:**

```
extra::Dictionary(
  `bookmarks::List<String>`,
  `tx_timeout::Integer`,
  `tx_metadata::Dictionary`
  `mode::String`,
  `db::String`
)
```

  - The `bookmarks` is a list of strings containg some kind of bookmark identification e.g [“neo4j-bookmark-transaction:1”, “neo4j-bookmark-transaction:2”]
  - The `tx_timeout` is an integer in that specifies a transaction timeout in ms.
  - The `tx_metadata` is a dictionary that can contain some metadata information, mainly used for logging.
  - The `mode` specifies what kind of server the `RUN` message is targeting. For write access use `"w"` and for read access use `"r"`. Defaults to write access if no mode is sent.
  - The `db` specifies the database name for multi-database to select where the transaction takes place. If no `db` is sent or empty string it implies that it is the default database.

**Detail Messages:**

No detail messages.

**Valid Summary Messages:**

* `SUCCESS`
* `FAILURE`

#### Synopsis

```
BEGIN {extra}
```

Example 1:

```
BEGIN {"tx_timeout": 123, "mode": "r", "db": "example_database", "tx_metadata": {"log": "example_log_data"}}
```


Example 2:

```
BEGIN {"db": "example_database", "tx_metadata": {"log": "example_log_data"}, "bookmarks": ["example-bookmark:1", "example-bookmark2"]}
```


#### Server Response `SUCCESS`

Example:

```
SUCCESS {}
```

#### Server Response `IGNORED`

Example:

```
IGNORED
```

#### Server Response `FAILURE`

TODO: A `FAILURE` message response indicates that the client is not permitted to exchange further messages before the server is in a 

Example:

```
FAILURE {"message": "example failure", "code": "Example.Failure.Code"}
```


### `COMMIT` - Message

The `COMMIT` message request that the **explicit transaction** is done.

**Signature:** `12`

**Fields:**

No fields.

**Detail Messages:**

No detail messages.

**Valid Summary Messages:**

* `SUCCESS`
* `IGNORED`
* `FAILURE`


#### Synopsis

```
COMMIT
```

Example:

```
COMMIT
```

#### Server Response `SUCCESS`

A `SUCCESS` message response indicates that the **explicit transaction** was completed.

  - `bookmark::String`, the bookmark after committing this transaction.

Example:

```
SUCCESS {"bookmark": "example-bookmark:1"}
```

#### Server Response `IGNORED`

Example:

```
IGNORED
```

#### Server Response `FAILURE`

TODO: A `FAILURE` message response indicates that the client is not permitted to exchange further messages before the server is in a 

Example:

```
FAILURE {"message": "example failure", "code": "Example.Failure.Code"}
```

### `ROLLBACK` - Message

The `ROLLBACK` message requests that the **explicit transaction** rolls back.

**Signature:** `13`

**Fields:**

No fields.

**Detail Messages:**

No detail messages.

**Valid Summary Messages:**

* `SUCCESS`
* `IGNORED`
* `FAILURE`


#### Synopsis

```
ROLLBACK
```

Example:

```
ROLLBACK
```

#### Server Response `SUCCESS`

A `SUCCESS` message response indicates that the **explicit transaction** was rolled back.

```
SUCCESS {}
```

#### Server Response `IGNORED`

Example:

```
IGNORED
```

#### Server Response `FAILURE`

TODO: A `FAILURE` message response indicates that the client is not permitted to exchange further messages before the server is in a 

Example:

```
FAILURE {"message": "example failure", "code": "Example.Failure.Code"}
```

### `SUCCESS` - Summary Message

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


### `IGNORED` - Summary Message

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

### `FAILURE` - Summary Message


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


### `RECORD` - Detail Message

A `RECORD` message carries a sequence of values corresponding to a single entry in a result.

**Signature:** `71`

These messages are currently only ever received in response to a `PULL` message and will always be followed by a **summary message**.

**Fields:**

```
data::List<Value>
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


# Version 4.1

## Deltas

The changes compared to Bolt protocol version 4.0 are listed below:

* The `BEGIN` message now requires the sub-field `routing::Dictionary(address::String)`.
* Support for `NOOP` chunk (empty chunk). Both server and client should support this.


## Overview

Bolt handshake should now timeout (off by default) on the server side.

The initial address that the client knows the server by is sent with the `HELLO` message to help with routing information.
See [Appendix - Example 3](#example-3).

The `NOOP` chunk (empty chunk) is used to send an empty chunk and the purpose is to be able to support a keep alive behaviour on the connection.

Example: **Two messages with a NOOP in between**


Message 1 data containing 16 bytes:

```
00 01 02 03 04 05 06 07 08 09 0A 0B 0C 0D 0E 0F
```

Message 2 data containing 8 bytes:

```
0F 0E 0D 0C 0B 0A 09 08
```

The two messages encoded with chunking and a `NOOP` (empty chunk) in between.

```
00 10 00 01 02 03 04 05 06 07 08 09 0A 0B 0C 0D 0E 0F 00 00
00 00
00 08 0F 0E 0D 0C 0B 0A 09 08 00 00
```


## Messages - Version 4.1


| Message       | Signature | Request Message | Summary Message | Detail Message | Fields                                                                                                                                                           | Description                                             |
|---------------|:---------:|:---------------:|:---------------:|:--------------:|------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------|
| `HELLO`       | `01`      | x               |                 |                | `extra::Dictionary(user_agent::String, scheme::String, principal::String, credentials::String, routing::Dictionary(address::String))`                            | initialize connection                                   |
| `GOODBYE`     | `02`      | x               |                 |                |                                                                                                                                                                  | close the connection, triggers a `<DISCONNECT>` signal  |
| `RESET`       | `0F`      | x               |                 |                |                                                                                                                                                                  | reset the connection, triggers a `<INTERRUPT>` signal   |
| `RUN`         | `10`      | x               |                 |                | `query::String`, `parameters::Dictionary`, `extra::Dictionary(bookmarks::List<String>, tx_timeout::Integer, tx_metadata::Dictionary, mode::String, db:String)`   | execute a query                                         |
| `DISCARD`     | `2F`      | x               |                 |                | `extra::Dictionary(n::Integer`, `qid::Integer)`                                                                                                                  | discard records                                         |
| `PULL`        | `3F`      | x               |                 |                | `extra::Dictionary(n::Integer`, `qid::Integer)`                                                                                                                  | fetch records                                           |
| `BEGIN`       | `11`      | x               |                 |                | `extra::Dictionary(bookmarks::List<String>, tx_timeout::Integer, tx_metadata::Dictionary, mode::String, db::String)`                                             | begin a new transaction                                 |
| `COMMIT`      | `12`      | x               |                 |                |                                                                                                                                                                  | commit a transaction                                    |
| `ROLLBACK`    | `13`      | x               |                 |                |                                                                                                                                                                  | rollback a transaction                                  |
| `SUCCESS`     | `70`      |                 | x               |                | `metadata::Dictionary`                                                                                                                                           | request succeeded                                       |
| `IGNORED`     | `7E`      |                 | x               |                |                                                                                                                                                                  | request was ignored                                     |
| `FAILURE`     | `7F`      |                 | x               |                | `metadata::Dictionary(code::String, message::String)`                                                                                                            | request failed                                          |
| `RECORD`      | `71`      |                 |                 | x              | `data::List`                                                                                                                                                     | data values                                             |



# Appendix - Bolt Message Exchange Examples

* The `C:` stands for client.
* The `S:` stands for server.


## Example 1

```
C: 60 60 B0 17
C: 00 00 00 04 00 00 00 00 00 00 00 00 00 00 00 00
S: 00 00 00 04
C: HELLO {"user_agent": "Example/4.0.0", "scheme": "basic", "principal": "user", "credentials": "password"}
S: SUCCESS {"server": "Neo4j/4.0.0", "connection_id": "example-connection-id:1"}
C: GOODBYE
```


## Example 2

```
C: 60 60 B0 17
C: 00 00 00 04 00 00 00 00 00 00 00 00 00 00 00 00
S: 00 00 00 04
C: HELLO {"user_agent": "Example/4.0.0", "scheme": "basic", "principal": "user", "credentials": "password"}
S: SUCCESS {"server": "Neo4j/4.0.0", "connection_id": "example-connection-id:1"}
C: RUN "RETURN $x AS example" {"x": 123} {"mode": "r", "db": "example_database"}
S: SUCCESS {"fields": ["example"]}
C: PULL {"n": -1}
S: RECORD [123]
S: SUCCESS {"bookmark": "example-bookmark:1", "t_last": 300, "type": "r", "db": "example_database"}
C: GOODBYE
```


## Example 3

```
C: 60 60 B0 17
C: 00 00 01 04 00 00 00 00 00 00 00 00 00 00 00 00
S: 00 00 01 04
C: HELLO {"user_agent": "Example/4.1.0", "scheme": "basic", "principal": "user", "credentials": "password", "routing": {"address": "x.example.com"}}
S: SUCCESS {"server": "Neo4j/4.1.0", "connection_id": "example-connection-id:1"}
C: RUN "RETURN $x AS example" {"x": 123} {"mode": "r", "db": "example_database"}
S: SUCCESS {"fields": ["example"]}
C: PULL {"n": -1}
S: RECORD [123]
S: SUCCESS {"bookmark": "example-bookmark:1", "t_last": 300, "type": "r", "db": "example_database"}
C: GOODBYE
```


## Example 4

```
C: 60 60 B0 17
C: 00 00 00 04 00 00 00 00 00 00 00 00 00 00 00 00
S: 00 00 00 04
C: HELLO {"user_agent": "Example/4.0.0", "scheme": "basic", "principal": "test", "credentials": "test"}
S: SUCCESS {"server": "Neo4j/4.0.0", "connection_id": "example-connection-id:1"}
C: BEGIN {"mode": "r", "db": "example_database", "tx_metadata": {"foo": "bar"}, "tx_timeout": 300}
S: SUCCESS {}
C: RUN "UNWIND [1,2,3,4] AS x RETURN x" {} {}
S: SUCCESS {"fields": ["x"], "qid": 0}
C: PULL {"n": 2}
S: RECORD [1]
S: RECORD [2]
S: SUCCESS {"has_more": true}
C: DISCARD {"n": -1, "qid": 0}
S: SUCCESS {"type": "r", "db": "test"}
C: COMMIT
S: SUCCESS {"bookmark": "neo4j:bookmark-test-1"}
```
