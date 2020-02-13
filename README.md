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

Replicache dramatically reduces the difficulty of building offline-first applications. Replicache's goals are to provide:
1. a natural offline-first programming model that is easy to reason about that requires
1. minimal changes to and opinions about existing database implementations, schemas, and deployments

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

The Replicache client maintains a local cache of the user's state against which the application runs read and write transactions. Transactions run immediately against the local state and write transactions are queued as *pending* application on the server. Periodically the client *syncs* to a *Replicache server*, transmitting pending transactions to be applied and receiving updated state in response. 

The Replicache server is a proxy in front of the storage layer that makes the downstream updates to the client more efficient. During sync the Replicache server applies the pending transactions received from the client to the storage layer and then fetches the resulting state from the storage layer. It diffs the new state with what the client has (if anything), and sends the delta downstream to the client.

A key feature that makes Replicache flexible and easy to adopt is that Replicache does not take ownership of the data. The storage layer owns the data and is the source of truth. Replicache runs alongside an existing document database (storage layer) and requires only minimal changes to it. Processes that Replicache knows nothing about can mutate state in the storage layer and Replicache clients will converge on the storage layer's canonical state.

## TransactionIDs

Updates in Replicache are transactional: multiple keys can be modified atomically. Transactions are identified by a _TransactionID_, which has two parts:

* A *Client ID*: A string generated by Replicache Server which uniquely identify clients.
* A *Transaction Ordinal*: An incrementing integer uniquely identifying each transaction originating on a particular client.

### TransactionID Serialization

TransactionIDs are serialized as a JSON array with two elements:

```
// client b4866c5a3b, ordinal 42
["b4866c5a3b", 42]
```

In cases where TransactionIDs need to be sent as strings, the JSON serialization is sent.

## Checksums

At various places in the Replicache protocol, checksums are used to verify two states are identical.

Replicache uses the [LtHash](https://engineering.fb.com/security/homomorphic-hashing/) algorithm to enable efficient incremental computation of checksums.

The serialization fed into the hash is as follows:

* The number of key/value pairs in the state, little-endian
* For each key/value pair in the state, in lexicographic order of key:
  * length of key in bytes, little-endian
  * key bytes
  * length of value when serialized as [CanonicalJSON](http://gibson042.github.io/canonicaljson-spec/) in bytes, little-endian
  * value as [CanonicalJSON](http://gibson042.github.io/canonicaljson-spec/)

## Replicache Client

The Replicache Client maintains:

* The current client ID
* A _bundle_ of JavaScript, provided by user code, containing _bundle functions_ which can be invoked to by user code to read or write data
* A versioned, transactional key/value store
  * Versioned meaning that we can go back to any previous version
  * Transactional meaning that we can read and write many keys atomically
  * It must also be possible to _fork_ from a historical state, apply many transactions, then commit those transactions atomically

Each version in the key/value store is a _commit_. There are two types of commits:
* *Snapshots* represent the current state received from server at some moment in time
* *Mutations* represent a change made on the client-side not yet known to be confirmed by server

Both transaction types contain:
* An immutable snapshot of the state
* The *TransactionID* of the transaction that created the state
* A *Checksum* over the state

Additionally, mutation commits contain:
* The name of a JavaScript function plus arguments that was used to create this change

## Replicache Server

Replicache Server is a multitenant distributed service, which maintains, for each connected client:

* A history of recent snapshots

Note: The history doesn't need to be complete. Missing history entries don't affect the correctness of the system, only sync performance.

For each snapshot:

* TransactionID
* Checksum
* State

## Customer Server

The Customer Server is a standard REST/GraphQL web service. In order to integrate with Replicache it has to maintain the following additional state:

* A table mapping ClientID->TransactionOrdinal

This table tracks the last applied transaction ordinal for every known client. This table grows with O(number of users) and cannot ever be pruned because it will stall sync of those users. The only recovery would be for those clients to give up and get new client ids.

# API

There are two APIs, the API available from the host language and the one available from JavaScript inside transactions:

```
// This is the subset of the API available inside JS transactions
class ReplicacheJS {
  ReplicacheJS()
  bool has(String key)
  JSON get(String key)
  void put(String key, JSON value)
  List<Entry> scan(ScanOptions options)
}

// The external API has everything the JS API has, plus:
class Replicache extends ReplicacheJS {
  Replicache(String accountID, String auth)
  
  // Bundle registration
  Blob bundle();
  void setBundle(Blob blob);

  // Transaction invocation
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

... and the API available to JavaScript running inside transactions:

# Execution Model

Execution is entirely client-side. Clients register a bundle of JavaScript using `putBundle`, usually sourced from an asset packaged with the app.

Clients make changes by calling `exec`, which executes one of the functions by name from the bundle, or `put`. This creates entries in the versioned key/value store.

Clients read state by calling `get`, `has`, `exec`, or `subscribe`. Subscriptions can be implemented efficiently by recording the reads the transaction made and checking whether puts could have possibly changed them during write transactions.

# Synchronization

There are three continuous decoupled processes happening at all times. Any of these processes can stop or stall indefinitely without affecting correctness.

Data conceptually flows in unidirectional loop: from device to Replicache Server, to Customer Server, back to Replicache Server, back to device.

## Client Sync

The client invokes the `sync` API on the Replicache Server, passing its own ClientID, any pending mutations and the last confirmed `TransactionID` it has, along with the corresponding `checksum`.

The Replicache Server applies the mutations (using `Server Push`), then obtains and stores a new snapshot (using `Server Pull`).

Replicache Server then validates the checksum of the last confirmed `TransactionID` passed by the client. If the commit is unknown or the checksum doesn't match, Replicache Server logs an error and returns the entire state. Otherwise, it computes and returns a [JSONPatch](http://jsonpatch.com/) that will bring the client into alignment with the latest server.

Client forks from basis `TransactionID`, applies the patch, checks the checksum, then re-runs any pending transactions to create final local state.

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
