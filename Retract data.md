## Retract data

Datascript/Datomic are immutable data stores (the default for any Clojure value). Thus values can only be added but never deleted, just like your code in version control.
The DB operates in value oriented fashion, where any value is time dependent.
Values flow and change over time, and the database keeps track of this value progression (maintains full history). The DB thus commits values much like a version control system such as git.

Facts can be "retracted", which means they are no longer visible from this point on but only available at any time point before the retraction. 
This works just like when you delete files from your git repo. You can still go back and find (recover) them in older commits.

