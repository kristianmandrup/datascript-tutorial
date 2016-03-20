## Schema creation

In Datascript we can define a schema using Hash as follows:

`{:aka {:db/cardinality :db.cardinality/many}`

Here `:aka` is the name of the entity.
`{:db/cardinality :db.cardinality/many}` creates a constraint that `:aka` can
refer to multiple values.

Datomic uses a special attribute `:db/ident` to identify attributes.
In this (Dartomic) example we define an attribute named `:person/name` of type `:db.type/string` with `:db.cardinality/one` that is intended to capture a person's name.

```clojure
{:db/id #db/id[:db.part/db]
 :db/ident :person/name
 :db/valueType :db.type/string
 :db/cardinality :db.cardinality/one
 :db/doc "A person's name"
 :db.install/_attribute :db.part/db}
```

In Datascript we can simply use the nested Hash form: `{:aka { <schema> }` as demonstrated above. This Datomic example in Datascript would be simplified to:

```
{:person/name {
  :db/valueType :db.type/string
  :db/cardinality :db.cardinality/one
  :db/doc "A person's name"
}}
```

We can however still use `:db/ident` as a regular attribute but give it special powers as per our own conventions.

### Identities

We can use `:db/ident` as a convention for lookup refs.
First we must set the schema of `:db/ident` to be a `:db/unique` of `:db.unique/identity` to ensure it is unique across the entire DB.

We can similarly use `:db.unique/value` for identity attributes that should only be guaranteed to be unique for the given attribute, such as `:person/email` or `:person/phone`. We could then reuse the same value in a similar (but different attribute) such as `:company/phone`.

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
