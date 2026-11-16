---
title: "Distributed Systems without Raft (part 3)"
date: "2026-07-13T18:40:03Z"
description: "Last year, I wrote about my work on FlowG, trying to implement replication without Raft:"
tags: ["distributed", "databases", "go", "open-source", "programming"]
author: "David Delassus"
---

Last year, I wrote about my work on FlowG, trying to implement replication without Raft:

{{< cta-link
  url="/posts/2025-03-06-distributed-systems-without-raft-part-1"
  title="Distributed Systems without Raft (part 1)"
  description="How does FlowG implements replication? A short introduction to the design distributed systems in Go."
>}}

{{< cta-link
  url="/posts/2025-05-14-distributed-systems-without-raft-part-2"
  title="Distributed Systems without Raft (part 2)"
  description="How does FlowG implements replication? A short introduction to the design distributed systems in Go."
>}}

We are now 1 year later, and the result is… failure.

---

The topic of writing a distributed system, especially one as critical as a database, is extremely complex.

While I managed to get some experiments working, the cost in performance (bandwidth, CPU and IO usage) was just too much and nowhere near production ready.

The complexity of the algorithms, and the complexity of the test suite itself was just too big.

The cherry on the cake was that the more nodes the cluster had, the less performant (log/s throughput) it was. The complete opposite of what we wanted!

In the end, we decided to take the safer road, use a proper distributed database, and our choice landed on **FoundationDB**.

{{< cta-link
  url="https://www.foundationdb.org/"
  title="FoundationDB | Home"
  description="Get Started Download FoundationDB Multi-model data store. FoundationDB is multi-model, meaning you can store many types…"
>}}

On the occasion of its release, I wrote on FlowG’s blog a “deep dive” article on the data model, and limitations we met:

{{< cta-link
  url="https://flowg.cloud/blog/replication-via-foundationdb"
  title="Replication via FoundationDB | FlowG"
  description="Replication is now generally available, thanks to a brand new storage backend"
>}}

Have a great read!
