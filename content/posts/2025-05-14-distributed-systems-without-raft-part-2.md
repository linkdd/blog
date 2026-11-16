---
title: "Distributed Systems without Raft (part 2)"
date: "2025-05-14T11:49:58Z"
description: "How does FlowG implements replication? A short introduction to the design distributed systems in Go."
tags: ["go", "distributed", "logging", "replication", "consensus"]
author: "David Delassus"
images: ["img/image-50a4a4bed843c6e8.png"]
---

- [Part 1: SWIM Protocol for Node Discovery](/posts/2025-03-06-distributed-systems-without-raft-part-1)
- **Part 2: Incremental updates of every node**

---

## Introduction and Context

Recently, I published the `v0.37.0` release of **FlowG**, a Free and OpenSource low-code log processing software:

{{< cta-link
  url="https://link-society.github.io/flowg"
  title="FlowG"
  description="Free and Open-Source Low-Code log processing solution"
>}}

![](img/image-50a4a4bed843c6e8.png)

FlowG’s pipeline editor

![](img/image-7f4d3f295f428b52.png)

FlowG’s log viewer

This version includes the first **experimental** draft of replication, allowing FlowG to run on multiple servers as a single “semantic” instance.

---

## Refresher on Node Discovery

We chose a [Gossip](https://en.wikipedia.org/wiki/Gossip_protocol) protocol, [SWIM](https://en.wikipedia.org/wiki/SWIM_Protocol) (*Scalable Weakly Consistent Infection-style Process Group Membership):*

{{< cta-link
  url="https://github.com/hashicorp/memberlist"
  title="GitHub - hashicorp/memberlist: Golang package for gossip based membership and failure detection"
  description="Golang package for gossip based membership and failure detection - hashicorp/memberlist"
>}}

Thanks to this library, we can tell a node to “join” another node, they will start exchanging information about the cluster architecture (aka: known nodes of the cluster).

A technical choice was also to use HTTP as the transport layer instead of the default TCP/UDP:

- the TCP part is replaced by a hijacked HTTP connection (similar to Websocket);
- the UDP part is replaced by normal HTTP requests.

“Why?” you may ask. Here are a few reasons:

- it becomes easy to add TLS, authentication, etc;
- Reverse Proxies are easy to setup;
- we might explore using HTTP3 later on if performance becomes an issue.

---

## Eventual Consistency

As said in the original article, we chose for FlowG to be *eventually consistent*. We initially planned to use an “operation log” and [**C**onflict-free **R**eplicated **D**ata **T**ypes](https://en.wikipedia.org/wiki/Conflict-free_replicated_data_type) (CRDTs) to implement the replication part.

This idea was temporarily dismissed. I’ll explain why.

Our storage backend is **BadgerDB**:

{{< cta-link
  url="https://docs.hypermode.com/badger/overview"
  title="Overview - Badger"
  description="Welcome to the Badger docs!"
>}}

It is an embeddable key/value database with full transaction support. Internally, it uses a [Log Structured Merge Tree](https://en.wikipedia.org/wiki/Log-structured_merge-tree) (LSM) to organize key/value pairs.

Every key/value pair is assigned a “version”, and every mutation on a key updates that version. This enables incremental backups quite easily:

```go
nextSince, err := db.Backup(writer, since)
nextSince, err = db.Backup(writer, nextSince + 1)
// ...
```

This also means that we don’t actually *need* an “operation log” to track the mutations done on a database.

As for the CRDT, FlowG has 3 kinds of storages:

- **auth:** stores users, permissions, tokens, roles, …
- **config:** stores configuration (pipelines, transformers, forwarders, …)
- **log:** stores actual logs

The first 2 storages aren’t used that much. FlowG is a “setup\&forget” kind of software. The 3rd one is **append-only** (in reality, we also have a purge operation to remove logs, but we never update logs).

The BadgerDB instance for those storage already acts like a CRDT. It is “last write wins” for the *auth* and *config* storages, and “append-only” for the *log* storage.

---

## Actual Replication

To implement replication, we will rely on the SWIM protocol, and on BadgerDB’s incremental backup feature.

The `hashicorp/memberlist` package provides what they call a “TCP Push/Pull”. Periodically, each node computes their local state, send them to other nodes, which then merge the remote state with their local state.

We will trigger the replication during this “TCP Push/Pull” process.

We defined a node state as: the set of last known version of other nodes.

For example:

```json
{
  "node_id": "flowg-0",
  "last_sync": {
    "flowg-1": {
      "auth": 1,
      "config": 2,
      "log": 324
    },
    "flowg-2": {
      "auth": 0,
      "config": 0,
      "log": 0
    }
  }
}
```

When `flowg-1` receives this state from `flowg-0`, it will start an HTTP request to `flowg-0`’s management interface and run (in parallel):

```go
// POST http://<remote>/cluster/sync/auth
newAuthSince, err := authDb.Backup(
    syncAuthRequestBodyWriter,
    remoteState.LastSync["flowg-1"].Auth,
)
// send `X-FlowG-Since: <newAuthSince>` as HTTP trailer

// POST http://<remote>/cluster/sync/config
newConfigSince, err := configDb.Backup(
    syncConfigRequestBodyWriter,
    remoteState.LastSync["flowg-1"].Config,
)
// send `X-FlowG-Since: <newConfigSince>` as HTTP trailer

// POST http://<remote>/cluster/sync/log
newLogSince, err := logDb.Backup(
    syncLogRequestBodyWriter,
    remoteState.LastSync["flowg-1"].Log,
)
// send `X-FlowG-Since: <newLogSince>` as HTTP trailer
```

On the node `flowg-0`, the HTTP handler for `/cluster/sync/...` will read the request body:

```go
nodeId := request.Header.Get("X-FlowG-NodeID")
err := db.Load(syncLogrequestBodyReader, 1)
updateLocalState("log", nodeId, request.Trailer.Get("X-FlowG-Since"))
```

And voilà! We have actual replication.

---

## A few considerations

This is only a draft, and it is still highly experimental. Without further testing to understand:

- what happens if the syncing fails?
- what happens during a network partition?
- what happens if a node fails?
- …

I would not recommend going to production. But you’re free to test and open bug reports:

{{< cta-link
  url="https://github.com/link-society/flowg/issues"
  title="Issues · link-society/flowg"
  description="Low Code / Visual Scripting log processing software - Issues · link-society/flowg"
>}}

It would greatly help 🙂

---

## Technical notes

The node’s local state is also stored in a BadgerDB instance, but it is not replicated. Only the 3 other storages are replicated.

The interval between 2 “TCP Push/Pull” has been arbitrarily set to 1s. Maybe it should be configurable, maybe it should be tuned, further testing is required.

We use a very niche feature of HTTP/1.1: Trailers

{{< cta-link
  url="https://developer.mozilla.org/fr/docs/Web/HTTP/Reference/Headers/Trailer"
  title="Trailer - HTTP | MDN"
  description="L'en-tête Trailer permet à l'expéditeur d'inclure des champs supplémentaires à la fin des blocs de messages pour…"
>}}

TL;DR: It allows the emitter to send headers **AFTER** the body of the request or response has been written:

```
HTTP/1.1 POST /
Transfer-Encoding: chunked
Trailer: Foo

... request body ...

Foo: bar
```

This feature is so niche that not many HTTP libraries support it:

- Python’s stdlib, or `requests` or `httpx` do not support it
- Web Browser’s `fetch` API do not support it
- …

Fortunately, Golang’s stdlib `net/http` package does support it, and FlowG is written in Go.

We need this feature because of the way BadgerDB incremental backups work. The `db.Backup()` method accepts an `io.Writer` and a `uint64` version as input. It will write the key/value pairs newer than the input version. At the end, it will return the highest seen version. But at that point, we already wrote data in the HTTP request body, we cannot send HTTP headers anymore. We can send a Trailer though.

---

## Conclusion

This concludes the second part of this series of articles. The next one may come in a long time, in order to have some real world usage data to present and give a feedback on how well (or how bad) this implementation worked.

Stay tuned and follow me if you want to read more about FlowG and distributed systems in Go.

Don’t hesitate to clap for this article to give it more visibility, and star the repository on Github:

{{< cta-link
  url="https://github.com/link-society/flowg"
  title="GitHub - link-society/flowg: Low Code / Visual Scripting log processing software"
  description="Low Code / Visual Scripting log processing software - link-society/flowg"
>}}

Any contribution are welcome, be it for documentation, bug reports, feature requests, or simply discussions.

Thank you for reading me, and have a great day (or night depending on when you read this article).
