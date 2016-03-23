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

## Clojure

Add the following dependencies to your `project.clj` file:

```
:dependencies [
    ;; SERVER
    [org.clojure/clojure "1.8.0"]

    ;; Server DB (datomic)
    [com.datomic/datomic-free "0.9.5327" :exclusions [joda-time]]
    ;; ...
]
```

Then for your app file ,such as `my-app.core`

See [Datomic API]((http://docs.datomic.com/clojure/#datomic.api) for API details.

```clojure
(ns my-app.core
  (:require [datomic :as d]
            ;; more libs...
            ))

;;; Create a DataScript "connection" (an atom with the current DB value)
(def conn (d/create-conn))

;; Define datoms to transact
(def datoms [{:db/id -1 :name "Bob" :age 30}
             {:db/id -2 :name "Sally" :age 15}])

;;; Add the datoms via transaction
(d/transact! conn datoms)

;;; Query to find names for entities (people) whose age is less than 18
(def q-young '[:find ?n
               :where
               :in ?min-age
               [?e :natedme ?n]
               [?e :age ?a]
               [(< ?a ?min-age)]])

;; execute query: q-young, passing 18 as min-age
(d/q q-young 18)

;; Query Result
;; => [["Sally"]]
```

As you can see, for this simple example the API is identical to the one used for our Datascript getting started code example.
