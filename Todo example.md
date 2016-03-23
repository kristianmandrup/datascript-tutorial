# Todo example

```clj
(ns datascript-todo.core
  (:require
    [clojure.set :as set]
    [clojure.string :as str]
    [datascript.core :as d]
    [rum.core :as rum]
    [datascript.transit :as dt]
```

## Define Todo schema

We start by defining the schema as a regular clojure Map then define a function using `defonce` which can create a DB connection from the schema.
`defonce` helps to ensure we will only ever have one connection in the app.

```clj
;; Define todo schema
;; - a todo can have many tags
;; - todo can reference (belong to) a project
;; - done and due are indexed
(def schema {:todo/tags    {:db/cardinality :db.cardinality/many}
             :todo/project {:db/valueType :db.type/ref}
             :todo/done    {:db/index true}
             :todo/due     {:db/index true}})

;; create connection to DB with schema
(defonce conn (d/create-conn schema))
```

## History

We can now add some basic infrastruture for history

```clj
;; History
;; a list of atoms (ie. app states)

(defonce history (atom []))
(def ^:const history-limit 10)
```

We also need various grouping filters

```clj
;; Grouping filters

(defmethod todos-by-group :completed [db _ _]
  (d/q '[:find [?todo ...]
         :where [?todo :todo/done true]]
    db))

(defmethod todos-by-group :all [db _ _]
  (d/q '[:find  [?todo ...]
         :where [?todo :todo/text]]
    db))

(defmethod todos-by-group :project [db _ pid]
  (d/q '[:find [?todo ...]
         :in   $ ?pid
         :where [?todo :todo/project ?pid]]
       db pid))

;; Since todos do not store month directly, we pass in
;; month boundaries and then filter todos with <= predicate
(defmethod todos-by-group :month [db _ [year month]]
  (d/q '[:find [?todo ...]
         :in   $ ?from ?to
         :where [?todo :todo/due ?due]
                [(<= ?from ?due ?to)]]
       db (u/month-start month year) (u/month-end month year)))
```

## Rum

Rum is an alternative React renderer, similar to Om, Reagent and Omniscient, but designed to be more flexible.

Rum let's us easily design React components with actions, much like redux.

The Add view let's us add a new Task (todo item). We can setProjects, tags and due date for the task. The `on-submit` of the form will call `add-todo` with the task.

```clj
;; Add view (ie. 'add' actions)
(rum/defc add-view []
  [:form.add-view {:on-submit (fn [_] (add-todo) false)}
    [:input.add-text    {:type "text" :placeholder "New task"}]
    [:input.add-project {:type "text" :placeholder "Project"}]
    [:input.add-tags    {:type "text" :placeholder "Tags"}]
    [:input.add-due     {:type "text" :placeholder "Due date"}]
    [:input.add-submit  {:type "submit" :value "Add task"}]])
```

The History view let's us go forward and backwards in time, by calling `u/find-prev` and `u/find-next` respectively on `@history` to retrieve a given atom state and then passing that state to `reset-conn!` such as `(reset-conn! prev)` to set our current app state to a specific time in history.

```
;; History view (go forward/backwards in time)
(rum/defc history-view [db]
  [:.history-view
    (for [state @history]
      [:.history-state
       { :class (when (identical? state db) "history-selected")
         :on-click (fn [_] (reset-conn! state)) }])
    (if-let [prev (u/find-prev @history #(identical? db %))]
      [:button.history-btn {:on-click (fn [_] (reset-conn! prev))} "‹ undo"]
      [:button.history-btn {:disabled true} "‹ undo"])
    (if-let [next (u/find-next @history #(identical? db %))]
      [:button.history-btn {:on-click (fn [_] (reset-conn! next))} "redo ›"]
      [:button.history-btn {:disabled true} "redo ›"])])
```

We call the main component `canvas`. 
It renders the `main`, `history` and `add task` views. The main view contains a `filter` pane and the panes `overview-pane` and `todo-pane`.

```clj
(rum/defc canvas [db]
  [:.canvas
    [:.main-view
      (filter-pane db)
      (let [db (filtered-db db)]
        (list
          (overview-pane db)
          (todo-pane db)))]
    (add-view)
    (history-view db)])
```

The main `render` function uses the current `@conn` as app data and mounts the canvas on the document `<body>` element.

```clj
(defn render
  ([] (render @conn))
  ([db]
    (profile "render"
      (rum/mount (canvas db) js/document.body))))
```

We set up a transaction report listener, to re-render on every transaction!

```
;; re-render on every DB change
(d/listen! conn :render
  (fn [tx-report]
    (render (:db-after tx-report))))
```

We set up another transaction report listener, to log transactions via `println` (which prints via `console.log`).

```
;; logging of all transactions (prettified)
(d/listen! conn :log
  (fn [tx-report]
    (let [tx-id  (get-in tx-report [:tempids :db/current-tx])
          datoms (:tx-data tx-report)
          datom->str (fn [d] (str (if (:added d) "+" "−")
                               "[" (:e d) " " (:a d) " " (pr-str (:v d)) "]"))]
      (println
        (str/join "\n" (concat [(str "tx " tx-id ":")] (map datom->str datoms)))))))
```

```
;; history
(d/listen! conn :history
  (fn [tx-report]
    (let [{:keys [db-before db-after]} tx-report]
      (when (and db-before db-after)
        (swap! history (fn [h]
          (-> h
            (u/drop-tail #(identical? % db-before))
            (conj db-after)
            (u/trim-head history-limit))))))))
```

We now define some utility methods `db->string` and `string->db` to serialize/deserialize the database to/from the string format used to save/restore the DB from localstorage. Note that we use `datascript-transit` for this, using the `dt` namespace alias, via `dt/write-transit-str` and `dt/read-transit-str`

```
;; transit serialization

(defn db->string [db]
  (profile "db serialization"
    (dt/write-transit-str db)))

(defn string->db [s]
  (profile "db deserialization"
    (dt/read-transit-str s)))
```

We now define a `persist` method which given the DB (app state), saves the db to localstorage at `datascript-todo/DB` by serializing the `db` to a string via `db->string` utility method.

```
;; persisting DB between page reloads
(defn persist [db]
  (js/localStorage.setItem "datascript-todo/DB" (db->string db)))
```

We then set up a listener called `:persistence` which on every transaction (`tx-report`) will persist the latest state to localstorage via `persist` we defined just before.

```clj
(d/listen! conn :persistence
  (fn [tx-report] ;; FIXME do not notify with nil as db-report
                  ;; FIXME do not notify if tx-data is empty
    (when-let [db (:db-after tx-report)]
      (js/setTimeout #(persist db) 0))))
```

On app load, the following will be executed, which tests if we have an existing app state stored in localstorage, and if so we will use this state to set the connection state and history. If not, we will initialize the DB by transacting some fixture data `u/fixtures`

```clj
;; restoring once persisted DB on page load
(or
  (when-let [stored (js/localStorage.getItem "datascript-todo/DB")]
    (let [stored-db (string->db stored)]
      (when (= (:schema stored-db) schema) ;; check for code update
        (reset-conn! stored-db)
        (swap! history conj @conn)
        true)))
  (d/transact! conn u/fixtures))
```

We then clear the local storage

```
#_(js/localStorage.clear)
```

And call the initial render method which displays the data of the current connection.

```
;; for interactive re-evaluation
(render)
```



