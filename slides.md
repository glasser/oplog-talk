# Making Mongo Realtime
## Oplog tailing in Meteor
## David Glasser

Meteor DevShop 10, 2013-Dec-05

@@@

## Meteor makes realtime the default

When data changes in your database, everybody's web UI updates automatically
without you having to write any custom code.

@@@

## Data flow through the Meteor stack

- Clients **subscribe** to record sets by name
```js
Meteor.subscribe("top-scores", "dominion");
```
- The server runs its **publish function**, which typically returns a cursor
```js
Meteor.publish("top-scores", function (gameId) {
  check(gameId, String);
  return Scores.find({game: gameId},
    {sort: {score: -1}, limit: 5, fields: {score: 1, user: 1}});
});
```
- The server watches that cursor and **observes its changes**
- When changes happens, the server sends **DDP data messages** to the client
- The client updates its **local cache**
- Changes to the local cache cause Meteor UI to **re-render templates**

## Most of these steps are straightforward.

Note:
the alien technology that kicks all this off is how we got the change from the
DB in the first place and how we can do this at scale. that's what i want to
talk about.  that thing is called observeChanges.  you don't usually see it.

@@@

## `observeChanges` is what makes our Mongo realtime


```js
handle = Messages.find({room: roomId}).observeChanges({
  added: function (id, fields) {...},
  changed: function (id, fields) {...},
  removed: function (id) {...}
})
```
- `observeChanges` executes an **arbitrary Mongo query** and calls the `added`
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

## `observeChanges` initial implementation

- Based around polling queries and diffing the results
- Essentially, we run a query over and over, and compare the results each time
```js
var results = {};
setInterval(function () {
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
}, 10 * 1000);
```

@@@

## When do we re-run the query?

- Every time we think the query may have changed: specifically, any time that
  the current Meteor server process writes to the collection
- Additionally, every 10 seconds, to catch writes from other processes

@@@

## Benefits of the query-polling `observeChanges` implementation

- Code is straightforward and correct
- Writes from the current process are reflected on clients immediately
- Writes from other processes are reflected on clients eventually

@@@

## Drawbacks of query-polling

1. High write volumes lead to frequent polling: number of polls grows with the
   frequency of data change
2. The cost of a single poll increases with the amount of data matching that
   query (Mongo bandwidth, CPU to parse BSON and perform recursive diffs, etc)
3. Latency depends on whether a write originated from the same server (very low)
   or another process (10 seconds)

@@@

## Optimizations to reduce polling frequency

We want to poll frequently enough to keep data updates real-time, while not
wasting CPU and Mongo bandwidth on redundant polls. We added a few optimizations
a year ago:

1. Instead of polling every query whenever we did any write to its collection,
   we analyze the query and the write and skip polling queries that are
   definitely unaffected. (Specifically, when both the write and the query
   specify a specific `_id`.)
2. Query de-duplication: if multiple connections want to subscribe to the same
   query, use the same poll-and-diff fiber for all of them.

These helped a lot with problem #1 (polling is unnecessarily frequent) but
didn't help with the other two problems: comparing large data sets takes a
while, and horizontal scaling still breaks "immediacy".

@@@

## Oplog tailing

- We have always known that "poll every query pretty often" is not the final
  answer for Meteor at scale
- Our next step: **oplog tailing**
- MongoDB replica sets communicate with each other via an **operation log**
  describing exactly what has changed in the database
- You can **tail** the oplog: follow along and find out about every change
  immediately

@@@

## Using oplog tailing for `observeChanges`

- So if we're trying to `observeChanges` some queries, we can tail the oplog and
  figure out for ourselves if the results of a query has changed, without having
  to re-run the entire query
- Only have to look at the pieces that just changed
- Driven by database changes, so "in process" and "out of process" changes are
  treated identically

@@@

## Oplog tailing, conceptually

```
Scores.find({game: "carrom"}, {sort: {score: -1}, limit: 3, fields: {score: 1, user: 1}})
```
Run the query and get
```
{_id: "xxx", user: "avital", score: 150}
{_id: "yyy", user: "naomi", score: 140}
{_id: "zzz", user: "slava", score: 130}
```
Tailing the oplog, we see (abstractly):
```
{op: "insert", id: "www", {game: "skee-ball", user: "glasser", score: 1000}}
   # ignore it: does not match query
{op: "update", id: "xxx", {$set: {user: "avi"}}}
   # invoke changed("xxx", {user: "avi"})
{op: "insert", id: "aaa", {game: "carrom", user: "glasser", score: 10}}
   # ignore it: query matches but it sorts below our limit
```

@@@

## Oplog tailing, conceptually

```
Scores.find({game: "carrom"}, {sort: {score: -1}, limit: 3, fields: {score: 1, user: 1}})
```
Our state now is:
```
{_id: "xxx", user: "avi", score: 150}
{_id: "yyy", user: "naomi", score: 140}
{_id: "zzz", user: "slava", score: 130}
```
More oplog entries:
```
{op: "remove", id: "ppp"}
   # ignore it: removing something we aren't publishing can't affect us
   #   (unless skip option is set!)
{op: "insert", id: "bbb", {user: "glasser", score 200}}
   # matches and sorts at the top!
   # invoke added("bbb", {user: "glasser", score 200})
   #    and removed("zzz")
```

@@@

## Oplog tailing, conceptually

```
Scores.find({game: "carrom"}, {sort: {score: -1}, limit: 3, fields: {score: 1, user: 1}})
```
Our state now is:
```
{_id: "bbb", user: "glasser", score: 200}
{_id: "xxx", user: "avi", score: 150}
{_id: "yyy", user: "naomi", score: 140}
```
More oplog entries:
```
{op: "update", id: "ccc", {$set: {color: "blue"}}
   # this is a document not currently in the cursor. this change
   # does not affect the selector or the sort criteria, so it can't
   # affect us.
{op: "update", id: "ddd", {$set: {game: "dominion"}}
   # this is a document not currently in the cursor. this change
   # is to a field from the selector, but it can't make it true.
{op: "update", id: "eee", {$set: {score: 500}}
   # does this match? only if "eee" has game="carrom"
   # we have to fetch doc "eee" from Mongo and check.
   # if it does, invoke added("eee", {user: "emily", score: 500})
   #                and removed("yyy")
```

@@@

## Understanding Mongo

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

### Oplog Tailing: On `devel` now!

- Aiming for release this month
- Current implementation is very conservative: only equality queries, no
  sort/limit support
- Plan to increase the supported cursors over time as we ensure that the
  implementation is 100% correct
- Before 1.0, will support all of the most common queries
- Query planner for live queries
- Current implementation runs automatically for dev-mode `meteor run` and can be
  enabled in production with `$MONGO_OPLOG_URL`
- Works with Galaxy!

Note:
Mention Arunoda.

@@@

### Benchmarks XXX

@@@

### Thank you

Any questions?
