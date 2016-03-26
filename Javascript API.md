# Datascript API for JavaScript

DataScript has a dedicated [public JavaScript API]([js public API](https://github.com/tonsky/datascript/blob/master/src/datascript/js.cljs#L61))

Some [test usage examples](https://github.com/tonsky/datascript/blob/master/test/js/tests.js#L101)

For proper use of Datascript in a javascript context, you really should be using it with immutable values if possible. One such option is to use [datascript-mori](https://github.com/typeetfunc/datascript-mori) introduced in the [[Morin Integration]] chapter.

## Javascript public API
We present the API in order of common use.
The API can be divided roughly into the following groups.

- Database
    - Filter DB
    - Connections
- Transactions (upsert)
- Query
    - Query data
    - Pull data
    - Fetch entity data
- Listeners
- Indexes
- IDs
    - Temp IDs
    - Unique IDs

### Database

[db](http://docs.datomic.com/clojure/#datomic.api/db) Retrieves a value of the database for reading.

`db(conn)`

Creates a new DB with initial data (datoms) and a schema.

`init_db(datoms..., schema)`

[entity-db](http://docs.datomic.com/clojure/#datomic.api/entity-db) Creates a completely empty DB.

`empty_db(schema)`

Returns the database value that is the basis for this entity

`entity_db(..)`

#### Filtered DBs (views)

A Database can be filtered as a view. Transactions can NOT be performed on a filtered DB as it is a view and read only.

*Filter*

[filter](http://docs.datomic.com/clojure/#datomic.api/filter) Returns the value of the database containing only datoms
satisfying the predicate.

`filter(db, pred)`

*Check if filtered*

[is-filtered](http://docs.datomic.com/clojure/#datomic.api/is-filtered) Check if database is filtered, ie. is an instance of `FilteredDB`

`is_filtered(db)`

#### DB Connection

[connect](http://docs.datomic.com/clojure/#datomic.api/connect) Create a connection (and DB) for a schema

`create_conn(schema)`

*Create a connection from a DB*

`conn_from_db(db)`

Get a connection for a new DB created from a list of datoms (initial data).

`conn_from_datoms(datoms, schema)`

*Reset a DB connection*

`reset_conn(conn, db, tx-meta)`

### Transactions

Transactions are used to "upsert" data, ie. insert/update data.

[transact](http://docs.datomic.com/clojure/#datomic.api/transact) Creates a transaction on a DB connection, passing a list of entities to be added or retracted.

`transact(conn, entities..., tx-meta)`

Execute a transaction to create a new entity.

```clojure
  (d/transact! conn [ { :db/id -1
                        :name  "Maksim"
                        :age   45
                        :aka   ["Maks Otto von Stirlitz", "Jack Ryan"] } ])
```

[with](http://docs.datomic.com/clojure/#datomic.api/with) Applies transaction data to the database. It is as if the data was
applied in a transaction, but the source of the database is
*unaffected*. Useful to simulate an update to the data without affecting the real underlying source.

`db_with(db, entities)`

### Queries

#### Query data

[q](http://docs.datomic.com/clojure/#datomic.api/q) query sources of data

`q(query, sources)`

Queries in Clojure are written usin EDN, which doesn't directly translate to JS. So in order to use queries from JS, we need to serialize a JS string to EDN. We can use a library such as [jsedn]()

### JS/EDN converter

In your `package.json` file include a dependency to jsedn:

`"jsedn" : "kristianmandrup/jsedn"`

Then use it `edn`  as follows:

```js
var edn = require('jsedn');

var query = edn.parse(`
    :find max(?id)
    :where [?e, :person/id ?id]`);

d.q([query], conn, 18)
```

### Babel plugin

[babel-plugin-datascript](https://github.com/typeetfunc/babel-plugin-datascript/) let's you write your queries like:

```js
var result = Datalog.Q`[:find  ?e ?email
                          :in    $ [[?n ?email]]
                          :where [?e :name ?n]]`;
```

and it will be compiled to code, which is wrapped in a Mori immutable data model:

```js
import { mori as _mori } from 'datascript-mori';
var Datalog = _mori.vector(_mori.keyword('find'), _mori.symbol('?e'), _mori.symbol('?email'), _mori.keyword('in'), _mori.symbol('$'), _mori.vector(_mori.vector(_mori.symbol('?n'), _mori.symbol('?email'))), _mori.keyword('where'), _mori.vector(_mori.symbol('?e'), _mori.keyword('name'), _mori.symbol('?n')));
```

Which you can then use with [datascript-mori](https://github.com/typeetfunc/datascript-mori)

### Query DSL for JS

We aim to enable building queries using a DSL more suitable for JS.

```js
var query = {find: 'max(?id)', where: ['?e', ':person/id' '?id']};
d.q(query, conn, 18)
```

Or using chain builder syntax:

```js
var query = find('max(?id)').where('?e', ':person/id' '?id').and('?e', ':person/id' '?id');
```

Or even better:

```js
var query = q()
    .find('max(?id) ?email')
    .where(entity: '?e')
    .has(':person/id': '?id')
    .and(':person/email': '?email')
    .build();
```

Please help out in this effort :)

#### Pull data

*Pull one*

Pull attributes matching a pattern for an entity by ID.

[pull](http://docs.datomic.com/clojure/#datomic.api/pull) Pull all attributes for the entity identified by `max-id`

`pull(db, pattern, eid)`

Example: `pull(db, ['*'], max-id)`

*Pull many*

[pull-many](http://docs.datomic.com/clojure/#datomic.api/pull-many) Pull attributes matching a pattern for entities matching a list of IDs.

`pull_many(db, pattern, eids)`

Get all attributes of entities: `max-id` and `alice-id`

Example: `pull_many(db, ['*'], [max-id, alice-id])`

#### Fetch entity data

[entity](http://docs.datomic.com/clojure/#datomic.api/entity) Returns a dynamic map of the entity's attributes for the given id, ident or lookup ref.

`entity_db(db, eid)`

Example: `entity_db(db, [:person/email 'hans.gruber@gmail.com'])`

*Touch*

[touch](http://docs.datomic.com/clojure/#datomic.api/touch) Usually DataScript will just return a "reference" to an entity. In order to get any attribute-values out, you will have to dereference the entity (by asking specifically for some attribute, e.g. (`:name eid`)). But sometimes you're just interested in all the values, so touch just recursively dereferences all attributes for you (e.g. `(touch eid) => {:name "...", :age "...", ...})`

`touch(entity-id)`

### Listeners

Datascript supports listening for transactions performed.

[add-listener](http://docs.datomic.com/clojure/#datomic.api/add-listener) Attach a listener to a DB connection.

`listen(conn, callback)`

Detach a listener from connection

`unlisten(conn, key)`

### Indexes

*Set index range*

[index-range](http://docs.datomic.com/clojure/#datomic.api/index-range) Set the range for an index over an attribute (See *Datascript architecture*).

`index_range(db, attr, start, end)`

Example: `index_range(db, :person/name, 0, 100)`

#### Find index datoms

Raw access to the index data, by index:

[datoms](http://docs.datomic.com/clojure/#datomic.api/datoms) . The index must be supplied, and optionally one or more leading components of the index can be
supplied to narrow the result.

`datoms(db, index, components)`

[seek-datoms](http://docs.datomic.com/clojure/#datomic.api/seek-datoms) Raw access to the index data, by index. The index must be supplied,
and, optionally, one or more leading components of the index can be supplied for the initial search. For advanced use only.

`seek_datoms(db, index, components)`

### IDs

Utility functions for creating/resolving IDs.

#### Temp Ids

*Resolve temp id*

[resolve-tempid](http://docs.datomic.com/clojure/#datomic.api/resolve-tempid) Resolve a tempid to the actual id assigned in a database.

`resolve_tempid(tempids, tempid)`

#### Unique IDs

Datascript supports Squiidsm (globally unique identifiers) as used in [Datomic](http://docs.datomic.com/identity.html)

[squuid](http://docs.datomic.com/clojure/#datomic.api/squuid) Generate a globally unique identifier (based in part on time in millis)

`squuid()`

[squuid-time-millis](http://docs.datomic.com/clojure/#datomic.api/squuid-time-millis) get the time part of a squuid.

`squuid_time_millis(uuid)`
