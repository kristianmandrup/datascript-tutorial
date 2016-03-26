# Schema creation

The facts that a Datomic/Datascript database stores are represented by datoms. Each datom is an addition or retraction of a relation between an entity, an attribute, a value, and a transaction. The set of possible attributes a datom can specify is defined by a database's schema.

Each database has a schema that describes the set of attributes that can be associated with entities. A schema only defines the characteristics of the attributes themselves. It does not define which attributes can be associated with which entities. Decisions about which attributes apply to which entities are made by an application.

## Attributes

Schema attributes are defined using the same data model used for application data. That is, attributes are themselves entities with associated attributes. Datomic defines a set of built-in system attributes that are used to define new attributes.

[Datomic schemas](http://docs.datomic.com/schema.html) require the following attributes:
- `:db/ident` Attribute Identifier (name)
- `:db/valueType` Value type
- `:db/cardinality` Cardinality

### Attribute ID

`:db/ident` specifies the unique name of an attribute. It's value is a namespaced keyword with the lexical form `:<namespace>/<name>`. 

It is possible to define a name without a namespace, as in `:<name>`, but a namespace is preferred in order to avoid naming collisions. 

*Namespaces* can be hierarchical, with segments separated by ".", as in `:<namespace>.<nested-namespace>/<name>`. The `:db` *namespace* is reserved for use by Datomic itself.

### Value types

The Datomic value types map to Java equivalents.

*Basic*

- `:db.type/keyword` such as `:color`
- `:db.type/string` "Hello"
- `:db.type/boolean` true|false
- `:db.type/ref` entity reference (entity id)
- `:db.type/instant` time in milliseconds

*Numbers*

- `:db.type/long` Long integer
- `:db.type/bigint` Big integer

- `:db.type/float` Float
- `:db.type/double` Double
- `:db.type/bigdec` Big Decimal

*Special*

- `:db.type/uuid` UUIDs (ie. unique IDs)
- `:db.type/uri` URIs
- `:db.type/bytes` binary data

### Cardinality

- `:db.cardinality/one` Reference ONE value
- `:db.cardinality/many` Reference MANY values

A value can be either a literal such as 3 or "Hello" or a reference to an entity via its entity ID.

### Optional schema attributes

In addition to these three required attributes, there are several optional attributes that can be associated with an attribute definitions:

- `:db/doc` documentation string.
- `:db/unique` uniqueness constraint for the values of an attribute. Also implies `:db/index`. Set to `:db.unique/value ` or `:db.unique/identity`
- `:db/index` to index this attribute (boolean)
- `:db/fulltext` fulltext search index (boolean) (Datomic only)
- `:db/isComponent` indicates that `db.type/ref` refers to a subcomponent of the entity and should apply cascading (recursive) retraction (boolean).
- `:db/noHistory` keep history or not (boolean)

All booleans are by default `false` (ie. not enabled).

## Datascript schemas

Datascript schemas are a little different.
We can define a schema using a `Map` as follows:

`{:aka {:db/cardinality :db.cardinality/many}`

Here `:aka` is the name of the entity, similar to `:db/ident`.
`{:db/cardinality :db.cardinality/many}` creates a constraint that `:aka` can
refer to multiple values.

Notice that `:aka` doesn't have a value type as required by Datomic. This is because we are in Javascript land and things are a bit more flexible and "type less" ;) You can still add the value type if you like!

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

`:db/ident` is just a regular attribute, but we can give it "special powers" as per our own conventions if we choose.

### Lookup refs

In many databases, entities have unique identifiers from the problem domain like an email address or an order number. Applications often need to find entities based on these external keys. You can do this with query, but it's easier to use a *lookup ref* such as:

`[:person/email "joe@example.com"]`

You can also use lookup refs to refer to existing entities, such as in:

```clojure
{:db/id [:person/email "joe@example.com"]
 :person/loves :pizza}
```

Lookup refs have the following restrictions:

- The specified attribute must be defined as either `:db.unique/value` or `:db.unique/identity`.

- Lookup refs cannot be used in the body of a query. (Datomic constraint only?)

Datomic performs automatic resolution of entity identifiers, so that you can generally use entity ids, idents, and lookup refs interchangeably.

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
(def schema {:person/email {:db/unique :db.unique/identity}})
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

## Schema Alteration
Because Datomic maintains a *single set of physical indexes*, and supports query across time, a db value utilizes the single schema associated with its basis, i.e. before any call to asOf/since/history, for all views of that db (including `asOf` `since` and `history`). Thus traveling back in time does not take the working schema back in time, as the infrastructure to support it may no longer exist.

You can only rename the `:db/ident` or alter certain schema attributes of attributes.

### Renaming an Identity
To rename a `:db/ident`, submit a transaction with the `:db/id` and the value of the new `:db/ident`.

The supported attributes to alter include `:db/cardinality`, `:db/isComponent`, `:db/noHistory`, `:db/index` and `:db/unique`. You can NOT alter `:db/valueType` or `:db/fulltext`.

Alters the `:db/ident` of `:person/name` to be `:person/full-name`.

```clojure
[{:db/id :person/name
  :db/ident :person/full-name}]
```
