# document-managment-system-api

## Documentation

The system manages documents, users and user roles. Each document defines access rights; the document defines which roles can access it. Also, each document specifies the date it was published.
Users are categorized by roles. Each user must have a role defined for them.

## Installation

* First install [node.js](http://nodejs.org/) and [mongodb](https://www.mongodb.org/downloads).

## Database Setup

Either [MongoDB Compass](https://docs.mongodb.com/compass/master/install/) or [Mongo Atlas](https://www.mongodb.com/cloud/atlas) can be used to store data. Follow the link to set them up.

## Setup
* Open up  the terminal in the root directory of the clone repository. Then run

```sh
$ npm install
```
* create a .env file in the root directory. Copy and paste the contents in .env_example file. Define the variables

```sh
PORT= 'the port on which to run the app eg. 5000'
NODE_ENV=development
LOCAL_DATABASE=mongodb://localhost/document_management_system
TEST_DATABASE=mongodb://localhost/document_management_system_tests
REMOTE_DATABASE=mongodb+srv://<username>:<password>@<name of remote database, atlas>db-ua8ev.mongodb.net/test?retryWrites=true
JWT_PRIVATE_KEY=<define a secret key eg. HELLO>
```


## Overview

### Connecting to MongoDB

First, we need to define a connection. If your app uses only one database, you should use `mongoose.connect`. If you need to create additional connections, use `mongoose.createConnection`.

Both `connect` and `createConnection` take a `mongodb://` URI, or the parameters `host, database, port, options`.

```js
const mongoose = require('mongoose');

mongoose.connect('mongodb://localhost/my_database', {useNewUrlParser: true});
```

Once connected, the `open` event is fired on the `Connection` instance. If you're using `mongoose.connect`, the `Connection` is `mongoose.connection`. Otherwise, `mongoose.createConnection` return value is a `Connection`.

**Note:** _If the local connection fails then try using 127.0.0.1 instead of localhost. Sometimes issues may arise when the local hostname has been changed._

**Important!** Mongoose buffers all the commands until it's connected to the database. This means that you don't have to wait until it connects to MongoDB in order to define models, run queries, etc.

### Defining a Model

Models are defined through the `Schema` interface.

```js
const Schema = mongoose.Schema;
const ObjectId = Schema.ObjectId;

const BlogPost = new Schema({
  author: ObjectId,
  title: String,
  body: String,
  date: Date
});
```

Aside from defining the structure of your documents and the types of data you're storing, a Schema handles the definition of:

* [Validators](http://mongoosejs.com/docs/validation.html) (async and sync)
* [Defaults](http://mongoosejs.com/docs/api.html#schematype_SchemaType-default)
* [Getters](http://mongoosejs.com/docs/api.html#schematype_SchemaType-get)
* [Setters](http://mongoosejs.com/docs/api.html#schematype_SchemaType-set)
* [Indexes](http://mongoosejs.com/docs/guide.html#indexes)
* [Middleware](http://mongoosejs.com/docs/middleware.html)
* [Methods](http://mongoosejs.com/docs/guide.html#methods) definition
* [Statics](http://mongoosejs.com/docs/guide.html#statics) definition
* [Plugins](http://mongoosejs.com/docs/plugins.html)
* [pseudo-JOINs](http://mongoosejs.com/docs/populate.html)

The following example shows some of these features:

```js
const Comment = new Schema({
  name: { type: String, default: 'hahaha' },
  age: { type: Number, min: 18, index: true },
  bio: { type: String, match: /[a-z]/ },
  date: { type: Date, default: Date.now },
  buff: Buffer
});

// a setter
Comment.path('name').set(function (v) {
  return capitalize(v);
});

// middleware
Comment.pre('save', function (next) {
  notify(this.get('email'));
  next();
});
```

Take a look at the example in `examples/schema.js` for an end-to-end example of a typical setup.

### Accessing a Model

Once we define a model through `mongoose.model('ModelName', mySchema)`, we can access it through the same function

```js
const myModel = mongoose.model('ModelName');
```

Or just do it all at once

```js
const MyModel = mongoose.model('ModelName', mySchema);
```

The first argument is the _singular_ name of the collection your model is for. **Mongoose automatically looks for the _plural_ version of your model name.** For example, if you use

```js
const MyModel = mongoose.model('Ticket', mySchema);
```

Then Mongoose will create the model for your __tickets__ collection, not your __ticket__ collection.

Once we have our model, we can then instantiate it, and save it:

```js
const instance = new MyModel();
instance.my.key = 'hello';
instance.save(function (err) {
  //
});
```

Or we can find documents from the same collection

```js
MyModel.find({}, function (err, docs) {
  // docs.forEach
});
```

You can also `findOne`, `findById`, `update`, etc. For more details check out [the docs](http://mongoosejs.com/docs/queries.html).

**Important!** If you opened a separate connection using `mongoose.createConnection()` but attempt to access the model through `mongoose.model('ModelName')` it will not work as expected since it is not hooked up to an active db connection. In this case access your model through the connection you created:

```js
const conn = mongoose.createConnection('your connection string');
const MyModel = conn.model('ModelName', schema);
const m = new MyModel;
m.save(); // works
```

vs

```js
const conn = mongoose.createConnection('your connection string');
const MyModel = mongoose.model('ModelName', schema);
const m = new MyModel;
m.save(); // does not work b/c the default connection object was never connected
```

### Embedded Documents

In the first example snippet, we defined a key in the Schema that looks like:

```
comments: [Comment]
```

Where `Comment` is a `Schema` we created. This means that creating embedded documents is as simple as:

```js
// retrieve my model
var BlogPost = mongoose.model('BlogPost');

// create a blog post
var post = new BlogPost();

// create a comment
post.comments.push({ title: 'My comment' });

post.save(function (err) {
  if (!err) console.log('Success!');
});
```

The same goes for removing them:

```js
BlogPost.findById(myId, function (err, post) {
  if (!err) {
    post.comments[0].remove();
    post.save(function (err) {
      // do something
    });
  }
});
```

Embedded documents enjoy all the same features as your models. Defaults, validators, middleware. Whenever an error occurs, it's bubbled to the `save()` error callback, so error handling is a snap!


### Middleware

See the [docs](http://mongoosejs.com/docs/middleware.html) page.

#### Intercepting and mutating method arguments

You can intercept method arguments via middleware.

For example, this would allow you to broadcast changes about your Documents every time someone `set`s a path in your Document to a new value:

```js
schema.pre('set', function (next, path, val, typel) {
  // `this` is the current Document
  this.emit('set', path, val);

  // Pass control to the next pre
  next();
});
```

Moreover, you can mutate the incoming `method` arguments so that subsequent middleware see different values for those arguments. To do so, just pass the new values to `next`:

```js
.pre(method, function firstPre (next, methodArg1, methodArg2) {
  // Mutate methodArg1
  next("altered-" + methodArg1.toString(), methodArg2);
});

// pre declaration is chainable
.pre(method, function secondPre (next, methodArg1, methodArg2) {
  console.log(methodArg1);
  // => 'altered-originalValOfMethodArg1'

  console.log(methodArg2);
  // => 'originalValOfMethodArg2'

  // Passing no arguments to `next` automatically passes along the current argument values
  // i.e., the following `next()` is equivalent to `next(methodArg1, methodArg2)`
  // and also equivalent to, with the example method arg
  // values, `next('altered-originalValOfMethodArg1', 'originalValOfMethodArg2')`
  next();
});
```

#### Schema gotcha

`type`, when used in a schema has special meaning within Mongoose. If your schema requires using `type` as a nested property you must use object notation:

```js
new Schema({
  broken: { type: Boolean },
  asset: {
    name: String,
    type: String // uh oh, it broke. asset will be interpreted as String
  }
});

new Schema({
  works: { type: Boolean },
  asset: {
    name: String,
    type: { type: String } // works. asset is an object with a type property
  }
});
```

## API Docs

Find the API docs [here](http://mongoosejs.com/docs/api.html), generated using [dox](https://github.com/tj/dox)
and [acquit](https://github.com/vkarpov15/acquit).
