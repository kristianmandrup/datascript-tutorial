## CRUD apps and domain models

How can we achieve something akin to *CRUD* with Datomic/Datascript?
The *Pull API* is designed to be convenient for simple CRUD-like queries.

We can easily get one or more entities with specific attributes as a high performant batch query (using indexes).

[clj-crud](https://github.com/thegeez/clj-crud) sample app.

- [datascript, react web app](http://thegeez.net/2014/04/30/datascript_clojure_web_app.html)
- [datascript, quiescent, react app](http://thegeez.net/2014/05/01/datascript_quiescent_standalone.html)

### Create

Datascript/Datomic create and update entity data via "upserts" (update or insert), ie. facts are inserted (created) or updated. An "update" is simply a new commit with an add (create). If the entity does not exist, a new entity is created at that time. If an existing entity exists, a new version (in time) of that entity is created.

### Update

See "upserts" for Create.

### Delete

Datascript/Datomic use `retract`

### Read

For CRUD apps, the pull API should be used where possible as it is simpler and faster. Here some typical CRUD query examples:

Get entity (Person) by ID, such as `GET: persons/1`.

```clj
d/pull(['*'], [:person/id, 1])
```

We recommend leveraging *Lookup refs* to have an domain specific id, such as `:person/id`, like a primary (unique) key for each entity (see *Lookup refs* section in [Create Schema] chapter).

Using the entity id directly, it would be something like: `GET: persons/17235393939`.

```clj
d/pull(['*'], 17235393939)
```

Which doesn't quite look and feel right.

### Get specific attributes only

We can simply supply a list of attribute names to match as the first (pattern) argument.

```clj
d/pull(['name', 'age', 'status'], [:person/id, 1])
```

```clj
d/pull(['name', 'age', 'status'], [[:person/id, 1], [:person/id, 2], ...])
```

Get for all entities

```clj
d/pull(['name', 'age', 'status'])
```

Limit to return only #n number of entities (TODO: check Pull many API syntax)

```clj
d/pull(['name', 'age', 'status'], {:limit 100})
```

### Query by predicates

If we want to query by predicates we need to switch to using Query.
Here we query to return name, age and status for all people who are
female 35 and single.

```clj
d/q('[:find ?name ?age ?status],
      :where [[?e :person/age 35]
              [?e :person/status :single]
              [?e :person/gender :female]])
```

Note that the `d/q` does not allow use of `limit`, so to limit the entries returned, we need manually apply a limit filter (or buffer) to the array returned as a step after.

