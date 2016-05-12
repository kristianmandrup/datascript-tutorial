# Datascript architecture and internals

From [datascript-internals](http://tonsky.me/blog/datascript-internals)

### Atoms
Minimal piece of information in DataScript is Datom. It’s defined in datascript.core as

`(defrecord Datom [e a v tx added])`

where `[e a v]` is entity, attribute and value, tx is a transaction id (integer) and added is a boolean flag to differentiate additions from retractions.

Datom has (`-hash`) and (`-eqiv`) methods overloaded so only `[e a v]` take part in comparisons. It helps to guarantee that each datom can be added to the database only once.

## DB
Database is just an immutable, persistent collection of datoms. It’s very much like a built-in ClojureScript data structure, e.g. set. Main operations supported by DB are:

- Add a datom (similar to conj)
- Retract a datom (similar to disj)
- Search for range of datoms

Both addition and retraction are implemented in `datascript.core/with-datom`. Searching is implemented as a part of `ISearch` protocol.

DataScript DB contains only currently relevant datoms. There’s no history tracking at DB level. When datom is removed from a DB, there’s no trace of it anywhere. Retracted means gone.

Internally DB contains three different indexes to help with various search patterns. Each index is a sorted set of datoms (it’s literally a set). They’re named after sort order: `EAVT`, `AEVT` and `AVET`. Every index contains all datoms of current DB, meaning all three indexes are the same set sorted differently. There’s no other datoms storage inside the DB besides indexes.

DB is defined in `datascript.core` as

```
(defrecord DB [schema
               eavt aevt avet
               max-eid
               max-tx
               rschema])
```

Besides indexes, there’s only schema (just a regular map) and some internal bookkeeping: max seen entity id, latest transaction id and reverse value-to-key schema map.

### BTSet
Each DataScript index is stored as datascript.btset. It’s an immutable, persistent implementation of *B+ tree* data structure. It’s a proper ClojureScript collection, with all protocols of sorted-set implemented.

`BTSet` was needed because DataScript does a lot of range scans over indexes. Due to usage of continuous js arrays as tree nodes, range scans over BTSet are ~3 times faster than over built-in sorted-set which is a Red-Black binary tree.

`BTSet` uses generative testing to validate its correctness: same operations are performed at BTSet and sorted-set, results are compared for equality (see `stresstest-...` in `datascript.test.btset`).

### Adding data to DB
DataScript has a lot of conventions about how to format data before adding it to the DB. Basically, you can provide vector of form `[op e a v]` or a map of form `{:db/id e, (a v)+}`. There’s also a lot of nuances like resolving *temporary ids*, calling `:db.fn/*` shortcuts, referencing current transaction `id`, using nested maps for components, understanding different attributes arity and reversing reverse references.

Biggest part of datascript.core, starting from (`defrecord TxReport`), is all about solving these problems. Main work happens in transact-tx-data, which uses recursion to reduce complexity of transaction data step-by-step, until it’s reduced to a set of plain datoms:

`entity map → op vector[s] → id resolution → datom[s]`

`transact-tx-data` loop also builds `TxReport` record along the way:

```
(defrecord TxReport [db-before
                     db-after
                     tx-data
                     tempids
                     tx-meta])
```

`TxReport` is a log of everything that has happened during transaction. It stores temporary entity ids resolution table (tempids) and raw datoms which were used to modify DB (`tx-data`). Given `TxReport` and `db-before`, it’s trivial to replay transaction and calculate `db-after`:

`db-before + tx-data = db-after`

`tx-data` is an end result of transaction data simplification, with all temporary and implicit ids allocated to actual entity ids, all maps expanded into vector forms, and all vectors converted to Datoms.

`TxReport` is also the only place where you can see datoms with added == false for datoms which were retracted.

### Entities
Entity is just a convenient interface to `EAVT` index. Basically this:

`(:person/name (d/entity db 1))`

is translated to this:

`(first (d/datoms db :eavt 1 :person/name))`

with some per-entity caching. Most of the namespace `datascript.impl.entity` is spent on implementing protocols to make entities look like normal ClojureScript maps.

### Queries
Query engine implements Datalog over DBs and collections. It is the most complicated part of DataScript so far.

Query engine operates on relations. Relation is just a collection of tuples. E.g. this:

`:where [?e :name ?v]`

Query engine is implemented in `datascript.query` and uses parser from `datascript.parser`.

Pull API
Pull API is implemented in datascript.pull-api by David Thomas Hume. It walks all entity ids, for each entity it walks all attributes from a pull pattern, and for each recursive attribute calls itself. It’s a recursive algorithm implemented as a state machine that emulates stack so it can handle unlimited recursion depth.

There’s also `datascript.pull-parser` where pull pattern is parsed and validated.

During implementation of DataScript pull pattern parser and query parser, we’ve helped Datomic team to fix couple of inaccuracies in their query grammar. This is why having alternative implementations is usually a good thing.

### Filtered DBs
Given database D and predicate `P: datom, DB → boolean`, you can construct special view `F = (datascript/filter D P)` that will work like a regular database but for all needs (query, entity, pull, index access) will look like as if it only contains datoms that satisfy `P`.

Filtered databases are implemented in datascript.core by just extending couple of protocols: `IDB, ISearch, IIndexAccess` — mostly by adding additional filter post-process step.

For advanced uses it means that it’s quite easy to pretend to be a database: just implement these protocols and rest of DataScript code will work as expected.

### Database mutation
Everything we’ve discussed so far works on top of immutable, persistent DB objects. There’s also a small piece of code to handle database mutations (as of `0.9.0`, it’s literally 25 lines).

`datascript/create-conn` returns a new database object wrapped with an atom. There’s also a listen! machinery for listening for DB changes. It was added because regular atom watchers receive value-before and value-after, but database changes have more useful information: list of datoms added and temporary ids resolution table.

DataScript “connection” is not a connection in any sense. I borrowed the term from Datomic, but now I’m thinking about changing it to `db-ref` or better, if only I could think of anything.

### JavaScript interop
Almost from the beginning DataScript has supported JavaScript interface. You include pre-compiled JS file *(270k raw/~60k gzipped as of 0.9.0)* and call `datascript.empty_db()`, `datascript.query()` and stuff.

This was made possible by datascript.js facade: it converts data coming from JS into ClojureScript data structures, calls proper datascript.* APIs, converts results into JS form and returns them back to JS caller. Not the most effective way to do things, but the only one manageable without writing js-native DataScript’s evil twin. Even with such approach, some things have leaked into the main DataScript code: it has to take into account that attributes can be strings, for example. But for the most part JS interop is isolated.

And JS interop is feature-full: everything you can do from CLJS, you can do from JS. Maybe not as well battle-tested, but nonetheless. See `test/js/tests.html` for how DataScript can be used from JS.

### Tests
DataScript does acceptance testing almost exclusively. Everything in `datascript.test.*` works with top-level interfaces of datascript and `datascript.core` namespaces. Exceptions are `datascript.test.btset` (generative tests of BTSet implementation) and `datascript.test.perf` (somewhat ad-hoc and messy performance tests of various parts of DataScript).

Having just acceptance tests works because semantics and APIs are already defined and documented by Datomic team whom DataScript is following. APIs do not change often, so extensive tests suite (2.8k loc `test/`` for 3k loc `src/`) is not a dead weight and requires little maintenance.

Tests allowed me to change underlying implementation without rewriting a single test: I completely changed index implementation and inner DB structure once already and replaced query engine twice so far. It gives tremendous confidence to pull such refactorings without breaking anyone else’s code.