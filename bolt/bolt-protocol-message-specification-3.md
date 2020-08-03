# Bolt Protocol Message Specification - Version 3

* [**Version 3**](#version-3)


**NOTE:** Byte values are represented using hexadecimal notation unless otherwise specified.


# Version 3

## 1. Changes

* `INIT` message has been removed.
* `ACK_FAILUER` message has been removed.

## 2. New

* `HELLO` message.
* `GOODBYE` message.
* `BEGIN` message.
* `COMMIT` message.
* `ROLLBACK` message.

## 3. Messages

| Message       | Signature | Request Message | Summary Message | Detail Message | Fields                                                                                                            | Description                                             |
|---------------|:---------:|:---------------:|:---------------:|:--------------:|-------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------|
| `HELLO`       | `01`      | x               |                 |                | `extra::Map( user_agent::String, scheme::String, principal::String, credentials::String)`                         | initialize connection                                   |
| `GOODBYE`     | `02`      | x               |                 |                |                                                                                                                   | close the connection, triggers a `<DISCONNECT>` signal  |
| `RESET`       | `0F`      | x               |                 |                |                                                                                                                   | reset the connection, triggers a `<INTERRUPT>` signal   |
| `RUN`         | `10`      | x               |                 |                | `query::String`, `parameters::Map`                                                                                | execute a query                                         |
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

