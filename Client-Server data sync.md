## Client/Server data sync

[datasync]() is a library by [@metasaurus]() which can sync the transactions logs of a Datascript client DB and Datomic server.

This can be used with a server such as FeathersJS, with hooks to monitor and filter transactions being synced, thus controlling which clients can access given data being synced across multiple client with variable authorization policies.

TODO: @metasaurus - notes and examples!