## Javascript integration

See the [Javascript-API](https://github.com/tonsky/datascript/wiki/Javascript-API) for a basic overview.

The [public API](https://github.com/tonsky/datascript/blob/master/src/datascript/js.cljs#L61) contains all available functions exported for use in JS land ;)

There is a good (but dated) post on js/cljs interop [here](http://www.spacjer.com/blog/2014/09/12/clojurescript-javascript-interop/)

## Simple example app

As you can see from the following code, the Javascript API looks almost identical to using the API from clojure, except for the missing `!` on side effect functions.

```js
import {datascript as d} from 'datascript'

// Use regular JS API for create connection and add data to DB',
  // schema is JS Object
var schema = {
  "aka": {":db/cardinality": ":db.cardinality/many"}, 
  "friend": {":db/valueType": ":db.type/ref"}
};

var conn = d.create_conn(schema);
var reports = []

d.listen(conn, "main", report => {
    reports.push(report)
})

var datoms = [{
      ":db/id": -1,
      "name": "Ivan",
      "age": 18,
      "aka": ["X", "Y"]
    },
    {
      ":db/id": -2,
      "name": "Igor",
      "aka": ["Grigory", "Egor"]
    },
    [":db/add", -1, "friend", -2]
];

// Tx is Js Array of Object or Array
d.transact(conn, datoms, "initial info about Igor and Ivan")

// report is regular JS object'
// query mori values from conn with CLJS API

var result = d.q(parse('[:find ?n :in $ ?a :where [?e "friend" ?f] [?e "age" ?a] [?f "name" ?n]]'), d.db(conn), 18);
```

