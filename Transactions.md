## Transactions
All writes to Datomic databases are protected by *ACID* transactions. Transactions are submitted to the system's *Transactor* component, which processes them serially and reflects changes out to all connected peers.

### Representation
Datomic represents transaction requests as data structures.
A transaction is simply a list of lists and/or maps, each of which is a statement in the transaction.

#### List forms
Each list a transaction contains represents either the addition or retraction of a specific fact about an entity, attribute, and value; or the invocation of a data function, as shown below.

Add transaction: `[:db/add entity-id attribute value]`

Retract transaction: `[:db/retract entity-id attribute value]`

#### Map forms
Each map a transaction contains is equivalent to a set of one or more `:db/add`operations. The map must include a `:db/id` key identifying the entity data is being added to (as described below). It may include any number of attribute, value pairs.

```clojure
{:db/id entity-id
 attribute value
 attribute value
 ... }
```

Internally, the *map structure is transformed to the list structure*. Each attribute, value pair becomes a `:db/add` list using the entity-id value associated with the `:db/id` key. So the above example is transformed to:

```clojure
[:db/add entity-id attribute value]
[:db/add entity-id attribute value]
...
```

The map structure is supported as a convenience when adding data.

### Identifying entities
All the statements in a transaction must specify the entity id they apply to. There are three possible values for an entity id:

- a *temporary id* for a new entity being added to the database
- an *existing id* for an entity that's already in the database
- an *identifier* (lookup ref) for an entity that's already in the database

#### Temporary ids
When you are adding data to a new entity, you identify it using a temporary id. Temporary ids get resolved to actual entity ids when a transaction is processed.

#### Making temporary ids
You can make a temporary id by calling the `datomic.Peer.tempid` method. The first argument to `Peer.tempid` is the name of the partition where the new entity will reside as an argument.

`temp_id = Peer.tempid(":db.part/user")`

Each call to tempid produces a unique temporary id.

There is an overloaded version of tempid that takes a *negative number* as a second argument. This version of tempid creates a temporary id based on the number you pass as input. I

Datascript only uses negative numbers for temp IDs, since there are no peers or partitions.

`[:db/id -1 ...]`

#### Temporary id resolution
When a transaction containing temporary ids is processed, each unique temporary id is mapped to an actual entity id. If a given temporary id is used more than once in a given transaction, all instances are mapped to the same actual entity id.

### Lookup refs
If the entity in question has a unique identifier, you can specify the entity id by using a *lookup ref*. Rather than querying the database, you can provide the unique attribute, value pair corresponding to the entity you want to assert (add) or retract a fact for. Note that a lookup ref specified in a transaction will be resolved by the transactor.

```clojure
;; Adding a customer using unique email as ID

{:db/id [:customer/email "joe@example.com"]
 :customer/status :active}
```

Using temp IDs to connect entities.

```clojure
[
 {:db/id #db/id[:db.part/user -1]
  :person/name "Bob"
  :person/spouse #db/id[:db.part/user -2]}
 {:db/id #db/id[:db.part/user -2]
  :person/name "Alice"
  :person/spouse #db/id[:db.part/user -1]}]
```

Here the `person/spouse` of Alice and Bob is used to reference each other using temp IDs (`-1`, `-2`) which are resolved to real entity IDs when the transaction is executed.

### Datascript transactions
Datascript comes with the following API transaction methods:

- `d/transact!`
- `:db/add`, `:db/retract`

```clojure
;; example

```

TODO: Describe more...

### Further reading

[Datomic transactions](http://docs.datomic.com/transactions.html)