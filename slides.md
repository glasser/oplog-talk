# Making Mongo Realtime
## Oplog tailing in Meteor
## David Glasser

Meteor DevShop 10, 2013-Dec-05

@@@

## Meteor makes realtime the default

When data changes in your database, everybody's web UI updates automatically
without you having to write any custom code.

@@@

## How the Meteor realtime stack works

- The server runs its **publish function**, which typically returns a cursor

```javascript
Meteor.publish("top-scores", function (gameId) {
  check(gameId, String);
  return Scores.find({game: gameId},
    {sort: {score: -1}, limit: 5, fields: {score: 1, user: 1}});
});
```

- The server watches the query in the database and **observes its changes**
- When changes happens, the server sends **DDP data messages** to the client
- The client updates its **local cache**
- Changes to the local cache cause Meteor UI to **re-render templates**

Note:
the alien technology that kicks all this off is how we got the change from the
DB in the first place and how we can do this at scale. that's what i want to
talk about.  that thing is called observeChanges.  you don't usually see it.

@@@

## `observeChanges` makes Mongo a realtime database

`observeChanges` is a brand new API in Meteor's Mongo client interface.

```javascript
handle = Messages.find({room: roomId}).observeChanges({
  added: function (id, fields) {...},
  changed: function (id, fields) {...},
  removed: function (id) {...}
})
```
- `observeChanges` executes  and calls the `added`
  callback for each matching document
- It continues to watches the database and notices when the query's results
  change
- When the results change, it calls the `added`, `changed`, and `removed`
  callbacks asynchronously
- This continues until you call `handle.stop()`

@@@

## `observeChanges` supports all Mongo queries

- Meteor turns the full query API of a real database into a live query API
- No more custom per-query code to monitor the database and see when it changes
- It's our job to make `observeChanges` as efficient as possible for as many
  queries as possible

@@@

## poll-and-diff

- Run a query over and over, and compare the results each time

```javascript
var results = {};
var pollAndDiff = function () {
  cursor.rewind();
  var oldResults = results;
  results = {};
  cursor.forEach(function (doc) {
    results[doc._id] = doc;
    if (_.has(oldResults, doc._id))
      callbacks.changed(doc._id, changedFieldsBetween(oldResults[doc._id], doc));
    else
      callbacks.added(doc._id, doc);
  });
  _.each(oldResults, function (doc, id) {
    if (!_.has(results, id))
      callbacks.removed(id);
  });
};
setInterval(pollAndDiff, 10 * 1000);
whenWeWriteToTheCollection(pollAndDiff);
```

@@@

## When do we re-run the query?

- Every time we think the query may have changed: specifically, any time that
  the current Meteor server process writes to the collection
- Additionally, every 10 seconds, to catch writes from other processes

@@@

## Benefits of poll-and-diff

- Code is short and correct
- Writes from the current process are reflected on clients immediately
- Writes from other processes are reflected on clients eventually

@@@

## Drawbacks of poll-and-diff

1. Cost proportional to poll frequency: number of polls grows with the frequency
   of data change
2. Cost proportional to query result size: Mongo bandwidth, CPU to parse BSON
   and perform recursive diffs, etc
3. Latency depends on whether a write originated from the same server (very low)
   or another process (10 seconds)

@@@

## Optimizations to poll-and-diff

1. Infer that a write does not affect an observe, then skip the poll.
   (eg, when both the write and the query specify a specific `_id`.)
2. Query de-duplication: if multiple connections want to subscribe to the same
   query, use the same poll-and-diff fiber for all of them.

@@@

## Oplog tailing

@@@

## The Mongo oplog

- MongoDB replication uses an **operation log** describing exactly what has
  changed in the database
- You can **tail** the oplog: follow along and find out about every change
  immediately

@@@

## Using oplog tailing for `observeChanges`

We're going to let the database tell us what changed

@@@

## Oplog tailing, conceptually

```javascript
Scores.find({game: "carrom"}, {sort: {score: -1}, limit: 3, fields: {score: 1, user: 1}})
```
Run the query and cache:
```javascript
{_id: "xxx", user: "avital", score: 150}
{_id: "yyy", user: "naomi", score: 140}
{_id: "zzz", user: "slava", score: 130}
```
Oplog says:
```javascript
{op: "insert", id: "www", {game: "skee-ball", user: "glasser", score: 1000}}
```
Ignore it: does not match selector.

@@@


## Oplog tailing, conceptually

```javascript
Scores.find({game: "carrom"}, {sort: {score: -1}, limit: 3, fields: {score: 1, user: 1}})
```
Cache is:
```javascript
{_id: "xxx", user: "avital", score: 150}
{_id: "yyy", user: "naomi", score: 140}
{_id: "zzz", user: "slava", score: 130}
```
Oplog says:

```javascript
{op: "insert", id: "aaa", {game: "carrom", user: "glasser", score: 10}}
```
Ignore it: selector matches, but the score is not high enough.


@@@


## Oplog tailing, conceptually

```javascript
Scores.find({game: "carrom"}, {sort: {score: -1}, limit: 3, fields: {score: 1, user: 1}})
```
Cache is:
```javascript
{_id: "xxx", user: "avital", score: 150}
{_id: "yyy", user: "naomi", score: 140}
{_id: "zzz", user: "slava", score: 130}
```
Oplog says:

```javascript
{op: "remove", id: "ppp"}
```

Ignore it: removing something we aren't publishing can't affect us (unless skip
option is set!)

@@@

## Oplog tailing, conceptually

```javascript
Scores.find({game: "carrom"}, {sort: {score: -1}, limit: 3, fields: {score: 1, user: 1}})
```
Cache is:
```javascript
{_id: "xxx", user: "avital", score: 150}
{_id: "yyy", user: "naomi", score: 140}
{_id: "zzz", user: "slava", score: 130}
```
Oplog says:


```javascript
{op: "update", id: "ccc", {$set: {color: "blue"}}}
```

This is a document not currently in the cursor. This change
does not affect the selector or the sort criteria, so it can't
affect the results. Ignore it!


@@@

## Oplog tailing, conceptually

```javascript
Scores.find({game: "carrom"}, {sort: {score: -1}, limit: 3, fields: {score: 1, user: 1}})
```
Cache is:
```javascript
{_id: "xxx", user: "avital", score: 150}
{_id: "yyy", user: "naomi", score: 140}
{_id: "zzz", user: "slava", score: 130}
```
Oplog says:


```javascript
{op: "update", id: "ddd", {$set: {game: "dominion"}}}
```

This is a document not currently in the cursor. This change
is to a field from the selector, but it can't make it true.
Ignore it!



@@@

## Oplog tailing, conceptually

```javascript
Scores.find({game: "carrom"}, {sort: {score: -1}, limit: 3, fields: {score: 1, user: 1}})
```
Cache is:
```javascript
{_id: "xxx", user: "avital", score: 150}
{_id: "yyy", user: "naomi", score: 140}
{_id: "zzz", user: "slava", score: 130}
```
Oplog says:


```javascript
{op: "update", id: "xxx", {$set: {user: "avi"}}}
```
Invoke `changed("xxx", {user: "avi"})`.

Cache is now:
```javascript
{_id: "xxx", user: "avi", score: 150}
{_id: "yyy", user: "naomi", score: 140}
{_id: "zzz", user: "slava", score: 130}
```

@@@

## Oplog tailing, conceptually

```javascript
Scores.find({game: "carrom"}, {sort: {score: -1}, limit: 3, fields: {score: 1, user: 1}})
```
Cache is:
```javascript
{_id: "xxx", user: "avi", score: 150}
{_id: "yyy", user: "naomi", score: 140}
{_id: "zzz", user: "slava", score: 130}
```
Oplog says:

```javascript
{op: "insert", id: "bbb", {user: "glasser", game: "carrom", score: 200}}
```

Matches and sorts at the top!  
Invoke `added("bbb", {user: "glasser", score: 200})` and `removed("zzz")`.

Cache is now:
```javascript
{_id: "bbb", user: "glasser", score: 200}
{_id: "xxx", user: "avi", score: 150}
{_id: "yyy", user: "naomi", score: 140}
```

@@@

## Oplog tailing, conceptually

```javascript
Scores.find({game: "carrom"}, {sort: {score: -1}, limit: 3, fields: {score: 1, user: 1}})
```
Cache is:
```javascript
{_id: "bbb", user: "glasser", score: 200}
{_id: "xxx", user: "avi", score: 150}
{_id: "yyy", user: "naomi", score: 140}
```

Oplog says:

```javascript
{op: "update", id: "eee", {$set: {score: 500}}}
```

This matches if "eee" has `game: "carrom"`.
We have to fetch doc "eee" from Mongo and check.  
If it does, invoke `added("eee", {user: "emily", score: 500})` and `removed("yyy")`.
Otherwise, do nothing.

@@@

## Minimongo on the server

- In order to process the oplog, we need to be able to interpret Mongo
  selectors, field specifiers, sort specifiers, etc
- This was not the case for poll-and-diff
- Fortunately, Meteor already can do this: minimongo, our client-side local
  database cache!
- When moving minimongo to the server, we need to be very careful that we
  perfectly match Mongo's implementation, even in complex cases (nested arrays,
  nulls, etc)
- Synchronizing between the "initial query" and the oplog tailing is very subtle


@@@

## Benchmarks

- Running benchmarks with various high write loads
- Benchmark with lots of inserts and few updates: 10x more connected clients
- Benchmark with lots of updates: 2.5x more connected clients
- Goal: Scale Meteor so that the DB is the limiting factor
- Bottleneck: Mongo server CPU/bandwidth
  - Can fix by reading from Mongo replicas
  - More unimplemented heuristics

@@@

## On `devel` today!

- Oplog tailing for an initial class of Mongo queries in the next release
- Other classes of queries will be supported by 1.0
- Current implementation runs automatically for dev-mode `meteor run` and can be
  enabled in production with `$MONGO_OPLOG_URL`
- Works with Galaxy!

Note:
Mention Arunoda.


@@@

# Thanks!

Any questions?
