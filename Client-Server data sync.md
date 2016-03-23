## Client/Server data sync

[datsync](https://github.com/metasoarous/datsync) is a library by [@metasaurus](https://twitter.com/metasoarous) which can be used synchronize the transactions logs of a Datascript client DB and Datomic server over a socket connection.

Datsync can be used with a server such as [FeathersJS](http://feathersjs.com/), using hooks to monitor/filter transactions being synced. This way we can control which clients get access to given data being synced across multiple client with variable authorization policies.

TODO: @metasaurus - notes and examples!

Only really works if we sync entire entities as per the CRUD model, not if we sync individual attributes.

### FeathersJs

Describe FeathersJs architecture and how it can solve our goal in a clean way, by acting as a cemtralized Sync controller between master and peers.

If a synced transaction fails on a peer, we can simply send the last (5?) transactions of that peer back to the originator of the failed transaction so that he can try to resolve the conflict. Conflict resolution is left up to the app (atomatic strategy or ask user).

### Local storage

Show how simple it is to save and load entire DB from local storage.

Acha-acha does this. Do the same for Todo.

Does it make sense to use Feathers for this as well, via [feathers-localstorage]() ??