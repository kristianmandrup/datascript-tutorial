# Datomic Architecture

## The players

- Transactor
- ...

TODO: Show diagram(s)

### Transactor

...

### Partitions

All entities created in a database reside within a partition. Partitions group data together, providing locality of reference when executing queries across a collection of entities. In general, you want to group entities based on how you'll use them.

The following partitions are built into Datomic.

- `:db.part/db` System partition, used for schema
- `:db.part/tx` Transaction partition
- `:db.part/user`   User partition, for application entities

Use `:db.part/user` for your own application entities (domain models).

#### Creating new partitions
A partition is represented by an entity with a :db/ident attribute, as shown below. A partition can be referred to by either its entity id or ident.

```clojure
{:db/id #db/id[:db.part/db]
 :db/ident :communities}
```

To install the `:communities` partition.

```clojure
[{:db/id #db/id[:db.part/db -1]
  :db/ident :communities}
 [:db/add :db.part/db :db.install/partition #db/id[:db.part/db -1]]]
```

