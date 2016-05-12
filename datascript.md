# Datascript


Partly extracted from [Tips-&-tricks](https://github.com/tonsky/datascript/wiki/Tips-&-tricks)

### Enums

Representing enums

As of 0.13.0, DataScript does not support idents (this may change in the future). Instead, if you need to represent enums, use keywords as values. E.g. instead of

```clojure
;; setting attribute type to reference
@(datomic.api/transact conn [[:db/add <attr> :db/valueType :db.type/ref]])

;; registering enum value
@(datomic.api/transact conn [[:db/add #db/id[:db.part/db] :db/ident :some-enum-value]])

;; transacting data
@(datomic.api/transact conn [[:db/add <eid> <attr> :some-enum-value]])

;; just do
(d/transact! conn [[:db/add <eid> <attr> :some-enum-value]])
```

### EDN serialization

DataScript DB support EDN serialization natively:

```
(pr-str (d/empty-db)) => "#datascript/DB {:schema {} :datoms[]}"
Do read database from a reader,
```

In Clojure use datascript.core/data-readers:

```
(clojure.edn/read-string
  {:readers d/data-readers}
  "#datascript/DB {:schema {} :datoms[]}")
```

In ClojureScript #datascript/DB handler is installed globally when using DataScript, so read-string just works:

```
(cljs.reader/read-string "#datascript/DB {:schema {} :datoms[]}")
```

### Transit serialization

[datascript-transit](https://github.com/tonsky/datascript-transit)

Add this to your `project.clj`:

```
[datascript "0.13.0"]
[datascript-transit "0.2.0"]
```

Usage in your app:

```
(require '[datascript.transit :as dt])

(-> (datascript.core/empty-db)
    (dt/write-transit-str) ;; => string
    (dt/read-transit-str)) ;; => DataScript DB
```

If you use your own serialization/deserialization, add these handlers to the mix:

```
datascript.transit/read-handlers
datascript.transit/write-handlers
```

## Filtering database entities

`datascript.core/filter` filters individual datoms, but what if you want to keep whole entities? For example, you want to filter a DB leaving out only persons whose name is `"Ivan"`:

```clj
(d/filter db
  (fn [db datom]
    (or
      ;; leaving all datoms that are not about :person/* as-is
      (not= "person" (namespace (:a datom)))
      ;; for :person/* attributes take entity id
      ;; and check :person/name on that entity using db value
      (let [eid    (:e datom)
            entity (d/entity db eid)]
        (= "Ivan" (:person/name entity))))))
```

## Querying on a missing attribute

Until DataScript supports `not` clause, use a combination of `get-else` and `nil?` predicate to find out entities which _do not have_ some attribute:

```clj
(d/q '[:find ?e
       :where [?e :document/id _]
              [(get-else $ ?e :document/users nil) ?u]
              [(nil? ?u)]]
     db)
```


### Getting top 10 entities by some attribute

If you have a big database and want to render 10 first posts on a page, instead of doing a query (which will have to materialize everything you have in a result set), use index directly:

`(take 10 (d/datoms :avet :post/timestamp))`

This will return 10 datoms with the smallest :post/timestamp value. You can then (mapv :e ...) to get their entity ids and pass them down to a `query/pull-many` call.

Index is already sorted and stored in ascending order. `(take 10 ...)` will return 10 smallest values very fast.

To get 10 biggest values, reverse the index:

`(take 10 (reverse (d/datoms :avet :post/timestamp)))`

## Preserving order

DataScript makes it easy to store unordered data. If you need to keep order, you have several options.

By convention `:db.cardinality/many` attributes are unordered. Sorting datoms by transaction id will give you insertion order:

```clj
;; schema
{ :aliases { :db/cardinality :db.cardinality/many } }

;; query
(->>
  (d/q '[:find ?v ?tx
         :in $ ?e
         :where [?e :aliases ?v ?tx]]
       db eid)
  (sort-by second)
  (map first))

;; index lookup
(->> (d/datoms :aevt eid :aliases)
     (sort-by :tx)
     (map :v))
```

You can store vector of values as a `:db.cardinality/one` value. Because it will be just a single value, you can keep inside any structure you want, including order:

```clj
;; schema
{ :aliases { :db/cardinality :db.cardinality/one } }

;; transaction
(d/transact! conn [[:db/add eid :aliases ["X" "Y" "Z"]]])

;; entity lookup
(:aliases (d/entity @conn eid)) ;; => ["X" "Y" "Z"]
```

You can introduce attribute you want to sort by, and store references in an `:db.cardinality/many` attribute:

```clj
;; schema
{ :aliases { :db/cardinality :db.cardinality/many
             :db/valueType   :db.type/ref } }

;; transaction
(d/transact! conn [{:db/id eid
                    :aliases [-1 -2 -3]}
                   {:db/id -1, :name "X", :order 3}
                   {:db/id -2, :name "Y", :order 7}
                   {:db/id -3, :name "Z", :order 1})

;; query
(->>
  (d/q '[:find [(pull ?a [:name :order]) ...]
         :in   $ ?e
         :where [?e :aliases ?a]
       db eid)
  (sort-by :order)
  (map :name))
```

## Querying a schema

Unlike Datomic, in DataScript schema is just a map kept separate from database datoms. It means you cannot query it as you would in Datomic, but you can query it just as any other collection. Use `[[?attr [[?aprop ?avalue] ...]] ...]` destructuring form to convert nested maps into a flat collection of facts.

For example, following query will return all datoms from a DB which attribute has cardinality many:

```clj
(def schema
  { :entry/id           {:db/unique      :db.unique/identity}
    :entry/child        {:db/cardinality :db.cardinality/many
                         :db/valueType   :db.type/ref}
    :entry/first-child  {:db/valueType   :db.type/ref} })

(def db (-> (d/empty-db schema)
            (d/db-with [[:db/add 1 :entry/id "a"]
                        [:db/add 1 :entry/child 2]
                        [:db/add 1 :entry/child 3]
                        [:db/add 1 :entry/first-child 2]])))

(d/q '[:find ?entity ?attr ?value
       :in $ [[?attr [[?aprop ?avalue] ...]] ...]
       :where [(= ?avalue :db.cardinality/many)]
              [?entity ?attr ?value]]
     db (:schema db))

=> #{[1 :entry/child 3] [1 :entry/child 2]}
```