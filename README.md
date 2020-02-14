# Table of Contents

* [Spinner-Free Applications](#spinner-free-applications)
* [Introducing Replicache](#introducing-replicache)
* [System Overview](#system-overview)
  * [Data Model](#data-model)
  * [Transaction IDs](#transaction-ids)
  * [Checksums](#checksums)
  * [Replicache Client](#replicache-client)
  * [Replicache Server](#replicache-service)
  * [Customer Server](#customer-service)
* [Execution Model](#execution-model)
* [Synchronization](#synchronization)
  * [Client Sync](#client-sync)
  * [Server Push](#server-push)
  * [Server Pull](#server-pull)
* [Images and other BLOBs](#images-and-other-blobs)
* [TODO](#todo)
  * [Pokes](#pokes)
  * [Auth](#auth)
  * [Images and other Bobs](#images-and-other-blobs)
  * [Non-Bundle Transactions?](#non-bundle-transactions)

# Spinner-Free Applications

"[Offline-First](https://www.google.com/search?q=offline+first)" describes a client/server architecture where
the application reads and writes to a local database on the device, and synchronizes with servers asynchronously whenever
there is connectivity.

These applications are highly desired by product teams and users because they are so much more responsive and
reliable than applications that are directly dependent upon servers. By using a local database as a buffer, offline-first
applications are instantaneously responsive and reliable in any network conditions.

Unfortunately, offline-first applications are also really hard to build. Bidirectional
sync is a famously difficult problem, and one which has eluded satisfying general
solutions. Existing attempts to build general solutions (Apple CloudKit, Android Sync, Google Cloud Firestore, Realm, PouchDB) all have one or more of the following serious problems:

* **Non-Convergence.** Many solutions do not guarantee that clients end up with a view of the state that is consistent with the server. It is up to developers to carefully construct a patch to send to clients that will bring them into the correct state. Client divergence is common and difficult to detect or fix.
* **Manual Conflict Resolution.** Consult the [Android Sync](http://www.androiddocs.com/training/cloudsave/conflict-res.html) or [PouchDB](https://pouchdb.com/guides/conflicts.html) docs for a taste of how difficult this is for even simple cases. Every single pair of operations in the application must be considered for conflicts, and the resulting conflict resolution code needs to be kept up to date as the application evolves. Developers are also responsible for ensuring the resulting merge is equivalent on all devices, otherwise the application ends up [split-brained](https://en.wikipedia.org/wiki/Split-brain_(computing)).
* **No Atomic Transactions.** Some solutions claim automatic conflict resolution, but lack atomic transactions. Without transactions, automatic merge means that any two sequences of writes might interleave. This is analogous to multithreaded programming without locks.
* **Difficult Integration with Existing Applications.** Some solutions effectively require a full committment to a non-standard or proprietary backend database or system design, which is not tractable for existing systems, and risky even for new systems.

For these reasons, existing products are often not practical options for application developers, leading them
to develop their own sync protocol at the application layer if they want an offline-first app. Given how expensive and risky this is, most applications delay offline-first until the business is very large and successful. Even then, many attempts fail.

# Introducing Replicache

Replicache dramatically reduces the difficulty of building offline-first applications. Replicache's goals are:
1. a truly offline-first programming model that is natural and easy to reason about
1. maximize compatability with existing application infrastructure and minimize the work to integrate

The key features that contribute to Replicant's leap in usability are:

* **Easy Integration**: Replicache runs alongside an existing server-side database. Its job is to provide bidirectional conflict-free sync between clients and servers. It does not take ownership of the data. This makes it very easy to adopt: you can try it for just a small piece of functionality, or a small slice of users, while leaving the rest of your application the same.
* **Standard Data Model**: The Replicache data model is a standard document database. From an API perspective, it's
very similar to Firestore, MongoDB, Couchbase, FaunaDB, and many others. You don't need to learn anything new, 
and can build arbitrarily complex data structures on this primitive that are still conflict-free.
* **Guaranteed Convegence**: The existing database is the single source of truth and Replicache guarantees that after a client sync the client's state will exactly match that of the server. Developers do not need to manually track changes or construct diffs on either the client or the server.
* **Transactions**: Replicache provides full [ACID](https://en.wikipedia.org/wiki/ACID_(computer_science)) multikey read/write transactions. On the server side, transactions are implemented as REST or GraphQL APIs. On the client, transactions are implemented as deterministic programmatic functions, which are executed serially and isolated from each other.
* **Much Easier Conflict-Resolution**: Replicache is a [Convergent Causal Consistent](https://jepsen.io/consistency/models/causal) system: after synchronization, transactions are guaranteed to have run in the same order on all nodes, resulting in the same database state. This feature, combined with transaction atomicity,
makes conflict resolution much easier. Conflicts do still happen, but in many cases resolution is a natural side-effect of serialized atomic transactions. In the remaining cases, reasoning about conflicts is made far simpler. These claims have been reviewed by independent Distributed Systems expert Kyle Kingsbury of Jepsen. See [Jepsen Summary](jepsen-summary.md) and [Jepsen Article](jepsen-article.pdf).

# System Overview

Replicache is a transaction synchronization, scheduling, and execution layer that runs in a mobile app and alongside an existing server-side key/value database. It takes a page from [academic](https://www.microsoft.com/en-us/research/publication/unbundling-transaction-services-in-the-cloud/) [research](http://cs.yale.edu/homes/thomson/publications/calvin-sigmod12.pdf) that separates the transaction layer from the data or physical storage layer. The piece in the mobile app is the *client* and the existing server-side database is the *storage layer*.

![Diagram](./diagram.png)

## Data Model

Replicache synchronizes updates to per-user *state* across an arbitrary number of clients. The state is a sorted map of key/value pairs. Keys are byte strings, values are JSON.

## The Big Picture

The Replicache client maintains a local cache of the user's state against which the application runs read and write transactions. Both read and write transactions run immediately against the local state and write transactions are additionally queued as *pending* application on the server. Periodically the client *syncs* to a *Replicache server*, pushing pending transactions to be applied and pulling updated state.

The Replicache server is a proxy in front of the storage layer that makes sync more efficient. During sync the Replicache server applies the pending transactions received from the client to the storage layer and returns the resulting state from the storage layer. It diffs the new state with what the client has and sends the delta downstream to the client.

A key feature that makes Replicache flexible and easy to adopt is that Replicache does not take ownership of the data. The storage layer owns the data and is the source of truth. Replicache runs alongside an existing document database (storage layer) and requires only minimal changes to it. Processes that Replicache knows nothing about can mutate state in the storage layer and Replicache clients will converge on the storage layer's canonical state and correctly apply client changes on top of it.

# Detailed Design

## TransactionIDs

Updates in Replicache are transactional: multiple keys can be modified atomically. Transactions are identified by a _TransactionID_, which has two parts:

* A *Client ID*: A string generated by Replicache Server which uniquely identifies a client
* A *Transaction Ordinal*: An incrementing integer uniquely identifying each transaction originating on a particular client

TransactionIDs are serialized as a JSON array with two elements:

```
// client b4866c5a3b, ordinal 42
["b4866c5a3b", 42]
```

In cases where TransactionIDs need to be sent as strings, the JSON serialization is sent.

## Checksums

At various places in the Replicache protocol checksums are used to verify that two states are identical. The checksum is over the keys and values in key order. 

## Replicache Client

The Replicache Client maintains:

* The client ID
* The next transaction ordinal. This ordinal ensures local causal consistency and idempotence at the server.
* A _bundle_ of JavaScript, provided by the app, containing _bundle functions_ implementing transactions which can be invoked to by the app to read or write data.
* A versioned, transactional, deterministically iterable key/value store that keeps the user's state
  * Versioned meaning that we can go back to any previous version
  * It must also be possible to _fork_ from a version, apply many transactions, then reveal this new version atomically (like git branch and merge)
  * Transactional meaning that we can read and write many keys atomically

### Commits

Each version in the key/value store is represented as a _commit_ which has:
* An immutable view of the user's state. In practice, this might be a reference to the state (eg, a content hash) but conceptually each commit has the state.
* A *Checksum* over the state

Commits come in two flavors, those from the client and those from the server:
* *Pending commits* represent a change made on the client that is not yet known to be applied on the server. Mutation commits include the transaction that caused them so that they may be replayed on top of new confirmed commits from the server.
* *Confirmed commits* represent a state received from the server. They carry a *Snapshot ID* uniquely identifying the snapshot.

### API Sketch

There are two primary APIs: one available within JavaScript transactions running on the client and the client interface itself.

```
// KVStore is accessible to JavaScript implementing transactions.
class KVStore {
  KVStore()
  bool has(String key)
  JSON get(String key)
  void put(String key, JSON value)
  List<Entry> scan(ScanOptions options)
}

// Replicache is the client interface, used by the app.
class Replicache extends KVStore {
  Replicache(AuthOpts authOpts)
  
  // Bundle registration
  Blob bundle();
  boolean setBundle(Blob blob);

  // Transaction invocation and result change notification.
  JSON exec(String functionName, List<JSON> args);
  Subscription subscribe(String functionName, List<JSON> args, void handler(JSON result));
}

struct Entry {
  String key
  JSON value
}

struct Subscription {
  void function(JSON result) handler;
  void cancel();
}

struct ScanOptions {
  String prefix
  ScanBound start
  Limit int
}

struct ScanBound {
  ScanID id
  uint64 index
}

struct ScanID {
  String value
  bool exclusive
}
```

## Replicache Server

Replicache Server is a multitenant distributed service. It proxies transactions from the client to the storage layer and sends state updates to the client in the form of deltas. The replicache server is an optimization that reduces upstream server round trips and downstream bandwidth; conceptually it is not required, though practically it is.

For each user the Replicache server maintains a *history* of previous states. Specifically it keeps the state, snapshot id, and checksum. Note that this history need not be complete; missing entries only affect sync performance, not correctness. 

The Replicache server provides three interfaces:
* *NewClient*: calls into the storage layer to create a new client id and returns it
* *Push*: accepts a batch of pending transactions from the client and applies them to the storage layer
* *Pull*: accepts a snapshot id from the client indicating the snapshot in use, pulls the latest user state from the storage layer, computes the delta from what the client has, and returns it (if any)

## Storage Layer

The storage layer is a standard REST/GraphQL web service. In order fto integrate Replicache the storage layer must:
1. implement an interface for each transaction in the bundle
1. maintain a mapping from client id to last confirmed transaction ordinal. The ordinal increases monotonically.
1. provide a way to fetch a user's full state along with the client's latest confirmed transaction ordinal

# Data Flow

Data flows in a loop from app into the client, up to the replicache server, into the storage layer, and back down from the storage layer to replicache server to the client. Transactions flow upstream while state updates flow downstream. Any of these processes can stop or stall indefinitely without affecting correctness.

The replicache client keeps a head commit representing the latest state of the local key-value database and runs transactions serially against this state. The head commmit can change in two ways:
1. write transactions are assigned a squentailly increasing transaction ordinal and add a new pending commit on top of the current head
1. state updates pulled from the server add a new confirmed commit on top of the latest confirmed commit and cause pending transactions to re-run on top of it

## Sync

In order to sync, the client first calls Push on the replicache server, passing all its pending transactions. The replicache server plays the pending transactions serially in order against the storage layer. When the storage layer executes a transcation it updates the client's latest confirmed transaction ordinal to match, as part of that transaction. If a transaction's ordinal is less than the client's latest confirmed transaction ordinal, the transaction is ignored. 

The client then calls Pull on the replicache server and passes the snapshot id of the state in its latest confirmed commit. The replicache server retrieves the client's latest transaction ordinal and the *entire* state for the user from the storage layer. The replicache server then checks its history to see if it has a cached copy of the state the client has (identified by snapshot id). If so the new and old states are diff'd and if there is a delta a new snapshot id is assigned, the delta is returned to the client, and the new state stored in the replicache server's history by snapshot id. In either case, the client's latest confirmed transaction ordinal is returned. 

If Pull returns new state to the client, the client forks from the old confirmed commit, adds a new confirmed commit with the delta on top of it, and (re)plays any pending transactions that haven't been confirmed on top, in order. The last commit in this new chain is then set as the new head. The client can now forget about all transactions that have been confirmed.

## Mutations outside the client

There is nothing in the design that requires that changes to user data come through Replicache. In fact we expect there is great utility in mutating the user's state outside of clients, eg in batch jobs or in response to changes in other users' clients. So long as all transactions that mutate the user's data run at a proper isolation level and leave the database in a valid state, replicache will faithfully converge all clients to the new state. 

## Conflicts

Conflicts are a reality of disconnected operation. There is no built in conflict resolution or signaling of exceptional conditions in replicache. Transactions should be written to accommodate conflicts and any signaling that may be required takes place in the user's data. For example, a transaction that reserves an hour on a user's calendar could update a status for the request in the user's data. The transaction might successfully reserve the hour when running locally for the first time, setting the status to RESERVED. Later, if still pending, the transaction might be replayed on top of a state where that hour is unavailable, in which case the transaction would update the status to UNAVAILABLE. Later during Push when played against the storage layer it will settle on one value or the other, and the client will converge on the value in the storage layer. App code can rely on a subscription to a read transaction to receive notifications about status changes. 

# Constraints

**Data size** A a primary constraint is the size of user data. In fetching all a user's data from the storage layer during each pull Replicache makes an explicit tradeoff of bandwidth for ease of implementation and integration. For this reason we are initially limiting user data to 20MB per user and recommend replicache server be deployed as close to the storage layer as possible.

**Blobs** Any truly offline first system must have first class bidirectional support for binary assets aka blobs (eg, profile pictures). Ideally these assets should be managed transactionally along with the user's data: either you get all the data and all the blobs referenced or you get none of it. However there is presently no special support for blobs in replicache -- users who need them would need to manage the assets themselves or encode them to json strings in the user data. We would like to address this shortcoming in the future. 

# Database properties TODO

TODO techinical discussion here and ref to jepsen?

# TODO

## Pokes

TODO: There should someday be some way for Customer Server to poke Replicache Server and/or client to tell it to sync.
TODO: We should also consider whether there are any advantages to making this whole thing socket-based. I'm not sure given interaction with background sync on mobile devices. It feels like simplicity wins to me, but not sure.

## Auth

TODO.

Currently Replicant server has its own auth mechanism based on JWT. However, this is for authing with Replicant. We now need to auth with customer.

Clients will still need to auth w/ Replicache Server, too, for various reasons (e.g., admin).

Hm.

## Images and Other Blobs

The Replicache State should typically contain mutable structured data. But what about images, and other large immutable objects?

TODO: There needs to be some way to sync blobs. We might be able to require that they are immutable.

## Non-JS Transactions

The bundle concept is nice because it allows transactions to be run in an isolated environment that enforces determinism. It also allows code sharing between client platforms. However, Replicache does not *require* determinism for correctness of sync. And the bundle is a cost for developers -- especially those that aren't at home in JavaScript, and don't have JS build infrastructure already setup.

Perhaps Replicache should enable transactions to be registered in the host language and run in the host environment.

# Specification

Here we collect some important but evolving implementation details.

## Checksums

Replicache currently uses the [LtHash](https://engineering.fb.com/security/homomorphic-hashing/) algorithm to enable efficient incremental computation of checksums.

The serialization fed into the hash is as follows:

* The number of key/value pairs in the state, little-endian
* For each key/value pair in the state, in lexicographic order of key:
  * length of key in bytes, little-endian
  * key bytes
  * length of value when serialized as [CanonicalJSON](http://gibson042.github.io/canonicaljson-spec/) in bytes, little-endian
  * value as [CanonicalJSON](http://gibson042.github.io/canonicaljson-spec/)
  
  Probably we should namespace the user keys so that we could add in other information eg previous state if we wanted to.


### Request

* `ClientID` (optional): The client's ID, unless this is the first sync
* `Basis` (optional): The last confirmed transaction ID the client has, if any
* `Checksum` (optional): The checksum client has for _basis_
* `Mutations`: Zero or more mutations to apply to customer server, each having:
  * `Ordinal`: The transaction ordinal for this mutation
  * `Path`: Path at customer server to invoke
  * `Payload`: Payload to supply in POST to customer server

### Response

* `ClientID` (optional): If the client didn't pass a ClientID, it is new, and the server returns an assigned ClientID here.
* `TransactionID`: The ID of the last transaction applied on the server. This doesn't have to be a transaction from the requesting client.
* `Patch`: The patch to apply to client state to bring it to *TransactionID*
* `Checksum`: Expected checksum to get over patched data

## Server Push

Replicache Server applies mutations to Customer Server by invoking standard REST/GraphQL HTTP APIs.

For each request, Replicache additionally passes the custom HTTP header `X-Replicache-TransactionID`.

Customer Server must:

* Decode TransactionID into ClientID and TransactionOrdinal
* Atomically, with no less than [snapshot isolation](https://jepsen.io/consistency/models/snapshot-isolation):
  * Read the last committed TransactionOrdinal for that client
  * If the last committed TransactionOrdinal is >= the supplied one:
    * Return HTTP 200 OK
  * If the last committed TransactionOrdinal is exactly one less than the supplied one:
    * Process the request as normal
    * Atomically, *as part of the same commit*, increment the TransactionOrdinal for this client
  * Otherwise, return HTTP 400 with a custom HTTP Header `X-Replicache-OutOfOrderMutation`

Customer Server does not need to return a response body, Replicache Server always ignores them.

Error handling details:

* HTTP 200:
  * Success
* Other HTTP 2xx
  * Success, but an informational message returned to caller and logged
* HTTP 3xx
  * Redirects are followed by Replicache
* HTTP 5xx, 408 (Request Timeout), 429 (Too Many Requests) and 400 with `X-Replicache-OutOfOrderMutation`:
  * Replicache will wait and retry the request later with exponential backoff
* All other HTTP 4xx:
  * Replicache returns the error to caller and logs it, then continues. That is for purposes of synchronization, this request has been handled.
    
### Response

Unused

## Server Pull

Replicache periodically polls Customer Server for the current state for a user. Customer Server returns the *entire* state (up to 20MB) each time.

### Request

Unused

### Response

```
{
  "transactionID": ["client17", 42],
  "data": {
    "key": "value",
    "pairs": [
      "arbitrary",
      "JSON",
      "for",
      "values": {"foo", 42: "bar": false},
    }
  }
}
```
