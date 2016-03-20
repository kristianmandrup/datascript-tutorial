## Pull queries

Datascript and Datomic both come with a special *Pull API* for pulling entity data at a more coarse grained level. The Pull API can be seen as a sort of *simple query* that is more batch oriented.

The Pull API comes in two variants:
- Pull One
- Pull Many

### Select patterns

The Pull API uses patterns for selecing attributes.
The special pattern `'*'` fetches *all* attributes.

### Limits

`:limit` can be used to limit how many results are returned. This is useful for paging and similar use cases where you want to limit (or throttle) the data load sent to the user.

### Pull Many

Pull Many can be used to retrieve a set of attributes from a set of entities.
Thus the pull:

`(d/pull ['*'])`

Is equivalent to the SQL:

`"SELECT * FROM People"`

### Pull One

Pull one can be used to retrieve a set of attributes from a particular entity.
Thus the pull:

`(d/pull ['*', maksim-id])`

Is equivalent to the SQL:

`["SELECT * FROM People WHERE People.id = ?", maksim-id]`

### Usage with Query

In Datascript, Pull can be used in combination with Queries to great effect.

TODO: Example



