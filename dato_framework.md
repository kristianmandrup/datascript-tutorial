# Dato framework

Dato is an alternative approach to building apps, heavily inspired by Meteor, Firebase, and Parse, but with a strong bent towards using FP to make app design, iteration, tooling, and implementing features considerable easier. By default it comes with lag-compensation, security rules, and server-side function call. It'll eventually extensible so that e.g. offline apps, Operational Transform (Etherpad/Google Docs-like functionality), and other behaviors should be accessible and efficient.

## Apps built with Dato

Demo Apps:

 * [TodoMVC in Dato](https://github.com/sgrove/datodomvc)

Dato is currently very framework-oriented. We aim to decompose it into smaller libraries that are useful by themselves.

 * A (transport independent) library for syncing datoms between DataScript and Datomic, with a middleware-like design that includes pluggable security and can be extended to handle things like offline-sync (effectively a Webpeer for Datomic, which has several very difficult challenges)
 * An efficient DataScript/React.js(or [Proact?](https://github.com/brandonbloom/proact))-based UI layer that provides an intuitive way to bind components to queries, and to parameterize those queries.
 * A (transport independent) RPC library that has both "fire-and-forget" and "legacy return-value-oriented" modes.

Everything in Dato-the-framework should be a combination of the above libraries.

# Rationale
In many ways, this is a radical departure from building apps as it's normally done. Sitting down to build the current production app that Dato is extracted from, I decided I wanted to do away with all of the cruft, the tedious non-app-specific work I had to do again and again whenever starting up a new app. There are also several frustrating pieces in current approaches to the frontend, in particular state, the shape of the state, synchonization of state, and triggering external effects (API calls, etc.). So the technical goals for the project (All videos below are from previous experiments or products that have informed Dato's designs or goals, not yet from Dato itself):

 * UI driven by a flat, homogeneous, queryable/navigable db-like immutable data structure. Think FB's GraphQL, but powered by DataScript.
 * Explicit separation of state transitions from effects in the UI (as much is possible in ClojureScript)
 * No REST endpoints, anywhere. At all. (This may change to something auto-generated from Dato's structures for API interop, but not for now). Instead, we expose the *entire* backend database to the user for natural data navigation/retrieval (similar to FB's Relay + GraphQL), and we use server-side functions for effects 
   * Datomic gives us a fantastic way to filter databases so that we have the illusion of having access to the full database and never have to worry security from the client's PoV.
   * SS functions are defined server-side (obviously), are used for effects while enforcing security, *not* for changing the database (although the Datomic db may be changed as a result of an effect). An example might be uploading a file, or requesting the server make an API call on the client's behalf. The client bootstraps the functions on load, and can seamlessly call them (asynchronously, but the user has core.async + macros if they want to clean that up), knowing that the server is in charge of security.
 * Neat (and incredibly useful!) tricks like client-side [time-traveling debuggers](https://dl.dropboxusercontent.com/u/412963/zensight/time_travel.mp4), [state save/restore](https://dl.dropboxusercontent.com/u/412963/om_save_restore.mov), [user-session replays](https://dl.dropboxusercontent.com/u/412963/zensight_replay_ctrl.mov), and predictive testing should all be possible without a developer spending any effort on it. Ideally generative testing for exploring the app's state-space should be possible with a small amount of effort as well.
 * Incredibly amenable to tooling - in-app inspection/editing of state, [queries](https://dl.dropboxusercontent.com/u/412963/sul/sul_public_1.mp4), [layout](https://dl.dropboxusercontent.com/u/412963/zensight/component_layout_tools.mp4), [component structure](https://dl.dropboxusercontent.com/u/412963/zenrise/zenrise_preview_intro1.mov) (All videos are from previous experiments or products that have informed Dato's design and goals)
 * [Collaborative apps](https://dl.dropboxusercontent.com/u/412963/mana/collab_mouse.mp4) should fall out of the design with (at most) minor additional effort on the part of the developer.

# Non-goals
I doubt Dato's design will ever scale to the needs of e.g. Twitter or FB. It should have similar (or better) characteristics to Meteor or Firebase given a significantly large enough server. For the vast, vast majority of apps, this is an acceptable trade-off for Dato's other goals.

# Overall usage

## Client-side flow:

See `components/root.cljs` for most of the app-specific implementation of this demo.

At bootup, the client will request two things from the server via a hard-coded (but replaceable) ss method, `bootstrap`: The current Datomic schema (which it will merge with a client-side schema and use to create the local DataScript DB), and the session-id. From there, everything falls into the following flow:

 1. Event is triggered (from the server, UI interaction, or something else)
 1. A (non-effectful, pure) transition handler is called with an immutable instance of the current database and the event payload. It returns additions/retractions that should happen as a response to the event.
 1. The additions/retractions are transacted into the database, producing a new immutable database
 1. A effecting handler is then called with a copy of the previous db, the current db, and the event payload. It's also given several functions from Dato to do things like make SS calls, trigger further events, etc.
 1. The UI is re-rendered with the new copy of the DB, and we wait for another event to loop back to step 1

![Dato State Flow](/docs/resources/dato_flow.png "Dato State Flow")

An example of the flow given when a user logs out:

```clj
    (defmethod transition :ui/user-logged-out
      [db payload]
      ;; This will cause out UI to re-render with the logged-out view
      [[:db/retract (:db/id (db/me db)) :user/me? true]])

    (defmethod effect! :ui/user-logged-out
      [context old-db new-db payload]
      ;; Retrieve the ss method and invoke it so our ss-session is also destroyed
      (let [log-out! (get-in context [:ss :log-out!])]
         ;; We simply call it without arguments or a handler, but we can pass anything that can be transit-serialized.
         ;; Because we don't provide a specific handler, the ss event will come back with an event-name of
         ;; :server/log-out!-succeeded or :server/log-out!-failed. We could implement specific handlers for those
         ;; cases if we'd like (rather than optimistically logging out the user, as in this example).
         (log-out!)))
```

Nearly all state transitions and effects can be modeled with these two simple concepts (although some transitions are cumbersome). But there are three distinct categories for transitions:

 1. Purely-local transition, not seen by the server or other users.
 1. Ephemeral transitions meant to be seen by other users (after security filters are applied), but not persisted as part of the global app state. Usually high-churn data such as mouse-position would fall into this category.
 1. Data that is meant to be persisted to the database, and then broadcast to other users (again, after security filters are applied)

The difference is simply indicated via meta-data on the transition handler's return-data, e.g.:

```clj
    (defmethod con/transition :ui/task-toggled
     [db {:keys [data]}]
     (let [task (:task data)]
       (with-meta
          [{:db/id (:db/id task)
            :task/completed? (not (:task/completed? task))}]
          ;; The change will be persisted to the durable Datomic DB server-side, the resultant server-side
          ;; tx will be broadcast out to clients who have indicated their interest. This client will also receive
          ;; an acknowledgment of the tx being persisted, which it can use to know whether we have any "pending" datoms
          ;; in our local DB.
          {:tx/persist? true})))

    (defmethod con/transition :ui/mouse-moved
     [db {:keys [data]}]
     (let [[x y] data
           ;; Get our local session from the db to update
           session (db/local-session db)]
       (with-meta
          [{:db/id (:db/id session)
             :mouse/position [x y]}]
          ;; The change will be inserted into our local db and sent to an in-memory instance of Datomic
          ;; (not persisted to disk), and the resultant server-side tx will be broadcast out to clients
          ;; who have indicated their interest. This client will not receive acknowledgment of the tx
          ;; (although this design aspect may change depending on real-world use cases)
          {:tx/broadcast? true})))
```

There are also times where it's useful to first get confirmation of server-side receipt before triggering another transition. This can be done via a `:tx/cb` key in the meta-data

```clj
    (defmethod con/transition :ui/content-created
     [db {:keys [data]}]
     (let [dato-guid (d/ruuid)
           content   (assoc (:content data) :db/id (d/tempid :db.part/user) :dato/guid dato-guid)]
       (with-meta
          [content]
          {:tx/persist? true
           :tx/cb (fn [new-db]
                    ;; This will be called after the server returns acknowledgment of the content entity above,
                    ;; that way we know we're not uploading a file to a piece of content that doesn't exist, and 
                    ;; we don't have to introduce racey workarounds.
                    (let [persisted-content (dsu/qe-by new-db :dato/guid dato-guid)]
                     (dato/cast! {:event :ui/file-ready-to-upload
                                  :data {:content content
                                         :file    (:file data)}})))})))


    (defmethod effect! :ui/file-ready-to-upload
      [{:keys [ss]} old-db new-db {:keys [file content]}]
      (let [upload-file! (:upload-file! ss)]
        (upload-file! content file)))
```

> (NB: the example above would break replay since the file/blob that flowed through the event bus would not be serializable. This breaks one of the goals of the project, and is an open area of research. Also, the signature of the `cast!` call there is likely to change.)

Using `cast!` as the basis of casting messages (or raising events, still deciding on the terminology), Dato provides some sugar on top of this when building the UI side of things. A typical database transaction in a live HTML form might look like:

```clj
    [:input.toggle {:type      "checkbox"
                    :checked   (:task/completed? task)
                    :on-change (fn [event]
                                 (transact! :task-toggled [{:db/id           (:db/id task)
                                                            :task/completed? (not (:task/completed? task))}]
                                            {:tx/persist? true}))}]
```

Where `transact!` takes advantage of a default (but overrideable) transition handler provided for by Dato, `:db/updated`. It simply dumps the datoms into the db and passes along the meta-data. Here's how it's implemented:

```clj
    (defmethod transition :db/updated
      [db {:keys [data] :as payload}]
      (assert (keyword? (:intent data)) "DB update must include an :intent key with a keyword value")
      (assert (vector? (:tx data)) "DB update must include an :tx key with a vector value of valid Datascript transaction data")
      (let [m (meta (:tx data))]
        (with-meta (:tx data) (assoc m :tx/intent (:intent data)))))
```

Implementing purely-functional state transition in the UI becomes trivial. Of course, you can build your own abstractions/handlers by using `cast!` and `(defmethod transition :event/name ...)` directly.


## Server-side flow:

The server creates an instance of `DatoServer`, providing a routing table for SS calls (although the default routing table will get you pretty far for simple apps), and configuration for where to open the WebSocket path/port. The server is responsible for providing the HTML to bootstrap the initial client-side bootstrap call (although more info could be provided on initial load as an optimization), and then creates a session upon a new incoming WebSocket request.

The code might look like:

```clj
    (def dato-server
      (dato/map->DatoServer {:routing-table (assoc dato/default-routing-table
                                                   ;; SS function to be made available/callable by the client
                                                   :my-app/file-upload (fn [context session incoming] ....))}))

    (defn run [& [port]]
      (when is-dev?
        (dev/start-figwheel!))
      (dato/start! {:server dato/dato-server
                    :port   8080})
      (run-web-server port))
```

Some of the built-in SS functions:

 * r-pull: Execute a single (non-live) pull request against the full db available to this user session
 * r-qes-by: A single query-entities-by call, e.g. to get a one-off report of all of the entities with a `task/title` attribute:

```clj
     (dato/r-qes-by dato {:name :find-tasks
                          :a    :task/title})
```

There are others, I'll write about them later (functions for contacting/messaging other session directly, e.g. to start a WebRTC negotiation process)
Also, each of these should have a "live" version whereby the client indicates they want to be kept up to date on any transactions that invalidate it, but that's work yet to be done.

# Credit
The vast majority of the heavy lifting around Datomic/DataScript has been done by Daniel Woelfel (@dwwoelfel). I put together this example repo and implemented a bit of sugar on top.