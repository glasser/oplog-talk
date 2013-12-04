# Meteor Oplog Tailing
## Making Mongo queries realtime
## David Glasser

Meteor DevShop 10, 2013-Dec-05

@@@

## Meteor makes realtime the default

- Data changing in your DB is automatically propagated all the way through to
  your web UI
- Just describe the Mongo queries that your server needs to publish, and Meteor
  does the rest.

```js
Meteor.publish("messages", function (roomId) {
  check(roomId, String);
  return Messages.find({room: roomId});
});
Meteor.publish("parties", function () {
  return Parties.find({$or: [{"public": true},
                             {invited: this.userId},
                             {owner: this.userId}]});
});
```

@@@

## Data through the Meteor stack

- Clients **subscribe** to record sets by name
```js
Meteor.subscribe("messages", "myRoom");
```
- The server runs its **publish function**, which typically returns a cursor
```js
Meteor.publish("messages", function (roomId) {
  check(roomId, String);
  return Messages.find({room: roomId});
});
```
- The server **observes changes** on that cursor
- When changes happens, the server sends **DDP data messages** to the client
- The client updates its **local cache**
- Changes to the local cache cause the Deps engine to **re-render templates**

@@@

## Almost all of this is always fast

- **subscribing** just sends a single JSON message over DDP
- **publish functions** are a tiny piece of code
- **DDP data messages** are easily calculated from the cursor's changes
- **local cache** updates are straightforward
- The Deps system used for **re-rendering templates** is fast (and will be even
  better when Meteor UI lands)

@@@

## Observing changes

```js
handle = Messages.find({room: roomId}).observeChanges({
  added: function (id, fields) {...},
  changed: function (id, fields) {...},
  removed: function (id) {...}
})
```
- `observeChanges` executes a Mongo query and calls the `added` callback for
  each matching document
- It continues to watches the database and notices when the query's results
  change
- When the results change, it calls the `added`, `changed`, and `removed`
  callbacks asynchronously
- This continues until you call `handle.stop()`

@@@

## How does it watch the database?

- Essentially, by running the query over and over
- The initial (unreleased) implementation just ran the query every 10 seconds
  and diffed the results
- Simple and definitely worked, but not very real-time
- So we added the **crossbar**: a mechanism that ensures we re-run all queries on
  a collection soon after any time your server process writes to it

@@@

## Pros and cons of the initial approach

- It worked! Your changes immediately show up in every browser everywhere
- ... as long as you only have one server process
- ... and the changes actually come from Meteor itself, not some other process
  writing to your database
- Other changes to your database are still detected by the 10-second poll
- Polling implementation was relatively simple and easy to trust
- Crossbar was overly sensitive: with high write traffic, queries are rerun very
  often, leading to high Meteor and Mongo CPU usage and high Mongo bandwidth use
- Large queries that don't change often still require in-process recursive diffs

@@@

## Improvements to polling

- We made the crossbar less sensitive by adding limited logic to detect that
  certain writes would not affect certain queries (namely, those which affect
  specific documents by `_id`
- Query de-duping: if multiple connections want to subscribe to the same query,
  use the same diff-and-poll fiber for all of them

@@@

## Oplog tailing

- We have always known that "poll every query pretty often" is not the final
  answer for Meteor at scale
- Our next step: **oplog tailing**
- MongoDB replica sets communicate with each other via an **operation log**
  describing exactly what has changed in the database
- You can **tail** the oplog: follow along and find out about every change
  immediately
- We already have a decent implementation of Mongo query logic: our client-side
  **minimongo** library
- So we can **interpret queries in Meteor** and use the oplog to keep the
  results up to date!



