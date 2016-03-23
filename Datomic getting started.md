# Datomic getting started

[datomic API](http://docs.datomic.com/clojure/#datomic.api)

## Javascript driver

You can also access Datomic from Javascript, such as from a Node.js app (which could be written in ClojureScript)

[datomicjs]()

## Usage

See `datomic.js`

`Datomic(server, port, alias, name)`

Create Schema:

```
schema.movies = '[
  {:db/id #db/id[:db.part/db]
   :db/ident :movie/title
   :db/valueType :db.type/string
   :db/cardinality :db.cardinality/one
   :db/doc "movie's title"
   :db.install/_attribute :db.part/db}'
   '
```

Then use...

```
Datomic = require('datomicjs').Datomic;
imdb = new Datomic('localhost', 8888, 'db', 'imdb');

// use the DB
imdb.transact(...);

datomic.transact(schema.movies).then((future) ->
  console.log(future);


// to build a Query
find = require('datomicjs').Query;
find('?m')
  .where('?m', ':movie/title', 'trainspotting')
  .toString()
...
```

### API

```
imdb.transact(data)
imdb.entity(id, opts)
imdb.q(query, opts)
```

### EDN

[parse EDN](https://github.com/jkroso/parse-edn) and [serialize EDN](https://github.com/jkroso/serialize-edn) are used to work with the Clojure Map structures of Datomic. The interop with Clojure is done by taking a string with the Map syntax and serializing it to EDN (for queries etc) and conversely parsing a received EDN structure from Datomic (ie. data).
