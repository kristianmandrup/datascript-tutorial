## Mori integration

Datascript can be based with [Mori](https://github.com/swannodette/mori), an immutable data model for Javascript created by legend *David Nolan* aka. [@swanodette](https://twitter.com/swannodette). 

Mori is typically used by [Om next](https://github.com/omcljs/om) as a powerful data layer for [ReactJS](https://facebook.github.io/react/).

The [datascript-mori](https://github.com/typeetfunc/datascript-mori) library achieves this integration by ...

@typeetfunc: add more here... plus examples!

### Getting started

```
import {
 datascript, // This is contain datascript object
 mori,       // This is contain mori object
 helpers     // This is contain helpers for conversions from CLJS
} from 'datascript-mori';

const {
 core, // This is pure DataScript CLJS API without any conversions
 js    // This is DataScript JS API
} = datascript;
```


```js
mport {datascript, mori, helpers} from '../datascript-mori'
import {assert} from 'chai'
var djs = datascript.js // use datascript_mori.datascript.js API
var dcljs = datascript.core // cljs API for quering
var {hashMap, vector, parse, toJs, equals, isMap, hasKey, isSet, set, getIn, get} = mori
var {DB_VALUE_TYPE, DB_TYPE_REF, DB_ADD, DB_ID, TEMPIDS} = helpers

// Use regular JS API for create connection and add data to DB',
  // schema is JS Object
var schema = {
  "aka": {":db/cardinality": ":db.cardinality/many"}, 
  "friend": {":db/valueType": ":db.type/ref"}
};

var conn = djs.create_conn(schema);
var reports = []

djs.listen(conn, "main", report => {
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
djs.transact(conn, datoms, "initial info about Igor and Ivan")

// report is regular JS object'
// query mori values from conn with CLJS API

var result = dcljs.q(parse('[:find ?n :in $ ?a :where [?e "friend" ?f] [?e "age" ?a] [?f "name" ?n]]'), djs.db(conn), 18);

// result is mori set')
set([vector("Igor")])) //result equals #{["Igor"]}')
```

## API usage

```js
import {datascript, mori, helpers} from '../datascript-mori'
import {assert} from 'chai'
var d = datascript.core // use datascript_mori.datascript.core API
var djs = datascript.js
var {hashMap, vector, parse, toJs, equals, isMap, hasKey, isSet, set, getIn, get} = mori
var {DB_VALUE_TYPE, DB_TYPE_REF, DB_ADD, DB_ID, TEMPIDS} = helpers
```

Add data to DB and query them
Scheme must be a mori structure or use `helpers.schema_to_clj({friend: {":db/valueType": ":db.type/ref"}})`

```js
  var scheme = hashMap(
    "friend", hashMap(
      DB_VALUE_TYPE, DB_TYPE_REF // or just keyword("db/valueType"), keyword("db.type/ref")
    )
  )
  var db = d.empty_db(scheme)
  // TX must be a Mori struct - vector of vector or hashMap

  var dbWithData = d.db_with(db, vector(
    vector(DB_ADD, -1, "name", "Ivan"),
    vector(DB_ADD, -1, "age", 17),
    hashMap(
      DB_ID, -2,
      "name", "Igor",
      "age", 35
    ),
    vector(DB_ADD, -1, "friend", -2)
  )) // add data to db.
```

or use `helpers.entities_to_clj( ....])`

```js
  var dbWithData = d.db_with(db,
helpers.entities_to_clj([
  [":db/add", 1, "name", "Ivan"],
  [":db/add", 1, "age", 17],
  {
    ":db/id": 2,
    "name": "Igor",
    "age": 35
  },
  [":db/add", 1, "friend", 2]
])
  );
```


Query all user name for users with age equals 17'
`parse` convert EDN string to mori structures. 
Also see [babel-plugin-datascript]() for compile-time converting

```js
var queryResponse = d.q(parse('[:find ?n :in $ ?a :where [?e "name" ?n] [?e "age" ?a]]'), dbWithData, 17)

isSet(queryResponse) // query response is Mori Set

set([vector("Ivan")])) // #{["Ivan"]}
```

query all user name and age

```js
var queryResponse = d.q(parse('[:find ?n ?a :where [?e "name" ?n] [?e "age" ?a]]'), dbWithData)

isSet(queryResponse) // query response is Mori Set

set([vector("Igor", 35), vector("Ivan", 17)])) // #{["Igor", 35] ["Ivan", 17]}
```

Create `conn` and use transact API

```js
  var scheme = hashMap(
    "friend", hashMap(
      DB_VALUE_TYPE, DB_TYPE_REF // or just keyword("db/valueType"), keyword("db.type/ref")
    )
  )
  var conn = d.create_conn(scheme)
  var reportsFromListen = []
  d.listen(conn, 'main', report => {  // or djs.listen(conn, "main", callback) is fully equal definition
    reportsFromListen.push(report);
  })

  var firstReport = d.transact(conn, vector(
    vector(DB_ADD, -1, "name", "Ivan"),
    vector(DB_ADD, -1, "age", 17)
  ))

  var secondReport = d.transact(conn, vector(
    hashMap(
      DB_ID, -1,
      "name", "Igor",
      "age", 35
    )
  ))
```

Reports equals callback argument and is mori datastucts

```js
reportsFromListen[0] // firstReport equal first callback arg of d.listen
reportsFromListen[1] // secondReport equal first callback arg of d.listen
isMap(secondReport) // reports is mori struct

var ivanId = getIn(firstReport, [TEMPIDS, -1])
var igorId = getIn(secondReport, [TEMPIDS, -1])

// resolve tempid from reports
ivanId === 1 // 'First Id === 1')
igorId === 2 // 'Second Id === 2')

d.transact(conn, vector(
    vector(DB_ADD, ivanId, "friend", igorId)
  ))
```

Query name friends of Igor from conn

```js
var result = d.q(parse('[:find ?n :in $ ?person :where [?e "name" ?n] [?e "friend" ?person]]'),
  djs.db(conn), igorId
)
isSet(result) //result of query is Mori Set
set([vector("Ivan")]) // 'result is #{["Ivan"]}'
  })
// or djs.unlisten(conn, "main", callback) is fully equal definition
d.unlisten(conn, "main")
```
