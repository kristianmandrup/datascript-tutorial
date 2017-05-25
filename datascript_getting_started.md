# Datascript getting started

DataScript follows the Datomic api very closely, so the [Datomic documentation](http://docs.datomic.com/clojure/#datomic.api) is a good resource

There is also a good [getting started guide](https://github.com/tonsky/datascript/wiki/Getting-started)

[@tonsky](http://www.twitter.com/tonsky) and the community will appreciate if you ask API/usage questions on the [#datascript gitter channel](https://gitter.im/tonsky/datascript). GitHub [issues](https://github.com/tonsky/datascript/issues) should be reserved for "real" issues (bugs/problems and such):

## Clojure quick start

A full `project.clj` file for a datascript only app would look like this.

```clj
(defproject my-app "0.1.0"
  :dependencies [
    [org.clojure/clojure "1.8.0"]
    [org.clojure/clojurescript "1.8.34"]
    [datascript "0.15.0"]
    [datascript-transit "0.2.0"]
    ;; ... more libs
  ]

  :plugins [
    [lein-cljsbuild "1.1.0"]
  ]

  :cljsbuild {
    :builds [
      { :id "advanced"
        :source-paths  ["src"]
        :compiler {
          :main          my-app.core
          :output-to     "target/my-app.js"
          :optimizations :advanced
          :pretty-print  false
        }}
  ]}

  :profiles {
    :dev {
      :cljsbuild {
        :builds [
          { :id "none"
            :source-paths  ["src"]
            :compiler {
              :main          my-app.core
              :output-to     "target/my-app.js"
              :output-dir    "target/none"
              :optimizations :none
              :source-map    true
            }}
      ]}
    }
  }
)
```

- Create a new DB connection (schema less)
- Transact datoms into DB
- Execute a query datoms in DB to fetch a result

```clojure
(ns my-app.core
  (:require [datascript.core :as d]
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
               :in $ ?min-age
               :where
               [?e :name ?n]
               [?e :age ?a]
               [(< ?a ?min-age)]])

;; execute query: q-young, passing 18 as min-age
(d/q q-young @conn 18)

;; Query Result
;; => [["Sally"]]
```

## Javascript quick start

- Create a new DB connection with a DB schema
- Setup listener
- Transact datoms into DB
- Execute a query datoms in DB to fetch a result

```js
import {datascript as d} from 'datascript';

// create DB schema, a regular JS Object
var schema = {
  "aka": {":db/cardinality": ":db.cardinality/many"},
  "friend": {":db/valueType": ":db.type/ref"}
};

// Use JS API to create connection and add data to DB
// create connection using schema
var conn = d.create_conn(schema);

// setup listener called main
// pushes each entity (report) to an Array of reports
// This is just a simple example. Make your own!
var reports = []
d.listen(conn, "main", report => {
    reports.push(report)
})

// define initial datoms to be used in transaction
var datoms = [{
      ":db/id": -1,
      "name": "Ivan",
      "age": 18,
      "aka": ["X", "Y"]
    },
    {
      ":db/id": -2,
      "name": "Igor",
      "aka": ["Grigory", "Egor"]
    },
    // use :db/add to link datom -2 as friend of datom -1
    [":db/add", -1, "friend", -2]
];

// Tx is Js Array of Object or Array
// pass datoms as transaction data
d.transact(conn, datoms, "initial info about Igor and Ivan")

// Fetch names of people who are friends with someone 18 years old
// query values from conn with JS API
var result = d.q('[:find ?n :in $ ?a :where [?e "friend" ?f] [?e "age" ?a] [?f "name" ?n]]'), d.db(conn), 18);

// print query result to console!
console.log(result); // [["Igor"]]
```

Please refer to the [[Javascript API]] for an overview of JS functions you can use to build your app. You can also have a loo

