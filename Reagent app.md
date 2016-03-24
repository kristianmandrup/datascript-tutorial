# Reagent integration

This example is taken from [Datascript/Reagent example app](https://gist.github.com/allgress/11348685)

We start off by requiring the libs needed, such as `datascript` and `reagent`.
We also enable console log printing via `(enable-console-print!)`

```clj
(ns reagent-test.core
  (:require [reagent.core :as reagent :refer [atom]]
            [datascript :as d]
            [cljs-uuid-utils :as uuid]))

(enable-console-print!)
```

## Binding to changes affecting local state

The infrastucture consists of a listener to listen on the `tx-report` for changes. On any novelty in the report, we reset the `state` to the latest version as per querying the current app state (ie `(:db-after tx-report)`.

```
(d/listen! conn k (fn [tx-report]
                         (let [novelty (d/q q (:tx-data tx-report))]
                           (when (not-empty novelty) ;; Only update if query results actually changed
                             (reset! state (d/q q (:db-after tx-report)))))))
```

The `bind` function let's our components subscribe to application changes for a given substate `state` of the app `[conn q state]`.
`q` is the query to identify the state. `state` is the local component atom to be updated on any detected change.
We use `bind` from a component like this, as shown in `peeps-view` below.

```
  (let [peeps (bind conn q-peeps)
```

Alternatively like this to auto-update a component atom (state):

```
  (let [y (atom nil)]
    (reagent/create-class
     {
      ;; Subscribe to db transactions.
      :component-will-mount
      (fn [] (bind conn q-young y))
```

Where state changes reflected in the query `q-young` cause an update of the atom `y` which causes a re-render of the reagent class via the `:render` function, which uses `@y`, as shown as cab be seen in `younguns-view` component below. We recommend placing `bind` and `unbind` in a utility/infrastructure namespace and not in `.core`.

```clj
(defn bind
  ([conn q]
   (bind conn q (atom nil)))
  ([conn q state]
   (let [k (uuid/make-random-uuid)]
     (reset! state (d/q q @conn))
     (d/listen! conn k (fn [tx-report]
                         (let [novelty (d/q q (:tx-data tx-report))]
                           (when (not-empty novelty) ;; Only update if query results actually changed
                             (reset! state (d/q q (:db-after tx-report)))))))
     (set! (.-__key state) k)
     state)))

(defn unbind
  [conn state]
  (d/unlisten! conn (.-__key state)))
```

### Transact initial data

We then create a new connection via `d/create-conn` and transact some initial data.

```clj
;;; Creates a DataScript "connection" (really an atom with the current DB value)
(def conn (d/create-conn))

;;; Add some initial data
(d/transact! conn [{:db/id -1 :name "Bob" :age 30}
                   {:db/id -2 :name "Sally" :age 25}])
```

### History

We setup a listener `:history` to manage the app `history` state list.

```clj
;;; Maintain DB history.
(def history (atom []))

(d/listen! conn :history (fn [tx-report] (swap! history conj tx-report)))
```

### Undo

We define an `undo` function to rewind the history and app state one step.
Here we perform `undo` it by inverting the transitions of each history transaction (as per `tx-data`). Undo is used in `peeps-view` (see below).

```clj
(defn undo
  []
  (when (not-empty @history)
    (let [prev (peek @history)
          before (:db-before prev)
          after (:db-after prev)
          ;; Invert transition, adds->retracts, retracts->adds
          tx-data (map (fn [{:keys [e a v t added]}] (d/Datom. e a v t (not added))) (:tx-data prev))]
      (reset! conn before)
      (swap! history pop)
      (doseq [[k l] @(:listeners (meta conn))]
        (when (not= k :history) ;; Don't notify history of undos
          (l (d/TxReport. after before tx-data)))))))
```

We now define a query which returns `name` and `age` of all peeps in the DB

```
;;; Query to get name and age of peeps in the DB
(def q-peeps '[:find ?n ?a
               :where
               [?e :name ?n]
               [?e :age ?a]])
```

### Peep views

The peep view displays all peeps in a list `<ul>` with `<li>` items, using `map`, iterating on `@peeps`.
The `<input>` elements for `name` and `age` update (`swap!`) the `temp` variable, an atom Map with empty `:name` and `:age` initially.
Fibally there is a button `"undo"` which is enabled only if there is a history of app state. It simply calls `undo` function on click to undo one app state in the history.

```
;; Simple reagent component. Returns a function that performs render
(defn peeps-view
  []
  (let [peeps (bind conn q-peeps)
        temp (atom {:name "" :age ""})]
    (fn []
      [:div
       [:h2 "Peeps!"]
       [:ul
        (map (fn [[n a]] [:li [:span (str "Name: " n " Age: " a)]]) @peeps)]
       [:div
        [:span "Name"][:input {:type "text"
                               :value (:name @temp)
                               :on-change #(swap! temp assoc-in [:name] (.. % -target -value))}]]
       [:div
        [:span "Age"][:input {:type "text"
                              :value (:age @temp)
                              :on-change #(swap! temp assoc-in [:age] (.. % -target -value))}]]
       [:button
        {:onClick (fn []
                    (d/transact! conn [{:db/id -1 :name (:name @temp) :age (js/parseInt (:age @temp))}])
                    (reset! temp {:name "" :age ""}))}
        "Add Peep"]
       [:button {:on-click #(undo) :disabled (= 0 (count @history))} "Undo"]])))
```

We define a query to find peeps whose age is < 18

```clj
;;; Query to find peeps whose age is less than 18
(def q-young '[:find ?n
               :where
               [?e :name ?n]
               [?e :age ?a]
               [(< ?a 18)]])
```

Then we create a `younguns-view` component, which displays all the young peep, using `q-young` as the data source.

We use `reagent/create-class` to create the reagent component (wrapper around a React class). We then `bind` our connection `conn` to the query `q-young` via `y`.

We use `bind` to subscribe to db transactions via Reagent. The rendering consists of listing the "young ones" by iterating over `@y`. Whenever `y` changes (via the binding/subscription), our component will re-render!

```clj
;;; Uses reagent/create-class to create a React component with lifecyle functions
(defn younguns-view
  []
  (let [y (atom nil)]
    (reagent/create-class
     {
      ;; Subscribe to db transactions.
      :component-will-mount
      (fn [] (bind conn q-young y))

      ;; Unsubscribe from db transactions.
      :component-will-unmount (fn [] (unbind conn y))

      :render
      (fn [_]
        [:div
         [:h2 "Young 'uns (under 18)"]
         [:ul
          (map (fn [[n]] [:li [:span n]]) @y)]])})))
```

The app state is defined as an atom, initially set to not display "younguns".

```clj
;;; Some non-DB state
(def state (atom {:show-younguns false}))
```

### Main app view

Our main app view component is called `uber` and renders subcomponents.
Note that no state is passed down to `younguns-view`, since state is managed internally via binding, so that the child component is completely self-contained!

```clj
;;; Uber component, contains/controls stuff and younguns.
(defn uber
  []
  [:div
   [:div [peeps-view]]
   [:div {:style {:margin-top "20px"}}
    [:input {:type "checkbox"
             :name "younguns"
             :onChange #(swap! state assoc-in [:show-younguns] (.. % -target -checked))}
     "Show Young'uns"]]
   (when (:show-younguns @state)
     [:div [younguns-view]])
   ])
```

Finally we mount the `uber` component on the document `<body>` element via `reagent/render-component`.

```clj
;;; Initial render
(reagent/render-component [uber] (.-body js/document))
```

