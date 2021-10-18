---
layout: default
---
# Driver API Specification

<div>
  <p style="background-color:#ffb950; padding:10px; border-radius:5px; color:black;">
  This Document is still in a draft state and the content can change at any time.
  </p>
</div>

This document describes the overview of a Driver and how different scenarios like update the routing table for a clustered Neo4j setup, available URI schemes and TLS settings, etc.


* [Driver Objects](#driver-objects)

* [URI Schemes](#uri-schemes)

* [Client Side Routing](#client-side-routing)

* [Client Side Logging](#client-side-logging)

* [Session](#session)

* [Transaction](#transaction)

* [Causal Chaining](#causal-chaining)



## Driver Objects

* **Driver**
* **Session**
* **Transaction**


* **DriverConfig**
  * `uri::String`
  * `auth::Dictionary`
  * `user_agent::String`
  * `encrypted::Boolean`
  * `trust::String`


* **SessionConfig**
  * `default_access_mode::String`
  * `database::String`
  * `fetch_size::Integer`
  * `bookmarks::List<String>`
  * `imp_user::String`


* TransactionConfig
  * `metadata::Dictionary`
  * `timeout::Integer`


* **AuthToken**
* **TransactionManager**


* **ConnectionPool**
* **Routing**


* **BoltProtocol**
* **PackStream**


* **Neo4jExceptions**
* **DriverExceptions**


## URI Schemes

No TLS enabled.

* `bolt`, connect to a single Neo4j instance.
* `neo4j`, connect to a Neo4j instance and use the routing table information for further connections.


Enable TLS and allow self signed certificate authority.

* `bolt+ssc`
* `neo4j+ssc`


Enable TLS and only allow system enabled certificate authorty and verify hostname.

* `bolt+s`
* `neo4j+s`


## Client Side Routing

Neo4j supports a clustered setup and uses [The Raft Consensus Algorithm](https://raft.github.io/).

See, [https://neo4j.com/docs/operations-manual/current/clustering-advanced/lifecycle/](https://neo4j.com/docs/operations-manual/current/clustering-advanced/lifecycle/).


Each Neo4j **Core** instance in a cluster supports `routing` and `reading`.

Only one Neo4j Core in a cluster can be selected to support `writing` operations. This selection can rotate over time.

The driver should support a **routing table**.

**Read Replicas** are not involved in the Raft Consensus Algorithm, but a Read Replica do return a routing table that only contain the Read Replica itself.


### Fetching Routing Tables

The procedure call to fetch the routing table has varied considerably throughout the various versions of Neo4j.


#### Route Message (4.3 and newer)

| Neo4j | Bolt | Bolt Message                                                       |
|------:|-----:|:-------------------------------------------------------------------|
| 4.3   | 4.3  | `ROUTE {$context} [$bookmarks] $db`                                |
| 4.4   | 4.4  | `ROUTE {$context} [$bookmarks] {"db": $db, "imp_user": $imp_user}` |

The table shows how to fetch the routing table for database `"foo"`:

| Neo4j | Bolt | Bolt Message                                             |
|------:|-----:|:---------------------------------------------------------|
| 4.3   | 4.3  | `ROUTE {"address": "example.org:7687"} ["neo4j-bookmark-transaction:1", "neo4j-bookmark-transaction:2"] "foo"` |
| 4.4   | 4.4  | `ROUTE {"address": "example.org:7687"} ["neo4j-bookmark-transaction:1", "neo4j-bookmark-transaction:2"] {"db": "foo"}` |


Example:

```
C: 60 60 B0 17
C: 00 00 04 04 00 00 00 00 00 00 00 00 00 00 00 00
S: 00 00 04 04
C: HELLO {"scheme": "basic", "principal": "user", "credentials": "password", "user_agent": "Example/4.4.0", "routing": {"address": "localhost:9001", "policy": "example_policy", "region": "example_region"}}
S: SUCCESS {"server": "Neo4j/4.4.0", "connection_id": "bolt-123456789"}
C: ROUTE {"address": "localhost:9001", "policy": "example_policy", "region": "example_region"} ["neo4j-bookmark-transaction:1", "neo4j-bookmark-transaction:2"], {}
S: SUCCESS {"rt": {"ttl": 300, "db": "foo", "servers": [{"addresses": ["127.0.0.1:9001"], "role": "WRITE"}, {"addresses": ["127.0.0.1:9002"], "role": "READ"}, {"addresses": ["127.0.0.1:9001", "127.0.0.1:9002"], "role": "ROUTE"}]}}
C: GOODBYE
```

#### Procedure Call (4.2 and older)

| Neo4j | Bolt | Neo4j Procedure Call                                                                                                                                                         |
|------:|-----:|:-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 3.5   | 3    | [`dbms.cluster.routing.getRoutingTable($context)`](https://neo4j.com/docs/operations-manual/3.5/reference/procedures/)                                                       |
| 4.0   | 4.0  | [`dbms.routing.getRoutingTable($context, $database)`](https://neo4j.com/docs/operations-manual/4.0/reference/procedures/#procedure_dbms_cluster_routing_getroutingtable)     |
| 4.1   | 4.1  | [`dbms.routing.getRoutingTable($context, $database)`](https://neo4j.com/docs/operations-manual/4.1/reference/procedures/#procedure_dbms_cluster_routing_getroutingtable)     |
| 4.2   | 4.2  | [`dbms.routing.getRoutingTable($context, $database)`](https://neo4j.com/docs/operations-manual/4.2/reference/procedures/#procedure_dbms_cluster_routing_getroutingtable)     |


The table shows how to fetch the routing table for database `"foo"`, (Neo4j 3.5 does not support multi-database)


| Neo4j | Bolt  | Bolt Message                                                                                                                          |
|------:|------:|:--------------------------------------------------------------------------------------------------------------------------------------|
| 3.5   | 3     | `RUN "CALL dbms.cluster.routing.getRoutingTable($context)" {"context": {}} {"mode": "r"}`                                             |
| 4.0   | 4.0   | `RUN "CALL dbms.routing.getRoutingTable($context, $database)" {"context": {}, "database": "foo"} {"db": "system", "mode": "r"}`       |
| 4.1   | 4.1   | `RUN "CALL dbms.routing.getRoutingTable($context, $database)" {"context": {}, "database": "foo"} {"db": "system", "mode": "r"}`       |
| 4.2   | 4.2   | `RUN "CALL dbms.routing.getRoutingTable($context, $database)" {"context": {}, "database": "foo"} {"db": "system", "mode": "r"}`       |


Example:

```
C: 60 60 B0 17
C: 00 00 01 04 00 00 00 00 00 00 00 00 00 00 00 00
S: 00 00 01 04
C: HELLO {"scheme": "basic", "principal": "user", "credentials": "password", "user_agent": "Example/4.1.0", "routing": {"address": "localhost:9001", "policy": "example_policy", "region": "example_region"}}
S: SUCCESS {"server": "Neo4j/4.1.0", "connection_id": "bolt-123456789"}
C: RUN "CALL dbms.routing.getRoutingTable($context)" {"context": {"address": "localhost:9001", "policy": "example_policy", "region": "example_region"}} {"mode": "r", "db": "system"}
C: PULL {"n": -1}
S: SUCCESS {"fields": ["ttl", "servers"]}
S: RECORD [300, [{"addresses": ["127.0.0.1:9001"], "role": "WRITE"}, {"addresses": ["127.0.0.1:9002"], "role": "READ"}, {"addresses": ["127.0.0.1:9001", "127.0.0.1:9002"], "role": "ROUTE"}]]
S: SUCCESS {"bookmark": "example-bookmark:1", "type": "r", "t_last": 5, "db": "system"}
C: GOODBYE
```



### Neo4j 4.0 Cluster and Multi-database

#### System Database

* The name of the **system database** is fixed and named `"system"`.
* The system database cannot be changed for a single instance or a cluster.
* The system database exists on each instance.


#### Cluster Member

A cluster contains **Core** members and **Read Replica** members.

* Each cluster member will host the exact same databases.
  
  If a cluster member `A` that is up-to-date has databases `"foo"` and `"system"`, then all other members that is up-to-date in the cluster should also have and only have `"foo"` and `"system"`.
  However at a certain time, the cluster members may or may not be up-to-date, as a result, cluster members may contain different databases.

* Only one Core at any time can be the `leader` (accept `writes`).

* Each database in a cluster has its own **raft group**, each database has its own routing table.

  In other words, the leader/core/read-replica for each database in a cluster can be different.

* There is a **default database** for a single instance and/or a cluster.

   By default, it is named as `"neo4j"`, but the name can be changed to something else such as `"foo"`.
   When changing the name, a restart of the single instance and/or cluster may or may not be required.
   The default database may or may not be allowed to be deleted. (We cannot assume that there is always a default database on each instance.)


* Any Core member in a cluster can provide a routing table for any database inside this cluster.

  Given a seed URL pointing to a Core member this can be used to find any databases in a cluster by fetching the routing table from a Core member.


#### Driver Routing Table

**The Driver should prevent the routing table from growing infinitely.**

The routing table for a specific database should be removed from the routing table if there is a failed to attempt to obtain routing information.

The routing table for a specific database should be removed from the routing table if it is invalid.

An invalid routing table could either be a:

 - Routing table that has timed out where the `TTL` (Time To Live) key for that routing table have ended.
 - Routing table that is pointing to a database that no longer exists.



Here is the workflow the driver should follow when fetching a routing table for database named `"foo"`.

1. Find the routing table for database `"foo"`.

2. If the database does not exist in the routing table, then create an empty routing table with seed URL as initial router.

3. If the routing table is stale, then refresh the routing table with a query to a cluster member that.

4. If any error happens, remove the key "foo" from routing table map.

  The only errors possible are:
  
  - `SECURITY_ERROR`
  - `ROUTING_ERROR`
  - `SERVICE_UNAVILABLE_ERROR`, happens when the driver failed to get routing table for all existing routers


## Client Side Logging

- Logging Levels
- Logging Syntax


## Session

- Connections
- Connection Pool


## Transaction

- Atomic unit of work
- Transaction Manager
- Transaction Functions


## Causal Chaining

- Bookmark
