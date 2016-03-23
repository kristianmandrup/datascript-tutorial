## Building Client/Server apps

A typical `project.clj` file for Client/Server apps would look something like this. Here we assume we are using datomic and datascript, async channels and ReactJS.

```clj
:dependencies [
    ;; clojure and clojurescript
    [org.clojure/clojure "1.8.0"]
    [org.clojure/clojurescript "1.8.34"]

    ;; Async channels
    [org.clojure/core.async "0.2.374"]

    ;; App component structure
    [com.stuartsierra/component "0.3.1"]

    ;; client DB (datascript)
    [datascript "0.15.0"]

    ;; Server DB (datomic)
    [com.datomic/datomic-free "0.9.5350" :exclusions [joda-time]]
```

Typically we would then add a client stack such as [React](https://facebook.github.io/react/) with some sort of "flux" data layer libs like `re-frame` and `posh`.

```clj
    ;; React "flux" libs
    [reagent "0.5.1"]
    [re-frame "0.5.0"]
    [posh "0.3.4"]
```

### React native

For React Native, we might well use [Realm](https://realm.io/news/introducing-realm-react-native/) or perhaps [FeathersJS](http://feathersjs.com/)

## Client/Server app examples

App examples in the wild:
- [acha-acha](http://tonsky.me/blog/acha-acha/)
- [cat chat](http://tonsky.me/blog/datascript-chat/)
- [todo](https://github.com/tonsky/datascript-todo)
- [photo viewer](https://github.com/piranha/showkr/tree/ds)

Stack integration examples:
- [with reagent](https://gist.github.com/allgress/11348685)
- ...

