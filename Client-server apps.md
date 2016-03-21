## Building Client/Server apps

A typical `project.clj` file would look something like this, assuming we are using datomic and datascript, async channels and ReactJS.

```clj
:dependencies [
    ;; clojure and clojurescript
    [org.clojure/clojure "1.8.0"]
    [org.clojure/clojurescript "1.7.145"]

    ;; Async channels
    [org.clojure/core.async "0.2.371"]

    ;; App component structure
    [com.stuartsierra/component "0.3.0"]

    ;; React "flux" libs
    [reagent "0.5.1"]
    [re-frame "0.5.0"]
    [posh "0.3.3.1"]

    ;; client DB (datascript)
    [datascript "0.14.0"]

    ;; Server DB (datomic)
    [com.datomic/datomic-free "0.9.5327" :exclusions [joda-time]]
```

App examples in the wild
- [acha-acha](http://tonsky.me/blog/acha-acha/)
- [cat chat](http://tonsky.me/blog/datascript-chat/)
- [todo](https://github.com/tonsky/datascript-todo)
- [photo viewer](https://github.com/piranha/showkr/tree/ds)

Usage/integrations
- [with reagent](https://gist.github.com/allgress/11348685)

