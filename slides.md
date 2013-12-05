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
Meteor.subscribe("top-scores", "scrabble");
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

### example of reacting to oplog

@@@

### but we do need to understand mongo.  fortunately we have minimongo

@@@

### on devel now, with restrictions

basically we're writing a query planner

@@@

### you'll be able to use it in next week's release

in meteor run.  you can run it in production with an url.  galaxy supports it !

@@@

### benchmarks

@@@

### near future: support more queries

@@@

### deeper future: in-mongo?

@@@

### any qs



big pro: we don't need deep understanding of selectors, field projections, sort
specifiers, options, etc: that's what mongo is for.

@@@

special nod to arunoda who explored this, we've copmared design notes etc

## How is that useful

we already wrote a mongo query engine

- We already have a decent implementation of Mongo query logic: our client-side
  **minimongo** library
- So we can **interpret queries in Meteor** and use the oplog to keep the
  results up to date!



i'm conservative.  we don't fuck around.

no spec.
