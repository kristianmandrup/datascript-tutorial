## Adding data

Data in the Datascript database are facts in the form of entities with attributes. The attributes can be references to other entities and thus easily form relational/graph relationships between entities.

We can add data via a an existing connection or database. Typically this is done via transactions, using `d/transact!`

### Generating entity IDs

The special attribute `:db/id` is used to set the DB generated unique identifier for a given entity.

By passing `-1` as the value for `:db/id`, we indicate that it is a temporary ID and that the DB transactor should generate a real ID for us.

We can then use the `-1` temp ID value as an alias for that entity in further transactions, such as when we want to link other entities to this entity.

Note that the attributes `:name` and `:age` can be assigned the entity freely (schema less). We set `:aka` to the list `["Maks Otto von Stirlitz", "Jack Ryan"]` which is a valid value according to the schema (from Chapter 1) `:cardinality/many` rule.

```clojure
  (d/transact! conn [ { :db/id -1
                        :name  "Maksim"
                        :age   45
                        :aka   ["Maks Otto von Stirlitz", "Jack Ryan"] } ])
```

### Linking entities

Here an example of linking entities via the temp entity IDs.

```clojure
let [maksim {:db/id -1
             :name  "Maksim"
             :age   45
             :wife -2
             :aka   ["Maks Otto von Stirlitz", "Jack Ryan"]}
     anna   {:db/id -2
             :name  "Anna"
             :age   31
             :husband -1
             :aka   ["Anitzka"]}
     }

  (d/transact! conn [maksim anna])
```

Using the `-1` and `-2` temp IDs we have thus formed a bi-directional link between husband `"Maksim"` and wife `"Anna"`.

