# Datascript getting started

DataScript follows the Datomic api very closely, so the Datomic documentation is a good resource: http://docs.datomic.com/clojure/#datomic.api/touch

There is also a lot of good [getting started guide](https://github.com/tonsky/datascript/wiki/Getting-started)

@tonsky and the community will appreciate if you ask API/usage questions on the [#datascript gitter channel](https://gitter.im/tonsky/datascript). GitHub [issues](https://github.com/tonsky/datascript/issues) should be reserved for "real" issues (bugs/problems and such):

## Clojure quick start

```
```
(ns reagent-test.core
  (:require [datascript :as d]
            ;; more libs...
            ))


;;; Creates a DataScript "connection" (an atom with the current DB value)
(def conn (d/create-conn))

;;; Add some data
(d/transact! conn [{:db/id -1 :name "Bob" :age 30}
                   {:db/id -2 :name "Sally" :age 25}])

;;; Query to find peeps whose age is less than 18
(def q-young '[:find ?n
               :where
               [?e :name ?n]
               [?e :age ?a]
               [(< ?a 18)]])
```

## Javascript quick start

```js
import {datascript as d} from 'datascript';

```

Show examples
