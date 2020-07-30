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


