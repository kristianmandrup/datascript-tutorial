# Entity identity

Any entity needs an identity to uniquely identify it, such that the entity can later be retrieved by id.

## External keys

All entities in a database have an *internal key*, the *entity id*. You can use `:db/unique` to define an attribute to represent an external key.
An entity may have any number of external keys.
External keys must be single attributes.

[Datomic Identity](http://docs.datomic.com/identity.html)

Datomic/Datascript provides a number of ways to model identity and uniqueness.

- All entities are given database-unique entity ids.
- *Unique identities* allow transactions to work with domain keys instead of entity ids.
- *Unique values* enforce a single holder of an identifying key.
- *Lookup Refs* represent a lookup on a unique attribute as an attribute, value pair.

*Datomic only*

- *Idents* can be used as *name identifiers*
- *Squuids* provide efficient, *globally unique identifiers*

## Lookup refs

In many databases, entities have unique identifiers from the problem domain like an `email` address or an `order` number. Applications often need to find entities based on these external keys. You can do this with query, but it's easier to use a *lookup ref* such as:

`[:person/email "joe@example.com"]`

You can also use lookup refs to refer to existing entities, such as in:

```clojure
{:db/id [:person/email "joe@example.com"]
 :person/loves :pizza}
```

Lookup refs have the following restrictions:

- The specified attribute must be defined as either `:db.unique/value` or `:db.unique/identity`.

- Lookup refs cannot be used in the body of a query. *(Datomic only?)*

Datomic performs *automatic resolution of entity identifiers*, so that you can generally use entity ids, idents, and lookup refs interchangeably.

For example, the following family of queries all locate Belgium and return the same results:

```clojure
;; query
[:find ?artist-name
 :in $ ?country
 :where [?artist :artist/name ?artist-name]
        [?artist :artist/country ?country]]

;; input option 1: via lookup ref
db, [:country/name "Belgium"]

;; input option 2: via ident
db, :country/BE

;; input option 3: via entity id
db, 17592186045516
```

### Identity via db/ident

We can use `:db/ident` as a convention for *lookup refs*, a more semantic kind of entity ids.

First we must set the schema of `:db/ident` to be a `:db/unique` of `:db.unique/identity` to ensure the attribute must have unique values.
This is often useful for attributes like `:person/email` or `:person/phone`. We could then reuse the same values in similar attributes such as `:company/phone` without conflict.

We can similarly use `:db.unique/value` for identity attributes that should be guaranteed to be unique cross the entire DB, such as Social security or Passport numbers.

```clojure
;; registering in schema
(def schema {:db/ident {:db/unique :db.unique/identity}})
```

We can now add an entity with `:db/ident` set to a unique key of our choosing

```
;; specifying ident on an entity
(d/transact! conn [[:db/add <eid> :db/ident 'my-unique-key']])
```

Then look up an entity by `:db/ident` value

```
(d/entity @conn [:db/ident 'my-unique-key'])
```

Add a new entity using the lookup ref `'my-unique-key'` as an implicit entity id.

```
(d/transact! conn [[:db/add [:db/ident 'my-unique-key'] <attr> <value>]])
```

Then find entity attributes `?a` and values `?v` given the lookup ref.

```
(d/q '[:find ?a ?v
       :where [[:db/ident 'my-unique-key'] ?a ?v]]
     @conn)
```

You can learn more about Datomic Identity rules [here](http://docs.datomic.com/identity.html). Most of it should apply to Datascript as well (except for `:db/ident`)

### Unique identities

Unique identity is specified through an attribute with `:db/unique` set to `:db.unique/identity`

A *unique identity* attribute is always indexed by value, in order to support uniqueness checks. Specifying `:db/index true` is *redundant* and not recommended.

Uniqueness checks are *per-attribute*, and do not prevent you from using the same value with a different attribute elsewhere.

*Unique value* is specified through an attribute with `:db/unique` set to `:db.unique/value`. A unique value represents a *database-wide value* that can be asserted only once.