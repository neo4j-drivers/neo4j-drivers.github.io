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

Each Neo4j core instance in a cluster supports `routing` and `reading`.

Only one Neo4j core in a cluster can be selected to support `writing` operations. This selection can rotate over time.

The client should support a `routing table`.


## Client Side Logging


