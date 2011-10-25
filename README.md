# Automatic, comprehensive, event-driven CouchDB exploration

Probe CouchDB is a Javascript library which digs into every corner of a CouchDB server and fire events when it find interesting things: users, configs, databases, design documents, etc.

Probe CouchDB is available as an NPM module.

    $ npm install probe_couchdb

You can also install it globally (`npm install -g`) to get a simple `probe_couchdb` command-line tool.

## Is it any good?

Yes.

## Usage

Probe CouchDB is an event-emitter. Give it a URL and tell it to start.

```javascript
var probe_couchdb = require("probe_couchdb");

var url = "https://admin:secret@example.iriscouch.com";
var couch = new probe_couchdb.CouchDB(url);

couch.start();
```

Next, handle any events you are interested in.

```javascript
couch.on('db', function(db) {
  console.log('Found a database: ' + db.url);
  db.on('metadata', function(data) {
    console.log(db.name + ' has ' + data.doc_count + ' docs, using ' + (data.disk_size/1024) + 'KB on disk');
  })
})
```

<a name="api"></a>
## API Overview

This is the object hierarchy: **CouchDB** &rarr; **Database** &rarr; **Design document**

* A *CouchDB* explores the main server, including zero or more `db` events, containing a *Database* probe.
* A *Database* explores a database, including zero or more `ddoc` events, containing a *design document* probe.
* A *Design document* explores a design document.

### Common Events

All events pass one parameter to your callback unless otherwise noted.

* **start** | The probe is beginning its work; *zero callback arguments*
* **end** | The probe has finished its work; *zero callback arguments*
* **error** | An Error object indicating a problem. Databases re-emit all design document errors, and CouchDBs re-emit all database errors.

### Common properties

* **url** | The url to this resource (either a couch, a database, or a design doc)
* **log** | A log4js logger. Databases inherit the log from CouchDBs, design documents inherit the log from databases.

### Common methods

**request(options, callback)** | A [request][req] wrapper. Headers for JSON are set, and the response body is JSON-parsed automatically.

## CouchDB Probes

You create these using the API.

### Events

* **couchdb** | The server "Welcome" message (`/` response)
* **session** | The session with this server (`/_session` response). Check `.userCtx` to see your login and roles.
* **config** | The server configuration (`/_config` response). If you are not the admin, this will be `null`.
* **users** | Object with all user documents (from the `_users` database). Keys are the document IDs, values are the documents. Always includes a `null` key with the anonymous user.
* **db** | A *Database* probe. If you care about that database, subscribe to its events!

These events are used internally and less useful:

* **dbs** | An array of databases on this server
* **end_dbs** | Indicates that all databases have been processed

### Properties

* **only_dbs** | *Either* an array, to probe only specific databases, *or* a `function(db_name)` which returns whether to probe that database.
* **max_users** | Emit an error if the server has more users than this number.

### Methods

* **start()** | Start probing the server
* **anonymous_user()** | Helper function to produce an anonymous userCtx: `{"name":null, "roles":[]}`

## Database Probes

CouchDB probes pass database probes to your callback on the *db* event.

### Events

* **metadata** | The database metadata (`/db` response), or `null` if you haven't read permission
* **security** | The security object (`/db/_security` response), or `null` if you haven't read permission
* **ddoc** | A *design document* probe. If you care about that design document, subscribe to its events!

These events are used internally and less useful:

* **ddoc_ids** | An array of design document IDs
* **end_ddocs** | Indicates that all design documents have been processed

### Properties

* **couch** | The database's parent CouchDB probe
* **name** | The database's name

### Methods

**all_docs(options, callback)** | Run an `_all_docs` query. The *options* object (if given) is querystring parameters, e.g. `{"include_docs":true, startkey:["name", "S"]}`

## Design Document Probes

Database probes pass design document probes to your callback on the *ddoc* event.

### Events

* **body** | The design document, as a Javascript object
* **info** | The design document metadata info (`/db/_design/ddoc/_info` response)
* **language** | A string representing the language this design document uses. This is whatever the `.language` field in the document is. Usually this is `"javascript"`, or else `undefined` if it was not specified
* **view** | *Two callback arguments:* the view name (e.g. `"by_name"`), and then the view object (e.g. `{"map":"function(doc) { ... }"}`
* **code_error** | Indicates that a Javascript view has a error in its source code (either syntax, or nonstandard function signature); *four callback arguments:*
  1. The error object
  1. The name of the view in question, e.g. `"by_name"`
  1. The name of the function in question, e.g. `"map"` or `"reduce"`
  1. The function source code, e.g. `"function(doc) { ... }"`

These events are used internally and less useful:

**end_views** | Indicates that all views have been processed

### Properties

* **db** | The design document's parent database probe
* **id** | The design document's ID, e.g. `"_design/example"`

### Methods

No methods.

[req]: https://github.com/mikeal/request
