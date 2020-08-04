# Bolt Protocol Message Specification - Version 3

* [**Version 3**](#version-3)


**NOTE:** Byte values are represented using hexadecimal notation unless otherwise specified.


# Version 3

Added BEGIN, COMMIT, ROLLBACK
INIT became HELLO, added GOODBYE
Removed ACK_FAILURE (using RESET instead)
Added extra metadata field to RUN and BEGIN

## 1. Deltas

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

## 2. Overview of Major Version Changes



## 3. Messages

| Message       | Signature | Request Message | Summary Message | Detail Message | Fields                                                                                                            | Description                                             |
|---------------|:---------:|:---------------:|:---------------:|:--------------:|-------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------|
| `HELLO`       | `01`      | x               |                 |                | `extra::Map( user_agent::String, scheme::String, principal::String, credentials::String)`                         | initialize connection                                   |
| `GOODBYE`     | `02`      | x               |                 |                |                                                                                                                   | close the connection, triggers a `<DISCONNECT>` signal  |
| `RESET`       | `0F`      | x               |                 |                |                                                                                                                   | reset the connection, triggers a `<INTERRUPT>` signal   |
| `RUN`         | `10`      | x               |                 |                | `query::String`, `parameters::Map`, `extra::Map`                                                                                | execute a query                                         |
| `DISCARD_ALL` | `2F`      | x               |                 |                |                                                                                                                   | discard all records                                     |
| `PULL_ALL`    | `3F`      | x               |                 |                |                                                                                                                   | fetch all records                                       |
| `BEGIN`       | `11`      | x               |                 |                | `extra::Map( bookmarks::List<String>, tx_timeout::Integer, tx_metadata::Map<String, Value>, mode::String)`        | begin a new transaction                                 |
| `COMMIT`      | `12`      | x               |                 |                |                                                                                                                   | commit a transaction                                    |
| `ROLLBACK`    | `13`      | x               |                 |                |                                                                                                                   | rollback a transaction                                  |
| `SUCCESS`     | `70`      |                 | x               |                | `metadata::Map`                                                                                                   | request succeeded                                       |
| `IGNORED`     | `7E`      |                 | x               |                |                                                                                                                   | request was ignored                                     |
| `FAILURE`     | `7F`      |                 | x               |                | `metadata::Map`                                                                                                   | request failed                                          |
| `RECORD`      | `71`      |                 |                 | x              | `data::List`                                                                                                      | data values                                             |




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
C: 00 00 00 03 00 00 00 00 00 00 00 00 00 00 00 00
S: 00 00 00 03
C: HELLO {"user_agent": "test", "scheme": "basic", "principal": "test", "credentials": "test"}
S: SUCCESS {"server": "Neo4j/3.5.0", "connection_id": "123e4567-e89b-12d3-a456-426655440000"}
C: GOODBYE
```