# Acha Acha

The [acha-acha](http://acha-acha.co/) app works as follows.

## Model

First we define a DB schema with `repo`, `user` and `achent` (ie. achievement).
All simple reference types. We then create a connection to a database with that schema.

```cljs
;; DB
(def ^:private schema {
   :ach/repo   {:db/valueType :db.type/ref}
   :ach/user   {:db/valueType :db.type/ref}
   :ach/achent {:db/valueType :db.type/ref}})

(def conn (d/create-conn schema))
```

Then we define the main app state atom:
- progress
- users
- first load or not?
- path

```cljs
(def app-state (atom {:progress    0
  :users {:visible 0, :total 0}
  :first-load? true
  :path        "/"}))
```

### Listen loop

The listen loop sets up 3 main `core.async` incoming data channels:
- `socket-ch` (events)
- `data-ch` (data)
- `ajax-ch` (batch data)

We then listen to the socket channel `/events` and set up listeners for:
- `on-open` puts `:open` on the `socket-ch` to notify start receiving data.
- `on-close` closes all channels.
- `on-message` puts the message on the `data-ch` for processing.

We then start an async `go` routine for the incoming `socket-ch` data.
We set progress initially to `0.3` (30%). An ajax call is made to `/api/db` to fetch all datomsand put them on the `ajax-ch` channel, setting progress to `0.6` when complete.
When we have received all data (or after 30 secs timeout), we reset the DB connection with the data `dump` and `schema`. Then calculate the numer of users, datoms and achievements using the indexes directly (faster!).
Then we set the `app-state` of the app and set progress to `1` (ie. completed).

Secondly we create an infinite loop (via `loop`, `recur`) to listen to data on the `data-ch` channel and transact each data item into our local DB, thus syncing the remote DB (repo) with the local DB. We also update the `:total` number of `:users` for each repo synced.
Finally we reset the progress to `-1` to make it disappear (see `progress-bar` below) and setup a timeout to call the `listen-loop` every second.

```cljs
(defn- listen-loop []
  (let [socket-ch (async/chan 1)
        data-ch   (async/chan 10)
        ajax-ch   (async/chan 1)
        socket (ws/connect "/events"
                 :on-open    #(async/put! socket-ch :open)
                 :on-close   #(doseq [ch [socket-ch ajax-ch data-ch]]
                                (async/close! ch))
                 :on-message #(async/put! data-ch %))]
    (go
      (when (async/<! socket-ch) ;; open socket
        (swap! app-state assoc :progress 0.3)
        (u/ajax "/api/db/" (fn [datoms] (async/put! ajax-ch datoms) (swap! app-state assoc :progress 0.6)))
        (let [[dump _] (async/alts! [ajax-ch (async/timeout 30000)])]
          (when dump  ;; wait ajax
            (profile "DB initialization"
              (reset! conn (d/init-db dump schema))
              (let [num-users (count (d/datoms @conn :aevt :user/name))
                    num-datoms (count (:eavt @conn))
                    num-achs (count (d/datoms @conn :aevt :ach/sha1))]
                (println "Pushed [datoms:" num-datoms "] [users:" num-users "] [achievements:" num-achs "]")
                (swap! app-state assoc
                  :users {:total num-users :visible (min 30 num-users)}
                  :first-load? false
                  :progress 1)))
            (loop []
              (when-let [tx-data (async/<! data)]  ;; listen for socket
                (try
                  (d/transact! conn tx-data)
                  (let [num-users (count (d/datoms @conn :aevt :user/name))]
                    (swap! app-state assoc-in [:users :total] num-users))
                  (catch js/Error e
                    (.error js/console e)))
                (recur))))
          (.close socket)))
     (swap! app-state assoc :progress -1)
     (js/setTimeout listen-loop 1000))))
```

### Start

The app is started by calling the public function `start`.
We mount the `application` component on the `<body>` element, passing the `app-state` and the DB connection `conn`. We also set up a DOM listener to update the `:path` of `app-state` when we do page navigation.
The `listen-loop` is started to listen on socket data.
We also listen to the DB `conn` for changes to the transaction report `tx-report`. When a datom is added we set a timeout to retract it after 10 seconds.

```cljs
(defn ^:export start []
  (rum/mount (application app-state conn) (.-body js/document))
  (dom/listen-nav #(swap! app-state assoc :path %))

  (listen-loop)

  (d/listen! conn
    (fn [tx-report]
      (doseq [datom (:tx-data tx-report)
              :when (and (= (.-a datom) :message/text)
                         (.-added datom))]
        (js/setTimeout (fn [] 
                         (d/transact! conn [[:db.fn/retractEntity (.-e datom)]]))
                       10000)))))
```

## Views

[Rum](https://github.com/tonsky/rum) is used as the React wrapper.

The main view is called `application`. We pass in the application state atom `app-state` and the DB connection `conn`. We then define them as the local vars `db`, `state` and then the `path` is extracted from state.

The `path` determines which page to show, one of either:
- `index-page`
- `user-page`
- `repo-page`

### Page

Each page is wrapped in a `header` and `footer`.

```cljs
(rum/defc application < rum/cursored-watch [app-state conn]
  (let [db @conn
        state @app-state
        path (:path state)]
    [:.window {:on-click (fn [e] (when (inner-nav? e) (dom/scroll-to-top)))}
      (header state)
      (when-not (:first-load? state)
        (list
          (messages-overlay db)
          (cond
            (index? path) (index-page db state)
            (u/starts-with? path "/user/") (user-page db (subs path (count "/user/")))
            (u/starts-with? path "/repo/") (repo-page db (subs path (count "/repo/"))))
          (footer db)))]))
```

A page contains:
- header
- footer (static)

The `header` shows the progress bar only when the app is first loading.

```cljs
(rum/defc header < rum/static [state]
  (if (:first-load? state)
    [:.header (progress-bar (:progress state))])
  [:.header
    [:a {:href "https://github.com/someteam/acha"
         :target "_blank"}
      [:div.ribbon]]
    (conj
      (if (index? (:path state))
        [:div.logo {:title "Acha-acha"}]
        [:a.logo   {:title "Acha-acha"
                    :href  "#/"}])
      [:h2 "Enterprise Git Achievement Solution" [:br] "Web Scale. In the Cloud"])
    (progress-bar (:progress state))])
```

The progress bar displays the percentage of the database loaded (received) from the server.
The progress bar is displayed if there is any (ie. non-negative) `progress`, stored in the app `state`.

The progress bar is displayed by using `style` with `width` set to progress (`0.0` - `1.0` "percentage") x `900` px.

```cljs
(rum/defc progress-bar < rum/static [progress]
  (if (neg? progress)
    [:.progress.progress__offline]
    [:.progress
      [:.progress__bar
       {:style {:width (* progress 900)}}]]))
```

## Pages

The pages for acha-acha, are:
- `index-page`
- `user-page`
- `repo-page`

The `index-page` displays the `user-pane` and `repo-pane`
The `title` is empty.

```cljs
(rum/defc index-page < rum/static [db state]
  (do
    (dom/set-title! nil)
    [:div
      (users-pane db state (u/qes-by db :user/email))
      (repo-pane  db)]))
```

The `repo-page` displays the `ach-pane` (achievements) and `repo-profile`.
The `title` is the repo name.

```cljs
(rum/defc repo-page [db url]
  (let [repo  (u/qe-by db :repo/url url)
        aches (->> (:ach/_repo repo) group-aches)]
    (dom/set-title! (repo-name url))
    [:div
      (ach-pane aches repo-achent)
      (repo-profile repo aches)]))
```

The `user-page` displays the `ach-pane` (achievements) using the `user-achent` view component. The user page displays user info for the user by email (unique id). The `user` is retrieved by querying the db for a user by email.
The `user-profile` is then displayed, passing the `user` and his/her achievements.
The `title` is the user name.

```cljs
(rum/defc user-page [db email]
  (let [user  (u/qe-by db :user/email email)
        aches (->> (:ach/_user user) group-aches)]
    (dom/set-title! (:user/name user))
    [:div
      (ach-pane aches user-achent)
      (user-profile user aches)]))
```

## Panes

The panes on the index page are:
- `repo-pane`
- `user-pane`

The `repo-pane` fetches the `repos` by querying the db for matches on repo url, using `u/ques-by` (queries by).

The `last-aches` (last achievements) are retrieved directly by a `d/datoms`lookup in the `avet` index, for attributes matching `:ach/assigned`, then taking the last 10 in reverse.

The repos are then displayed in a `li` list and with a form input underneath to clone the url for a given repo.

The last achievements are iterated and each displayed via `last-ach`, passing the achievement by calling `d/entity` to feth the full entity of the achievement (using `(.-e ach)`).

```cljs
(rum/defc repo-pane < rum/static [db]
  (let [repos      (u/qes-by db :repo/url)
        last-aches (->> (d/datoms db :avet :ach/assigned) (take-last 10) reverse)]
    [:.repo_pane.pane
      [:h1 "Repos"]
      [:ul
        (map (fn [r] [:li {:key (:db/id r)} (repo r)]) repos)]
      [:form.add_repo {:on-submit (fn [e] (add-repo) (.preventDefault e))}
        [:input {:id "add_repo__input" :type :text :placeholder "Clone URL"}]]

      (when (not-empty last-aches)
        (list
          [:h1 {:style {:margin-top 100 :margin-bottom 40}} "Last 10 achievements"]
          [:.laches
            (for [ach last-aches]
              (last-ach (d/entity db (.-e ach))))]))]))
```

The `users-pane` displays a pane for a list of `users`, by mapping the `users` and rendering a `user` component for each.

```cljs
(rum/defc users-pane < rum/static user-pane-appender-mixin [db state users]
  (let [ach-cnt (u/qmap '[:find  ?u (count ?a)
                          :where [?e :ach/achent ?a]
                                 [?e :ach/user ?u]] db)
        visible (get-in state [:users :visible])
        users (->> users (sort-by #(ach-cnt (:db/id %) -1)) reverse (take visible))]
    [:.users_pane.pane
      [:h1 "Users"]
      (if (not-empty users)
        [:ul
          (map (fn [u] [:li {:key (:db/id u)} (user u (ach-cnt (:db/id u)))]) users)]
        [:.empty "Nobody achieved anything yet"])]))
```

The `ach-cnt` defined as a `u/qmap` function, passing a query `q` which finds and counts all `:ach/achent` (achievements) for a given user `u?` in `:ach/user`.

```(defn qmap
  "Convert returned 2-tuples to a map"
  [q & sources]
  (into {} (apply -q q sources)))
```

The `users` are the defined by sorting by incoming `users` by their achievement count (ie. `ach-cnt`).

The users are then displayed by mapping over them and displaying each in a `li` list.

## Repos

Repos are displayed via:
- `repo-pane`
- `repo-profile`

The `repo-profile` uses `repo-status` helper.

```cljs
(rum/defc repo-profile < rum/static [repo aches]
  [:.rp.pane
    (let [repo-url (:repo/url repo)]
      (list
        [:.rp__name (repo-name repo-url)
          [:.id (:db/id repo)]
          (repo-status repo)]
        (if-let [web-url (repo-web-url repo-url)]
          [:a.rp__url {:href web-url :target "_blank"} repo-url]
          [:.rp__url repo-url])))
    (when-not (str/blank? (:repo/reason repo))
      (list
        [:.rp__reason (:repo/reason repo)]
        [:button.rp__delete {:on-click (fn [_] (delete-repo (:db/id repo)))}]))
    [:.rp__hr]
    [:.rp__achs
      (for [[achent _] aches]
        [:img.rp__ach {:key   (:db/id achent)
                       :src   (achent-img achent)
                       :title (str (:achent/name achent) ":\n\n" (:achent/desc achent))}])]])
```

A `repo-achent` achievement view takes an achievement `achent` and a list of achievements `aches` and displays the details of the achievement, then iterating over the achievement list and displaying user details for each.

```cljs
(rum/defc repo-achent < rum/static [achent aches]
  [:.ach { :key (:db/id achent)}
    [:.ach__logo
      [:img {:src (achent-img achent)}]]
    [:.ach__name (:achent/name achent)]
    [:.ach__desc (:achent/desc achent)]
    [:div.ach__users
      (for [ach aches
            :let [user   (:ach/user ach)
                  avatar (avatar (:user/email user) 114)]]
        [:a {:key  (:db/id user)
             :href (str "#" (user-link user))}
          [:img.ach__user {:src avatar
                           :title (ach-details ach)}]])]])
```

## Users

Users are displayed via:
- user-page
- user-profile

The `user-page` displays an achievement pane for each user and the user profile.

```cljs
(rum/defc user-page [db email]
  (let [user  (u/qe-by db :user/email email)
        aches (->> (:ach/_user user) group-aches)]
    (dom/set-title! (:user/name user))
    [:div
      (ach-pane aches user-achent)
      (user-profile user aches)]))
```

The `user` component displays details about a given user

```cljs
(rum/defc user < rum/static [user ach-cnt]
  [:a.user {:key (:db/id user)
            :href (str "#" (user-link user))}
    [:.user__avatar
      [:img {:src (avatar (:user/email user) 114)}]]
    [:.user__name (:user/name user) [:.id (:db/id user)]]
    [:.user__email (:user/email user)]
    (when ach-cnt [:.user__ach ach-cnt])])
```

The `user-profile` displays user info and loops over user achievements in `aches` and displays each as an image.

```cljs
(rum/defc user-profile < rum/static [user aches]
  [:.up.pane
    [:.up__avatar
      [:img {:src (avatar (:user/email user) 228)}]]
    [:.up__name (:user/name user) [:.id (:db/id user)]]
    [:.up__email (:user/email user)]
    [:.up__hr]
    [:.up__achs
     (for [[achent _] aches]
       [:img.up__ach {:key   (:db/id achent)
                      :src   (achent-img achent)
                      :title (str (:achent/name achent) ":\n\n" (:achent/desc achent))}])]])
```


The `user-achent` displays a list of user achievements

```cljs
(rum/defc user-achent < rum/static [achent aches]
  [:.ach { :key (:db/id achent)}
    [:.ach__logo
      [:img {:src (achent-img achent)}]]
    [:.ach__name (:achent/name achent)
      (let [max-lvl (reduce max 0 (map #(:ach/level % 0) aches))]
        (when (pos? max-lvl)
          [:.ach__level {:title (:achent/level-desc achent)} (str "lvl " max-lvl)]))]

    [:.ach__desc (:achent/desc achent)]
    [:div.ach__links
      (for [ach aches]
        (ach-link ach))]])
```

The `user` component displays user info and uses the `avatar` helper to display linked avatar image.

```cljs
(rum/defc user < rum/static [user ach-cnt]
  [:a.user {:key (:db/id user)
            :href (str "#" (user-link user))}
    [:.user__avatar
      [:img {:src (avatar (:user/email user) 114)}]]
    [:.user__name (:user/name user) [:.id (:db/id user)]]
    [:.user__email (:user/email user)]
    (when ach-cnt [:.user__ach ach-cnt])])
```

The `user-profile` displays user info and achievements looping over each achievement to display an image.

```cljs
(rum/defc user-profile < rum/static [user aches]
  [:.up.pane
    [:.up__avatar
      [:img {:src (avatar (:user/email user) 228)}]]
    [:.up__name (:user/name user) [:.id (:db/id user)]]
    [:.up__email (:user/email user)]
    [:.up__hr]
    [:.up__achs
     (for [[achent _] aches]
       [:img.up__ach {:key   (:db/id achent)
                      :src   (achent-img achent)
                      :title (str (:achent/name achent) ":\n\n" (:achent/desc achent))}])]])
```

`users-pane` displays all users, by mapping over the `users` list and rendering `user` component for each.

```cljs
(rum/defc users-pane < rum/static user-pane-appender-mixin [db state users]
  (let [ach-cnt (u/qmap '[:find  ?u (count ?a)
                          :where [?e :ach/achent ?a]
                                 [?e :ach/user ?u]] db)
        visible (get-in state [:users :visible])
        users (->> users (sort-by #(ach-cnt (:db/id %) -1)) reverse (take visible))]
    [:.users_pane.pane
      [:h1 "Users"]
      (if (not-empty users)
        [:ul
          (map (fn [u] [:li {:key (:db/id u)} (user u (ach-cnt (:db/id u)))]) users)]
        [:.empty "Nobody achieved anything yet"])]))
```

User achievements are displayed via `user-achent`

```cljs
(rum/defc user-achent < rum/static [achent aches]
  [:.ach { :key (:db/id achent)}
    [:.ach__logo
      [:img {:src (achent-img achent)}]]
    [:.ach__name (:achent/name achent)
      (let [max-lvl (reduce max 0 (map #(:ach/level % 0) aches))]
        (when (pos? max-lvl)
          [:.ach__level {:title (:achent/level-desc achent)} (str "lvl " max-lvl)]))]

    [:.ach__desc (:achent/desc achent)]
    [:div.ach__links
      (for [ach aches]
        (ach-link ach))]])
```


## Achievements

Achievements are displayed via `ach-pane` component which takes a list of achievements in `aches` and maps over them.

```cljs
(rum/defc ach-pane < rum/static [aches component]
  [:.ach_pane.pane
    [:h1 "Achievements"]
    (if (not-empty aches)
      (map (fn [[achent aches]] (component achent aches)) aches)
      [:.empty "No achievements yet. Try harder!"])])
```

The achievements are displayed using `ach-*` components.

```cljs
(rum/defc ach-link < rum/static [ach]
  (let [sha1     (:ach/sha1 ach)
        repo-url (get-in ach [:ach/repo :repo/url])
        text     (str (repo-name repo-url) "/" (subs sha1 0 7))
        title    (ach-details ach)]
    [:div {:key (:db/id ach)}
      (if-let [commit-url (commit-web-url repo-url sha1)]
        [:a.ach__text { :target "_blank"
                        :href   commit-url
                        :title  title}
         text]
        [:.ach__text {:title title} text])]))

(rum/defc last-ach < rum/static [ach]
  (let [achent (:ach/achent ach)
        user   (:ach/user   ach)]
    [:.lach {:key (:db/id ach)}
      [:a.lach__user {:href (str "#" (user-link user))}
        [:img.lach__user__img {:src (avatar (:user/email user) 114)}]
        [:.lach__user__name (:user/name user)]
        [:.lach__user__email (:user/email user)]]
      [:.lach__ach
        [:img.lach__ach__img {:src (achent-img achent)}]
        [:.lach__ach__name (:achent/name achent)]
        [:.lach__ach__desc (:achent/desc achent)]
        [:.lach__ach__links
          (ach-link ach)]]]))
```
