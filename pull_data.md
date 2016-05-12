## Pull

Datascript and Datomic both come with a special *Pull API* for pulling entity data at a more coarse grained level. The Pull API can be seen as a sort of *simple query* that is more batch oriented.

The Pull API comes in two variants:
- Pull One
- Pull Many

### Select patterns

The Pull API uses patterns for selecing attributes.
The special pattern `'*'` fetches *all* attributes.

### Limits

`:limit` can be used to limit how many results are returned. This is useful for paging and similar use cases where you want to limit (or throttle) the data load sent to the user.

### Pull Many

Pull Many can be used to retrieve a set of attributes from a set of entities.

SQL: `"SELECT * FROM People"`

`(d/pull [db '*'])`

#### Selecting specific attributes

`"SELECT name, age FROM People"`

`(d/pull [db [:name :age]])`

### Pull One

Pull one can be used to retrieve a set of attributes from a particular entity.
SQL: `["SELECT * FROM People WHERE People.id = ?", maksim-id]`

`(d/pull [db '*' maksim-id])`

#### Selecting specific attributes

`(d/pull [db [:name :age] maksim-id)`

### Fetch entity by ID

`entity(db eid)`

### Pull and Query combined

In Datascript, Pull can be used in combination with Query to great effect.
Let's say we want to extract all entity data of all entities.

`(d/pull-many db '[*] (d/q '[:find ?e :where [?e]]))`

The `d/q` will match all entity ids in the db
Then `pull-many` pulls everything for all of them

We can f.ex also use a pull to find attributes to be used in a `:find` clause.
Here we use `pull` on the `:aliases` attribute value `?a` in the `:where` clause to pull only `:name` and `:order` values, which will be used as the attributes to `find`.

```clj
;; query
(->>
  (d/q '[:find [(pull ?a [:name :order]) ...]
         :in   $ ?e
         :where [?e :aliases ?a]
       db eid)
  (sort-by :order)
  (map :name))
```

We can then sort and map the query result independently (and/or include limit/paging logic).
