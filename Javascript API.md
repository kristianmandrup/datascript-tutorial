# Datascript API for JavaScript

DataScript has a dedicated public JavaScript API.

Some examples:
- [tests](https://github.com/tonsky/datascript/blob/master/test/js/tests.js#L101)

[datascript-mori](https://github.com/typeetfunc/datascript-mori) contains some usage examples:

- [basic usage](https://github.com/typeetfunc/datascript-mori/blob/master/release-js/test/onlyCljsApiUsage.spec.js)
- [mori integration](https://github.com/typeetfunc/datascript-mori/blob/master/release-js/test/combineJsAndCljsApi.spec.js)

### [js public API](https://github.com/tonsky/datascript/blob/master/src/datascript/js.cljs#L61)

### Database

[db](http://docs.datomic.com/clojure/#datomic.api/db)

`db(conn)` Retrieves a value of the database for reading. 

`init_db(datoms..., schema)` Initialize new DB with initial data and schema

`empty_db(schema)` Creates emty db

`entity_db(..)` ???


[with](http://docs.datomic.com/clojure/#datomic.api/with)

`db_with(db, entities)`

### Queries

[q](http://docs.datomic.com/clojure/#datomic.api/q)

`q(query, sources)` query sources of data

*Pull one*
Pull attributes matching a pattern for an entity by ID.

[pull](http://docs.datomic.com/clojure/#datomic.api/pull)

`pull(db, pattern, eid)`

Pull all attributes for the entity identified by `max-id`

`pull(db, ['*'], max-id)`

*Pull many*
Pull attributes matching a pattern for entities matching a list of IDs.

[pull-many](http://docs.datomic.com/clojure/#datomic.api/pull-many)

`pull_many(db, pattern, eids)`

Get all attributes of entities: `max-id` and `alice-id`

`pull_many(db, ['*'], [max-id, alice-id])`

[entity](http://docs.datomic.com/clojure/#datomic.api/entity)

Returns a dynamic map of the entity's attributes for the given id, ident or lookup ref.

`entity(db, eid)`

Examples:

`entity(db, [:person/email 'hans.gruber@gmail.com'])`

### Filter

[filter](http://docs.datomic.com/clojure/#datomic.api/filter)

`filter(db, pred)`

Returns the value of the database containing only datoms
satisfying the predicate.

Examples:
TODO

### Filtered DBs (views)

A Database can be filtered as a view. Transactions can NOT be performed on a filtered DB as it is a view and read only.

Check if database is filtered, ie. is an instance of `FilteredDB`

[is-filtered](http://docs.datomic.com/clojure/#datomic.api/is-filtered)

`is_filtered(db)` Returns true if db has had a filter set with filter

### Miscelaneous

*Resolve temp id*

[resolve-tempid](http://docs.datomic.com/clojure/#datomic.api/resolve-tempid)

`resolve_tempid(tempids, tempid)`

Resolve a tempid to the actual id assigned in a database.

*Touch*

`touch(entity-id)`

Marks an entity as "touched", see [Datomic touch](http://docs.datomic.com/clojure/#datomic.api/touch)

Usually DataScript will just return a "reference" to an entity (e.g. #{1}). In order to get any attribute-values out, you will have to dereference the entity (by asking specifically for some attribute, e.g. (:name eid)). But sometimes you're just interested in all the values, so touch just recursively dereferences all attributes for you (e.g. `(touch eid) => {:name "...", :age "...", ...})`

*Set index range*

[index-range](http://docs.datomic.com/clojure/#datomic.api/index-range)

`index_range(db, attr, start, end)`

Set the range for an index over an attribute (See Datascript architecture).

Example: `index_range(db, :person/name, 0, 100)`

### Connection

Create a connection (and DB) for a schema

`create_conn(schema)`

Create a connection from a DB

`conn_from_db(db)`

Create a connection (and DB) for a list of datoms (initial data).

`conn_from_datoms(datoms, schema)`

Reset a DB connection

`reset_conn(conn, db, tx-meta)`

### Transaction

Create a transaction on a DB connection, passing a list of entities to be added or retracted.

[transact](http://docs.datomic.com/clojure/#datomic.api/transact)

`transact(conn, entities..., tx-meta)`

Examples:
TODO

### Listeners

Datascript supports adding listeners for each transaction performed.

[add-listener](http://docs.datomic.com/clojure/#datomic.api/add-listener)

Attach a listener to connection

`listen(conn, callback)`

Detach a listener from connection

`unlisten(conn, key)`

### Datoms

See Datomic API?

[datoms](http://docs.datomic.com/clojure/#datomic.api/datoms)

`datoms(db, index, components)`

Raw access to the index data, by index. The index must be supplied,
and, optionally, one or more leading components of the index can be
narrow the result.

[seek-datoms](http://docs.datomic.com/clojure/#datomic.api/seek-datoms)

`seek_datoms(db, index, components)`

Raw access to the index data, by index. The index must be supplied,
and, optionally, one or more leading components of the index can be supplied for the initial search. For advanced use only.

### Unique IDs

Datascript supports Squiidsm (globally unique identifiers) as used in [Datomic](http://docs.datomic.com/identity.html)

[squuid](http://docs.datomic.com/clojure/#datomic.api/squuid)

`squuid()` Generate a globally unique identifier (based in part on time in millis)

[squuid-time-millis](http://docs.datomic.com/clojure/#datomic.api/squuid-time-millis)

`squuid_time_millis(uuid)` get the time part of a squuid.

