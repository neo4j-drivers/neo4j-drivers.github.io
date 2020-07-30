# Bolt Protocol Server State Specification (DRAFT!!!)


* [Version 4.1](#version-41)
* [Version 4.0](#version-40)
* [Version 3](#version-3)
* [Version 2](#version-2)
* [Version 1](#version-1)


# Version 4.1

## 1. Changes

* no changes

## 2. New

## 3. Server States


# Version 4.0

## 1. Changes

* no changes

## 2. New

* `STREAMING` state

## 3. Server States


# Version 3

## 1. Changes


## 2. New


## 3. Server States



# Version 2

## 1. Changes

No changes.


# Version 1



## Server States

Each connection maintained by a Bolt server will occupy one of several states throughout its lifetime.
This state is used to determine what actions may be undertaken by the client.

| State             | Logic State    | Description                                         |
|-------------------|:--------------:|-----------------------------------------------------|
| `DISCONNECTED`    | x              | no socket connection                                |
| `CONNECTED`       | x              | protocol handshake has been completed successfully  |
| `DEFUNCT`         | x              | the socket connection has been permanently closed   |
| `READY`           |                | ready to accept a `RUN` message                     |
| `STREAMING`       |                | a result is available for streaming from the server |
| `FAILED`          |                | a connection is in a temporarily unusable state     |
| `INTERUPTED`      |                |                                                     |


### `DISCONNECTED`

No **socket connection** has yet been established.
This is the initial state and exists only in a logical sense prior to the socket being opened.

#### Transitions from `DISCONNECTED`

- `<CONNECT>` to `CONNECTED` or `DEFUNCT`


### `CONNECTED`

After a new **protocol connection** has been established and handshake has been completed successfully, the server enters the `CONNECTED` state.
The connection has not yet been authenticated and permits only one transition, through successful initialization, into the `READY` state.

#### Transitions from `CONNECTED`

- `INIT` to `READY` or `DEFUNCT`
- `<DISCONNECT>` to `DEFUNCT`


### `DEFUNCT`

This is not strictly a connection state, but is instead a logical state that exists after a connection has been closed.
When `DEFUNCT`, a connection is permanently not usable.
This may arise due to a graceful shutdown or can occur after an unrecoverable error or protocol violation.
Clients and servers should clear up any resources associated with a connection on entering this state, including closing any open sockets.
This is a terminal state on which no further transitions may be carried out.


### `STREAMING`

When `STREAMING`, a result is available for streaming from server to client.
This result must be fully consumed or discarded by a client before the server can re-enter the `READY` state and allow any further statements to be executed.

#### Transitions from `STREAMING`

- `PULL_ALL` to `READY` or `FAILED`
- `DISCARD_ALL` to `READY` or `FAILED`
- `<INTERRUPT>` to `INTERRUPTED`
- `<DISCONNECT>` to `DEFUNCT`


### `FAILED`

When `FAILED`, a connection is in a temporarily unusable state.
This is generally as the result of encountering a recoverable error.
No more work will be processed until the failure has been acknowledged by `ACK_FAILURE` or until the connection has been `RESET`.
This mode ensures that only one failure can exist at a time, preventing cascading issues from batches of work.

#### Transitions from `FAILED`

- `ACK_FAILURE` to `READY` or `DEFUNCT`
- `<INTERRUPT>` to `INTERRUPTED`
- `<DISCONNECT>` to `DEFUNCT`


### `INTERRUPTED`

This state occurs between the server receiving the jump-ahead `<INTERRUPT>` and the queued `RESET` message, (the `RESET` message triggers an `<INTERRUPT>`).
Most incoming messages are ignored when the server are in an `INTERRUPTED` state, with the exception of the `RESET` that allows transition back to `READY`.

#### Transitions from `INTERRUPTED`

- `RUN` to `INTERRUPTED`
- `DISCARD_ALL` to `INTERRUPTED`
- `PULL_ALL` to `INTERRUPTED`
- `RESET` to `READY` or `DEFUNCT`
- `<INTERRUPT>` to `INTERRUPTED`
- `<DISCONNECT>` to `DEFUNCT`


#### 4.1.2. `INIT` State Transitions

| Initial State | Final State | Response     |
|---------------|-------------|--------------|
| `CONNECTED`   | `READY`     | `SUCCESS {}` |
| `CONNECTED`   | `DEFUNCT`   | `FAILURE {}` |


#### 4.2.2. `RUN` State Transitions

| Initial State | Final State   | Response     |
|---------------|---------------|--------------|
| `READY`       | `STREAMING`   | `SUCCESS {}` |
| `READY`       | `FAILED`      | `FAILURE {}` |
| `INTERRUPTED` | `INTERRUPTED` | `IGNORED`    |


#### 4.3.2. `DISCARD_ALL` State Transitions

| Initial State | Final State   | Response     |
|---------------|---------------|--------------|
| `STREAMING`   | `READY`       | `SUCCESS {}` |
| `STREAMING`   | `FAILED`      | `FAILURE {}` |
| `INTERRUPTED` | `INTERRUPTED` | `IGNORED`    |


#### 4.4.2. `PULL_ALL` State Transitions

| Initial State | Final State   | Response                      |
|---------------|---------------|-------------------------------|
| `STREAMING`   | `READY`       | \[`RECORD` ...\] `SUCCESS {}` |
| `STREAMING`   | `FAILED`      | \[`RECORD` ...\] `FAILURE {}` |
| `INTERRUPTED` | `INTERRUPTED` | `IGNORED`                     |


#### 4.5.2. `ACK_FAILURE` State Transitions

| Initial State | Final State   | Response     |
|---------------|---------------|--------------|
| `FAILED`      | `READY`       | `SUCCESS {}` |
| `FAILED`      | `DEFUNCT`     | `FAILURE {}` |


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

