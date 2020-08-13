# Driver API Specification (DRAFT !!!)


## Driver Objects

* Driver
* Session
* Transaction


* DriverConfig
  * uri::String
  * auth::Map
  * user\_agent::String
  * encrypted::Boolean
  * trust::String


* SessionConfig
  * default\_access\_mode::String
  * database::String
  * fetch\_size::Integer
  * bookmarks::List\<String\>


* TransactionConfig
  * metadata::Map
  * timeout::Integer


* AuthToken
* TransactionManager


* ConnectionPool
* Routing


* BoltProtocol
* PackStream


* Neo4jExceptions
* DriverExceptions


## URI Schemes

* `bolt`
* `neo4j`

## Client Side Routing

Neo4j supports a clustered setup and uses [The Raft Consensus Algorithm](https://raft.github.io/).

See, [https://neo4j.com/docs/operations-manual/current/clustering-advanced/lifecycle/](https://neo4j.com/docs/operations-manual/current/clustering-advanced/lifecycle/).


Each Neo4j **Core** instance in a cluster supports `routing` and `reading`.

Only one Neo4j Core in a cluster can be selected to support `writing` operations. This selection can rotate over time.

The driver should support a **routing table**.

**Read Replicas** are not involved in the Raft Consensus Algorithm, but a Read Replica do return a routing table that only contain the Read Replica itself.


### Routing Table Procedure Call

The procedure call to fetch the routing table has varied considerably throughout the various versions of Neo4j.


| Neo4j | Bolt | Neo4j Procedure Call                                                                                                          |
|------:|-----:|:------------------------------------------------------------------------------------------------------------------------------|
| 3.5   | 3    | [`dbms.cluster.routing.getRoutingTable($context)`](https://neo4j.com/docs/operations-manual/3.5/reference/procedures/)        |
| 4.0   | 4.0  | [`dbms.routing.getRoutingTable($context, $database)`](https://neo4j.com/docs/operations-manual/4.0/reference/procedures/)     |
| 4.1   | 4.1  | [`dbms.routing.getRoutingTable($context, $database)`](https://neo4j.com/docs/operations-manual/4.1/reference/procedures/)     |
| 4.2   | 4.2  | ?                                                                                                                             |


The table shows how to fetch the routing table for database `"foo"`, (Neo4j 3.5 does not support multi-database)


| Neo4j | Bolt  | Bolt Message                                                                                                                          |
|------:|------:|:--------------------------------------------------------------------------------------------------------------------------------------|
| 3.5   | 3     | `RUN "CALL dbms.cluster.routing.getRoutingTable($context)" {"context": {}} {"mode": "r"}`                                             |
| 4.0   | 4.0   | `RUN "CALL dbms.routing.getRoutingTable($context, $database)" {"context": {}, "database": "foo"} {"database": "system", "mode": "r"}` |
| 4.1   | 4.1   | `RUN "CALL dbms.routing.getRoutingTable($context, $database)" {"context": {}, "database": "foo"} {"database": "system", "mode": "r"}` |
| 4.2   | 4.2   | ?                                                                                                                                     |



```


C: HELLO {"scheme": "basic", "principal": "test", "credentials": "test", "user_agent": "test", "routing": {"address": "localhost:9001", "policy": "my_policy", "region": "china"}}
S: SUCCESS {"server": "Neo4j/4.1.0", "connection_id": "bolt-123456789"}
C: RUN "CALL dbms.routing.getRoutingTable($context)" {"context": {"address": "localhost:9001", "policy": "my_policy", "region": "china"}} {"mode": "r", "db": "system"}
C: PULL {"n": -1}
S: SUCCESS {"fields": ["ttl", "servers"]}
S: RECORD [4321, [{"addresses": ["127.0.0.1:9001"],"role": "WRITE"}, {"addresses": ["127.0.0.1:9002"], "role": "READ"}, {"addresses": ["127.0.0.1:9001", "127.0.0.1:9002"], "role": "ROUTE"}]]
S: SUCCESS {"bookmark": "neo4j:bookmark-test-1", "type": "r", "t_last": 5, "db": "system"}
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

  In other words, the leader/core/read-replica of each database in a cluster can be different from each other.

* There is a **default database** for a single instance and/or a cluster.

   By default, it is named as `"neo4j"`, but the name can be changed to something else such as `"foo"`.
   When changing the name, a restart of the single instance and/or cluster may or may not be required.
   The default database may or may not be allowed to be deleted. (We cannot assume that there is always a default database on each instance.)


* Any Core member in a cluster can provide a routing table for any database inside this cluster.

  Given a seed URL pointing to a Core member this can be used to find any databases in a cluster by fetching the routing table from a Core member.


#### Driver Routing Table

We should prevent the driver routing table from growing infinitely.

Till now we only remove routing table from the map when we failed to obtain a routing table.

However it is possible that the driver will hold some routing tables that is no longer valid anymore.

An invalid routing table could either be a routing table that is timed out, or a routing table that pointing to a database that does not exist anymore. 

For example, a user could create a database using a driver, and then do some work on the newly created database, but finally delete the database.

If this user repeat doing this, then the driver could hold infinite long routing table map.

If we trim the driver routing table map by removing all routing tables that pointing an non-existing database, then the driver will still hold routing tables less or equal to the amount of routing tables on the server side.

When should we remove a routing table from routing table map?

Should we allowed to remove a routing table from the map after TTL (default to 5 mins)? Currently we will remove the routing table after a timeout = TTL + 30s.  To be discussed.


Here is the workflow the driver should follow when fetching a routing table for database named "foo".

1. Find routing table in routing table map with key "foo".
   If the key does not exist, then create an empty routing table with seed URL as initial router.

2. If the routing table is stale, then refresh the routing table. 

3. If any error happens, remove the key "foo" from routing table map.

  The only errors possible are `SECURITY_ERROR`, `ROUTING_ERROR`, or `SERVICE_UNAVILABLE_ERROR` which only happens when the driver failed to get routing table with all existing routers,
  

## Client Side Logging


