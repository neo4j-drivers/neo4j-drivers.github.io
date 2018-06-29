# Bolt

"Bolt" is the name of a group of related protocols and serialisation format used by Neo4j.

## Version Negotiation

On connection, clients and servers undergo negotiation to determine the protocol version to use for communication.
Note that the major version number of a Neo4j driver directly correlates to the version of Bolt used by that driver.

- [The Bolt Handshake Protocol](bolt-handshake-protocol.md)


## Version 1

The first version of the protocol is used by the 1.x series drivers and the 3.x series servers.

- [PackStream](packstream-v1.md)
- [The Bolt Messaging Protocol](bolt-messaging-protocol-v1.md)
