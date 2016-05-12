## Basic Queries

Queries are executed via `d/q`. It takes a quoted vector `'[...]` containing the query.

A query is built up of the following parts:
- `:find`
- `:in` (optional: to pass query parameters)
- `:where` (optional: to define constraints)

Let's make a query to find the `:name` and `:age` of any entity who has the alias (`:aka`) of `"Maks Otto von Stirlitz"`.

The result is: `["Maksim" 45]`

```clojure
  (d/q '[ :find  ?n ?a
          :where [?e :aka "Maks Otto von Stirlitz"]
                 [?e :name ?n]
                 [?e :age  ?a] ]
       @conn))

;; => #{ ["Maksim" 45] }
```

The symbol `_` can be used as a wildcard for the parts of the data pattern that you wish to ignore. You can also omit trailing values in a data pattern.Therefore, the previous query is equivalent to this next query, because we ignore the transaction part of the datoms.

```clojure
[:find ?e
 :where [?e :person/name "Ridley Scott" _]]
```

Omitting the trailing `_`, it becomes:

```clojure
[:find ?e
 :where [?e :person/name "Ridley Scott"]]
```


Learn more at:
- [Basic queries](http://www.learndatalogtoday.org/chapter/1)

### Query parameters

We can pass in query parameters as the 2nd argument. Here we pass in `red` as the `?color` (as per `:in ?color`). We then find `?e` (the entity id) and 
the `?rgb` value of any entity which has a `:color/name` attribute matching the `?color` passed `:in`.

```clojure
(d/q '[ :find ?e ?rgb
        :in ?color
        :where [?e :color/name ?color]
      ]
      'red'
;; => [[27 '#ff0000'] [42 '#ff0205']]
```

We can also pass a list of values as query parameters, using the special `[?x ...]` form. Here we find the maximum value of a list of integers passed in.
See more on aggregation functions below.

```clojure
(d/q '[ :find max(?x)
        :in [?x ...]
      ]
      [1 2 3 4] ;; query params

;; => [4]
```

You can learn more at:

- [Parameterized queries](http://www.learndatalogtoday.org/chapter/3)


### Aggregation

Aggregation is possible via functions just like in most other DBs.
We will be using the aggregation functions `max` and `min`.

In the `:find` we demand to return the `?color` (name) and the `max` and `min` values (denoted as `?x`) of that color.

The `:in` clause defines how to use the query parameters.
We tell the query to expect a list of tuples `[[?color ?x]]`
Then we pass in the list of tuples: `[:red 10] ... [:red 50]` and the query result is as expected.

```clojure
(d/q '[ :find ?color (min ?x) (max ?x)
        :in [[?color ?x]]
      ]
      [[:red 10]  [:red 20] [:red 30] [:blue 20] [:blue 50]] ;; query params

;; => [[:red  [10 30]] [:blue  [20 50]]]
```
