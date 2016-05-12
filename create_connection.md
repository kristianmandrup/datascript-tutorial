## Connection to a database

A DB connection `conn` can be created using `d/create-conn`.

`(d/create-conn db)`

You can also add a schema in the same operation:

`(d/create-conn schema)`


`d/create-conn` takes an optional initial database (set of values) as a 2nd argument `(d/create-conn schema data)`.


The full example:

```clojure
(require '[datascript.core :as d])

;; Implicit join, multi-valued attribute

(let [schema {:aka {:db/cardinality :db.cardinality/many}}
      conn   (d/create-conn schema)]
  (d/transact! conn [ { :db/id -1
                        :name  "Maksim"
                        :age   45
                        :aka   ["Maks Otto von Stirlitz", "Jack Ryan"] } ])
```
