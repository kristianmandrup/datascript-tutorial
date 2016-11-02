# Datascript

Datascript is based on the Datomic DB created by Cognitect.

### Datalog

Datomic's query and rules system is an extended form of Datalog. Datalog is a deductive query system, typically consisting of:

- A database of facts
- A set of rules for deriving new facts from existing facts
- A query processor that, given some partial specification of a fact or rule:
finds all instances of that specification implied by the database and rules
i.e. all the matching facts

### Data model

The data model in Datascript/Datomic is based around atomic facts called datoms. A datom is a 4-tuple consisting of:

- Entity ID
- Attribute
- Value
- Transaction ID

Example:

```
[<e-id>  <attribute>      <value>          <tx-id>]
...
[ 167    :person/id       168373838          102  ]
[ 167    :person/name     "James Cameron"    102  ]
...
[ 234    :movie/id        173532083          102  ]
[ 234    :movie/title     "Die Hard"         103  ]
[ 234    :movie/year      1987               103  ]
```

The following query finds all entity-ids that have the attribute `:person/name`with a value of `"Ridley Scott"`:

```clojure
[:find ?e
 :where
 [?e :person/name "Ridley Scott"]]
```

### Useful resources

[Learn datalog queries](http://www.learndatalogtoday.org/)