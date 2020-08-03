# Bolt Protocol Message Specification - Version 4


* [**Version 4.0**](#version-40)

* [**Version 4.1**](#version-41)


**NOTE:** Byte values are represented using hexadecimal notation unless otherwise specified.


# Version 4.0

## 1. Changes

* `DISCARD_ALL` message renamed to `DISCARD` and introduced new fields.
* `PULL_ALL` message renamed to `PULL` and introduced new fields.

## 2. New

* The `DISCARD` message can now discard an arbitrary number of records.
* The `PULL` message can now fetch an arbitrary number of records.

## 3. Messages

| Message       | Signature | Request Message | Summary Message | Detail Message | Fields                                                                                              | Description                                             |
|---------------|:---------:|:---------------:|:---------------:|:--------------:|-----------------------------------------------------------------------------------------------------|---------------------------------------------------------|
| `HELLO`       | `01`      | x               |                 |                | `extra::Map( user_agent::String, scheme::String, principal::String, credentials::String)`           | initialize connection                                   |
| `GOODBYE`     | `02`      | x               |                 |                |                                                                                                     | close the connection, triggers a `<DISCONNECT>` signal  |
| `RESET`       | `0F`      | x               |                 |                |                                                                                                     | reset the connection, triggers a `<INTERRUPT>` signal   |
| `RUN`         | `10`      | x               |                 |                | `query::String`, `parameters::Map`                                                                  | execute a query                                         |
| `DISCARD`     | `2F`      | x               |                 |                | `n::Integer`, `qid::Integer`                                                                        | discard records                                         |
| `PULL`        | `3F`      | x               |                 |                | `n::Integer`, `qid::Integer`                                                                        | fetch records                                           |
| `BEGIN`       | `11`      | x               |                 |                | \<missing\>                                                                                         | begin a new transaction                                 |
| `COMMIT`      | `12`      | x               |                 |                |                                                                                                     | commit a transaction                                    |
| `ROLLBACK`    | `13`      | x               |                 |                |                                                                                                     | rollback a transaction                                  |
| `SUCCESS`     | `70`      |                 | x               |                | `metadata::Map`                                                                                     | request succeeded                                       |
| `IGNORED`     | `7E`      |                 | x               |                |                                                                                                     | request was ignored                                     |
| `FAILURE`     | `7F`      |                 | x               |                | `metadata::Map`                                                                                     | request failed                                          |
| `RECORD`      | `71`      |                 |                 | x              | `data::List`                                                                                        | data values                                             |


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

### 3.3. `RESET`

### 3.4. `RUN`

### 3.5. `DISCARD`

### 3.6. `PULL`

### 3.7. `BEGIN`

### 3.8. `COMMIT`

### 3.9. `ROLLBACK`

### 3.10. `SUCCESS`

### 3.11. `IGNORED`

### 3.12. `FAILURE`

### 3.13. `RECORD`


# Version 4.1

## 1. Changes

* `BEGIN` message now requires the field address.

## 2. New

## 3. Messages


| Message       | Signature | Request Message | Summary Message | Detail Message | Fields                                                                                              | Description                                             |
|---------------|:---------:|:---------------:|:---------------:|:--------------:|-----------------------------------------------------------------------------------------------------|---------------------------------------------------------|
| `HELLO`       | `01`      | x               |                 |                | `extra::Map( user_agent::String, scheme::String, principal::String, credentials::String)`           | initialize connection                                   |
| `GOODBYE`     | `02`      | x               |                 |                |                                                                                                     | close the connection, triggers a `<DISCONNECT>` signal  |
| `RESET`       | `0F`      | x               |                 |                |                                                                                                     | reset the connection, triggers a `<INTERRUPT>` signal   |
| `RUN`         | `10`      | x               |                 |                | `query::String`, `parameters::Map`                                                                  | execute a query                                         |
| `DISCARD`     | `2F`      | x               |                 |                | `n::Integer`, `qid::Integer`                                                                        | discard records                                         |
| `PULL`        | `3F`      | x               |                 |                | `n::Integer`, `qid::Integer`                                                                        | fetch records                                           |
| `BEGIN`       | `11`      | x               |                 |                | \<missing\>                                                                                         | begin a new transaction                                 |
| `COMMIT`      | `12`      | x               |                 |                |                                                                                                     | commit a transaction                                    |
| `ROLLBACK`    | `13`      | x               |                 |                |                                                                                                     | rollback a transaction                                  |
| `SUCCESS`     | `70`      |                 | x               |                | `metadata::Map`                                                                                     | request succeeded                                       |
| `IGNORED`     | `7E`      |                 | x               |                |                                                                                                     | request was ignored                                     |
| `FAILURE`     | `7F`      |                 | x               |                | `metadata::Map`                                                                                     | request failed                                          |
| `RECORD`      | `71`      |                 |                 | x              | `data::List`                                                                                        | data values                                             |


**Server Signals**

Jump ahead is that the signal will imediatly be available before any messages are processed in the message queue.

| Server Signal   | Jump Ahead | Description            |
|:---------------:|:----------:|------------------------|
| `<INTERRUPT>`   | x          | an interrupt signal    |
| `<DISCONNECT>`  |            | a disconnect signal    |

