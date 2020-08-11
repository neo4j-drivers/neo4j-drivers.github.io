# Bolt Protocol Server State Specification - Version 4

* [**Version 4.0**](#version-40)

* [**Version 4.1**](#version-41)

* [**Appendix - Bolt Message State Transitions**](#appendix---bolt-message-state-transitions)


# Version 4.0

## Deltas

Compared to version 3 the `RUN`, `PULL` and `DISCARD` now can re-enter `STREAMING` or `TX_STREAMING`.

## Server States

Each **connection** maintained by a Bolt server will occupy one of several states throughout its lifetime.

This state is used to determine what actions may be undertaken by the client.

| State                                             | Logic State    | Description                                                                              |
|---------------------------------------------------|:--------------:|------------------------------------------------------------------------------------------|
| [`DISCONNECTED`](#server-state---disconnected)    | x              | no socket connection                                                                     |
| [`CONNECTED`](#server-state---connected)          | x              | protocol handshake has been completed successfully                                       |
| [`DEFUNCT`](#server-state---defunct)              | x              | the socket connection has been permanently closed                                        |
| [`READY`](#server-state---ready)                  |                | ready to accept a `RUN` message                                                          |
| [`STREAMING`](#server-state---streaming)          |                | **Auto-commit Transaction**, a result is available for streaming from the server         |
| [`TX_READY`](#server-state---tx_ready)            |                | **Explicit Transaction**, ready to accept a `RUN` message                                |
| [`TX_STREAMING`](#server-state---tx_streaming)    |                | **Explicit Transaction**, a result is available for streaming from the server            |
| [`FAILED`](#server-state---failed)                |                | a connection is in a temporarily unusable state                                          |
| [`INTERRUPTED`](#server-state---interrupted)      |                |                                                                                          |


## Server State - `DISCONNECTED`

No **socket connection** has yet been established.
This is the initial state and exists only in a logical sense prior to the socket being opened.

### Transitions from `DISCONNECTED`

- handshake completed successfully to `CONNECTED`
- handshake did not complete successfully to `DEFUNCT`


## Server State - `CONNECTED`

After a new **protocol connection** has been established and handshake has been completed successfully, the server enters the `CONNECTED` state.
The connection has not yet been authenticated and permits only one transition, through successful initialization, into the `READY` state.

### Transitions from `CONNECTED`

- `<DISCONNECT>` to `DEFUNCT`
- `HELLO` to `READY` or `DEFUNCT`


## Server State - `DEFUNCT`

This is not strictly a connection state, but is instead a logical state that exists after a connection has been closed.

When `DEFUNCT`, a connection is permanently not usable.

This may arise due to a graceful shutdown or can occur after an unrecoverable error or protocol violation.

Clients and servers should clear up any resources associated with a connection on entering this state, including closing any open sockets.

This is a terminal state on which no further transitions may be carried out.

The `<DISCONNECT>` signal will set the connection in the `DEFUNCT` server state.


## Server State - `READY`

### Transitions from `READY`

- `<INTERRUPT>` to `INTERRUPTED`
- `<DISCONNECT>` to `DEFUNCT`
- `RUN` to `STREAMING` or `FAILED`
- `BEGIN` to `TX_READY` or `FAILED`


## Server State - `STREAMING`

When `STREAMING`, a result is available for streaming from server to client.

This result must be fully consumed or discarded by a client before the server can re-enter the `READY` state and allow any further queries to be executed.

### Transitions from `STREAMING`

- `<INTERRUPT>` to `INTERRUPTED`
- `<DISCONNECT>` to `DEFUNCT`
- `PULL` to `READY`, `FAILED` or `STREAMING`
- `DISCARD` to `READY`, `FAILED` or `STREAMING`


#### `<INTERRUPT>` Signal State Transitions

| Initial State | Final State   | Response |
|---------------|---------------|----------|
| `STREAMING`   | `INTERRUPTED` | *n/a*    |

#### `<DISCONNECT>` Signal State Transitions

| Initial State | Final State   | Response |
|---------------|---------------|----------|
| `STREAMING`   | `DEFUNCT`     | *n/a*    |

#### Request Message - `DISCARD` - State Transitions

| Initial State | Final State   | Response                     |
|---------------|---------------|------------------------------|
| `STREAMING`   | `READY`       | `SUCCESS {"has_more": false}`|
| `STREAMING`   | `FAILED`      | `FAILURE {}`                 |
| `STREAMING`   | `STREAMING`   | `SUCCESS {"has_more": true}` |

#### Request Message - `PULL` - State Transitions

| Initial State | Final State   | Response                                      |
|---------------|---------------|-----------------------------------------------|
| `STREAMING`   | `READY`       | \[`RECORD` ...\] `SUCCESS {"has_more": false}`|
| `STREAMING`   | `FAILED`      | \[`RECORD` ...\] `FAILURE {}`                 |
| `STREAMING`   | `STREAMING`   | \[`RECORD` ...\] `SUCCESS {"has_more": true}` |


## Server State - `TX_READY`

### Transitions from `TX_READY`

- `<INTERRUPT>` to `INTERRUPTED`
- `<DISCONNECT>` to `DEFUNCT`
- `RUN` to `TX_STREAMING` or `FAILED`
- `COMMIT` to `READY` or `FAILED`
- `ROLLBACK` to `READY` or `FAILED`

#### `<INTERRUPT>` Signal State Transitions

| Initial State | Final State   | Response |
|---------------|---------------|----------|
| `TX_READY`    | `INTERRUPTED` | *n/a*    |

#### `<DISCONNECT>` Signal State Transitions

| Initial State | Final State   | Response |
|---------------|---------------|----------|
| `TX_READY`    | `DEFUNCT`     | *n/a*    |

#### `RUN` Message State Transitions

| Initial State | Final State      | Response                       |
|---------------|------------------|--------------------------------|
| `TX_READY`    | `TX_STREAMING`   | `SUCCESS {"qid": id::Integer}` |
| `TX_READY`    | `FAILED`         | `FAILURE {}`                   |

#### `COMMIT` Message State Transitions

| Initial State | Final State      | Response     |
|---------------|------------------|--------------|
| `TX_READY`    | `READY`          | `SUCCESS {}` |
| `TX_READY`    | `FAILED`         | `FAILURE {}` |

#### `ROLLBACK` Message State Transitions

| Initial State | Final State      | Response     |
|---------------|------------------|--------------|
| `TX_READY`    | `READY`          | `SUCCESS {}` |
| `TX_READY`    | `FAILED`         | `FAILURE {}` |


## Server State - `TX_STREAMING`

When `TX_STREAMING`, a result is available for streaming from server to client.

This result must be fully consumed or discarded by a client before the server can transition to the `TX_READY` state.

### Transitions from `TX_STREAMING`

- `<INTERRUPT>` to `INTERRUPTED`
- `<DISCONNECT>` to `DEFUNCT`
- `RUN` to `FAILED` or `TX_STREAMING`
- `PULL` to `TX_READY`, `FAILED` or `TX_STREAMING`
- `DISCARD` to `TX_READY`, `FAILED` or `TX_STREAMING`

#### Request Message - `RUN` - State Transitions

| Initial State    | Final State      | Response                       |
|------------------|------------------|--------------------------------|
| `TX_STREAMING`   | `TX_STREAMING`   | `SUCCESS {"qid": id::Integer}` |
| `TX_STREAMING`   | `FAILED`         | `FAILURE {}`                   |

#### Request Message - `DISCARD` - State Transitions

| Initial State    | Final State                                                     | Response                     |
|------------------|-----------------------------------------------------------------|------------------------------|
| `TX_STREAMING`   | `TX_READY` or `TX_STREAMING` if there is other streams open     | `SUCCESS {"has_more": false}`|
| `TX_STREAMING`   | `FAILED`                                                        | `FAILURE {}`                 |
| `TX_STREAMING`   | `TX_STREAMING`                                                  | `SUCCESS {"has_more": true}` |

#### Request Message - `PULL` - State Transitions

`TX_READY` or `TX_STREAMING` if there is other streams open

| Initial State    | Final State                                                    | Response                                      |
|------------------|----------------------------------------------------------------|-----------------------------------------------|
| `TX_STREAMING`   | `TX_READY` or `TX_STREAMING` if there is other streams open    | \[`RECORD` ...\] `SUCCESS {"has_more": false}`|
| `TX_STREAMING`   | `FAILED`                                                       | \[`RECORD` ...\] `FAILURE {}`                 |
| `TX_STREAMING`   | `TX_STREAMING`                                                 | \[`RECORD` ...\] `SUCCESS {"has_more": true}` |


## Server State - `FAILED`

When `FAILED`, a connection is in a temporarily unusable state.

This is generally as the result of encountering a recoverable error.

This mode ensures that only one failure can exist at a time, preventing cascading issues from batches of work.


### Transitions from `FAILED`

- `<INTERRUPT>` to `INTERRUPTED`
- `<DISCONNECT>` to `DEFUNCT`
- `RUN` to `FAILED`
- `PULL` to `FAILED`
- `DISCARD` to `FAILED`


## Server State - `INTERRUPTED`

This state occurs between the server receiving the jump-ahead `<INTERRUPT>` and the queued `RESET` message, (the `RESET` message triggers an `<INTERRUPT>`).

Most incoming messages are ignored when the server are in an `INTERRUPTED` state, with the exception of the `RESET` that allows transition back to `READY`.

The `<INTERRUPT>` signal will set the connection in the `INTERRUPTED` server state.


### Transitions from `INTERRUPTED`

- `<INTERRUPT>` to `INTERRUPTED`
- `<DISCONNECT>` to `DEFUNCT`
- `RUN` to `INTERRUPTED`
- `DISCARD` to `INTERRUPTED`
- `PULL` to `INTERRUPTED`
- `BEGIN` to `INTERRUPTED`
- `COMMIT` to `INTERRUPTED`
- `ROLLBACK` to `INTERRUPTED`
- `RESET` to `READY` or `DEFUNCT`


# Version 4.1

No changes compared to version 4.0.


# Appendix - Bolt Message State Transitions


| Initial State  | Client Message | Triggers Signal | Server Response Summary Message | Final State                                                 |
|----------------|----------------|-----------------|---------------------------------|-------------------------------------------------------------|
| `CONNECTED`    | `HELLO`        |                 | `SUCCESS {}`                    | `READY`                                                     |
| `CONNECTED`    | `HELLO`        |                 | `FAILURE {}`                    | `DEFUNCT`                                                   |
|                |                |                 |                                 |                                                             |
| `READY`        | `RUN`          |                 | `SUCCESS {}`                    | `STREAMING`                                                 |
| `READY`        | `RUN`          |                 | `FAILURE {}`                    | `FAILED`                                                    |
| `READY`        | `BEGIN`        |                 | `SUCCESS {}`                    | `TX_READY`                                                  |
| `READY`        | `BEGIN`        |                 | `FAILURE {}`                    | `FAILED`                                                    |
| `READY`        | `RESET`        | `<INTERRUPT>`   | *n/a*                           |                                                             |
| `READY`        | `GOODBYE`      | `<DISCONNECT>`  | *n/a*                           | `DEFUNCT`                                                   |
|                |                |                 |                                 |                                                             |
| `STREAMING`    | `PULL`         |                 | `SUCCESS {"has_more": true}`    | `STREAMING`                                                 |
| `STREAMING`    | `PULL`         |                 | `SUCCESS {"has_more": false}`   | `READY`                                                     |
| `STREAMING`    | `PULL`         |                 | `FAILURE {}`                    | `FAILED`                                                    |
| `STREAMING`    | `DISCARD`      |                 | `SUCCESS {"has_more": true}`    | `STREAMING`                                                 |
| `STREAMING`    | `DISCARD`      |                 | `SUCCESS {"has_more": false}`   | `READY`                                                     |
| `STREAMING`    | `DISCARD`      |                 | `FAILURE {}`                    | `FAILED`                                                    |
| `STREAMING`    | `RESET`        | `<INTERRUPT>`   | *n/a*                           |                                                             |
| `STREAMING`    | `GOODBYE`      | `<DISCONNECT>`  | *n/a*                           | `DEFUNCT`                                                   |
|                |                |                 |                                 |                                                             |
| `TX_READY`     | `RUN`          |                 | `SUCCESS {"qid": id::Integer}`  | `TX_STREAMING`                                              |
| `TX_READY`     | `RUN`          |                 | `FAILURE {}`                    | `FAILED`                                                    |
| `TX_READY`     | `COMMIT`       |                 | `SUCCESS {}`                    | `READY`                                                     |
| `TX_READY`     | `COMMIT`       |                 | `FAILURE {}`                    | `FAILED`                                                    |
| `TX_READY`     | `ROLLBACK`     |                 | `SUCCESS {}`                    | `READY`                                                     |
| `TX_READY`     | `ROLLBACK`     |                 | `FAILURE {}`                    | `FAILED`                                                    |
| `TX_READY`     | `RESET`        | `<INTERRUPT>`   | *n/a*                           |                                                             |
| `TX_READY`     | `GOODBYE`      | `<DISCONNECT>`  | *n/a*                           | `DEFUNCT`                                                   |
|                |                |                 |                                 |                                                             |
| `TX_STREAMING` | `RUN`          |                 | `SUCCESS {"qid": id::Integer}`  | `TX_STREAMING`                                              |
| `TX_STREAMING` | `RUN`          |                 | `FAILURE {}`                    | `FAILED`                                                    |
| `TX_STREAMING` | `PULL`         |                 | `SUCCESS {"has_more": true}`    | `TX_STREAMING`                                              |
| `TX_STREAMING` | `PULL`         |                 | `SUCCESS {"has_more": false}`   | `TX_READY` or `TX_STREAMING` if there is other streams open |
| `TX_STREAMING` | `PULL`         |                 | `FAILURE {}`                    | `FAILED`                                                    |
| `TX_STREAMING` | `DISCARD`      |                 | `SUCCESS {"has_more": true}`    | `TX_STREAMING`                                              |
| `TX_STREAMING` | `DISCARD`      |                 | `SUCCESS {"has_more": false}`   | `TX_READY` or `TX_STREAMING` if there is other streams open |
| `TX_STREAMING` | `DISCARD`      |                 | `FAILURE {}`                    | `FAILED`                                                    |
| `TX_STREAMING` | `RESET`        | `<INTERRUPT>`   | *n/a*                           |                                                             |
| `TX_STREAMING` | `GOODBYE`      | `<DISCONNECT>`  | *n/a*                           | `DEFUNCT`                                                   |
|                |                |                 |                                 |                                                             |
| `FAILED`       | `RUN`          |                 | `IGNORED`                       | `FAILED`                                                    |
| `FAILED`       | `PULL`         |                 | `IGNORED`                       | `FAILED`                                                    |
| `FAILED`       | `DISCARD`      |                 | `IGNORED`                       | `FAILED`                                                    |
| `FAILED`       | `RESET`        | `<INTERRUPT>`   | *n/a*                           |                                                             |
| `FAILED`       | `GOODBYE`      | `<DISCONNECT>`  | *n/a*                           | `DEFUNCT`                                                   |
|                |                |                 |                                 |                                                             |
| `INTERRUPTED`  | `RUN`          |                 | `IGNORED`                       | `INTERRUPTED`                                               |
| `INTERRUPTED`  | `PULL`         |                 | `IGNORED`                       | `INTERRUPTED`                                               |
| `INTERRUPTED`  | `DISCARD`      |                 | `IGNORED`                       | `INTERRUPTED`                                               |
| `INTERRUPTED`  | `BEGIN`        |                 | `IGNORED`                       | `INTERRUPTED`                                               |
| `INTERRUPTED`  | `COMMIT`       |                 | `IGNORED`                       | `INTERRUPTED`                                               |
| `INTERRUPTED`  | `ROLLBACK`     |                 | `IGNORED`                       | `INTERRUPTED`                                               |
| `INTERRUPTED`  | `RESET`        | `<INTERRUPT>`   | `SUCCESS {}`                    | `READY`                                                     |
| `INTERRUPTED`  | `RESET`        | `<INTERRUPT>`   | `FAILURE {}`                    | `DEFUNCT`                                                   |
| `INTERRUPTED`  | `GOODBYE`      | `<DISCONNECT>`  | *n/a*                           | `DEFUNCT`                                                   |


The `<INTERRUPT>` signal,


| Initial State  | Signal         | Server Response Summary Message | Final State   |
|----------------|----------------|---------------------------------|---------------|
| `READY`        | `<INTERRUPT>`  | *n/a*                           | `INTERRUPTED` |
| `STREAMING`    | `<INTERRUPT>`  | *n/a*                           | `INTERRUPTED` |
| `TX_READY`     | `<INTERRUPT>`  | *n/a*                           | `INTERRUPTED` |
| `TX_STREAMING` | `<INTERRUPT>`  | *n/a*                           | `INTERRUPTED` |
| `FAILED`       | `<INTERRUPT>`  | *n/a*                           | `INTERRUPTED` |
| `INTERRUPTED`  | `<INTERRUPT>`  | *n/a*                           | `INTERRUPTED` |