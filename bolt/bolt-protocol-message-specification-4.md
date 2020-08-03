# Bolt Protocol Message Specification - Version 4


* [**Version 4.0**](#version-40)

* [**Version 4.1**](#version-41)


**NOTE:** Byte values are represented using hexadecimal notation unless otherwise specified.


# Version 4.0

## 1. Changes

* `DISCARD_ALL` message renamed to `DISCARD` and introduced new fields.
* `PULL_ALL` message renamed to `PULL` and introduced new fields.

## 2. New

* The `BEGIN` message now have a field `db` to specify a database name.
* The `RUN` message now have a field `db` to specify a database name.
* **Explicit transaction** (`BEGIN`+`RUN`) can now get a server response with a `SUCCESS` and metadata key `qid` (query identification).
* The `DISCARD` message can now discard an arbitrary number of records. New fields `n` and `qid`.
* The `DISCARD` message can now get a server response with a `SUCCESS` and metadata key `has_more`. 
* The `PULL` message can now fetch an arbitrary number of records. New fields `n` and `qid`.
* The `PULL` message can now get a server response with a `SUCCESS` and metadata key `has_more`. 


## 3. Messages

| Message       | Signature | Request Message | Summary Message | Detail Message | Fields                                                                                                                                                        | Description                                             |
|---------------|:---------:|:---------------:|:---------------:|:--------------:|---------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------|
| `HELLO`       | `01`      | x               |                 |                | `extra::Map( user_agent::String, scheme::String, principal::String, credentials::String )`                                                                    | initialize connection                                   |
| `GOODBYE`     | `02`      | x               |                 |                |                                                                                                                                                               | close the connection, triggers a `<DISCONNECT>` signal  |
| `RESET`       | `0F`      | x               |                 |                |                                                                                                                                                               | reset the connection, triggers a `<INTERRUPT>` signal   |
| `RUN`         | `10`      | x               |                 |                | `query::String`, `parameters::Map`, `extra::Map( bookmarks::List<String>, tx_timeout::Integer, tx_metadata::Map<String, Value>, mode::String, db:String )`    | execute a query                                         |
| `DISCARD`     | `2F`      | x               |                 |                | `n::Integer`, `qid::Integer`                                                                                                                                  | discard records                                         |
| `PULL`        | `3F`      | x               |                 |                | `n::Integer`, `qid::Integer`                                                                                                                                  | fetch records                                           |
| `BEGIN`       | `11`      | x               |                 |                | `extra::Map( bookmarks::List<String>, tx_timeout::Integer, tx_metadata::Map<String, Value>, mode::String, db::String )`                                       | begin a new transaction                                 |
| `COMMIT`      | `12`      | x               |                 |                |                                                                                                                                                               | commit a transaction                                    |
| `ROLLBACK`    | `13`      | x               |                 |                |                                                                                                                                                               | rollback a transaction                                  |
| `SUCCESS`     | `70`      |                 | x               |                | `metadata::Map<String, Value>`                                                                                                                                | request succeeded                                       |
| `IGNORED`     | `7E`      |                 | x               |                |                                                                                                                                                               | request was ignored                                     |
| `FAILURE`     | `7F`      |                 | x               |                | `metadata::Map( code::String, message::String )`                                                                                                              | request failed                                          |
| `RECORD`      | `71`      |                 |                 | x              | `data::List<Value>`                                                                                                                                           | data values                                             |


**Server Signals**

Jump ahead is that the signal will imediatly be available before any messages are processed in the message queue.

| Server Signal   | Jump Ahead | Description            |
|:---------------:|:----------:|------------------------|
| `<INTERRUPT>`   | x          | an interrupt signal    |
| `<DISCONNECT>`  |            | a disconnect signal    |


### 3.1. `HELLO`

The `HELLO` message request the connection to be authorized for use with the remote database.

The server **must** be in the `CONNECTED` state to be able to process a `HELLO` message.
For any other states, receipt of an `HELLO` request **must** be considered a protocol violation and lead to connection closure.

Clients should send `HELLO` message to the server immediately after connection and process the corresponding response before using that connection in any other way.

If authentication fails, the server **must** respond with a `FAILURE` message and immediately close the connection.

Clients wishing to retry initialization should establish a new connection.

**Signature:** `01`

**Fields:**

```
extra::Map(
  user_agent::String,
  scheme::String,
  principal::String,
  credentials::String,
)
```

  - The `user_agent` should conform to `"name/version"` for example `"my-client/1.2.3"`.
  - The `scheme` is the authentication scheme. Predefined schemes are “none”, “basic”, “kerberos”

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
HELLO {"user_agent": "example-agent/4.0.0", "scheme": "basic", "principal": "user-id-1", "credentials": "user-password"}
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


### 3.2. `GOODBYE`

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

### 3.3. `RESET`

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

### 3.4. `RUN`

The `RUN` message requests that a Cypher query is executed with a set of parameters and additional extra data.

This message could both be used for running an explicit transaction or an autocommit transaction. The transaction type is implied by the order of message sequence.

**Signature:** `10`

**Fields:**

```
query::String
paramaters::Map<String, Value>
extra::Map(
  bookmarks::List<String>,
  tx_timeout::Integer,
  tx_metadata::Map<String, Value>,
  mode::String,
  db:String,
)
```

  - The `query` can be a Cypher syntax or a procedure call.
  - The `parameters` is a map of parameters to be used in the `query` string.

  An **explicit transaction** (`BEGIN`+`RUN`) does not carry any data in the extra `extra` field.

  For **autocommit transaction** (`RUN`) the `extra` field carries:

  - The `bookmarks` is a list of strings containg some kind of bookmark identification e.g [“neo4j-bookmark-transaction:1”, “neo4j-bookmark-transaction:2”]
  - The `tx_timeout` is an integer in that specifies a transaction timeout in ms.
  - The `tx_metadata` is a map that can contain some metadata information, mainly used for logging.
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

### 3.5. `DISCARD`

The `DISCARD` message requests that the remainder of the result stream should be thrown away.

**Signature:** `2F`

**Fields:**

```
extra::Map{
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


### 3.6. `PULL`

The `PULL` message requests that the remainder of the result stream should be thrown away.

**Signature:** `3F`

**Fields:**

```
extra::Map{
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
  - `stats::Map<String, Value>`, counter information, such as db-hits etc.
  - `plan::Map<String, Value>`, plan result.
  - `profile::Map<String, Value>`, profile result.
  - `notifications::Map<String, Value>`, any notification generated during execution of this statement.
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


### 3.7. `BEGIN`

The `BEGIN` message request the creation of a new **explicit transaction**.

This message should then be followed by a `RUN` message.

The **explicit transaction** is closed with either the `COMMIT` message or `ROLLBACK` message.


**Signature:** `11`

**Fields:**

```
extra::Map(
  `bookmarks::List<String>`,
  `tx_timeout::Integer`,
  `tx_metadata::Map<String, Value>`
  `mode::String`,
  `db::String`
)
```

  - The `bookmarks` is a list of strings containg some kind of bookmark identification e.g [“neo4j-bookmark-transaction:1”, “neo4j-bookmark-transaction:2”]
  - The `tx_timeout` is an integer in that specifies a transaction timeout in ms.
  - The `tx_metadata` is a map that can contain some metadata information, mainly used for logging.
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


### 3.8. `COMMIT`

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

### 3.9. `ROLLBACK`

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

### 3.10. `SUCCESS` - summary message

The `SUCCESS` message indicates that the corresponding request has succeeded as intended.

It may contain metadata relating to the outcome.

Metadata keys are described in the section of this document relating to the message that began the exchange.

**Signature:** `70`

**Fields:**

```
metadata::Map<String, Value>
```

#### Synopsis

```
SUCCESS {metadata}
```

Example:

```
SUCCESS {"example": "see specific message for server response metadata"}
```


### 3.11. `IGNORED` - summary message

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

### 3.12. `FAILURE` - summary message


**Signature:** `7F`

**Fields:**

```
metadata::Map(
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


### 3.13. `RECORD` - detail message

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

## 1. Changes

* The `BEGIN` message fields have change. 

## 2. New

* The `BEGIN` message now requires the sub field `routing::Map( address::String, )`.


## 3. Messages


| Message       | Signature | Request Message | Summary Message | Detail Message | Fields                                                                                                                                                        | Description                                             |
|---------------|:---------:|:---------------:|:---------------:|:--------------:|---------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------|
| `HELLO`       | `01`      | x               |                 |                | `extra::Map( user_agent::String, scheme::String, principal::String, credentials::String, routing::Map(address::String) )`                                     | initialize connection                                   |
| `GOODBYE`     | `02`      | x               |                 |                |                                                                                                                                                               | close the connection, triggers a `<DISCONNECT>` signal  |
| `RESET`       | `0F`      | x               |                 |                |                                                                                                                                                               | reset the connection, triggers a `<INTERRUPT>` signal   |
| `RUN`         | `10`      | x               |                 |                | `query::String`, `parameters::Map`, `extra::Map( bookmarks::List<String>, tx_timeout::Integer, tx_metadata::Map<String, Value>, mode::String, db:String )`    | execute a query                                         |
| `DISCARD`     | `2F`      | x               |                 |                | `n::Integer`, `qid::Integer`                                                                                                                                  | discard records                                         |
| `PULL`        | `3F`      | x               |                 |                | `n::Integer`, `qid::Integer`                                                                                                                                  | fetch records                                           |
| `BEGIN`       | `11`      | x               |                 |                | `extra::Map( bookmarks::List<String>, tx_timeout::Integer, tx_metadata::Map<String, Value>, mode::String, db::String )`                                       | begin a new transaction                                 |
| `COMMIT`      | `12`      | x               |                 |                |                                                                                                                                                               | commit a transaction                                    |
| `ROLLBACK`    | `13`      | x               |                 |                |                                                                                                                                                               | rollback a transaction                                  |
| `SUCCESS`     | `70`      |                 | x               |                | `metadata::Map<String, Value>`                                                                                                                                | request succeeded                                       |
| `IGNORED`     | `7E`      |                 | x               |                |                                                                                                                                                               | request was ignored                                     |
| `FAILURE`     | `7F`      |                 | x               |                | `metadata::Map( code::String, message::String )`                                                                                                              | request failed                                          |
| `RECORD`      | `71`      |                 |                 | x              | `data::List<Value>`                                                                                                                                           | data values                                             |

**Server Signals**

Jump ahead is that the signal will imediatly be available before any messages are processed in the message queue.

| Server Signal   | Jump Ahead | Description            |
|:---------------:|:----------:|------------------------|
| `<INTERRUPT>`   | x          | an interrupt signal    |
| `<DISCONNECT>`  |            | a disconnect signal    |



# Appendix - Message Exchange Examples

* The `C:` stands for client.
* The `S:` stands for server.


## Example 1

```
C: 60 60 B0 17
C: 00 00 00 04 00 00 00 00 00 00 00 00 00 00 00 00
S: 00 00 00 04 00
C: HELLO {"user_agent": "test", "scheme": "basic", "principal": "test", "credentials": "test"}
S: SUCCESS {"server": "Neo4j/4.0.0", "connection_id": "example-connection-id:1"}
C: GOODBYE
```


## Example 2

```
C: 60 60 B0 17
C: 00 00 00 04 00 00 00 00 00 00 00 00 00 00 00 00
S: 00 00 00 04 00
C: HELLO {"user_agent": "test/4.0.0", "scheme": "basic", "principal": "test", "credentials": "test"}
S: SUCCESS {"server": "Neo4j/4.0.0", "connection_id": "example-connection-id:1"}
C: RUN "RETURN $x AS example" {"x": 123} {"mode": "r", "db": "example_database"}
S: SUCCESS {"fields": ["example"]}
C: PULL {"n": -1}
S: RECORD [123]
S: SUCCESS {"bookmark": "example-bookmark:1", "t_last": 300, "type": "r", "db": "example_database"}
C: GOODBYE
```
