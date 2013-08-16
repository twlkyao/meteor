<template name="api">
{{#better_markdown}}

<h1 id="api">The Meteor API</h1>

Your JavaScript code can run in two environments: the *client* (browser), and
the *server* (a [Node.js](http://nodejs.org/) container on a server).  For each
function in this API reference, we'll indicate if the function is available just
on the client, just on the server, or *Anywhere*.

<h2 id="core"><span>Meteor Core</span></h2>

{{> api_box isClient}}
{{> api_box isServer}}

<div class="note">
`Meteor.isServer` can be used to limit where code runs, but it does not
prevent code from being sent to the client. Any sensitive code that you
don't want served to the client, such as code containing passwords or
authentication mechanisms, should be kept in the `server` directory.
</div>


{{> api_box startup}}

On a server, the function will run as soon as the server process is
finished starting. On a client, the function will run as soon as the DOM
is ready.

The `startup` callbacks are called in the same order as the calls to
`Meteor.startup` were made.

On a client, `startup` callbacks from packages will be called
first, followed by `<body>` templates from your `.html` files,
followed by your application code.

    // On server startup, if the database is empty, create some initial data.
    if (Meteor.isServer) {
      Meteor.startup(function () {
        if (Rooms.find().count() === 0) {
          Rooms.insert({name: "Initial room"});
        }
      });
    }

{{> api_box absoluteUrl}}

{{> api_box settings}}

{{> api_box release}}

<h2 id="publishandsubscribe"><span>Publish and subscribe</span></h2>

These functions control how Meteor servers publish sets of records and
how clients can subscribe to those sets.

{{> api_box publish}}

To publish records to clients, call `Meteor.publish` on the server with
two parameters: the name of the record set, and a *publish function*
that Meteor will call each time a client subscribes to the name.

Publish functions can return a
[`Collection.Cursor`](#meteor_collection_cursor), in which case Meteor
will publish that cursor's documents. You can also return an array of
`Collection.Cursor`s, in which case Meteor will publish all of the
cursors.

<div class="warning">
If you return multiple cursors in an array, they currently must all be from
different collections. We hope to lift this restriction in a future release.
</div>

    // server: publish the rooms collection, minus secret info.
    Meteor.publish("rooms", function () {
      return Rooms.find({}, {fields: {secretInfo: 0}});
    });

    // ... and publish secret info for rooms where the logged-in user
    // is an admin. If the client subscribes to both streams, the records
    // are merged together into the same documents in the Rooms collection.
    Meteor.publish("adminSecretInfo", function () {
      return Rooms.find({admin: this.userId}, {fields: {secretInfo: 1}});
    });

    // publish dependent documents and simulate joins
    Meteor.publish("roomAndMessages", function (roomId) {
      check(roomId, String);
      return [
        Rooms.find({_id: roomId}, {fields: {secretInfo: 0}}),
        Messages.find({roomId: roomId})
      ];
    });

Otherwise, the publish function should call the functions
[`added`](#publish_added) (when a new document is added to the published record
set), [`changed`](#publish_changed) (when some fields on a document in the
record set are changed or cleared), and [`removed`](#publish_removed) (when
documents are removed from the published record set) to inform subscribers about
documents.  These methods are provided by `this` in your publish function.



<!-- TODO discuss ready -->

Example:

    // server: publish the current size of a collection
    Meteor.publish("counts-by-room", function (roomId) {
      var self = this;
      check(roomId, String);
      var count = 0;
      var initializing = true;
      var handle = Messages.find({roomId: roomId}).observeChanges({
        added: function (id) {
          count++;
          if (!initializing)
            self.changed("counts", roomId, {count: count});
        },
        removed: function (id) {
          count--;
          self.changed("counts", roomId, {count: count});
        }
        // don't care about moved or changed
      });

      // Observe only returns after the initial added callbacks have
      // run.  Now return an initial value and mark the subscription
      // as ready.
      initializing = false;
      self.added("counts", roomId, {count: count});
      self.ready();

      // Stop observing the cursor when client unsubs.
      // Stopping a subscription automatically takes
      // care of sending the client any removed messages.
      self.onStop(function () {
        handle.stop();
      });
    });

    // client: declare collection to hold count object
    Counts = new Meteor.Collection("counts");

    // client: subscribe to the count for the current room
    Deps.autorun(function () {
      Meteor.subscribe("counts-by-room", Session.get("roomId"));
    });

    // client: use the new collection
    console.log("Current room has " +
                Counts.findOne(Session.get("roomId")).count +
                " messages.");

<div class="warning">
Meteor will emit a warning message if you call `Meteor.publish` in a
project that includes the `autopublish` package.  Your publish function
will still work.
</div>

{{> api_box subscription_userId}}

This is constant. However, if the logged-in user changes, the publish
function is rerun with the new value.

{{> api_box subscription_added}}
{{> api_box subscription_changed}}
{{> api_box subscription_removed}}
{{> api_box subscription_ready}}

{{> api_box subscription_onStop}}

If you call [`observe`](#observe) or [`observeChanges`](#observe_changes) in your
publish handler, this is the place to stop the observes.

{{> api_box subscription_error}}
{{> api_box subscription_stop}}

{{> api_box subscribe}}

When you subscribe to a record set, it tells the server to send records to the
client.  The client stores these records in local [Minimongo
collections](#meteor_collection), with the same name as the `collection`
argument used in the publish handler's `added`, `changed`, and `removed`
callbacks.  Meteor will queue incoming attributes until you declare the
[`Meteor.Collection`](#meteor_collection) on the client with the matching
collection name.

    // okay to subscribe (and possibly receive data) before declaring
    // the client collection that will hold it.  assume "allplayers"
    // publishes data from server's "players" collection.
    Meteor.subscribe("allplayers");
    ...
    // client queues incoming players records until ...
    ...
    Players = new Meteor.Collection("players");

The client will see a document if the document is currently in the published
record set of any of its subscriptions.

The `onReady` callback is called with no arguments when the server
[marks the subscription as ready](#publish_ready). The `onError` callback is
called with a [`Meteor.Error`](#meteor_error) if the subscription fails or is
terminated by the server.

`Meteor.subscribe` returns a subscription handle, which is an object with the
following methods:

<dl class="callbacks">
{{#dtdd "stop()"}}
Cancel the subscription. This will typically result in the server directing the
client to remove the subscription's data from the client's cache.
{{/dtdd}}

{{#dtdd "ready()"}}
True if the server has [marked the subscription as ready](#publish_ready). A
reactive data source.
{{/dtdd}}
</dl>

If you call `Meteor.subscribe` within a [reactive computation](#reactivity),
for example using
[`Deps.autorun`](#deps_autorun), the subscription will automatically be
cancelled when the computation is invalidated or stopped; it's not necessary
to call `stop` on
subscriptions made from inside `autorun`. However, if the next iteration
of your run function subscribes to the same record set (same name and
parameters), Meteor is smart enough to skip a wasteful
unsubscribe/resubscribe. For example:

    Deps.autorun(function () {
      Meteor.subscribe("chat", {room: Session.get("current-room")});
      Meteor.subscribe("privateMessages");
    });

This subscribes you to the chat messages in the current room and to your private
messages. When you change rooms by calling `Session.set("current-room",
"new-room")`, Meteor will subscribe to the new room's chat messages,
unsubscribe from the original room's chat messages, and continue to
stay subscribed to your private messages.

If more than one subscription sends conflicting values for a field (same
collection name, document ID, and field name), then the value on the client will
be one of the published values, chosen arbitrarily.

<h2 id="methods_header"><span>Methods</span></h2>

Methods are remote functions that Meteor clients can invoke.

{{> api_box methods}}

Example:

    Meteor.methods({
      foo: function (arg1, arg2) {
        check(arg1, String);
        check(arg2, [Number]);
        // .. do stuff ..
        if (you want to throw an error)
          throw new Meteor.Error(404, "Can't find my pants");
        return "some return value";
      },

      bar: function () {
        // .. do other stuff ..
        return "baz";
      }
    });

Calling `methods` on the server defines functions that can be called remotely by
clients.  They should return an [EJSON](#ejson)-able value or throw an
exception.  Inside your method invocation, `this` is bound to a method
invocation object, which provides the following:

* `isSimulation`: a boolean value, true if this invocation is a stub.
* `unblock`: when called, allows the next method from this client to
begin running.
* `userId`: the id of the current user.
* `setUserId`: a function that associates the current client with a user.

Calling `methods` on the client defines *stub* functions associated with
server methods of the same name.  You don't have to define a stub for
your method if you don't want to.  In that case, method calls are just
like remote procedure calls in other systems, and you'll have to wait
for the results from the server.

If you do define a stub, when a client invokes a server method it will
also run its stub in parallel.  On the client, the return value of a
stub is ignored.  Stubs are run for their side-effects: they are
intended to *simulate* the result of what the server's method will do,
but without waiting for the round trip delay.  If a stub throws an
exception it will be logged to the console.

You use methods all the time, because the database mutators
([`insert`](#insert), [`update`](#update), [`remove`](#remove)) are implemented
as methods. When you call any of these functions on the client, you're invoking
their stub version that update the local cache, and sending the same write
request to the server. When the server responds, the client updates the local
cache with the writes that actually occurred on the server.

{{> api_box method_invocation_userId}}

The user id is an arbitrary string &mdash; typically the id of the user record
in the database. You can set it with the `setUserId` function. If you're using
the [Meteor accounts system](#accounts_api) then this is handled for you.

{{> api_box method_invocation_setUserId}}

Call this function to change the currently logged in user on the
connection that made this method call. This simply sets the value of
`userId` for future method calls received on this connection. Pass
`null` to log out the connection.

If you are using the [built-in Meteor accounts system](#accounts_api) then this
should correspond to the `_id` field of a document in the
[`Meteor.users`](#meteor_users) collection.

`setUserId` is not retroactive. It affects the current method call and
any future method calls on the connection. Any previous method calls on
this connection will still see the value of `userId` that was in effect
when they started.

{{> api_box method_invocation_isSimulation}}

{{> api_box method_invocation_unblock}}

On the server, methods from a given client run one at a time. The N+1th
invocation from a client won't start until the Nth invocation
returns. However, you can change this by calling `this.unblock`. This
will allow the N+1th invocation to start running in a new fiber.

{{> api_box error}}

If you want to return an error from a method, throw an exception.  Methods can
throw any kind of exception.  But `Meteor.Error` is the only kind of error that
a server will send to the client. If a method function throws a different
exception, then it will be mapped to a sanitized version on the
wire. Specifically, if the `sanitizedError` field on the thrown error is set to
a `Meteor.Error`, then that error will be sent to the client. Otherwise, if no
sanitized version is available, the client gets
`Meteor.Error(500, 'Internal server error')`.

{{> api_box meteor_call}}

This is how to invoke a method.  It will run the method on the server.  If a
stub is available, it will also run the stub on the client.  (See also
[`Meteor.apply`](#meteor_apply), which is identical to `Meteor.call` except that
you specify the parameters as an array instead of as separate arguments and you
can specify a few options controlling how the method is executed.)

If you include a callback function as the last argument (which can't be
an argument to the method, since functions aren't serializable), the
method will run asynchronously: it will return nothing in particular and
will not throw an exception.  When the method is complete (which may or
may not happen before `Meteor.call` returns), the callback will be
called with two arguments: `error` and `result`. If an error was thrown,
then `error` will be the exception object.  Otherwise, `error` will be
undefined and the return value (possibly undefined) will be in `result`.

    // async call
    Meteor.call('foo', 1, 2, function (error, result) { ... } );

If you do not pass a callback on the server, the method invocation will
block until the method is complete.  It will eventually return the
return value of the method, or it will throw an exception if the method
threw an exception. (Possibly mapped to 500 Server Error if the
exception happened remotely and it was not a `Meteor.Error` exception.)

    // sync call
    var result = Meteor.call('foo', 1, 2);

On the client, if you do not pass a callback and you are not inside a
stub, `call` will return `undefined`, and you will have no way to get
the return value of the method. That is because the client doesn't have
fibers, so there is not actually any way it can block on the remote
execution of a method.

Finally, if you are inside a stub on the client and call another
method, the other method is not executed (no RPC is generated, nothing
"real" happens).  If that other method has a stub, that stub stands in
for the method and is executed. The method call's return value is the
return value of the stub function.  The client has no problem executing
a stub synchronously, and that is why it's okay for the client to use
the synchronous `Meteor.call` form from inside a method body, as
described earlier.

Meteor tracks the database writes performed by methods, both on the client and
the server, and does not invoke `asyncCallback` until all of the server's writes
replace the stub's writes in the local cache. In some cases, there can be a lag
between the method's return value being available and the writes being visible:
for example, if another method still outstanding wrote to the same document, the
local cache may not be up to date until the other method finishes as well. If
you want to process the method's result as soon as it arrives from the server,
even if the method's writes are not available yet, you can specify an
`onResultReceived` callback to [`Meteor.apply`](#meteor_apply).

{{> api_box meteor_apply}}

`Meteor.apply` is just like `Meteor.call`, except that the method arguments are
passed as an array rather than directly as arguments, and you can specify
options about how the client executes the method.

<h2 id="connections"><span>Server connections</span></h2>

These functions manage and inspect the network connection between the
Meteor client and server.

{{> api_box status}}

This method returns the status of the connection between the client and
the server. The return value is an object with the following fields:

<dl class="objdesc">
{{#dtdd name="connected" type="Boolean"}}
  True if currently connected to the server. If false, changes and
  method invocations will be queued up until the connection is
  reestablished.
{{/dtdd}}

{{#dtdd name="status" type="String"}}
  Describes the current reconnection status. The possible
  values are `connected` (the connection is up and
  running), `connecting` (disconnected and trying to open a
  new connection), `failed` (permanently failed to connect; e.g., the client
  and server support different versions of DDP), `waiting` (failed
  to connect and waiting to try to reconnect) and `offline` (user has disconnected the connection).
{{/dtdd}}

{{#dtdd name="retryCount" type="Number"}}
  The number of times the client has tried to reconnect since the
  connection was lost. 0 when connected.
{{/dtdd}}

{{#dtdd name="retryTime" type="Number or undefined"}}
  The estimated time of the next reconnection attempt. To turn this
  into an interval until the next reconnection, use
  `retryTime - (new Date()).getTime()`. This key will
  be set only when `status` is `waiting`.
{{/dtdd}}

{{#dtdd name="reason" type="String or undefined"}}
  If `status` is `failed`, a description of why the connection failed.
{{/dtdd}}
</dl>

Instead of using callbacks to notify you on changes, this is
a [reactive](#reactivity) data source. You can use it in a
[template](#templates) or [computation](#deps_autorun)
to get realtime updates.

{{> api_box reconnect}}

{{> api_box disconnect}}

Call this method to disconnect from the server and stop all
live data updates. While the client is disconnected it will not receive
updates to collections, method calls will be queued until the
connection is reestablished, and hot code push will be disabled.

Call [Meteor.reconnect](#meteor_reconnect) to reestablish the connection
and resume data transfer.

This can be used to save battery on mobile devices when real time
updates are not required.

{{> api_box connect}}

To call methods on another Meteor application or subscribe to its data
sets, call `DDP.connect` with the URL of the application.
`DDP.connect` returns an object which provides:

* `subscribe` -
  Subscribe to a record set. See
  [Meteor.subscribe](#meteor_subscribe).
* `call` -
  Invoke a method. See [Meteor.call](#meteor_call).
* `apply` -
  Invoke a method with an argument array. See
  [Meteor.apply](#meteor_apply).
* `methods` -
  Define client-only stubs for methods defined on the remote server. See
  [Meteor.methods](#meteor_methods).
* `status` -
  Get the current connection status. See
  [Meteor.status](#meteor_status).
* `reconnect` -
  See [Meteor.reconnect](#meteor_reconnect).
* `disconnect` -
  See [Meteor.disconnect](#meteor_disconnect).
* `onReconnect` - Set this to a function to be called as the first step of
  reconnecting. This function can call methods which will be executed before
  any other outstanding methods. For example, this can be used to re-establish
  the appropriate authentication context on the new connection.

By default, clients open a connection to the server from which they're loaded.
When you call `Meteor.subscribe`, `Meteor.status`, `Meteor.call`, and
`Meteor.apply`, you are using a connection back to that default
server.

<h2 id="collections"><span>Collections</span></h2>

Meteor stores data in *collections*.  To get started, declare a
collection with `new Meteor.Collection`.

{{> api_box meteor_collection}}

Calling this function is analogous to declaring a model in a traditional ORM
(Object-Relation Mapper)-centric framework. It sets up a *collection* (a storage
space for records, or "documents") that can be used to store a particular type
of information, like users, posts, scores, todo items, or whatever matters to
your application.  Each document is a EJSON object.  It includes an `_id`
property whose value is unique in the collection, which Meteor will set when you
first create the document.

    // common code on client and server declares livedata-managed mongo
    // collection.
    Chatrooms = new Meteor.Collection("chatrooms");
    Messages = new Meteor.Collection("messages");

The function returns an object with methods to [`insert`](#insert)
documents in the collection, [`update`](#update) their properties, and
[`remove`](#remove) them, and to [`find`](#find) the documents in the
collection that match arbitrary criteria. The way these methods work is
compatible with the popular Mongo database API.  The same database API
works on both the client and the server (see below).

    // return array of my messages
    var myMessages = Messages.find({userId: Session.get('myUserId')}).fetch();

    // create a new message
    Messages.insert({text: "Hello, world!"});

    // mark my first message as "important"
    Messages.update(myMessages[0]._id, {$set: {important: true}});

If you pass a `name` when you create the collection, then you are
declaring a persistent collection &mdash; one that is stored on the
server and seen by all users. Client code and server code can both
access the same collection using the same API.

Specifically, when you pass a `name`, here's what happens:

* On the server, a collection with that name is created on a backend
Mongo server. When you call methods on that collection on the server,
they translate directly into normal Mongo operations (after checking that
they match your [access control rules](#allow)).

* On the client, a Minimongo instance is
created. Minimongo is essentially an in-memory, non-persistent
implementation of Mongo in pure JavaScript. It serves as a local cache
that stores just the subset of the database that this client is working
with. Queries on the client ([`find`](#find)) are served directly out of
this cache, without talking to the server.

* When you write to the database on the client ([`insert`](#insert),
[`update`](#update), [`remove`](#remove)), the command is executed
immediately on the client, and, simultaneously, it's shipped up to the
server and executed there too.  The `livedata` package is
responsible for this.

If you pass `null` as the `name`, then you're creating a local
collection. It's not synchronized anywhere; it's just a local scratchpad
that supports Mongo-style [`find`](#find), [`insert`](#insert),
[`update`](#update), and [`remove`](#remove) operations.  (On both the
client and the server, this scratchpad is implemented using Minimongo.)

By default, Meteor automatically publishes every document in your
collection to each connected client.  To turn this behavior off, remove
the `autopublish` package:

    $ meteor remove autopublish

and instead call [`Meteor.publish`](#meteor_publish) to specify which parts of
your collection should be published to which users.

    // Create a collection called Posts and put a document in it. The
    // document will be immediately visible in the local copy of the
    // collection. It will be written to the server-side database
    // a fraction of a second later, and a fraction of a second
    // after that, it will be synchronized down to any other clients
    // that are subscribed to a query that includes it (see
    // Meteor.subscribe and autopublish)
    Posts = new Meteor.Collection("posts");
    Posts.insert({title: "Hello world", body: "First post"});

    // Changes are visible immediately -- no waiting for a round trip to
    // the server.
    assert(Posts.find().count() === 1);

    // Create a temporary, local collection. It works just like any other
    // collection, but it doesn't send changes to the server, and it
    // can't receive any data from subscriptions.
    Scratchpad = new Meteor.Collection;
    for (var i = 0; i < 10; i++)
      Scratchpad.insert({number: i * 2});
    assert(Scratchpad.find({number: {$lt: 9}}).count() === 5);


If you specify a `transform` option to the `Collection` or any of its retrieval
methods, documents are passed through the `transform` function before being
returned or passed to callbacks.  This allows you to add methods or otherwise
modify the contents of your collection from their database representation.  You
can also specify `transform` on a particular `find`, `findOne`, `allow`, or
`deny` call.

    // An Animal class that takes a document in its constructor
    Animal = function (doc) {
      _.extend(this, doc);
    };
    _.extend(Animal.prototype, {
      makeNoise: function () {
        console.log(this.sound);
      }
    });

    // Define a Collection that uses Animal as its document
    Animals = new Meteor.Collection("Animals", {
      transform: function (doc) { return new Animal(doc); }
    });

    // Create an Animal and call its makeNoise method
    Animals.insert({name: "raptor", sound: "roar"});
    Animals.findOne({name: "raptor"}).makeNoise(); // prints "roar"

`transform` functions are not called reactively.  If you want to add a
dynamically changing attribute to an object, do it with a function that computes
the value at the time it's called, not by computing the attribute at `transform`
time.

<div class="warning">
In this release, Minimongo has some limitations:

* `$pull` in modifiers can only accept certain kinds
of selectors.
* `$` to denote the matched array position is not
supported in modifier.
* `findAndModify`, upsert, aggregate functions, and
map/reduce aren't supported.

All of these will be addressed in a future release. For full
Minimongo release notes, see packages/minimongo/NOTES
in the repository.
</div>

<div class="warning">
Minimongo doesn't currently have indexes. It's rare for this to be an
issue, since it's unusual for a client to have enough data that an
index is worthwhile.
</div>

{{> api_box find}}

`find` returns a cursor.  It does not immediately access the database or return
documents.  Cursors provide `fetch` to return all matching documents, `map` and
`forEach` to iterate over all matching documents, and `observe` and
`observeChanges` to register callbacks when the set of matching documents
changes.

<div class="warning">
Collection cursors are not query snapshots.  If the database changes
between calling `Collection.find` and fetching the
results of the cursor, or while fetching results from the cursor,
those changes may or may not appear in the result set.
</div>

Cursors are a reactive data source.  The first time you retrieve a
cursor's documents with `fetch`, `map`, or `forEach` inside a
reactive computation (eg, a template or
[`autorun`](#deps_autorun)), Meteor will register a
dependency on the underlying data.  Any change to the collection that
changes the documents in a cursor will trigger a recomputation.  To
disable this behavior, pass `{reactive: false}` as an option to
`find`.

{{> api_box findone}}

Equivalent to `find(selector, options).fetch()[0]` with
`options.limit = 1`.

{{> api_box insert}}

Add a document to the collection. A document is just an object, and
its fields can contain any combination of EJSON-compatible datatypes
(arrays, objects, numbers, strings, `null`, true, and false).

`insert` will generate a unique ID for the object you pass, insert it
in the database, and return the ID. When `insert` is called from
untrusted client code, it will be allowed only if passes any
applicable [`allow`](#allow) and [`deny`](#deny) rules.

On the server, if you don't provide a callback, then `insert` blocks
until the database acknowledges the write, or throws an exception if
something went wrong.  If you do provide a callback, `insert` still
returns the ID immediately.  Once the insert completes (or fails), the
callback is called with error and result arguments.  In an error case,
`result` is undefined.  If the insert is successful, `error` is
undefined and `result` is the new document ID.

On the client, `insert` never blocks.  If you do not provide a callback
and the insert fails on the server, then Meteor will log a warning to
the console.  If you provide a callback, Meteor will call that function
with `error` and `result` arguments.  In an error case, `result` is
undefined.  If the insert is successful, `error` is undefined and
`result` is the new document ID.

Example:

    var groceriesId = Lists.insert({name: "Groceries"});
    Items.insert({list: groceriesId, name: "Watercress"});
    Items.insert({list: groceriesId, name: "Persimmons"});

{{> api_box update}}

Modify documents that match `selector` according to `modifier` (see
[modifier documentation](#modifiers)).

The behavior of `update` differs depending on whether it is called by
trusted or untrusted code. Trusted code includes server code and
method code. Untrusted code includes client-side code such as event
handlers and a browser's JavaScript console.

- Trusted code can modify multiple documents at once by setting
  `multi` to true, and can use an arbitrary [Mongo
  selector](#selectors) to find the documents to modify. It bypasses
  any access control rules set up by [`allow`](#allow) and
  [`deny`](#deny).

- Untrusted code can only modify a single document at once, specified
  by its `_id`. The modification is allowed only after checking any
  applicable [`allow`](#allow) and [`deny`](#deny) rules.

On the server, if you don't provide a callback, then `update` blocks
until the database acknowledges the write, or throws an exception if
something went wrong.  If you do provide a callback, `update` returns
immediately.  Once the update completes, the callback is called with a
single error argument in the case of failure, or no arguments if the
update was successful.

On the client, `update` never blocks.  If you do not provide a callback
and the update fails on the server, then Meteor will log a warning to
the console.  If you provide a callback, Meteor will call that function
with an error argument if there was an error, or no arguments if the
update was successful.

Client example:

    // When the givePoints button in the admin dashboard is pressed,
    // give 5 points to the current player. The new score will be
    // immediately visible on everyone's screens.
    Template.adminDashboard.events({
      'click .givePoints': function () {
        Players.update(Session.get("currentPlayer"), {$inc: {score: 5}});
      }
    });

Server example:

    // Give the "Winner" badge to each user with a score greater than
    // 10. If they are logged in and their badge list is visible on the
    // screen, it will update automatically as they watch.
    Meteor.methods({
      declareWinners: function () {
        Players.update({score: {$gt: 10}},
                       {$addToSet: {badges: "Winner"}},
                       {multi: true});
      }
    });

<div class="warning">
The Mongo `upsert` feature is not implemented.
</div>

{{> api_box remove}}

Find all of the documents that match `selector` and delete them from
the collection.

The behavior of `remove` differs depending on whether it is called by
trusted or untrusted code. Trusted code includes server code and
method code. Untrusted code includes client-side code such as event
handlers and a browser's JavaScript console.

- Trusted code can use an arbitrary [Mongo selector](#selectors) to
  find the documents to remove, and can remove more than one document
  at once by passing a selector that matches multiple documents. It
  bypasses any access control rules set up by [`allow`](#allow) and
  [`deny`](#deny).

  As a safety measure, if `selector` is omitted (or is `undefined`),
  no documents will be removed. Set `selector` to `{}` if you really
  want to remove all documents from your collection.

- Untrusted code can only remove a single document at a time,
  specified by its `_id`. The document is removed only after checking
  any applicable [`allow`](#allow) and [`deny`](#deny) rules.

On the server, if you don't provide a callback, then `remove` blocks
until the database acknowledges the write, or throws an exception if
something went wrong.  If you do provide a callback, `remove` returns
immediately.  Once the remove completes, the callback is called with a
single error argument in the case of failure, or no arguments if the
remove was successful.

On the client, `remove` never blocks.  If you do not provide a callback
and the remove fails on the server, then Meteor will log a warning to
the console.  If you provide a callback, Meteor will call that function
with an error argument if there was an error, or no arguments if the
remove was successful.

Client example:

    // When the remove button is clicked on a chat message, delete
    // that message.
    Template.chat.events({
      'click .remove': function () {
        Messages.remove(this._id);
      }
    });

Server example:

    // When the server starts, clear the log, and delete all players
    // with a karma of less than -2.
    Meteor.startup(function () {
      if (Meteor.isServer) {
        Logs.remove({});
        Players.remove({karma: {$lt: -2}});
      }
    });

{{> api_box allow}}

When a client calls `insert`, `update`, or `remove` on a collection, the
collection's `allow` and [`deny`](#deny) callbacks are called
on the server to determine if the write should be allowed. If at least
one `allow` callback allows the write, and no `deny` callbacks deny the
write, then the write is allowed to proceed.

These checks are run only when a client tries to write to the database
directly, for example by calling `update` from inside an event
handler. Server code is trusted and isn't subject to `allow` and `deny`
restrictions. That includes methods that are called with `Meteor.call`
&mdash; they are expected to do their own access checking rather than
relying on `allow` and `deny`.

You can call `allow` as many times as you like, and each call can
include any combination of `insert`, `update`, and `remove`
functions. The functions should return `true` if they think the
operation should be allowed. Otherwise they should return `false`, or
nothing at all (`undefined`). In that case Meteor will continue
searching through any other `allow` rules on the collection.

The available callbacks are:

<dl class="callbacks">
{{#dtdd "insert(userId, doc)"}}
The user `userId` wants to insert the document `doc` into the
collection. Return `true` if this should be allowed.
{{/dtdd}}

{{#dtdd "update(userId, doc, fieldNames, modifier)"}}

The user `userId` wants to update a document `doc`. (`doc` is the
current version of the document from the database, without the
proposed update.) Return `true` to permit the change.

`fieldNames` is an array of the (top-level) fields in `doc` that the
client wants to modify, for example
`['name',`&nbsp;`'score']`. `modifier` is the raw Mongo modifier that
the client wants to execute, for example `{$set: {'name.first':
"Alice"}, $inc: {score: 1}}`.

Only Mongo modifiers are supported (operations like `$set` and `$push`).
If the user tries to replace the entire document rather than use
$-modifiers, the request will be denied without checking the `allow`
functions.

{{/dtdd}}

{{#dtdd "remove(userId, doc)"}}

The user `userId` wants to remove `doc` from the database. Return
`true` to permit this.

{{/dtdd}}

</dl>

When calling `update` or `remove` Meteor will by default fetch the
entire document `doc` from the database. If you have large documents
you may wish to fetch only the fields that are actually used by your
functions. Accomplish this by setting `fetch` to an array of field
names to retrieve.

Example:

    // Create a collection where users can only modify documents that
    // they own. Ownership is tracked by an 'owner' field on each
    // document. All documents must be owned by the user that created
    // them and ownership can't be changed. Only a document's owner
    // is allowed to delete it, and the 'locked' attribute can be
    // set on a document to prevent its accidental deletion.

    Posts = new Meteor.Collection("posts");

    Posts.allow({
      insert: function (userId, doc) {
        // the user must be logged in, and the document must be owned by the user
        return (userId && doc.owner === userId);
      },
      update: function (userId, doc, fields, modifier) {
        // can only change your own documents
        return doc.owner === userId;
      },
      remove: function (userId, doc) {
        // can only remove your own documents
        return doc.owner === userId;
      },
      fetch: ['owner']
    });

    Posts.deny({
      update: function (userId, docs, fields, modifier) {
        // can't change owners
        return _.contains(fields, 'owner');
      },
      remove: function (userId, doc) {
        // can't remove locked documents
        return doc.locked;
      },
      fetch: ['locked'] // no need to fetch 'owner'
    });

If you never set up any `allow` rules on a collection then all client
writes to the collection will be denied, and it will only be possible to
write to the collection from server-side code. In this case you will
have to create a method for each possible write that clients are allowed
to do. You'll then call these methods with `Meteor.call` rather than
having the clients call `insert`, `update`, and `remove` directly on the
collection.

Meteor also has a special "insecure mode" for quickly prototyping new
applications. In insecure mode, if you haven't set up any `allow` or `deny`
rules on a collection, then all users have full write access to the
collection. This is the only effect of insecure mode. If you call `allow` or
`deny` at all on a collection, even `Posts.allow({})`, then access is checked
just like normal on that collection. __New Meteor projects start in insecure
mode by default.__ To turn it off just run `$ meteor remove insecure`.

{{> api_box deny}}

This works just like [`allow`](#allow), except it lets you
make sure that certain writes are definitely denied, even if there is an
`allow` rule that says that they should be permitted.

When a client tries to write to a collection, the Meteor server first
checks the collection's `deny` rules. If none of them return true then
it checks the collection's `allow` rules. Meteor allows the write only
if no `deny` rules return `true` and at least one `allow` rule returns
`true`.

<h2 id="meteor_collection_cursor"><span>Cursors</span></h2>

To create a cursor, use [`find`](#find).  To access the documents in a
cursor, use [`forEach`](#foreach), [`map`](#map), or [`fetch`](#fetch).

{{> api_box cursor_foreach}}

When called from a reactive computation, `forEach` registers dependencies on
the matching documents.

Examples:

    // Print the titles of the five top-scoring posts
    var topPosts = Posts.find({}, {sort: {score: -1}, limit: 5});
    var count = 0;
    topPosts.forEach(function (post) {
      console.log("Title of post " + count + ": " + post.title);
      count += 1;
    });

{{> api_box cursor_map}}

When called from a reactive computation, `map` registers dependencies on
the matching documents.

<!-- The following is not yet implemented, but users shouldn't assume
     sequential execution anyway because that will break. -->
On the server, if `callback` yields, other calls to `callback` may occur while
the first call is waiting. If strict sequential execution is necessary, use
`forEach` instead.

{{> api_box cursor_fetch}}

When called from a reactive computation, `fetch` registers dependencies on
the matching documents.

{{> api_box cursor_count}}

    // Display a count of posts matching certain criteria. Automatically
    // keep it updated as the database changes.
    var frag = Meteor.render(function () {
      var highScoring = Posts.find({score: {$gt: 10}});
      return "<p>There are " + highScoring.count() + " posts with " +
        "scores greater than 10</p>";
    });
    document.body.appendChild(frag);

Unlike the other functions, `count` registers a dependency only on the
number of matching documents.  (Updates that just change or reorder the
documents in the result set will not trigger a recomputation.)

{{> api_box cursor_rewind}}

The `forEach`, `map`, or `fetch` methods can only be called once on a
cursor.  To access the data in a cursor more than once, use `rewind` to
reset the cursor.

{{> api_box cursor_observe}}

Establishes a *live query* that invokes callbacks when the result of
the query changes. The callbacks receive the entire contents of the
document that was affected, as well as its old contents, if
applicable. If you only need to receive the fields that changed, see
[`observeChanges`](#observe_changes).

`callbacks` may have the following functions as properties:

<dl class="callbacks">
<dt><span class="name">added(document)</span> <span class="or">or</span></dt>
<dt><span class="name">addedAt(document, atIndex, before)</span></dt>
<dd>
{{#better_markdown}}
A new document `document` entered the result set. The new document
appears at position `atIndex`. It is immediately before the document
whose `_id` is `before`. `before` will be `null` if the new document
is at the end of the results.
{{/better_markdown}}
</dd>

<dt><span class="name">changed(newDocument, oldDocument)
    <span class="or">or</span></span></dt>
<dt><span class="name">changedAt(newDocument, oldDocument, atIndex)</span></dt>
<dd>
{{#better_markdown}}
The contents of a document were previously `oldDocument` and are now
`newDocument`. The position of the changed document is `atIndex`.
{{/better_markdown}}
</dd>

<dt><span class="name">removed(oldDocument)</span>
  <span class="or">or</span></dt>
<dt><span class="name">removedAt(oldDocument, atIndex)</span></dt>
<dd>
{{#better_markdown}}
The document `oldDocument` is no longer in the result set. It used to be at position `atIndex`.
{{/better_markdown}}
</dd>

{{#dtdd "movedTo(document, fromIndex, toIndex, before)"}}
A document changed its position in the result set, from `fromIndex` to `toIndex`
(which is before the document with id `before`). Its current contents is
`document`.
{{/dtdd}}
</dl>

Use `added`, `changed`, and `removed` when you don't care about the
order of the documents in the result set. They are more efficient than
`addedAt`, `changedAt`, and `removedAt`.

Before `observe` returns, `added` (or `addedAt`) will be called zero
or more times to deliver the initial results of the query.

`observe` returns a live query handle, which is an object with a `stop` method.
Call `stop` with no arguments to stop calling the callback functions and tear
down the query. **The query will run forever until you call this.**  If
`observe` is called from a `Deps.autorun` computation, it is automatically
stopped when the computation is rerun or stopped.
(If the cursor was created with the option `reactive` set to false, it will
only deliver the initial results and will not call any further callbacks;
it is not necessary to call `stop` on the handle.)


{{> api_box cursor_observe_changes}}

Establishes a *live query* that invokes callbacks when the result of
the query changes. In contrast to [`observe`](#observe),
`observeChanges` provides only the difference between the old and new
result set, not the entire contents of the document that changed.

`callbacks` may have the following functions as properties:

<dl class="callbacks">
<dt><span class="name">added(id, fields)</span>
  <span class="or">or</span></dt>
<dt><span class="name">addedBefore(id, fields, before)</span></dt>
<dd>
{{#better_markdown}}
A new document entered the result set. It has the `id` and `fields`
specified. `fields` contains all fields of the document excluding the
`_id` field. The new document is before the document identified by
`before`, or at the end if `before` is `null`.
{{/better_markdown}}
</dd>

{{#dtdd "changed(id, fields)"}}
The document identified by `id` has changed. `fields` contains the
changed fields with their new values. If a field was removed from the
document then it will be present in `fields` with a value of
`undefined`.
{{/dtdd}}

{{#dtdd "movedBefore(id, before)"}}
The document identified by `id` changed its position in the ordered result set,
and now appears before the document identified by `before`.
{{/dtdd}}

{{#dtdd "removed(id)"}}
The document identified by `id` was removed from the result set.
{{/dtdd}}
</dl>

`observeChanges` is significantly more efficient if you do not use
`addedBefore` or `movedBefore`.

Before `observeChanges` returns, `added` (or `addedBefore`) will be called
zero or more times to deliver the initial results of the query.

`observeChanges` returns a live query handle, which is an object with a `stop`
method.  Call `stop` with no arguments to stop calling the callback functions
and tear down the query. **The query will run forever until you call this.**
If
`observeChanges` is called from a `Deps.autorun` computation, it is automatically
stopped when the computation is rerun or stopped.
(If the cursor was created with the option `reactive` set to false, it will
only deliver the initial results and will not call any further callbacks;
it is not necessary to call `stop` on the handle.)

<div class="note">
Unlike `observe`, `observeChanges` does not provide absolute position
information (that is, `atIndex` positions rather than `before`
positions.) This is for efficiency.
</div>

Example:

    // Keep track of how many administrators are online.
    var count = 0;
    var query = Users.find({admin: true, onlineNow: true});
    var handle = query.observeChanges({
      added: function (id, user) {
        count++;
        console.log(user.name + " brings the total to " + count + " admins.");
      },
      removed: function () {
        count--;
        console.log("Lost one. We're now down to " + count + " admins.");
      }
    });

    // After five seconds, stop keeping the count.
    setTimeout(function () {handle.stop();}, 5000);

{{> api_box collection_object_id}}

`Meteor.Collection.ObjectID` follows the same API as the [Node MongoDB driver
`ObjectID`](http://mongodb.github.com/node-mongodb-native/api-bson-generated/objectid.html)
class. Note that you must use the `equals` method (or [`EJSON.equals`](#ejson_equals)) to
compare them; the `===` operator will not work. If you are writing generic code
that needs to deal with `_id` fields that may be either strings or `ObjectID`s, use
[`EJSON.equals`](#ejson_equals) instead of `===` to compare them.

<div class="note">
  `ObjectID` values created by Meteor will not have meaningful answers to their `getTimestamp`
  method, since Meteor currently constructs them fully randomly.
</div>

{{#api_box_inline selectors}}

In its simplest form, a selector is just a set of keys that must
match in a document:

    // Matches all documents where deleted is false
    {deleted: false}

    // Matches all documents where the name and cognomen are as given
    {name: "Rhialto", cognomen: "the Marvelous"}

    // Matches every document
    {}

But they can also contain more complicated tests:

    // Matches documents where age is greater than 18
    {age: {$gt: 18}}

    // Also matches documents where tags is an array containing "popular"
    {tags: "popular"}

    // Matches documents where fruit is one of three possibilities
    {fruit: {$in: ["peach", "plum", "pear"]}}

See the [complete
documentation](http://www.mongodb.org/display/DOCS/Advanced+Queries).

{{/api_box_inline}}

{{#api_box_inline modifiers}}

A modifier is an object that describes how to update a document in
place by changing some of its fields. Some examples:

    // Set the 'admin' property on the document to true
    {$set: {admin: true}}

    // Add 2 to the 'votes' property, and add "Traz"
    // to the end of the 'supporters' array
    {$inc: {votes: 2}, $push: {supporters: "Traz"}}

But if a modifier doesn't contain any $-operators, then it is instead
interpreted as a literal document, and completely replaces whatever was
previously in the database. (Literal document modifiers are not currently
supported by [validated updates](#allow).)

    // Find the document with id "123", and completely replace it.
    Users.update({_id: "123"}, {name: "Alice", friends: ["Bob"]});

See the [full list of
modifiers](http://www.mongodb.org/display/DOCS/Updating#Updating-ModifierOperations).

{{/api_box_inline}}

{{#api_box_inline sortspecifiers}}

Sorts may be specified using your choice of several syntaxes:

    // All of these do the same thing (sort in ascending order by
    // key "a", breaking ties in descending order of key "b")

    [["a", "asc"], ["b", "desc"]]
    ["a", ["b", "desc"]]
    {a: 1, b: -1}

The last form will only work if your JavaScript implementation
preserves the order of keys in objects. Most do, most of the time, but
it's up to you to be sure.

{{/api_box_inline}}

{{#api_box_inline fieldspecifiers}}

On the server, queries can specify a particular set of fields to include
or exclude from the result object.  (The field specifier is currently
ignored on the client.)

To exclude certain fields from the result objects, the field specifier
is a dictionary whose keys are field names and whose values are `0`.

    // Users.find({}, {fields: {password: 0, hash: 0}})

To return an object that only includes the specified field, use `1` as
the value.  The `_id` field is still included in the result.

    // Users.find({}, {fields: {firstname: 1, lastname: 1}})

It is not possible to mix inclusion and exclusion styles.

{{/api_box_inline}}

<h2 id="session"><span>Session</span></h2>

`Session` provides a global object on the client that you can use to
store an arbitrary set of key-value pairs. Use it to store things like
the currently selected item in a list.

What's special about `Session` is that it's reactive. If
you call [`Session.get`](#session_get)`("currentList")`
from inside a template, the template will automatically be rerendered
whenever [`Session.set`](#session_set)`("currentList", x)` is called.

{{> api_box set}}

Example:

    Deps.autorun(function () {
      Meteor.subscribe("chat-history", {room: Session.get("currentRoomId")});
    });

    // Causes the function passed to Deps.autorun to be re-run, so
    // that the chat-history subscription is moved to the room "home".
    Session.set("currentRoomId", "home");

{{> api_box setDefault}}

This is useful in initialization code, to avoid re-initializing a session
variable every time a new version of your app is loaded.

{{> api_box get}}

Example:

    Session.set("enemy", "Eastasia");
    var frag = Meteor.render(function () {
      return "<p>We've always been at war with " +
        Session.get("enemy") + "</p>";
    });

    // Page will say "We've always been at war with Eastasia"
    document.body.append(frag);

    // Page will change to say "We've always been at war with Eurasia"
    Session.set("enemy", "Eurasia");

{{> api_box equals}}

If value is a scalar, then these two expressions do the same thing:

    (1) Session.get("key") === value
    (2) Session.equals("key", value)

... but the second one is always better. It triggers fewer invalidations
(template redraws), making your program more efficient.

Example:

    <template name="postsView">
    {{! Show a dynamically updating list of items. Let the user click on an
        item to select it. The selected item is given a CSS class so it
        can be rendered differently. }}

    {{#each posts}}
      {{> postItem }}
    {{/each}}
    </{{! }}template>

    <template name="postItem">
      <div class="{{postClass}}">{{title}}</div>
    </{{! }}template>

    ///// in JS file
    Template.postsView.posts = function() {
      return Posts.find();
    };

    Template.postItem.postClass = function() {
      return Session.equals("selectedPost", this._id) ?
        "selected" : "";
    };

    Template.postItem.events({
      'click': function() {
        Session.set("selectedPost", this._id);
      }
    });

    // Using Session.equals here means that when the user clicks
    // on an item and changes the selection, only the newly selected
    // and the newly unselected items are re-rendered.
    //
    // If Session.get had been used instead of Session.equals, then
    // when the selection changed, all the items would be re-rendered.

For object and array session values, you cannot use `Session.equals`; instead,
you need to use the `underscore` package and write
`_.isEqual(Session.get(key), value)`.



<h2 id="accounts_api"><span>Accounts</span></h2>

The Meteor Accounts system builds on top of the `userId` support in
[`publish`](#publish_userId) and [`methods`](#method_userId). The core
packages add the concept of user documents stored in the database, and
additional packages add [secure password
authentication](#accounts_passwords), [integration with third party
login services](#meteor_loginwithexternalservice), and a [pre-built user
interface](#accountsui).

The basic Accounts system is in the `accounts-base` package, but
applications typically include this automatically by adding one of the
login provider packages: `accounts-password`, `accounts-facebook`,
`accounts-github`, `accounts-google`, `accounts-meetup`,
`accounts-twitter`, or `accounts-weibo`.


{{> api_box user}}

Retrieves the user record for the current user from
the [`Meteor.users`](#meteor_users) collection.

On the client, this will be the subset of the fields in the document that
are published from the server (other fields won't be available on the
client). By default the server publishes `username`, `emails`, and
`profile`. See [`Meteor.users`](#meteor_users) for more on
the fields used in user documents.

{{> api_box userId}}

{{> api_box users}}

This collection contains one document per registered user. Here's an example
user document:

    {
      _id: "bbca5d6a-2156-41c4-89da-0329e8c99a4f",  // Meteor.userId()
      username: "cool_kid_13", // unique name
      emails: [
        // each email address can only belong to one user.
        { address: "cool@example.com", verified: true },
        { address: "another@different.com", verified: false }
      ],
      createdAt: 1349761684042,
      profile: {
        // The profile is writable by the user by default.
        name: "Joe Schmoe"
      },
      services: {
        facebook: {
          id: "709050", // facebook id
          accessToken: "AAACCgdX7G2...AbV9AZDZD"
        },
        resume: {
          loginTokens: [
            { token: "97e8c205-c7e4-47c9-9bea-8e2ccc0694cd",
              when: 1349761684048 }
          ]
        }
      }
    }

A user document can contain any data you want to store about a user. Meteor
treats the following fields specially:

- `username`: a unique String identifying the user.
- `emails`: an Array of Objects with keys `address` and `verified`;
  an email address may belong to at most one user. `verified` is
  a Boolean which is true if the user has [verified the
  address](#accounts_verifyemail) with a token sent over email.
- `createdAt`: a numeric timestamp (milliseconds since January 1 1970)
   of the time the user document was created.
- `profile`: an Object which (by default) the user can create
  and update with any data.
- `services`: an Object containing data used by particular
  login services. For example, its `reset` field contains
  tokens used by [forgot password](#accounts_forgotpassword) links,
  and its `resume` field contains tokens used to keep you
  logged in between sessions.

Like all [Meteor.Collection](#collections)s, you can access all
documents on the server, but only those specifically published by the server are
available on the client.

By default, the current user's `username`, `emails` and `profile` are
published to the client. You can publish additional fields for the
current user with:

    Meteor.publish("userData", function () {
      return Meteor.users.find({_id: this.userId},
                               {fields: {'other': 1, 'things': 1}});
    });

If the autopublish package is installed, information about all users
on the system is published to all clients. This includes `username`,
`profile`, and any fields in `services` that are meant to be public
(eg `services.facebook.id`,
`services.twitter.screenName`). Additionally, when using autopublish
more information is published for the currently logged in user,
including access tokens. This allows making API calls directly from
the client for services that allow this.

Users are by default allowed to specify their own `profile` field with
[`Accounts.createUser`](#accounts_createuser) and modify it with
`Meteor.users.update`. To allow users to edit additional fields, use
[`Meteor.users.allow`](#allow). To forbid users from making any modifications to
their user document:

    Meteor.users.deny({update: function () { return true; }});


{{> api_box loggingIn}}

For example, [the `accounts-ui` package](#accountsui) uses this to display an
animation while the login request is being processed.

{{> api_box logout}}

{{> api_box loginWithPassword}}

This function is provided by the `accounts-password` package. See the
[Passwords](#accounts_passwords) section below.


{{> api_box loginWithExternalService}}

These functions initiate the login process with an external
service (eg: Facebook, Google, etc), using OAuth. When called they open a new pop-up
window that loads the provider's login page. Once the user has logged in
with the provider, the pop-up window is closed and the Meteor client
logs in to the Meteor server with the information provided by the external
service.

<a id="requestpermissions" name="requestpermissions" />

In addition to identifying the user to your application, some services
have APIs that allow you to take action on behalf of the user. To
request specific permissions from the user, pass the
`requestPermissions` option the login function. This will cause the user
to be presented with an additional page in the pop-up dialog to permit
access to their data. The user's `accessToken` &mdash; with permissions
to access the service's API &mdash; is stored in the `services` field of
the user document. The supported values for `requestPermissions` differ
for each login service and are documented on their respective developer
sites:

- Facebook: <http://developers.facebook.com/docs/authentication/permissions/>
- GitHub: <http://developer.github.com/v3/oauth/#scopes>
- Google: <https://developers.google.com/accounts/docs/OAuth2Login#scopeparameter>
- Meetup: <http://www.meetup.com/meetup_api/auth/#oauth2-scopes>
- Twitter, Weibo: `requestPermissions` currently not supported

External login services typically require registering and configuring
your application before use. The easiest way to do this is with the
[`accounts-ui` package](#accountsui) which presents a step-by-step guide
to configuring each service. However, the data can be also be entered
manually in the `Accounts.loginServiceConfiguration` collection. For
example:

    // first, remove configuration entry in case service is already configured
    Accounts.loginServiceConfiguration.remove({
      service: "weibo"
    });
    Accounts.loginServiceConfiguration.insert({
      service: "weibo",
      clientId: "1292962797",
      secret: "75a730b58f5691de5522789070c319bc"
    });


Each external service has its own login provider package and login function. For
example, to support GitHub login, run `$ meteor add accounts-github` and use the
`Meteor.loginWithGithub` function:

    Meteor.loginWithGithub({
      requestPermissions: ['user', 'public_repo']
    }, function (err) {
      if (err)
        Session.set('errorMessage', err.reason || 'Unknown error');
    });



{{> api_box currentUser}}
{{> api_box loggingInTemplate}}
{{> api_box accounts_config}}
{{> api_box accounts_ui_config}}

Example:

    Accounts.ui.config({
      requestPermissions: {
        facebook: ['user_likes'],
        github: ['user', 'repo']
      },
      requestOfflineToken: {
        google: true
      },
      passwordSignupFields: 'USERNAME_AND_OPTIONAL_EMAIL'
    });

{{> api_box accounts_validateNewUser}}

This can be called multiple times. If any of the functions return `false` or
throw an error, the new user creation is aborted. To set a specific error
message (which will be displayed by [`accounts-ui`](#accountsui)), throw a new
[`Meteor.Error`](#meteor_error).

Example:

    // Validate username, sending a specific error message on failure.
    Accounts.validateNewUser(function (user) {
      if (user.username && user.username.length >= 3)
        return true;
      throw new Meteor.Error(403, "Username must have at least 3 characters");
    });
    // Validate username, without a specific error message.
    Accounts.validateNewUser(function (user) {
      return user.username !== "root";
    });

{{> api_box accounts_onCreateUser}}

Use this when you need to do more than simply accept or reject new user
creation. With this function you can programatically control the
contents of new user documents.

The function you pass will be called with two arguments: `options` and
`user`. The `options` argument comes
from [`Accounts.createUser`](#accounts_createuser) for
password-based users or from an external service login flow. `options` may come
from an untrusted client so make sure to validate any values you read from
it. The `user` argument is created on the server and contains a
proposed user object with all the automatically generated fields
required for the user to log in.

The function should return the user document (either the one passed in or a
newly-created object) with whatever modifications are desired. The returned
document is inserted directly into the [`Meteor.users`](#meteor_users) collection.

The default create user function simply copies `options.profile` into
the new user document. Calling `onCreateUser` overrides the default
hook. This can only be called once.

Example:

<!-- XXX replace d6 with _.random once we have underscore 1.4.2 -->

    // Support for playing D&D: Roll 3d6 for dexterity
    Accounts.onCreateUser(function(options, user) {
      var d6 = function () { return Math.floor(Random.fraction() * 6) + 1; };
      user.dexterity = d6() + d6() + d6();
      // We still want the default hook's 'profile' behavior.
      if (options.profile)
        user.profile = options.profile;
      return user;
    });


<h2 id="accounts_passwords"><span>Passwords</span></h2>

The `accounts-password` package contains a full system for password-based
authentication. In addition to the basic username and password-based
sign-in process, it also supports email-based sign-in including
address verification and password recovery emails.

Unlike most web applications, the Meteor client does not send the user's
password directly to the server. It uses the [Secure Remote Password
protocol](http://en.wikipedia.org/wiki/Secure_Remote_Password_protocol)
to ensure the server never sees the user's plain-text password. This
helps protect against embarrassing password leaks if the server's
database is compromised.

To add password support to your application, run `$ meteor add
accounts-password`. You can construct your own user interface using the
functions below, or use the [`accounts-ui` package](#accountsui) to
include a turn-key user interface for password-based sign-in.


{{> api_box accounts_createUser}}

On the client, this function logs in as the newly created user on
successful completion. On the server, it returns the newly created user
id.

On the client, you must pass `password` and one of `username` or `email`
&mdash; enough information for the user to be able to log in again
later. On the server, you can pass any subset of these options, but the
user will not be able to log in until it has an identifier and a
password.

To create an account without a password on the server and still let the
user pick their own password, call `createUser` with the `email` option
and then
call [`Accounts.sendEnrollmentEmail`](#accounts_sendenrollmentemail). This
will send the user an email with a link to set their initial password.

By default the `profile` option is added directly to the new user document. To
override this behavior, use [`Accounts.onCreateUser`](#accounts_oncreateuser).

This function is only used for creating users with passwords. The external
service login flows do not use this function.


{{> api_box accounts_changePassword}}

{{> api_box accounts_forgotPassword}}

This triggers a call
to [`Accounts.sendResetPasswordEmail`](#accounts_sendresetpasswordemail)
on the server. Pass the token the user receives in this email
to [`Accounts.resetPassword`](#accounts_resetpassword) to
complete the password reset process.

If you are using the [`accounts-ui` package](#accountsui), this is handled
automatically. Otherwise, it is your responsiblity to prompt the user for the
new password and call `resetPassword`.

{{> api_box accounts_resetPassword}}

This function accepts tokens generated
by [`Accounts.sendResetPasswordEmail`](#accounts_sendresetpasswordemail)
and
[`Accounts.sendEnrollmentEmail`](#accounts_sendenrollmentemail).

{{> api_box accounts_setPassword}}

{{> api_box accounts_verifyEmail}}

This function accepts tokens generated
by [`Accounts.sendVerificationEmail`](#accounts_sendverificationemail). It
sets the `emails.verified` field in the user record.

{{> api_box accounts_sendResetPasswordEmail}}

The token in this email should be passed
to [`Accounts.resetPassword`](#accounts_resetpassword).

To customize the contents of the email, see
[`Accounts.emailTemplates`](#accounts_emailtemplates).

{{> api_box accounts_sendEnrollmentEmail}}

The token in this email should be passed
to [`Accounts.resetPassword`](#accounts_resetpassword).

To customize the contents of the email, see
[`Accounts.emailTemplates`](#accounts_emailtemplates).

{{> api_box accounts_sendVerificationEmail}}

The token in this email should be passed
to [`Accounts.verifyEmail`](#accounts_verifyemail).

To customize the contents of the email, see
[`Accounts.emailTemplates`](#accounts_emailtemplates).

{{> api_box accounts_emailTemplates}}

This is an `Object` with several fields that are used to generate text
for the emails sent by `sendResetPasswordEmail`, `sendEnrollmentEmail`,
and `sendVerificationEmail`.

Override fields of the object by assigning to them:

- `from`: A `String` with an [RFC5322](http://tools.ietf.org/html/rfc5322) From
   address. By default, the email is sent from `no-reply@meteor.com`. If you
   wish to receive email from users asking for help with their account, be sure
   to set this to an email address that you can receive email at.
- `siteName`: The public name of your application. Defaults to the DNS name of
   the application (eg: `awesome.meteor.com`).
- `resetPassword`: An `Object` with two fields:
 - `resetPassword.subject`: A `Function` that takes a user object and returns
   a `String` for the subject line of a reset password email.
 - `resetPassword.text`: A `Function` that takes a user object and a url, and
   returns the body text for a reset password email.
- `enrollAccount`: Same as `resetPassword`, but for initial password setup for
   new accounts.
- `verifyEmail`: Same as `resetPassword`, but for verifying the users email
   address.


Example:

    Accounts.emailTemplates.siteName = "AwesomeSite";
    Accounts.emailTemplates.from = "AwesomeSite Admin <accounts@example.com>";
    Accounts.emailTemplates.enrollAccount.subject = function (user) {
        return "Welcome to Awesome Town, " + user.profile.name;
    };
    Accounts.emailTemplates.enrollAccount.text = function (user, url) {
       return "You have been selected to participate in building a better future!"
         + " To activate your account, simply click the link below:\n\n"
         + url;
    };


<h2 id="templates_api"><span>Templates</span></h2>

A template that you declare as `<{{! }}template name="foo"> ... </{{!
}}template>` can be accessed as the function `Template.foo`, which
returns a string of HTML when called.

The same template may occur many times on the page, and these
occurrences are called template instances.  Template instances have a
life cycle of being created, put into the document, and later taken
out of the document and destroyed.  Meteor manages these stages for
you, including determining when a template instance has been removed
or replaced and should be cleaned up.  You can associate data with a
template instance, and you can access its DOM nodes when it is in the
document.

Additionally, Meteor will maintain a template instance and its state
even if its surrounding HTML is re-rendered into new DOM nodes.  As
long as the structure of template invocations is the same, Meteor will
not consider any instances to have been created or destroyed.  You can
request that the same DOM nodes be retained as well using `preserve`
and `constant`.

There are a number of callbacks and directives that you can specify on
a named template and that apply to all instances of the template.
They are described below.

{{> api_box template_call}}

When called inside a template helper, the body of `Meteor.render`, or
other settings where reactive HTML is being generated, the resulting
HTML is annotated so that it renders as reactive DOM elements.
Otherwise, the HTML is unadorned and static.


{{> api_box template_rendered}}

This callback is called once when an instance of Template.*myTemplate* is
rendered into DOM nodes and put into the document for the first time, and again
each time any part of the template is re-rendered.

In the body of the callback, `this` is a [template
instance](#template_inst) object that is unique to this occurrence of
the template and persists across re-renderings.  Use the `created` and
`destroyed` callbacks to perform initialization or clean-up on the
object.

{{> api_box template_created}}

This callback is called when an invocation of *myTemplate* represents
a new occurrence of the template and not a re-rendering of an existing
template instance.  Inside the callback, `this` is the new [template
instance](#template_inst) object.  Properties you set on this object
will be visible from the `rendered` and `destroyed` callbacks and from
event handlers.

This callback fires once and is the first callback to fire.  Every
`created` has a corresponding `destroyed`; that is, if you get a
`created` callback with a certain template instance object in `this`,
you will eventually get a `destroyed` callback for the same object.

{{> api_box template_destroyed}}

This callback is called when an occurrence of a template is taken off
the page for any reason and not replaced with a re-rendering.  Inside
the callback, `this` is the [template instance](#template_inst) object
being destroyed.

This callback is most useful for cleaning up or undoing any external
effects of `created`.  It fires once and is the last callback to fire.


{{> api_box template_events}}

Declare event handers for instances of this template. Multiple calls add
new event handlers in addition to the existing ones.

See [Event Maps](#eventmaps) for a detailed description of the event
map format and how event handling works in Meteor.

{{> api_box template_helpers}}

Each template has a local dictionary of helpers that are made available to it,
and this call specifies helpers to add to the template's dictionary.

Example:

    Template.myTemplate.helpers({
      foo: function () {
        return Session.get("foo");
      }
    });

In Handlebars, this helper would then be invoked as `{{foo}}`.

The following syntax is equivalent, but won't work for reserved property
names:

    Template.myTemplate.foo = function () {
      return Session.get("foo");
    };

{{> api_box template_preserve}}

You can "preserve" a DOM element during re-rendering, leaving the
existing element in place in the document while replacing the
surrounding HTML.  This means that re-rendering a template need not
disturb text fields, iframes, and other sensitive elements it
contains.  The elements to preserve must be present both as nodes in
the old DOM and as tags in the new HTML.  Meteor will patch the DOM
around the preserved elements.

<div class="note">
By default, new Meteor apps automatically include the
`preserve-inputs` package.  This preserves all elements of type
`input`, `textarea`, `button`, `select`, and `option` that have unique
`id` attributes or that have `name` attributes that are unique within
an enclosing element with an `id` attribute.  To turn off this default
behavior, simply remove the `preserve-inputs` package.
</div>

Preservation is useful in a variety of cases where replacing a DOM
element with an identical or modified element would not have the same
effect as retaining the original element.  These include:

* Input text fields and other form controls
* Elements with CSS animations
* Iframes
* Nodes with references kept in JavaScript code

If you want to preserve a whole region of the DOM, an element and its
children, or nodes not rendered by Meteor, use a [constant
region](#constant) instead.

To preserve nodes, pass a list of selectors, each of which should match
at most one element in the template.  When the template is re-rendered,
the selector is run on the old DOM and the new DOM, and Meteor will
reuse the old element in place while working in any HTML changes around
it.

A second form of `preserve` takes a labeling function for each selector
and allows the selectors to match multiple nodes. The node-labeling
function takes a node and returns a label string that is unique for each
node, or `false` to exclude the node from preservation.

For example, to preserve all `<input>` elements with ids in template 'foo', use:

    Template.foo.preserve({
      'input[id]': function (node) { return node.id; }
    });

Selectors are interpreted as rooted at the top level of the template.
Each occurrence of the template operates independently, so the selectors
do not have to be unique on the entire page, only within one occurrence
of the template.  Selectors will match nodes even if they are in
sub-templates.

Preserving a node does *not* preserve its attributes or contents. They
will be updated to reflect the new HTML. Text in input fields is not
preserved unless the input field has focus, in which case the cursor and
selection are left intact. Iframes retain their navigation state and
animations continue to run as long as their parameters haven't changed.

There are some cases where nodes can not be preserved because of
constraints inherent in the DOM API. For example, an element's tag name
can't be changed, and it can't be moved relative to its parent or other
preserved nodes.  For this reason, nodes that are re-ordered or
re-parented by an update will not be preserved.

<div class="note">
Previous versions of Meteor had an implicit page-wide `preserve`
directive that labeled nodes by their "id" and "name" attributes.
This has been removed in favor of the explicit, opt-in mechanism.
</div>


<h2 id="template_inst"><span>Template instances</span></h2>

A template instance object represents an occurrence of a template in
the document.  It can be used to access the DOM and it can be
assigned properties that persist across page re-renderings.

Template instance objects are found as the value of `this` in the
`created`, `rendered`, and `destroyed` template callbacks and as an
argument to event handlers.

In addition to the properties and functions described below, you can
assign additional properties of your choice to the object.  Property names
starting with `_` are guaranteed to be available for your use.  Use
the `created` and `destroyed` callbacks to perform initialization or
clean-up on the object.

You can only access `findAll`, `find`, `firstNode`, and `lastNode`
from the `rendered` callback and event handlers, not from `created`
and `destroyed`, because they require the template instance to be
in the DOM.

{{> api_box template_findAll}}

Returns an array of DOM elements matching `selector`.

The template instance serves as the document root for the selector. Only
elements inside the template and its sub-templates can match parts of
the selector.

{{> api_box template_find}}

Returns one DOM element matching `selector`, or `null` if there are no
such elements.

The template instance serves as the document root for the selector. Only
elements inside the template and its sub-templates can match parts of
the selector.

{{> api_box template_firstNode}}

The two nodes `firstNode` and `lastNode` indicate the extent of the
rendered template in the DOM.  The rendered template includes these
nodes, their intervening siblings, and their descendents.  These two
nodes are siblings (they have the same parent), and `lastNode` comes
after `firstNode`, or else they are the same node.

{{> api_box template_lastNode}}

{{> api_box template_data}}

This property provides access to the data context at the top level of
the template.  It is updated each time the template is re-rendered.
Access is read-only and non-reactive.


{{> api_box render}}

`Meteor.render` creates a `DocumentFragment` (a sequence of DOM nodes)
that automatically updates in realtime. Most Meteor apps don't need to
call this directly; they use templates and Meteor handles the rendering.

Pass in `htmlFunc`, a function that returns an HTML
string. `Meteor.render` calls the function and turns the output into
DOM nodes. Meanwhile, it tracks the data that was used when `htmlFunc`
ran, and automatically wires up callbacks so that whenever any of the
data changes, `htmlFunc` is re-run and the DOM nodes are updated in
place.

You may insert the returned `DocumentFragment` directly into the DOM
wherever you would like it to appear. The inserted nodes will continue
to update until they are taken off the screen. Then they will be
automatically cleaned up. For more details about clean-up, see
[`Deps.flush`](#deps_flush).

`Meteor.render` tracks the data dependencies of `htmlFunc` by running
it in a reactive computation, so it can respond to changes in any reactive
data sources used by that function. For more information, or to learn
how to make your own reactive data sources, see
[Reactivity](#reactivity).

Example:

    // Client side: show the number of players online.
    var frag = Meteor.render(function () {
      return "<p>There are " + Players.find({online: true}).count() +
        " players online.</p>";
    });
    document.body.appendChild(frag);

    // Server side: find all players that have been idle for a while,
    // and mark them as offline. The count on the screen will
    // automatically update on all clients.
    Players.update({idleTime: {$gt: 30}}, {$set: {online: false}});

{{> api_box renderList}}

Creates a `DocumentFragment` that automatically updates as the results
of a database query change. Most Meteor apps use `{{#each}}` in
a template instead of calling this directly.

`renderList` is more efficient than using `Meteor.render` to render HTML
for a list of documents.  For example, if a new document is created in
the database that matches the query, a new item will be rendered and
inserted at the appropriate place in the DOM without re-rendering the
other elements.  Similarly, if a document changes position in a sorted
query, the DOM nodes will simply be moved and not re-rendered.

`docFunc` is called as needed to generate HTML for each document.  If
you provide `elseFunc`, then whenever the query returns no results, it
will be called to render alternate content. You might use this to show
a message like "No records match your query."

Each call to `docFunc` or `elseFunc` is run in its own reactive
computation so that if it has other external data dependencies, it will be
individually re-run when the data changes.

Example:

    // List the titles of all of the posts that have the tag
    // "frontpage". Keep the list updated as new posts are made, as tags
    // change, etc.  Display the selected post differently.
    var frag = Meteor.renderList(
      Posts.find({tags: "frontpage"}),
      function(post) {
        var style = Session.equals("selectedId", post._id) ? "selected" : "";
        // A real app would need to quote/sanitize post.name
        return '<div class="' + style + '">' + post.name + '</div>';
      });
    document.body.appendChild(frag);

    // Select a post.  This will cause only the selected item and the
    // previously selected item to update.
    var somePost = Posts.findOne({tags: "frontpage"});
    Session.set("selectedId", somePost._id);


{{#api_box_inline eventmaps}}

Several functions take event maps. An event map is an object where
the properties specify a set of events to handle, and the values are
the handlers for those events. The property can be in one of several
forms:

<dl>
{{#dtdd "<em>eventtype</em>"}}
Matches a particular type of event, such as 'click'.
{{/dtdd}}

{{#dtdd "<em>eventtype selector</em>"}}
Matches a particular type of event, but only when it appears on
an element that matches a certain CSS selector.
{{/dtdd}}

{{#dtdd "<em>event1, event2</em>"}}
To handle more than one type of event with the same function, use a
comma-separated list.
{{/dtdd}}
</dl>

The handler function receives two arguments: `event`, an object with
information about the event, and `template`, a [template
instance](#template_inst) for the template where the handler is
defined.  The handler also receives some additional context data in
`this`, depending on the context of the current element handling the
event.  In a Handlebars template, an element's context is the
Handlebars data context where that element occurs, which is set by
block helpers such as `#with` and `#each`.

Example:

    {
      // Fires when any element is clicked
      'click': function (event) { ... },

      // Fires when any element with the 'accept' class is clicked
      'click .accept': function (event) { ... },

      // Fires when 'accept' is clicked, or a key is pressed
      'keydown, click .accept': function (event) { ... }
    }

Most events bubble up the document tree from their originating
element.  For example, `'click p'` catches a click anywhere in a
paragraph, even if the click originated on a link, span, or some other
element inside the paragraph.  The originating element of the event
is available as the `target` property, while the element that matched
the selector and is currently handling it is called `currentTarget`.

    {
      'click p': function (event) {
        var paragraph = event.currentTarget; // always a P
        var clickedElement = event.target; // could be the P or a child element
      }
    }

If a selector matches multiple elements that an event bubbles to, it
will be called multiple times, for example in the case of `'click
div'` or `'click *'`.  If no selector is given, the handler
will only be called once, on the original target element.

The following properties and methods are available on the event object
passed to handlers:

<dl class="objdesc">
{{#dtdd name="type" type="String"}}
The event's type, such as "click", "blur" or "keypress".
{{/dtdd}}

{{#dtdd name="target" type="DOM Element"}}
The element that originated the event.
{{/dtdd}}

{{#dtdd name="currentTarget" type="DOM Element"}}
The element currently handling the event.  This is the element that
matched the selector in the event map.  For events that bubble, it may
be `target` or an ancestor of `target`, and its value changes as the
event bubbles.
{{/dtdd}}

{{#dtdd name="which" type="Number"}}
For mouse events, the number of the mouse button (1=left, 2=middle, 3=right).
For key events, a character or key code.
{{/dtdd}}

{{#dtdd "stopPropagation()"}}
Prevent the event from propagating (bubbling) up to other elements.
Other event handlers matching the same element are still fired, in
this and other event maps.
{{/dtdd}}

{{#dtdd "stopImmediatePropagation()"}}
Prevent all additional event handlers from being run on this event,
including other handlers in this event map, handlers reached by
bubbling, and handlers in other event maps.
{{/dtdd}}

{{#dtdd "preventDefault()"}}
Prevents the action the browser would normally take in response to this
event, such as following a link or submitting a form.  Further handlers
are still called, but cannot reverse the effect.
{{/dtdd}}

{{#dtdd "isPropagationStopped()"}}
Returns whether `stopPropagation()` has been called for this event.
{{/dtdd}}

{{#dtdd "isImmediatePropagationStopped()"}}
Returns whether `stopImmediatePropagation()` has been called for this event.
{{/dtdd}}

{{#dtdd "isDefaultPrevented()"}}
Returns whether `preventDefault()` has been called for this event.
{{/dtdd}}
</dl>

Returning `false` from a handler is the same as calling
both `stopImmediatePropagation` and `preventDefault` on the event.

Event types and their uses include:

<dl class="objdesc">
{{#dtdd "<code>click</code>"}}
Mouse click on any element, including a link, button, form control, or div.
Use `preventDefault()` to prevent a clicked link from being followed.
Some ways of activating an element from the keyboard also fire `click`.
{{/dtdd}}

{{#dtdd "<code>dblclick</code>"}}
Double-click.
{{/dtdd}}

{{#dtdd "<code>focus, blur</code>"}}
A text input field or other form control gains or loses focus.  You
can make any element focusable by giving it a `tabindex` property.
Browsers differ on whether links, checkboxes, and radio buttons are
natively focusable.  These events do not bubble.
{{/dtdd}}

{{#dtdd "<code>change</code>"}}
A checkbox or radio button changes state.  For text fields, use
`blur` or key events to respond to changes.
{{/dtdd}}

{{#dtdd "<code>mouseenter, mouseleave</code>"}} The pointer enters or
leaves the bounds of an element.  These events do not bubble.
{{/dtdd}}

{{#dtdd "<code>mousedown, mouseup</code>"}}
The mouse button is newly down or up.
{{/dtdd}}

{{#dtdd "<code>keydown, keypress, keyup</code>"}}
The user presses a keyboard key.  `keypress` is most useful for
catching typing in text fields, while `keydown` and `keyup` can be
used for arrow keys or modifier keys.
{{/dtdd}}

{{#dtdd "<code>tap</code>"}} Tap on an element.  On touch-enabled
devices, this is a replacement to `click` that fires immediately.
These events are synthesized from `touchmove` and `touchend`.
{{/dtdd}}

</dl>

Other DOM events are available as well, but for the events above,
Meteor has taken some care to ensure that they work uniformly in all
browsers.

{{/api_box_inline}}



{{#api_box_inline constant}}

You can mark a region of a template as "constant" and not subject to
re-rendering using the
`{{#constant}}...{{/constant}}` block helper.
Content inside the `#constant` block helper is preserved exactly as-is
even if the enclosing template is re-rendered.  Changes to other parts
of the template are patched in around it in the same manner as
`preserve`.  Unlike individual node preservation, a constant region
retains not only the identities of its nodes but also their attributes
and contents.  The contents of the block will only be evaluated once
per occurrence of the enclosing template.

Constant regions allow non-Meteor content to be embedded in a Meteor
template.  Many third-party widgets create and manage their own DOM
nodes programmatically. Typically, you put an empty element in your
template, which the widget or library will then populate with
children. Normally, when Meteor re-renders the enclosing template it
would remove the new children, since the template says it should be
empty. If the container is wrapped in a `#constant` block, however, it
is left alone; whatever content is currently in the DOM remains.

<div class="note">
Constant regions are intended for embedding non-Meteor content.
Event handlers and reactive dependencies don't currently work
correctly inside constant regions.
</div>


{{/api_box_inline}}

{{#api_box_inline isolate}}

Each template runs as its own reactive computation.  When the template
accesses a reactive data source, such as by calling `Session.get` or
making a database query, this establishes a data dependency that will
cause the whole template to be re-rendered when the data changes.
This means that the amount of re-rendering for a particular change
is affected by how you've divided your HTML into templates.

Typically, the exact extent of re-rendering is not crucial, but if you
want more control, such as for performance reasons, you can use the
`{{#isolate}}...{{/isolate}}` helper.  Data
dependencies established inside an `#isolate` block are localized to
the block and will not in themselves cause the parent template to be
re-rendered.  This block helper essentially conveys the reactivity
benefits you would get by pulling the content out into a new
sub-template.

{{/api_box_inline}}

<h2 id="match"><span>Match</span></h2>

Meteor methods and publish functions take arbitrary [EJSON](#ejson) types as
arguments, but most arguments are expected to be of a particular type. Meteor's
`check` package is a lightweight library for checking that arguments and other
values are of the expected type. For example:

    Meteor.publish("chats-in-room", function (roomId) {
      // Make sure roomId is a string, not an arbitrary mongo selector object.
      check(roomId, String);
      return Chats.find({room: roomId});
    });

    Meteor.methods({addChat: function (roomId, message) {
      check(roomId, String);
      check(message, {
        text: String,
        timestamp: Date,
        // Optional, but if present must be an array of strings.
        tags: Match.Optional([String])
      });

      // ... do something with the message ...
    }});

{{> api_box check}}

If the match fails, `check` throws a `Match.Error` describing how it failed. If
this error gets sent over the wire to the client, it will appear only as
`Meteor.Error(400, "Match Failed")`; the failure details will be written to the
server logs but not revealed to the client.

{{> api_box match_test}}

{{#api_box_inline matchpatterns}}

The following patterns can be used as pattern arguments to `check` and `Match.test`:


<dl>
{{#dtdd "<code>Match.Any</code>"}}
Matches any value.
{{/dtdd}}

{{#dtdd "<code>String</code>, <code>Number</code>, <code>Boolean</code>, <code>undefined</code>, <code>null</code>"}}
Matches a primitive of the given type.
{{/dtdd}}

{{#dtdd "<code>Match.Integer</code>"}}
Matches a signed 32-bit integer. Doesn't match `Infinity`, `-Infinity`, or `NaN`.
{{/dtdd}}

{{#dtdd "<code>[<em>pattern</em>]</code>"}}
A one-element array matches an array of elements, each of which match
*pattern*. For example, `[Number]` matches a (possibly empty) array of numbers;
`[Match.Any]` matches any array.
{{/dtdd}}

{{#dtdd "<code>{<em>key1</em>: <em>pattern1</em>, <em>key2</em>: <em>pattern2</em>, ...}</code>"}}
Matches an Object with the given keys, with values matching the given patterns.
If any *pattern* is a `Match.Optional`, that key does not need to exist
in the object. The value may not contain any keys not listed in the pattern.
The value must be a plain Object with no special prototype.
{{/dtdd}}

{{#dtdd "<code>Match.ObjectIncluding({<em>key1</em>: <em>pattern1</em>, <em>key2</em>: <em>pattern2</em>, ...})</code>"}}
Matches an Object with the given keys; the value may also have other keys
with arbitrary values.
{{/dtdd}}

{{#dtdd "<code>Object</code>"}}
Matches any plain Object with any keys; equivalent to
`Match.ObjectIncluding({})`.
{{/dtdd}}

{{#dtdd "<code>Match.Optional(<em>pattern</em>)</code>"}} Matches either
`undefined` or something that matches pattern. If used in an object this matches
only if the key is not set as opposed to the value being set to `undefined`.

    // In an object
    var pat = { name: Match.Optional(String) };
    check({ name: "something" }, pat) // OK
    check({}, pat) // OK
    check({ name: undefined }, pat) // Throws an exception

    // Outside an object
    check(undefined, Match.Optional(String)); // OK

{{/dtdd}}

{{#dtdd "<code>Match.OneOf(<em>pattern1</em>, <em>pattern2</em>, ...)</code>"}}
Matches any value that matches at least one of the provided patterns.
{{/dtdd}}

{{#dtdd "Any constructor function (eg, <code>Date</code>)"}}
Matches any element that is an instance of that type.
{{/dtdd}}

{{#dtdd "<code>Match.Where(<em>condition</em>)</code>"}}
Calls the function *condition* with the value as the argument. If *condition*
returns true, this matches. If *condition* throws a `Match.Error` or returns
false, this fails. If *condition* throws any other error, that error is thrown
from the call to `check` or `Match.test`. Examples:

    check(buffer, Match.Where(EJSON.isBinary));

    NonEmptyString = Match.Where(function (x) {
      check(x, String);
      return x.length > 0;
    }
    check(arg, NonEmptyString);
{{/dtdd}}
</dl>

{{/api_box_inline}}

<h2 id="timers"><span>Timers</span></h2>

Meteor uses global environment variables
to keep track of things like the current request's user.  To make sure
these variables have the right values, you need to use
`Meteor.setTimeout` instead of `setTimeout` and `Meteor.setInterval`
instead of `setInterval`.

These functions work just like their native JavaScript equivalents.
You'll get an error if you call the native function.

{{> api_box setTimeout}}

Returns a handle that can be used by `Meteor.clearTimeout`.

{{> api_box setInterval}}

Returns a handle that can be used by `Meteor.clearInterval`.

{{> api_box clearTimeout}}
{{> api_box clearInterval}}

<h2 id="deps"><span>Deps</span></h2>

Meteor has a simple dependency tracking system which allows it to
automatically rerun templates and other computations whenever
[`Session`](#session) variables, database queries, and other data
sources change.

Unlike most other systems, you don't have to manually declare these
dependencies &mdash; it "just works". The mechanism is simple and
efficient. When you call a function that supports reactive updates
(such as a database query), it automatically saves the current
Computation object, if any (representing, for example, the current
template being rendered).  Later, when the data changes, the function
can "invalidate" the Computation, causing it to rerun (rerendering the
template).

Applications will find [`Deps.autorun`](#deps_autorun) useful, while more
advanced facilities such as `Deps.Dependency` and `onInvalidate`
callbacks are intended primarily for package authors implementing new
reactive data sources.

{{> api_box deps_autorun }}

`Deps.autorun` allows you to run a function that depends on reactive data
sources, in such a way that if there are changes to the data later,
the function will be rerun.

For example, you can monitor a cursor (which is a reactive data
source) and aggregate it into a session variable:

    Deps.autorun(function () {
      var oldest = _.max(Monkeys.find().fetch(), function (monkey) {
        return monkey.age;
      });
      if (oldest)
        Session.set("oldest", oldest.name);
    });

Or you can wait for a session variable to have a certain value, and do
something the first time it does, calling `stop` on the computation to
prevent further rerunning:

    Deps.autorun(function (c) {
      if (! Session.equals("shouldAlert", true))
        return;

      c.stop();
      alert("Oh no!");
    });

The function is invoked immediately, at which point it may alert and
stop right away if `shouldAlert` is already true.  If not, the
function is run again when `shouldAlert` becomes true.

A change to a data dependency does not cause an immediate rerun, but
rather "invalidates" the computation, causing it to rerun the next
time a flush occurs.  A flush will occur automatically as soon as
the system is idle if there are invalidated computations.  You can
also use [`Deps.flush`](#deps_flush) to cause an immediate flush of
all pending reruns.

If you nest calls to `Deps.autorun`, then when the outer call stops or
reruns, the inner call will stop automatically.  Subscriptions and
observers are also automatically stopped when used as part of a
computation that is rerun, allowing new ones to be established.  See
[`Meteor.subscribe`](#meteor_subscribe) for more information about
subscriptions and reactivity.

If the initial run of an autorun throws an exception, the computation
is automatically stopped and won't be rerun.

{{> api_box deps_flush }}

Normally, when you make changes (like writing to the database),
their impact (like updating the DOM) is delayed until the system is
idle. This keeps things predictable &mdash; you can know that the DOM
won't go changing out from under your code as it runs. It's also one
of the things that makes Meteor fast.

`Deps.flush` forces all of the pending reactive updates to complete.
For example, if an event handler changes a Session
variable that will cause part of the user interface to rerender, the
handler can call `flush` to perform the rerender immediately and then
access the resulting DOM.

An automatic flush occurs whenever the system is idle which performs
exactly the same work as `Deps.flush`.  The flushing process consists
of rerunning any invalidated computations.  If additional
invalidations happen while flushing, they are processed as part of the
same flush until there is no more work to be done.  Callbacks
registered with [`Meteor.afterFlush`](#deps_afterflush) are called
after processing outstanding invalidations.

Any auto-updating DOM elements that are found to not be in the
document during a flush may be cleaned up by Meteor (meaning that
Meteor will stop tracking and updating the elements, so that the
browser's garbage collector can delete them).  So, if you manually
call `flush`, you need to make sure that any auto-updating elements
that you have created by calling [`Meteor.render`](#meteor_render)
have already been inserted in the main DOM tree.

It is illegal to call `flush` from inside a `flush` or from a running
computation.

{{> api_box deps_nonreactive }}

Calls `func()` with `Deps.currentComputation` temporarily set to
`null`.  If `func` accesses reactive data sources, these data sources
will never cause a rerun of the enclosing computation.

{{> api_box deps_active }}

This value is useful for data source implementations to determine
whether they are being accessed reactively or not.

{{> api_box deps_currentcomputation }}

It's very rare to need to access `currentComputation` directly.  The
current computation is used implicitly by
[`Deps.active`](#deps_active) (which tests whether there is one),
[`dependency.depend()`](#dependency_depend) (which registers that it depends on a
dependency), and [`Deps.onInvalidate`](#deps_oninvalidate) (which
registers a callback with it).

{{> api_box deps_oninvalidate }}

See [*`computation`*`.onInvalidate`](#computation_oninvalidate) for more
details.

{{> api_box deps_afterflush }}

Functions scheduled by multiple calls to `afterFlush` are guaranteed
to run in the order that `afterFlush` was called.  Functions are
guaranteed to be called at a time when there are no invalidated
computations that need rerunning.  This means that if an `afterFlush`
function invalidates a computation, that computation will be rerun
before any other `afterFlush` functions are called.

<h2 id="deps_computation"><span>Deps.Computation</span></h2>

A Computation object represents code that is repeatedly rerun in
response to reactive data changes.  Computations don't have return
values; they just perform actions, such as rerendering a template on
the screen.  Computations are created using [`Deps.autorun`](#deps_autorun).
Use [`stop`](#computation_stop) to prevent further rerunning of a
computation.

Each time a computation runs, it may access various reactive data
sources that serve as inputs to the computation, which are called its
dependencies.  At some future time, one of these dependencies may
trigger the computation to be rerun by invalidating it.  When this
happens, the dependencies are cleared, and the computation is
scheduled to be rerun at flush time.

The *current computation*
([`Deps.currentComputation`](#deps_currentcomputation)) is the
computation that is currently being run or rerun (computed), and the
one that gains a dependency when a reactive data source is accessed.
Data sources are responsible for tracking these dependencies using
[`Deps.Dependency`](#deps_dependency) objects.

Invalidating a computation sets its `invalidated` property to true
and immediately calls all of the computation's `onInvalidate`
callbacks.  When a flush occurs, if the computation has been invalidated
and not stopped, then the computation is rerun by setting the
`invalidated` property to `false` and calling the original function
that was passed to `Deps.autorun`.  A flush will occur when the current
code finishes running, or sooner if `Deps.flush` is called.

Stopping a computation invalidates it (if it is valid) for the purpose
of calling callbacks, but ensures that it will never be rerun.

Example:

    // if we're in a computation, then perform some clean-up
    // when the current computation is invalidated (rerun or
    // stopped)
    if (Deps.active) {
      Deps.onInvalidate(function () {
        x.destroy();
        y.finalize();
      });
    }

{{> api_box computation_stop}}

Stopping a computation is irreversible and guarantees that it will
never be rerun.  You can stop a computation at any time, including
from the computation's own run function.  Stopping a computation that
is already stopped has no effect.

Stopping a computation causes its `onInvalidate` callbacks to run
immediately if it is not currently invalidated.

Nested computations are stopped automatically when their enclosing
computation is rerun.

{{> api_box computation_invalidate }}

Invalidating a computation marks it to be rerun at
[flush time](#deps_flush), at
which point the computation becomes valid again.  It is rare to
invalidate a computation manually, because reactive data sources
invalidate their calling computations when they change.  Reactive data
sources in turn perform this invalidation using one or more
[`Deps.Dependency`](#deps_dependency) objects.

Invalidating a computation immediately calls all `onInvalidate`
callbacks registered on it.  Invalidating a computation that is
currently invalidated or is stopped has no effect.  A computation can
invalidate itself, but if it continues to do so indefinitely, the
result will be an infinite loop.

{{> api_box computation_oninvalidate }}

`onInvalidate` registers a one-time callback that either fires
immediately or as soon as the computation is next invalidated or
stopped.  It is used by reactive data sources to clean up resources or
break dependencies when a computation is rerun or stopped.

To get a callback after a computation has been recomputed, you can
call [`Deps.afterFlush`](#deps_afterflush) from `onInvalidate`.

{{> api_box computation_stopped }}

{{> api_box computation_invalidated }}

This property is initially false.  It is set to true by `stop()` and
`invalidate()`.  It is reset to false when the computation is
recomputed at flush time.

{{> api_box computation_firstrun }}

This property is a convenience to support the common pattern where a
computation has logic specific to the first run.

<h2 id="deps_dependency"><span>Deps.Dependency</span></h2>

A Dependency represents an atomic unit of reactive data that a
computation might depend on.  Reactive data sources such as Session or
Minimongo internally create different Dependency objects for different
pieces of data, each of which may be depended on by multiple
computations.  When the data changes, the computations are
invalidated.

Dependencies don't store data, they just track the set of computations to
invalidate if something changes.  Typically, a data value will be
accompanied by a Dependency object that tracks the computations that depend
on it, as in this example:

    var weather = "sunny";
    var weatherDep = new Deps.Dependency;

    var getWeather = function () {
      weatherDep.depend()
      return weather;
    };

    var setWeather = function (w) {
      weather = w;
      // (could add logic here to only call changed()
      // if the new value is different from the old)
      weatherDep.changed();
    };

This example implements a weather data source with a simple getter and
setter.  The getter records that the current computation depends on
the `weatherDep` dependency using `depend()`, while the setter
signals the dependency to invalidate all dependent computations by
calling `changed()`.

The reason Dependencies do not store data themselves is that it can be
useful to associate multiple Dependencies with the same piece of data.
For example, one Dependency might represent the result of a database
query, while another might represent just the number of documents in
the result.  A Dependency could represent whether the weather is sunny
or not, or whether the temperature is above freezing.
[`Session.equals`](#session_equals) is implemented this way for
efficiency.  When you call `Session.equals("weather", "sunny")`, the
current computation is made to depend on an internal Dependency that
does not change if the weather goes from, say, "rainy" to "cloudy".

Conceptually, the only two things a Dependency can do are gain a
dependent and change.

A Dependency's dependent computations are always valid (they have
`invalidated === false`).  If a dependent is invalidated at any time,
either by the Dependency itself or some other way, it is immediately
removed.

{{> api_box dependency_changed }}

{{> api_box dependency_depend }}

`dep.depend()` is used in reactive data source implementations to record
the fact that `dep` is being accessed from the current computation.

{{> api_box dependency_hasdependents }}

For reactive data sources that create many internal Dependencies,
this function is useful to determine whether a particular Dependency is
still tracking any dependency relationships or if it can be cleaned up
to save memory.

<h2 id="ejson"><span>EJSON</span></h2>

EJSON is an extension of JSON to support more types. It supports all JSON-safe
types, as well as:

 - **Date** (JavaScript `Date`)
 - **Binary** (JavaScript `Uint8Array` or the
   result of [`EJSON.newBinary`](#ejson_new_binary))
 - **User-defined types** (see [`EJSON.addType`](#ejson_add_type).  For example,
 [`Meteor.Collection.ObjectID`](#collection_object_id) is implemented this way.)

All EJSON serializations are also valid JSON.  For example an object with a date
and a binary buffer would be serialized in EJSON as:

    {
      "d": {"$date": 1358205756553},
      "b": {"$binary": "c3VyZS4="}
    }

Meteor supports all built-in EJSON data types in publishers, method arguments
and results, Mongo databases, and [`Session`](#session) variables.

{{> api_box ejsonParse}}

{{> api_box ejsonStringify}}

{{> api_box ejsonFromJSONValue}}

{{> api_box ejsonToJSONValue}}

{{> api_box ejsonEquals}}

{{> api_box ejsonClone}}

{{> api_box ejsonNewBinary}}

Buffers of binary data are represented by `Uint8Array` instances on JavaScript
platforms that support them.  On implementations of JavaScript that do not
support `Uint8Array`, binary data buffers are represented by standard arrays
containing numbers ranging from 0 to 255, and the `$Uint8ArrayPolyfill` key
set to `true`.

{{> api_box ejsonIsBinary}}

{{> api_box ejsonAddType}}

When you add a type to EJSON, Meteor will be able to use that type in:

 - publishing objects of your type if you pass them to publish handlers.
 - allowing your type in the return values or arguments to
   [methods](#methods_header).
 - storing your type client-side in Minimongo.
 - allowing your type in [`Session`](#session) variables.

<div class="note">

  MongoDB cannot store most user-defined types natively on the server.  Your
  type will work in Minimongo, and you can send it to the client using a custom
  publisher, but MongoDB can only store the types defined in
  [BSON](http://bsonspec.org/).

</div>

Instances of your type should implement the following interface:

{{> api_box ejsonTypeClone}}

{{> api_box ejsonTypeEquals}}

The `equals` method should define an [equivalence
relation](http://en.wikipedia.org/wiki/Equivalence_relation).  It should have
the following properties:

 - *Reflexivity* - for any instance `a`: `a.equals(a)` must be true.
 - *Symmetry* - for any two instances `a` and `b`: `a.equals(b)` if and only if `b.equals(a)`.
 - *Transitivity* - for any three instances `a`, `b`, and `c`: `a.equals(b)` and `b.equals(c)` implies `a.equals(c)`.

{{> api_box ejsonTypeName}}
{{> api_box ejsonTypeToJSONValue}}

For example, the `toJSONValue` method for
[`Meteor.Collection.ObjectID`](#collection_object_id) could be:

    function () {
      return this.toHexString();
    };

<h2 id="http"><span>HTTP</span></h2>

`HTTP` provides an HTTP request API on the client and server.  To use
these functions, add the HTTP package to your project with `$ meteor add
http`.

{{> api_box httpcall}}

This function initiates an HTTP request to a remote server. It returns
a result object with the contents of the HTTP response.  The result
object is detailed below.

On the server, this function can be run either synchronously or
asynchronously.  If the callback is omitted, it runs synchronously,
and the results are returned once the request completes. This is
useful when making server-to-server HTTP API calls from within Meteor
methods, as the method can succeed or fail based on the results of the
synchronous HTTP call.  In this case, consider using
[`this.unblock()`](#method_unblock) to allow other methods to run in
the mean time.  On the client, this function must be used
asynchronously by passing a callback.

Both HTTP and HTTPS protocols are supported.  The `url` argument must be
an absolute URL including protocol and host name on the server, but may be
relative to the current host on the client.  The `query` option
replaces the query string of `url`.  Parameters specified in `params`
that are put in the URL are appended to any query string.
For example, with a `url` of `"/path?query"` and
`params` of `{foo:"bar"}`, the final URL will be `"/path?query&foo=bar"`.

The `params` are put in the URL or the request body, depending on the
type of request.  In the case of request with no bodies, like GET and
HEAD, the parameters will always go in the URL.  For a POST or other
type of request, the parameters will be encoded into the body with a
standard `x-www-form-urlencoded` content type, unless the `content`
or `data` option is used to specify a body, in which case the
parameters will be appended to the URL instead.

The callback receives two arguments, `error` and `result`.  The
`error` argument will contain an Error if the request fails in any
way, including a network error, time-out, or an HTTP status code in
the 400 or 500 range.  In case of a 4xx/5xx HTTP status code, the
`response` property on `error` matches the contents of the result
object.  When run in synchronous mode, either `result` is returned
from the function, or `error` is thrown.

Contents of the result object:

<dl class="objdesc">

<dt><span class="name">statusCode</span>
  <span class="type">Number</span></dt>
<dd>Numeric HTTP result status code, or <code>null</code> on error.</dd>

<dt><span class="name">content</span>
  <span class="type">String</span></dt>
<dd>The body of the HTTP response as a string.</dd>

<dt><span class="name">data</span>
  <span class="type">Object or <code>null</code></span></dt>
<dd>If the response headers indicate JSON content, this contains the body of the document parsed as a JSON object.</dd>

<dt><span class="name">headers</span>
  <span class="type">Object</span></dt>
<dd>A dictionary of HTTP headers from the response.</dd>

</dl>

Example server method:

    Meteor.methods({checkTwitter: function (userId) {
      check(userId, String);
      this.unblock();
      var result = HTTP.call("GET", "http://api.twitter.com/xyz",
                             {params: {user: userId}});
      if (result.statusCode === 200)
         return true
      return false;
    }});

Example asynchronous HTTP call:

    HTTP.call("POST", "http://api.twitter.com/xyz",
              {data: {some: "json", stuff: 1}},
              function (error, result) {
                if (result.statusCode === 200) {
                  Session.set("twizzled", true);
                }
              });


{{> api_box http_get}}
{{> api_box http_post}}
{{> api_box http_put}}
{{> api_box http_del}}


<h2 id="email"><span>Email</span></h2>

The `email` package allows sending email from a Meteor app. To use it, add the
package to your project with `$ meteor add email`.

The server reads from the `MAIL_URL` environment variable to determine how to
send mail. Currently, Meteor supports sending mail over SMTP; the `MAIL_URL`
environment variable should be of the form
`smtp://USERNAME:PASSWORD@HOST:PORT/`. For apps deployed with `meteor deploy`,
`MAIL_URL` defaults to an account (provided by
[Mailgun](http://www.mailgun.com/)) which allows apps to send up to 200 emails
per day; you may override this default by assigning to `process.env.MAIL_URL`
before your first call to `Email.send`.

If `MAIL_URL` is not set (eg, when running your application locally),
`Email.send` outputs the message to standard output instead.

{{> api_box email_send }}

You must provide the `from` option and at least one of `to`, `cc`, and `bcc`;
all other options are optional.

`Email.send` only works on the server. Here is an example of how a
client could use a server method call to send an email. (In an actual
application, you'd need to be careful to limit the emails that a
client could send, to prevent your server from being used as a relay
by spammers.)

    // In your server code: define a method that the client can call
    Meteor.methods({
      sendEmail: function (to, from, subject, text) {
        check([to, from, subject, text], [String]);

        // Let other method calls from the same client start running,
        // without waiting for the email sending to complete.
        this.unblock();

        Email.send({
          to: to,
          from: from,
          subject: subject,
          text: text
        });
      }
    });

    // In your client code: asynchronously send an email
    Meteor.call('sendEmail',
                'alice@example.com',
                'bob@example.com',
                'Hello from Meteor!',
                'This is a test of Email.send.');

{{/better_markdown}}

<h2 id="assets"><span>Assets</span></h2>

{{#better_markdown}}
`Assets` allows server code in a Meteor application to access static server
assets, which are located in the `private` subdirectory of an application's
tree.

{{> api_box assets_getText }}
{{> api_box assets_getBinary }}

Static server assets are included by placing them in the application's `private`
subdirectory. For example, if an application's `private` subdirectory includes a
directory called `nested` with a file called `data.txt` inside it, then server
code can read `data.txt` by running:

    var data = Assets.getText('nested/data.txt');
{{/better_markdown}}

</template>






<template name="api_box">
<div class="api {{bare}}">
<h3 id="{{id}}">
  <a class="name selflink" href="#{{id}}">{{{name}}}</a>
{{#if locus}}
  <span class="locus">{{locus}}</span>
{{/if}}
</h3>

<div class="desc">
{{#each descr}}{{#better_markdown}}{{{this}}}{{/better_markdown}}{{/each}}
</div>

{{#if args}}
<h4>Arguments</h4>
{{> api_box_args args}}
{{/if}}

{{#if options}}
<h4>Options</h4>
{{> api_box_args options}}
{{/if}}

{{#if body}}
{{#better_markdown}}{{{body}}}{{/better_markdown}}
{{/if}}

</div>

</template>



<template name="api_box_args">
<dl class="args">
{{#each this}}
<dt><span class="name">{{{name}}}</span>
  <span class="type">
    {{#if type_link}}
      <a href="#{{type_link}}">{{{type}}}</a>
    {{else}}
      {{{type}}}
    {{/if}}
  </span></dt>
<dd>{{#better_markdown}}{{{descr}}}{{/better_markdown}}</dd>
{{/each}}
</dl>
</template>