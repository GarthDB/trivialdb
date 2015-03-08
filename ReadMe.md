# TrivialDB

[![Build Status](https://travis-ci.org/Morgul/trivialdb.svg?branch=master)](https://travis-ci.org/Morgul/trivialdb)

A lightweight key/value json storage with persistence. Conceptually, it's just a thin API wrapper around plain javascript
objects; with the added bonus of doing throttled asynchronous writes on changes. Its on disk format is simply "json on
disk"; basically the json version of the plain object, saved to a file on disk. This makes making hand edits not
just possible, but simple.

## Use Case

TrivialDB is intended for simple storage needs. It's in-process, small, and very fast for small data sets. It takes almost
nothing to get up and going with it, and it has just enough features to make it worth while. Personally I've found its
a great fit for a development database for websites, or even to power a simple blog.

The one caveat to keep in mind is this: _every database your work with is stored in memory_. Since TrivialDB is in-process,
you might run into the memory limit of node; on versions before 0.11+ there's a 1.4GB limit. If you try and load a
database of all your cat pictures, you might run out of memory pretty quickly.

That being said, this isn't actually much of a limitation. Generally, you're working with a large amount of your data
in memory anyway; your data sets can get relatively large before you even need to worry about this.

## Installation

Simply install with npm:

```bash
$ npm install --save trivialdb:
```

## API

The TrivialDB API is inspired (spiritually) by [RethinkDB](http://rethinkdb.com/) and it's node.js ORM,
[thinky](http://thinky.io/). These are two great projects, and once you outgrow TrivialDB, I strongly encourage you to
check them out!

There are two different APIs, the low-level [Database API](#database-api), and the higher level [Model API](#model-api).
Previous versions of TrivialDB only had the Database API, so if you want to use what you're used to, feel free to keep using
it. (You can even use both together in various ways.) That being said, I strongly encourage you to check out the
[Model API](#model-api), as it's got some really nice validation features, and is a great way to work.

### Model API

The model API was added to make TrivialDB feel much more like the larger ORM style solutions out there. I'm a very big fan
of working with models as opposed to direct database calls, and I've taken some of my favorite features and baked them
directly into TrivialDB. Here's the current feature list:

* Model validation
* Primary Key support
* Simple model definitions
* Models serialize as JSON objects
* Get & Filter functions that return model instances
* Automatic syncing of database changes to Model instances
* Manual syncing of Model instances with the database

Don't worry, you are not required to use the Model API, or even know it's there.

#### Defining a model

* `defineModel(databaseName, modelDefinition, databaseOptions)` - Returns a `JDBModel`.

Defining a model in TrivialDB is very simple. Models and databases have a one to one relationship, so you can think of the
`databaseName` as the name of the model, though they don't have to have an relation to each other. As for the
`databaseOptions`, these are defined [below](#options), and give you the ability to pass any of those options along to
the underlying database.

_Note_: You will need to save the return value of `defineModel` and use that for querying or creating new instances.

```javascript
// Define a user model
var User = trivialdb.defineModel('users', {
    name: { type: String, required: true },
    age: Number,
    admin: { type: Boolean, required: true, default: false }
});

// Define an in-memory only model
var Session = trivialdb.defineModel('sessions', {
    userID: { type: String, required: true }
}, { writeToDisk: false });
```

##### Type Definition Object

The model definition is a simple object where the value is either a javascript type (String, Boolean, Number, etc.), or
a type definition object. The supported options are:

* `type` - Required. This must be a javascript type (String, Boolean, Number, etc).
* `required` - Optional. If false (the default), null and undefined are allowed values for this field.
* `default` - Optional. A value to default the field to. (This can easily be overwritten simply by assigning to the
field.)
* `choice` - Optional. A list of values that are valid for the field. (Arrays will have their contents individually
checked.)

##### Retrieving the Schema

You can retrieve the schema a model was defined with by using the `schema` property on the model. (Note: this property
becomes `$$schema` on model instances.)

```javascript
// Define a user model
var User = trivialdb.defineModel('users', {
    name: { type: String, required: true },
    age: Number,
    admin: { type: Boolean, required: true, default: false }
});

// Access the schema
console.log('Schema:', User.schema);

// Make a user instance
var user = new User({ name: "Foo", age: 23, admin: true });

// Access the schema
console.log('Schema:', user.$$schema);
```

This can be useful in applications that need to handle converting of user input to appropriate types, or the generation
of web forms based on the model.

If should be noted that all models have a read-only `id` field. If you do not specify a key this will be a generated
UUID.

#### Defining a Primary Key

Often times, a generic uuid is fine as the key for your model. However, sometimes it's more intuitive to make one of the
fields the primary key, instead. TrivialDB allows you to do this, simply by passing the `pk` option in to the database
options object, when you define a model.

```javascript
var User = trivialdb.defineModel('users', {
    name: { type: String, required: true },
    age: Number,
    admin: { type: Boolean, required: true, default: false }
}, { pk: 'name' });

// Make a user instance
var user = new User({ name: "Foo", age: 23, admin: true });

// The `id` property is equal to `name`:
console.log(user.id === user.name);
```

There are some interesting caveats about specifying primary keys. First and foremost: **TrivialDB will not ensure that your
keys are unique.** If you attempt to save a model whose primary key overwrites another value, TrivialDB will simply do what
you asked, and overwrite. The reason for this is that TrivialDB can't detect the difference between a new insert with a
non-unique primary key, and an update. If you are going to use primary keys, it is up to you to ensure uniqueness.

Additionally, when converting to JSON, the primary key will be duplicated in both the `id` property, and the property
specified as the primary key. This is by design; you can always refer to the `id` property as the identifier of the
object, regardless of if you're using a primary key. The small duplication is considered acceptable for the convenience.

#### Creating a Model

To create a model, you simply create a new instance of the return value of `defineModel`, optionally passing in an
option to populate the model with.

```javascript
// Create a regular user
var user = new User({ name: "Some Person", age: 23 });
```

#### Updating a model

Updating a model is as simple as modifying the model instance, and then calling save.

```javascript
// Create a regular user
var user = new User({ name: "Some Person", age: 23 });

// Make the user an admin
user.admin = true;

// Save the user
user.save();
```

#### Validating a Model

* `model.validate()` - Returns a promise that either resolves, or rejects with a `ValidationError`.

All model instances have a `validate` function, which you can use to check to make sure the model is valid. When there
is a validation error, you will get a `ValidationError` object with a `key` property that indicates the key that failed
validation. You can catch this error using `.catch`

Currently, we only validate based on type, or the type definition object's options.

```javascript
// Create a regular user
var user = new User({ name: "Some Person", age: 23 });

// Make the user an admin
user.admin = true;

// This will resolve correctly.
user.validate().then(function()
{
    // We get here.
});

// Set the admin field to an invalid type
user.admin = 'not valid';

// This will throw an error.
user.validate()
    .then(function()
    {
        // We don't get here.
    })
    .catch(trivialdb.errors.ValidationError, function()
    {
        // We catch the error, and handle it here.
    });
```

#### Saving a Model

* `model.save(skipValidation)` - Returns a promise that either resolves, or rejects with an error.

Saving a model is as simple as calling `save`. It first calls `validate`, which means that it may throw a
`ValidationError`.

```javascript
// Create a regular user
var user = new User({ name: "Some Person", age: 23 });

// Make the user an admin
user.admin = true;

// Save the user
user.save()
    .then(function()
    {
        // Saved!
    });
```

#### Removing a Model

* `model.remove()` - Returns a promise that always resolves.

Removing a model is as simple as calling `remove`. The object is removed from the database, and it's values are emptied.

```javascript
// Get a user
User.get('some-id')
    .then(function(user)
    {
        return user.remove();
    });
```

Models can also be removed via a filter:

```javascript
// Remove all admins
User.remove({ admin: true });
```

#### Removing all Model Instances

* `Model.removeAll()` - returns a promise that resolves once the sync has completed.

This is provided as a convenience. It removes all instances of a model.

#### Syncing

Normally, in an ORM, if you either create a new model instance, or get a model instance somehow, and something else
changes the database, your model instance is out of date and doesn't get those changes. However, because of the nature
of TrivialDB, it was very easy to make the models smart enough to update themselves whenever something changes in the
database.

That being said, there is an issue of how do we merge your unsaved changes when something else has updated the database?
The design we've chosen is that whenever you model is dirty (`model.$dirty` is true), we ignore the `sync` event. You
can still manually sync the object.

##### Manual Syncing

* `model.sync(force)` - returns a promise that resolves once the sync has completed.

If you want to sync your model instance (because you manually updated the database, for example), you can simply call
`sync`. However, if your model is dirty, you will need to force the sync by passing `true` to the `sync` function.

```javascript
// Create a regular user
var user = new User({ name: "Some Person", age: 23 });

// Make the user an admin
user.admin = true;

// Force a sync
user.sync(true)
    .then(function()
    {
        // user.admin is false here.
    });
```

Keep in mind, when you force a sync, _all changes to the model instance will be lost_.

#### Getting a Model by ID

* `Model.get(modelID)` - returns a promise that either resolves with a model instance, ore rejects with a
`DocumentNotFound` error.

You can easily get model instances by id. Simply call `Model.get` and pass in the id you're looking for. If the id does
not exist, the promise will be rejected with a `DocumentNotFound` error. You can easily test for this, and handle it
using `.catch`.

```javascript
// Get an existing id
User.get('existing-id')
    .then(function(user)
    {
        // Work with user here
    });

// Get a non-existent id
User.get('existing-id')
    .catch(trivialdb.errors.DocumentNotFound, function(error)
    {
        // Handle not found case
    });
```

#### Getting all Models

* `Model.all()` - returns a promise that resolves to a list, or an error.

This is a nice shortcut to getting all instances of the Model.

```javascript
User.all()
    .then(function(users)
    {
        // Work with all users here
    });
```

#### Filtering models

* `Model.filter(filter)` - returns a promise that resolves to a list, or an error.

Instead, if you want to get a list of possible values, you should use `filter`. Just like the Database API function of
the same name, we support filter functions, `_.pluck` and `_.where` style filters. The results are _always_ as list, so
if nothing matched your filter, you will get back an empty list.

```javascript
// Get a list of admins
User.filter({ admin: true })
    .then(function(admins)
    {
        // Work with the list of admins.
    });
```

#### Working with the database

You can work with the database object for a particular model by using the `$$db` object. While I don't recommend doing
things this way, should you need it, you can use it.

### Database API

This is the API that previous versions of TrivialDB pioneered. It's relatively low-level, and if that's how you'd rather
work with your database, that's fine. It is still the primary focus of TrivialDB.

#### **New in 1.0.0**

The Database API has recently changed. As I've been building projects, I've discovered that Promise-based APIs are both elegant
and incredibly convenient. I have opted to make the almost entire API promise-based. This has a small performance hit on
individual operations, however, it also makes all calls asynchronous, which helps with TrivialDB's ability to handle load.

If you do not like promises, or disagree with this change, then I recommend using
[v0.9.0](https://github.com/Morgul/trivialdb/releases/tag/v0.9.0).

##### Promises

Because of the change to promises, TrivialDB now exposes our internal Promise object as `trivialdb.Promise`, so you can
leverage it  if you want to. (We use [bluebird](https://github.com/petkaantonov/bluebird).)

```javascript
var trivialdb = require('trivialdb');
var Promise = trivialdb.Promise;

// Work with `Promise` here
```

#### Loading or saving databases

* `db(databaseName, options)` - Returns a database instance.

TrivialDB lazily loads databases. TrivialDB also creates databases if they don't exist. To load or create a database:

```javascript
// Open or create a database
var db = trivialdb.db('some_db');

// Open or create a database, with options
var db = trivialdb.db('some_db', { writeToDisk: false });
```

This will look for a file named `"./some_db.json"`. (If your database lives somewhere else, you can pass the `rootPath`
option in to the `db` call.)

##### New in 1.0.0

You can now request the same database multiple times, and get back the same instance. This allows you to request the
database by name in different places in your code, and not worry about the two database instance fighting with each
other. (The previous behavior was clearly broken, and resulted in very strange issues.)

_Note_: When you request a database any other time than the first, the options are ignored. There is currently no way to
change a database's options at run time.

##### Options

The options supported by the `db` call are:

```javascript
{
    writeToDisk: true | false,  // Whether or not to persist the database to disk. (Default: `true`)
    loadFromDisk: true | false, // Whether or not to read the database in from disk on load. (Default: `true`)
    rootPath: "...",            // The path to a folder that will contain the persisted database json files. (Default: './')
    writeDelay: ...,            // A number in milliseconds to wait between writes to the disk. (Default: 0)
    prettyPrint: true | false,  // Whether or not the json on disk should be pretty printed. (Default: `true`)
    pk: "...",                  // The field in the object to use as the primary key. (Default: `undefined`)
    idFunc: function(){...}     // The function to use to generate unique ids. (Default: `uuid.v4()`)
}
```

##### Custom ID Generation

If you want to generate your own ids, and not use the uuids TrivialDB generates by default, you can specify your own
function in the database options. By specifying `idFunc`, TrivialDB will use this function to generate all ids, when needed.
The `idFunc` function is passed the object, so you can generate ids based on the object's content, if you wish. (An
example of this would be generating a slug from an article's name.)

```javascript
function slugify(article)
{
    return article.name.toString().toLowerCase()
        .replace(/\s+/g, '-')
        .replace(/[^\w\-]+/g, '')
        .replace(/\-\-+/g, '-')
        .replace(/^-+/, '')
        .replace(/-+$/, '');
} // end slugify

// Declare a new database, using the slugify function above.
db = new JDB("articles", { writeToDisk: false, idFunc: slugify });

// Now, we save an object
db.store({ name: "TrivialDB: now with id generation functions!", body: "Read the title, dude." })
    .then(function(id)
    {
        // This prints the id: 'trivialdb-now-with-id-generation-functions'.
        console.log('id:', id);
    });
```

Be careful; it is up to you to ensure your generated ids are unique. Additionally, if your generation function blows up,
TrivialDB may return some nonsensical errors. (This may improve in the future.)

#### Storing Values

* `store(value)` - Returns a promise resolved with `key`.
* `store(key, value)` - Returns a promise resolved with `key`.

Since TrivialDB is a key/value storage, all values are stored under a key. This key is not part of the value that gets stored,
since TrivialDB never modifies your value. Also, while you can specify a key, you will need to ensure it's unique (otherwise
it will silently overwrite). Instead, I recommend you let TrivialDB create the key for you (by not passing one).

When you let TrivialDB auto generate the key, you can find out what that key was by using `.then()`, which will be passed
the newly generated key. This auto generation is done using the `idFunc` function passed in the options. If not
specified, it will use `node-uuid` to generate uuids.

```javascript
// Store an object
db.store({ foo: "bar!", test: "Apples" })
    .then(function(key)
    {
        // Work with `key` here
    };

var key = undefined
// We support auto generating keys whenever the key parameter is undefined.
db.store(key, { foo: "bar!", test: "Apples" })
    .then(function(key)
    {
        // Work with `key` here
    };

// Store an object with key
db.store('my_key', { foo: "bar!", test: "Apples" })
    .then(function(key)
    {
        // `key` == 'my_key'
    };
```

#### Retrieving Values

* `get(key)` - Returns a promise resolved to the value or `undefined`.

TrivialDB only supports direct lookups by key. It returns a promise resolved to the value stored.

```javascript
// Get an object
db.get('my_key')
    .then(function(val)
    {
        // Work with `val` here
    });
```

#### Updating Values

* `merge(key, partialObj)` - Returns a promise resolved to the new value.

TrivialDB support partial object updates. TrivialDB will take whatever object you pass in, and merge that object with the value
stored at that key. If there is no value, it works exactly like `store`. The resulting object is returned.

```javascript
// Update an object
db.merge('my_key', { test: "Oranges" })
    .then(function(obj)
    {
        // Work with `obj` here
    });

```

#### Filter Queries

* `filter(filter)` - Returns a promise resolved to an object of filtered values.

Sometimes, you need to query based on more than just the key. To do that, TrivialDB gives you a very simple filter query. It
iterates over every value in the database, and passes that into your filter function. If the function returns true, that
value is included in the results, otherwise it's omitted.

```javascript
// Filter Function
db.filter(function(value, key)
{
    // Decide if you want this object
    return value.foo === 'bar!';
}).then(function(results)
{
    // Work with `results` here.
});
```

##### New in 1.0.0

You can now also pass in filter objects. We switched to using lodash under the hood, so we support their `_.pluck` &
`_.where` style callbacks as well!

```javascript
// Filter object
db.filter({ foo: 'bar!' })
    .then(function(results)
    {
        // Work with `results` here.
    });
```

#### Direct Access

* `sync()` - Returns a promise resolved once the database is considered 'settled'.

You can directly access the key/value store with the `values` property on the database instance. This is exposed
explicitly to allow you as much freedom to work with your data as you might want. However, TrivialDB can't detect any
changes you make directly, so you will need to call the `sync` function to get your changes to persist to disk.

```javascript
// Add a new key manually
db.values['foobar'] = { test: "something" };

// Sync that new key to disk
db.sync();
```

The `sync` function returns a promise that is resolved once the database has 'settled', as in, there are not more
scheduled writes. Because of this behavior, you should consider whether or not you want to wait on it's promise. Under
high load, (or with a high `writeDelay`) it's possible for a `sync` promise's resolution to be considerably delayed.

```javascript
// Add a new key manually
db.values['foobar'] = { test: "something" };

// Sync that new key to disk
db.sync()
    .then(function()
    {
        // Sync is done, db is settled
    });
```

Also, you should feel free to iterate over the values object if you need to do any advanced filtering. All the same
caveats of working with a plain javascript object apply. Just remember to call `sync` if you've made any modifications.

##### New in 1.0.0

Whenever `store` or `merge` are called, a `sync` event is fired from the database object. You can use this should you
need to know when TrivialDB is syncing to disk.

## Status

TrivialDB is reasonably stable, and since the code base is small enough, it's relatively immune to the most common forms
of 'code rot'. I make improvements when they're needed, or if someone files an issue. That being said, I consider
TrivialDB production ready, provided you meet the intended use case.

## Contributing

While I only work on TrivialDB in my spare time (what little there is), I use it for several of my projects. I'm more than
happy to accept merge requests, and/or any issues filed. If you want to fork it and improve part of the API, I'm ok with
that too, however I ask you open an issue to discuss your proposed changes _first_. And, since it's MIT licensed, you
can, of course, take the code and use it in your own projects.