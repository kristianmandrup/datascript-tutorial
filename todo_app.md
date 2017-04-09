# Todo app

The `core` namespace includes `datascript`, `datascript-transit` and `rum` as the main libraries. `datascript-transit` is used to serialize to the awesome [transit]() format. `rum` is a React component framework, similar to reagent or Om.

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

We start off by defining the schema as a regular clojure Map then define a function using `defonce` which can create a DB connection from the schema.
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

We declare the render and persist functions so we can reference them before
they are defined.

```clj
(declare render persist)
```

The `reset-conn!` function is used to set the current connection to point to a new DB atom, rendering it and persisting it to local storage.

```clj
(defn reset-conn! [db]
  (reset! conn db)
  (render db)
  (persist db))
```

The `set-system-attrs!` function will be used by the `filter-pane` to set system attributes on the "special system entity" with id `0` (just a specific convention we use here).
```clj
;; Entity with id=0 is used for storing auxilary view information
;; like filter value and selected group

(defn set-system-attrs! [& args]
  (d/transact! conn
    (for [[attr value] (partition 2 args)]
      (if value
        [:db/add 0 attr value]
        [:db.fn/retractAttribute 0 attr]))))
```

The `system-attr` function is used to read this attribute value.

```clj
(defn system-attr
  ([db attr]
    (get (d/entity db 0) attr))
  ([db attr & attrs]
    (mapv #(system-attr db %) (concat [attr] attrs))))
```

## History

We can now add some basic infrastruture for history

```clj
;; History
;; a list of atoms (ie. app states)

(defonce history (atom []))
(def ^:const history-limit 10)
```

We now create some filter functions.


```clj
;; Rules are used to implement OR semantic of a filter
;; ?term must match either :project/name OR :todo/tags
(def filter-rule
 '[[(match ?todo ?term)
    [?todo :todo/project ?p]
    [?p :project/name ?term]]
   [(match ?todo ?term)
    [?todo :todo/tags ?term]]])

;; terms are passed as a collection to query,
;; each term futher interpreted with OR semantic
(defn todos-by-filter [db terms]
  (d/q '[:find [?e ...]
         :in $ % [?term ...]
         :where [?e :todo/text]
                (match ?e ?term)]
    db filter-rule terms))

(defn filter-terms [db]
  (not-empty
    (str/split (system-attr db :system/filter) #"\s+")))
```

The `filtered-db` function filters a given db by the terms from `filter-terms` crating a whitelist via

```clj
(defn filtered-db [db]
  (if-let [terms   (filter-terms db)]
    (let[whitelist (set (todos-by-filter db terms))
         pred      (fn [db datom]
                     (or (not= "todo" (namespace (:a datom)))
                         (contains? whitelist (:e datom))))]
      (d/filter db pred))
    db))
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

### Toggle Todo done

The `toggle-todo-tx` function, swaps the value of `:todo/done` attribute via a transaction: `[:db/add eid :todo/done (not done?)]`

```clj
;; This transaction function swaps the value of :todo/done attribute.
;; Transaction funs are handy in situations when to decide what to do
;; you need to analyse db first. They deliver atomicity and linearizeability
;; to such calculations
(defn toggle-todo-tx [db eid]
  (let [done? (:todo/done (d/entity db eid))]
    [[:db/add eid :todo/done (not done?)]]))
```

The `toggle-todo` function transacts the toggling of `todo/done` status.

```clj
(defn toggle-todo [eid]
  (d/transact! conn [[:db.fn/call toggle-todo-tx eid]]))
```

### Extract Todo item values

The `extract-todo` function extract a Todo Map (object) from the *Add task* form (when submitted).

```clj
(defn extract-todo []
  (when-let [text (dom/value (dom/q ".add-text"))]
    {:text    text
     :project (dom/value (dom/q ".add-project"))
     :due     (dom/date-value  (dom/q ".add-due"))
     :tags    (dom/array-value (dom/q ".add-tags"))}))
```

### Cleanup Todo form

The `clean-todo` function simply resets the *Add task* form.

```clj
(defn clean-todo []
  (dom/set-value! (dom/q ".add-text") nil)
  (dom/set-value! (dom/q ".add-project") nil)
  (dom/set-value! (dom/q ".add-due") nil)
  (dom/set-value! (dom/q ".add-tags") nil))
```

## Add new Todo item

The `add-todo` function adds a `todo` to the application state.
First the todo object is extracted via `(when-let [todo (extract-todo)]`.
Then `when-let` means that when clause is only executed if the `let` assigns a truthy value (ie. not `null` or `false`).
Having extracted a todo Map (object), we then proceed to extract/calculate individual todo attributes such as `project`, `project-id` etc.
the `project-id` is found by calling `(u/e-by-av @conn :project/name project)`, ie. get entity from `@conn` by attribute and value match: `:project/name project`.

Then we set `project-tx` to be a new entity to be added, `[[:db/add -1 :project/name project]])` if there is yet no project indicated for the todo task being added.

We then define the todo `entity` to transact (ie. to be upserted) using these and other todo values such as `due` and `tags`. Then the entities are transact via `(d/transact! conn (concat project-tx [entity])))`

The todo entity is concatenated to the `project-tx` transaction (see above) or on its own, referencing an existing project entity via `todo/project`.
Finally we call `clean-todo` to reset the form.

```clj
(defn add-todo []
  (when-let [todo (extract-todo)]
    ;; This is slightly complicated logic where we need to identify
    ;; if a project with such name already exist. If yes, we need its
    ;; id to reference from entity, if not, we need to create it first
    ;; and then use its id to reference. We’re doing both in a single
    ;; transaction to avoid inconsistencies
    (let [project    (:project todo)
          project-id (when project (u/e-by-av @conn :project/name project))
          project-tx (when (and project (nil? project-id))
                       [[:db/add -1 :project/name project]])
          entity (->> {:todo/text (:text todo)
                       :todo/done false
                       :todo/project (when project (or project-id -1))
                       :todo/due  (:due todo)
                       :todo/tags (:tags todo)}
                      (u/remove-vals nil?))]
      (d/transact! conn (concat project-tx [entity])))
    (clean-todo)))
```    

## Rum: React components

We create our React view components via the lightweight, flexible  [Rum]() framework also by [@tonsky]().

Rum is an alternative React renderer, similar to Om, Reagent and Omniscient, but designed to be more flexible.

Rum let's us easily design React components with actions, much like redux.

First we define a `filter-pane` component which contains an input element that triggers `(set-system-attrs! :system/filter value)`

```clj
;; Keyword filter

(rum/defc filter-pane [db]
  [:.filter-pane
    [:input.filter {:type "text"
                    :value (or (system-attr db :system/filter) "")
                    :on-change (fn [_]
                                 (set-system-attrs! :system/filter (dom/value (dom/q ".filter"))))
                    :placeholder "Filter"}]])
```

## Displaying filtered list of Todos

The `todo-pane` component shows the todos in the current app state with current filters applied. It first retrieves the `:system/group` and `:system/group-item` system entity attributes and calls `todos-by-group` with these filter values to get the list of `todos` to display.
Then the todos are iterated by entity id (and sorted) via `(for [eid (sort todos) ...`. For each entity id we fetch the full Todo entity via `(d/entity db eid)`, stored in the local `td` Map (object). We use `td` to display the Todo item and project it belongd to. The display includes a checkbox to toggle `:todo/done` status for the todo, via `(toggle-todo eid)` (see above).

```clj
(rum/defc todo-pane [db]
  [:.todo-pane
    (let [todos (let [[group item] (system-attr db :system/group :system/group-item)]
                  (todos-by-group db group item))]
      (for [eid (sort todos)
            :let [td (d/entity db eid)]]
        [:.todo {:class (if (:todo/done td) "todo_done" "")}
          [:.todo-checkbox {:on-click #(toggle-todo eid)} "✔︎"]
          [:.todo-text (:todo/text td)]
          [:.todo-subtext
            (when-let [due (:todo/due td)]
              [:span (.toDateString due)])
            ;; here we’re using entity ref navigation, going from
            ;; todo (td) to project to project/name
            (when-let [project (:todo/project td)]
              [:span (:project/name project)])
            (for [tag (:todo/tags td)]
              [:span tag])]]))])
```

### Add view

The *Add* view let's us add a new Task (todo item). It builds a form where we can name the task and set: `projects`, `tags` and `due date` for the task.
We set the `on-submit` function of the form to call `add-todo` with the task (ie. form values) to be added.

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

## History view

The History view let's us go forward and backwards in time, by calling `u/find-prev` and `u/find-next` respectively on the `@history` state to retrieve a given atom state and then passing that state to `reset-conn!` such as `(reset-conn! prev)` to set our current app state to a specific time in history.

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

## Main app view

The main component of the app is defined as `canvas`.
The `canvas` renders the `main`, `history` and `add task` views. The `main` view contains a `filter` pane and the panes `overview-pane` and `todo-pane`.

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

## App rendering

The main `render` function uses the current `@conn` as app data and mounts the canvas on the document `<body>` element.

```clj
(defn render
  ([] (render @conn))
  ([db]
    (profile "render"
      (rum/mount (canvas db) js/document.body))))
```

## Render trigger

We set up a transaction report listener `:render`, to re-render on every transaction!

```
;; re-render on every DB change
(d/listen! conn :render
  (fn [tx-report]
    (render (:db-after tx-report))))
```

## Logging

We set up another transaction report listener `:log`, to log transactions via `println` (which prints via `console.log`).

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

## History update trigger

Finally the `:history` listener is configured to trigger when we have both a `:db-before` and `:db-after` in the `tx-report`, ie. `(when (and db-before db-after) ...)`, ie. we have a previous (version) history and a new history entry.

Then we `swap!` (ie update) the existing history with a new history that has the new history entry appended (via `(conj db-after)`). We finally ensure that the history is trimmed, limited to the `history-limit` size (default `10`) via `(u/trim-head history-limit)`.

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

## Localstorage serialization

We now define the utility serialize methods `db->string` and `string->db` to serialize/deserialize the database to/from the string format used to save/restore the DB from *localstorage*. Note that we use `datascript-transit` for this, using the `dt` namespace alias, via `dt/write-transit-str` and `dt/read-transit-str`

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
  (fn [tx-report]
    (when-let [db (:db-after tx-report)]
      (js/setTimeout #(persist db) 0))))
```

## App load

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

We then clear the local storage (once on on app load, after state has been loaded)

```
#_(js/localStorage.clear)
```

Finally we call the initial `render` method, which displays the data of the current connection.

```
;; for interactive re-evaluation
(render)
```

This example should demonstrate a typical setup for a modern FRP (CES) application, using reactive UI components, a "flux"-like model, using actions and immutable app state.