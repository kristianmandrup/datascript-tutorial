# Reagent integration

[Datascript Reagent example app](https://gist.github.com/allgress/11348685)

Go through it step by step...

```js
(ns reagent-test.core
  (:require [reagent.core :as reagent :refer [atom]]
            [datascript :as d]
            [cljs-uuid-utils :as uuid]))

(enable-console-print!)

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

;;; Creates a DataScript "connection" (really an atom with the current DB value)
(def conn (d/create-conn))

;;; Add some data
(d/transact! conn [{:db/id -1 :name "Bob" :age 30}
                   {:db/id -2 :name "Sally" :age 25}])

;;; Maintain DB history.
(def history (atom []))

(d/listen! conn :history (fn [tx-report] (swap! history conj tx-report)))

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

;;; Query to get name and age of peeps in the DB
(def q-peeps '[:find ?n ?a
               :where
               [?e :name ?n]
               [?e :age ?a]])

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

;;; Query to find peeps whose age is less than 18
(def q-young '[:find ?n
               :where
               [?e :name ?n]
               [?e :age ?a]
               [(< ?a 18)]])

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

;;; Some non-DB state
(def state (atom {:show-younguns false}))

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

;;; Initial render
(reagent/render-component [uber] (.-body js/document))
```

