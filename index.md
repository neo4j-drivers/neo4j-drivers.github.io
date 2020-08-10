# Overview of Driver Technology

The sections below contain links to current, historic, and proposed future specifications. 


## PackStream

PackStream is a binary [presentation](https://en.wikipedia.org/wiki/Presentation_layer) format for the exchange of richly-typed data.
It provides a syntax layer for the Bolt messaging protocol.

- [**Version 1**](packstream/packstream-specification-1.md), corresponds to the first releases of the PackStream specification.


## Bolt

Bolt is an [application protocol](https://en.wikipedia.org/wiki/Application_layer) for the execution of database queries via a database query language, such as [Cypher](https://www.opencypher.org/).
It is generally carried over a regular [TCP](https://tools.ietf.org/html/rfc793) or [WebSocket](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API) connection.

Bolt inherits its core type system from PackStream, over which its messages are generally carried.
Each version of Bolt provides a number of type system extensions, via the PackStream type extension mechanism.  

### Bolt Protocol Handshake Specification

All Bolt connections begin with a handshake to negotiate which version of the messaging protocol to use.
Following a successful negotiation, the agreed messaging protocol then takes ownership of the connection for the remainder of its lifetime.
The handshake itself is not versioned. 

* **Version 1**, corresponds to the first releases of the handshake specification.
* **Version 4.0**, incorporates an updated handshake specification. Now supports Major and Minor versions.

[**Bolt Protocol Handshake Specification**](bolt/bolt-protocol-handshake-specification.md)


### Bolt Protocol Message Specification

* [**Version 1**](bolt/bolt-protocol-message-specification-1.md), corresponds to the first releases of the message specification. Uses **PackStream Version 1**.
* [**Version 2**](bolt/bolt-protocol-message-specification-2.md), incorporates no changes to the message specification.
* [**Version 3**](bolt/bolt-protocol-message-specification-3.md), incorporates an updated message specification.
* [**Version 4.0**](bolt/bolt-protocol-message-specification-4.md), incorporates an updated message specification.
* [**Version 4.1**](bolt/bolt-protocol-message-specification-4.md), incorporates an updated message specification.


### Bolt Protocol Server State Specification

For the server, each connection using the Bolt Protocol will occupy one of several states throughout its lifetime.

This state is used to determine what actions may be undertaken by the client. Each server state specification corresponds to a message specification with the same version.

* [**Version 1**](bolt/bolt-protocol-server-state-specification-1.md), first version that defines the server states.
* [**Version 2**](bolt/bolt-protocol-server-state-specification-2.md), incorporates no changes to the server state specification.
* [**Version 3**](bolt/bolt-protocol-server-state-specification-3.md), incorporates major changes to the server state specification.
* [**Version 4.0**](bolt/bolt-protocol-server-state-specification-4.md), incorporates some changes to the server state specification.
* [**Version 4.1**](bolt/bolt-protocol-server-state-specification-4.md), incorporates no changes to the server state specification.


## Bolt Protocol and Neo4j Compatibility


| Neo4j Version | Bolt `1` | Bolt `2` | Bolt `3` | Bolt `4.0` | Bolt `4.1` | Bolt `4.2`  |
|:-------------:|:--------:|:--------:|:--------:|:----------:|:----------:|:-----------:|
| `3.0`         | `x`      |          |          |            |            |             |
| `3.1`         | `x`      |          |          |            |            |             |
| `3.2`         | `x`      |          |          |            |            |             |
| `3.3`         | `x`      |          |          |            |            |             |
| `3.4`         | `(x)`    | `x`      |          |            |            |             |
| `3.5`         |          | `(x)`    | `x`      |            |            |             |
| `4.0`         |          |          | `(x)`    | `x`        |            |             |
| `4.1`         |          |          | `(x)`    | `(x)`      | `x`        |             |
| `4.2`         |          |          | `(x)`    | `(x)`      | `(x)`      | `x`         |


The `(x)` denotes that support could be removed in next version of Neo4j.


[//]: ## Neo4j Driver API

[//]: The official Neo4j drivers export a uniform API.

[//]: This allows driver concepts and naming to be shared across ecosystems, making transition between languages and multi-language support easier and more consistent.

[//]: [Driver API Specification](driver\_api/driver-api-specification.md)


## Neo4j Drivers

[Java Driver](https://github.com/neo4j/neo4j-java-driver)

[JavaScript Driver](https://github.com/neo4j/neo4j-javascript-driver)

[.NET Driver](https://github.com/neo4j/neo4j-dotnet-driver)

[Python Driver](https://github.com/neo4j/neo4j-python-driver)

[Go Driver](https://github.com/neo4j/neo4j-go-driver)

