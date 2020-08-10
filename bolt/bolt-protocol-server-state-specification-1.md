# Bolt Protocol Server State Specification - Version 1

* [**Version 1**](#version-1)
* [**Appendix - Bolt Message State Transitions**](#appendix---bolt-message-state-transitions)

# Version 1

## Server States

Each **connection** maintained by a Bolt server will occupy one of several states throughout its lifetime.

This state is used to determine what actions may be undertaken by the client.

| State                                             | Logic State    | Description                                         |
|:--------------------------------------------------|:--------------:|:----------------------------------------------------|
| [`DISCONNECTED`](#server-state---disconnected)    | x              | no socket connection                                |
| [`CONNECTED`](#server-state---connected)          | x              | protocol handshake has been completed successfully  |
| [`DEFUNCT`](#server-state---defunct)              | x              | the socket connection has been permanently closed   |
| [`READY`](#server-state---ready)                  |                | ready to accept a `RUN` message                     |
| [`STREAMING`](#server-state---streaming)          |                | a result is available for streaming from the server |
| [`FAILED`](#server-state---failed)                |                | a connection is in a temporarily unusable state     |
| [`INTERRUPTED`](#server-state---interrupted)      |                | the server got a `<INTERRUPT>` signal               |


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
- `INIT` to `READY` or `DEFUNCT`


#### `<DISCONNECT>` Signal State Transitions

| Initial State | Final State   | Response |
|---------------|---------------|----------|
| `CONNECTED`   | `DEFUNCT`     | *n/a*    |

#### `INIT` Message State Transitions

| Initial State | Final State | Response     |
|---------------|-------------|--------------|
| `CONNECTED`   | `READY`     | `SUCCESS {}` |
| `CONNECTED`   | `DEFUNCT`   | `FAILURE {}` |


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

#### `<INTERRUPT>` Signal State Transitions

| Initial State | Final State   | Response |
|---------------|---------------|----------|
| `READY`       | `INTERRUPTED` | *n/a*    |

#### `<DISCONNECT>` Signal State Transitions

| Initial State | Final State   | Response |
|---------------|---------------|----------|
| `READY`       | `DEFUNCT`     | *n/a*    |

#### `RUN` Message State Transitions

| Initial State | Final State   | Response     |
|---------------|---------------|--------------|
| `READY`       | `STREAMING`   | `SUCCESS {}` |
| `READY`       | `FAILED`      | `FAILURE {}` |


## Server State - `STREAMING`

When `STREAMING`, a result is available for streaming from server to client.

This result must be fully consumed or discarded by a client before the server can re-enter the `READY` state and allow any further queries to be executed.

### Transitions from `STREAMING`

- `<INTERRUPT>` to `INTERRUPTED`
- `<DISCONNECT>` to `DEFUNCT`
- `PULL_ALL` to `READY` or `FAILED`
- `DISCARD_ALL` to `READY` or `FAILED`


#### `<INTERRUPT>` Signal State Transitions

| Initial State | Final State   | Response |
|---------------|---------------|----------|
| `STREAMING`   | `INTERRUPTED` | *n/a*    |

#### `<DISCONNECT>` Signal State Transitions

| Initial State | Final State   | Response |
|---------------|---------------|----------|
| `STREAMING`   | `DEFUNCT`     | *n/a*    |

#### `DISCARD_ALL` Message State Transitions

| Initial State | Final State   | Response     |
|---------------|---------------|--------------|
| `STREAMING`   | `READY`       | `SUCCESS {}` |
| `STREAMING`   | `FAILED`      | `FAILURE {}` |

#### `PULL_ALL` Message State Transitions

| Initial State | Final State   | Response                      |
|---------------|---------------|-------------------------------|
| `STREAMING`   | `READY`       | \[`RECORD` ...\] `SUCCESS {}` |
| `STREAMING`   | `FAILED`      | \[`RECORD` ...\] `FAILURE {}` |


## Server State - `FAILED`

When `FAILED`, a connection is in a temporarily unusable state.

This is generally as the result of encountering a recoverable error.

No more work will be processed until the failure has been acknowledged by `ACK_FAILURE` or until the connection has been `RESET`.

This mode ensures that only one failure can exist at a time, preventing cascading issues from batches of work.


### Transitions from `FAILED`

- `<INTERRUPT>` to `INTERRUPTED`
- `<DISCONNECT>` to `DEFUNCT`
- `ACK_FAILURE` to `READY` or `DEFUNCT`


#### `<INTERRUPT>` Signal State Transitions

| Initial State | Final State   | Response |
|---------------|---------------|----------|
| `FAILED`      | `INTERRUPTED` | *n/a*    |

#### `<DISCONNECT>` Signal State Transitions

| Initial State | Final State   | Response |
|---------------|---------------|----------|
| `FAILED`      | `DEFUNCT`     | *n/a*    |

#### `ACK_FAILURE` Message State Transitions

| Initial State | Final State   | Response     |
|---------------|---------------|--------------|
| `FAILED`      | `READY`       | `SUCCESS {}` |
| `FAILED`      | `DEFUNCT`     | `FAILURE {}` |


## Server State - `INTERRUPTED`

This state occurs between the server receiving the jump-ahead `<INTERRUPT>` and the queued `RESET` message, (the `RESET` message triggers an `<INTERRUPT>`).

Most incoming messages are ignored when the server are in an `INTERRUPTED` state, with the exception of the `RESET` that allows transition back to `READY`.

The `<INTERRUPT>` signal will set the connection in the `INTERRUPTED` server state.


### Transitions from `INTERRUPTED`

- `<INTERRUPT>` to `INTERRUPTED`
- `<DISCONNECT>` to `DEFUNCT`
- `RUN` to `INTERRUPTED`
- `PULL_ALL` to `INTERRUPTED`
- `DISCARD_ALL` to `INTERRUPTED`
- `ACK_FAILURE` to `INTERRUPTED`
- `RESET` to `READY` or `DEFUNCT`


#### `<INTERRUPT>` Signal State Transitions

| Initial State | Final State   | Response |
|---------------|---------------|----------|
| `INTERRUPTED` | `INTERRUPTED` | *n/a*    |

#### `<DISCONNECT>` Signal State Transitions

| Initial State | Final State   | Response |
|---------------|---------------|----------|
| `INTERRUPTED` | `DEFUNCT`     | *n/a*    |

#### `RUN` Message State Transitions

| Initial State | Final State   | Response     |
|---------------|---------------|--------------|
| `INTERRUPTED` | `INTERRUPTED` | `IGNORED`    |

#### `DISCARD_ALL` Message State Transitions

| Initial State | Final State   | Response     |
|---------------|---------------|--------------|
| `INTERRUPTED` | `INTERRUPTED` | `IGNORED`    |

#### `PULL_ALL` Message State Transitions

| Initial State | Final State   | Response     |
|---------------|---------------|--------------|
| `INTERRUPTED` | `INTERRUPTED` | `IGNORED`    |

#### `RESET` Message State Transitions

| Initial State | Final State   | Response     |
|---------------|---------------|--------------|
| `INTERRUPTED` | `READY`       | `SUCCESS {}` |
| `INTERRUPTED` | `DEFUNCT`     | `FAILURE {}` |


# Appendix - Bolt Message State Transitions


| Initial State | Client Message | Triggers Signal | Server Response Summary Message | Final State   |
|---------------|----------------|-----------------|---------------------------------|---------------| 
| `CONNECTED`   | `INIT`         |                 | `SUCCESS {}`                    | `READY`       |
| `CONNECTED`   | `INIT`         |                 | `FAILURE {}`                    | `DEFUNCT`     |
| `READY`       | `RUN`          |                 | `SUCCESS {}`                    | `STREAMING`   |
| `READY`       | `RUN`          |                 | `FAILURE {}`                    | `FAILED`      |
| `READY`       | `RESET`        | `<INTERRUPT>`   | *n/a*                           |               |
| `STREAMING`   | `PULL_ALL`     |                 | `SUCCESS {}`                    | `READY`       |
| `STREAMING`   | `PULL_ALL`     |                 | `FAILURE {}`                    | `FAILED`      |
| `STREAMING`   | `DISCARD_ALL`  |                 | `SUCCESS {}`                    | `READY`       |
| `STREAMING`   | `DISCARD_ALL`  |                 | `FAILURE {}`                    | `FAILED`      |
| `STREAMING`   | `RESET`        | `<INTERRUPT>`   | *n/a*                           |               |
| `FAILED`      | `RUN`          |                 | `IGNORED`                       | `FAILED`      |
| `FAILED`      | `PULL_ALL`     |                 | `IGNORED`                       | `FAILED`      |
| `FAILED`      | `DISCARD_ALL`  |                 | `IGNORED`                       | `INTERRUPTED` |
| `FAILED`      | `ACK_FAILURE`  |                 | `SUCCESS {}`                    | `READY`       | 
| `FAILED`      | `ACK_FAILURE`  |                 | `FAILURE {}`                    | `DEFUNCT`     |
| `FAILED`      | `RESET`        | `<INTERRUPT>`   | *n/a*                           |               |
| `INTERRUPTED` | `RUN`          |                 | `IGNORED`                       | `INTERRUPTED` |
| `INTERRUPTED` | `PULL_ALL`     |                 | `IGNORED`                       | `INTERRUPTED` |
| `INTERRUPTED` | `DISCARD_ALL`  |                 | `IGNORED`                       | `INTERRUPTED` |
| `INTERRUPTED` | `ACK_FAILURE`  |                 | `IGNORED`                       | `INTERRUPTED` |
| `INTERRUPTED` | `RESET`        | `<INTERRUPT>`   | `SUCCESS {}`                    | `READY`       |
| `INTERRUPTED` | `RESET`        | `<INTERRUPT>`   | `FAILURE {}`                    | `DEFUNCT`     |


The `<INTERRUPT>` signal,


| Initial State | Signal         | Server Response Summary Message | Final State   |
|---------------|----------------|---------------------------------|---------------|
| `READY`       | `<INTERRUPT>`  | *n/a*                           | `INTERRUPTED` |
| `STREAMING`   | `<INTERRUPT>`  | *n/a*                           | `INTERRUPTED` |
| `FAILED`      | `<INTERRUPT>`  | *n/a*                           | `INTERRUPTED` |
| `INTERRUPTED` | `<INTERRUPT>`  | *n/a*                           | `INTERRUPTED` |
