# Possible DB

PossibleDB is a Database Server built with DataScript, RethinkDB, Clojure, ClojureScript, and NodeJS.

- [Possible DB video](https://vimeo.com/107237345)

[PossibleDB](https://clojars.org/possibledb-client) is the bridge between DataScript and RethinkDB. 

Transactions are persisted in DataScript (memory) and RethinkDB (disk). Queries are resolved against DataScript. DataScript is synced with RethinkDB via changeset mechanism (I think? or should be...).

## Leinigen

`[possibledb-client "1.6"]`

## Client for Server

```clj
(:require [possibledb-client.core :as db])

(db/connect!
 "Connect to a PossibleDB server"
 [host port])

(db/get
 "Returns an entire database."
 [^:String db-name])

(db/q
 "Same q call as in DataScript"
 [^:String db-name
  query-coll])

(db/transact!
 "Same transact! call as in DataScript"
 [^:String db-name
  data-coll])

(db/create-db!
 "Create a PossibleDB db. If schema, same as DataScript."
 ([^:String db-name])
 ([^:String db-name
   ^:HashMap schema]))

(db/destroy-db!
 "Remove a DB and all of it's data"
 [^:String db-name])

(db/backup-db!
 "Writes an EDN representation of a DB to a file"
 [^:String db-name
  ^:String save-file-path])

(db/spawn-db!
 "Create a new database with all the data from original-db-name"
 [^:String original-db-name
  ^:String new-db-name])

(db/reset-db!
 "Destroy a DB and create it again"
 ([^:String db-name])
 ([^:String db-name
   ^:HashMap schema]))```

