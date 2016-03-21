# Datascript API for JavaScript

DataScript has a dedicated public JavaScript API.

Some examples:
- [tests](https://github.com/tonsky/datascript/blob/master/test/js/tests.js#L101)

[datascript-mori](https://github.com/typeetfunc/datascript-mori) contains some usage examples:

- [basic usage](https://github.com/typeetfunc/datascript-mori/blob/master/release-js/test/onlyCljsApiUsage.spec.js)
- [mori integration](https://github.com/typeetfunc/datascript-mori/blob/master/release-js/test/combineJsAndCljsApi.spec.js)

### [js public API](https://github.com/tonsky/datascript/blob/master/src/datascript/js.cljs#L61)

### Database

```js
db(conn)
init_db(datoms..., schema)
empty_db(schema)
db_with(db, entities)
entity_db(..)
```

### Queries

`q(query, sources)` query sources of data

*Pull one*
Pull attributes matching a pattern for an entity by ID.

`pull(db, pattern, eid)`

Pull all attributes for the entity identified by `max-id`

`pull(db, ['*'], max-id)`

*Pull many*
Pull attributes matching a pattern for entities matching a list of IDs.

`pull_many(db, pattern, eids)`

Get all attributes of entities: `max-id` and `alice-id`

`pull_many(db, ['*'], [max-id, alice-id])`

Find and return an entity by ID

`entity(db, eid)`

Examples:

`entity(db, [:person/email 'hans.gruber@gmail.com'])`

### Filter

Return a database filtered by a predicate?

`filter(db, pred)`

Examples:
TODO

### Filtered DBs (views)

A Database can be filtered as a view. Transactions can NOT be performed on a filtered DB as it is a view and read only.

Check if database is filtered, ie. is an instance of `FilteredDB`

`is_filtered(db)`


### Miscelaneous

`resolve_tempid(tempids, tempid)`

Marks an entity as "touched" [what?](https://github.com/tonsky/datascript/issues/148)

`touch(entity-id)`

Set the range for an index over an attribute (See Datascript architecture).

`index_range(db, attr, start, end)`

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

`transact(conn, entities..., tx-meta)`

Examples:
TODO

### Listeners

Datascript supports adding listeners for each transaction performed.

```js
listen(conn, callback)
unlisten(conn, key)
```

### Datoms

```
datoms(db, index, components)
seek_datoms(db, index, components)
```

TODO: investigate and document!

### Unique IDs

Datascript supports Squiidsm (globally unique identifiers) as used in [Datomic](http://docs.datomic.com/identity.html)

```js
squuid()
squuid_time_millis(uuid)
```

