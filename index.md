# Overview of Driver Technology

The sections below contain links to current, historic, and proposed future specifications. 

Unlinked entries in the lists below generally represent known future artifacts that have not yet been designed or current or historic artifacts that have no specification available.
Such entries exist for completeness.


## PackStream

PackStream is a binary [presentation](https://en.wikipedia.org/wiki/Presentation_layer) format for the exchange of richly-typed data.
It provides a syntax layer for the Bolt messaging protocol.

- [PackStream Specification v1](packstream/packstream-specification-v1.md) (Neo4j 3.0 to 3.5)
- PackStream Specification v2 (Neo4j 4.0)


### Jolt

Jolt is a proposed PackStream spin-off that provides identical data exchange capabilities to PackStream within a pure [JSON](http://json.org/) context.
This introduces readability at the expense of a slightly higher byte count.

Jolt is intended primarily for use over an HTTP connection and can be useful within network environments that have a requirement for the automatic inspection of traffic.  
 
- [Jolt Specification v1](jolt/jolt-specification-v1.md)


## Bolt

Bolt is an [application protocol](https://en.wikipedia.org/wiki/Application_layer) for the execution of database queries via a database query language, such as [Cypher](https://www.opencypher.org/).
It is generally carried over a regular [TCP](https://tools.ietf.org/html/rfc793) or [WebSocket](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API) connection.

Bolt inherits its core type system from PackStream, over which its messages are generally carried.
Each version of Bolt provides a number of type system extensions, via the PackStream type extension mechanism.  

### Bolt Handshake

All Bolt connections begin with a handshake to negotiate which version of the messaging protocol to use.
Following a successful negotiation, the agreed messaging protocol then takes ownership of the connection for the remainder of its lifetime.
The handshake itself is not versioned. 

- [Bolt Handshake Protocol Specification](bolt/bolt-handshake-protocol-specification.md)

### Bolt v1 (Neo4j 3.0 to 3.3)

Version 1 corresponds to the first releases of the messaging protocol and the type system.

- [Bolt Messaging Protocol Specification v1](bolt/bolt-messaging-protocol-specification-v1.md)
- [Bolt Type System Extensions v1](types/bolt-type-system-extensions-v1.md)

### Bolt v2 (Neo4j 3.4)

Version 2 incorporates an updated type system, but retains the messaging protocol from version 1.
There is consequently no second version of the Bolt Messaging Protocol Specification.

- Bolt Type System Extensions v2

### Bolt v3 (Neo4j 3.5)

Version 3 incorporates both an updated type system and an updated messaging protocol.

- [Bolt Messaging Protocol Specification v3](bolt/bolt-messaging-protocol-specification-v3.md)
- Bolt Type System Extensions v3

### Bolt v4 (Neo4j 4.0)

Version 4 incorporates both an updated type system and an updated messaging protocol.

- Bolt Messaging Protocol Specification v4
- Bolt Type System Extensions v4


## Driver API:

The official Neo4j drivers export a uniform API.
This allows driver concepts and naming to be shared across ecosystems, making transition between languages and multi-language support easier and more consistent.

- v1.0
- v1.1
- v1.2
- v1.3
- v1.4
- v1.5
- v1.6
- v1.7
- v2.0


## Connectors

Connectors are low-level libraries that provide Bolt messaging and routing capabilities.
They are primarily intended for use by drivers and other tooling.
It is recommended that application developers choose a driver over a connector for general purpose integration with Neo4j.

- [Seabolt](connectors/seabolt.md) (C Connector)


## Tools

The links below provide extra resources for driver authors.

- BoltKit
