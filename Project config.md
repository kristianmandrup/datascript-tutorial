# Clojure project configuration

See [[Datascript getting started]] for more on project setup.

```clj
(defproject my-app "0.0.1"
  :description "Datomic <-> DataScript syncing/replication utilities"
  :url "http://github.com/<account-id>/my-app"
  :license {:name "Eclipse Public License"
            :url "http://www.eclipse.org/legal/epl-v10.html"}
  :min-lein-version "2.0.0"
  :dependencies [
    [org.clojure/clojure "1.8.0"]
    ...
```

CLJS build configuration:

```
  :cljsbuild {
    ;; RELEASE build
    :builds [
      { :id "release"
        ;; where to grab the source files
        :source-paths ["src" "bench/src"]
        :assert false
        :compiler {
          ;; location of resulting build file
          :output-to     "release-js/my-app.bare.js"

          ;; Advanced optimization for Google Closure
          ;; Will minimize file size and uglify
          :optimizations :advanced

          ;; No pretty printing for production builds!
          :pretty-print  false
          :elide-asserts true
          :output-wrapper false
          :parallel-build true
        }
        ;:notify-command ["release-js/wrap_bare.sh"]
        }
  ]}
```

```
  :profiles {
    ;; DEV build
    :dev {
      :source-paths ["bench/src" "test" "dev" "src"]
      :plugins [
        [lein-cljsbuild "1.1.2"]
        [lein-typed "0.3.5"]
      ]
      :cljsbuild {
        :builds [
          ;; Optimized DEV build
          { :id "advanced"
            :source-paths ["src" "bench/src" "test"]
            :compiler {
              :output-to     "target/my-app.js"
              :optimizations :advanced
              :source-map    "target/my-app.js.map"
              :pretty-print  true
              :recompile-dependents false
              :parallel-build true
            }}
          ;; DEV build without optimizations
          { :id "none"
            :source-paths ["src" "bench/src" "test" "dev"]
            :compiler {
              :main          my-app.test
              :output-to     "target/my-app.js"
              :output-dir    "target/none"
              :optimizations :none
              :source-map    true
              :recompile-dependents false
              :parallel-build true
            }}
        ]
      }
    }
  }

  :clean-targets ^{:protect false} [
    "target"
    "release-js/my-app.bare.js"
    "release-js/my-app.js"
  ]

  ;; Once we're ready use Type checker
  ;:core.typed {:check []
               ;:check-cljs []}

  :resource-paths ["resources" "resources-index/prod"]
  :target-path "target/%s"

  ;; Build aliases
  :aliases {"package"
            ["with-profile" "prod" "do"
             "clean" ["cljsbuild" "once"]]})