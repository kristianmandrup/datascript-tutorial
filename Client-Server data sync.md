## Client/Server data sync

[datsync](https://github.com/metasoarous/datsync) is a library by [@metasaurus](https://twitter.com/metasoarous) which can be used synchronize the transactions logs of a Datascript client DB and Datomic server over a socket connection.

Datsync can be used with a server such as [FeathersJS](http://feathersjs.com/), using hooks to monitor/filter transactions being synced. This way we can control which clients get access to given data being synced across multiple client with variable authorization policies.

TODO: @metasaurus - notes and examples!