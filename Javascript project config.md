# Javascript project configuration

For a JavaScript project, the hottest build/packaging tool of 2016
seems to be webpack.

[Webpack Clojure](https://twitter.com/brandonbloom/status/598654885203062784)

- _Dear ClojureScript community, please study Webpack. It's pretty awesome._
- _@brandonbloom already did and it's not that awesome wrt JS bits ;)_

A comparison of Webpack with Google Closure modules can be found in this @swannodette blog post: [hello google closure modules](http://swannodette.github.io/2015/02/23/hello-google-closure-modules/)

"We'll briefly look at webpack's support for splitting and compare it to a little known feature of the Google Closure Compiler: [Google Closure Modules](https://github.com/google/closure-library/wiki/goog.module:-an-ES6-module-like-alternative-to-goog.provide)."

Google Closure Modules maintains the simple Closure Compiler philosophy:

_"don't let a human do anything a computer can do for you"_

Code splits are not defined in the source code and the modules you end up with may have nothing to do with the modules you actually wrote. This is a good thing. Closure Compiler _may freely move code between the modules you wrote to get optimized modules that you would have never written by hand that contain precisely what is needed_.

We'd like to split our application into three pieces, the shared bit, the bit for `hello-world.foo` and the bit for `hello-world.bar`. So in our `project.clj` file we would define a `:modules` entry like so:

```clj
{
  ...
  :modules {:foo {:output-to "out/foo.js"
                  :entries #{hello-world.foo}}
            :bar {:output-to "out/bar.js"
                  :entries #{hello-world.bar}}}
}
```

More info can be found on the [ClojureScript wiki](https://github.com/clojure/clojurescript/wiki/Compiler-Options#modules)

The final gzipped sizes of the modules for the example above:
- `cljs_base.js`, 22K
- `foo.js`, 3K
- `bar.js`, 0.04K

For large ClojureScript applications I think it's an understatement to say that this is a "game changer".

So no reason to use Webpack!

PS: For *dynamic* (runtime) module loading, check this [blog post](https://rasterize.io/blog/cljs-dynamic-module-loading.html)

