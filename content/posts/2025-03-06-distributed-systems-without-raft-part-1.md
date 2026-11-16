---
title: "Distributed Systems without Raft (part 1)"
date: "2025-03-06T17:11:27Z"
description: "How does FlowG implements replication? A short introduction to the design distributed systems in Go."
tags: ["go", "distributed", "logging", "replication", "consensus"]
author: "David Delassus"
images: ["img/image-0f39c0df05194955.png"]
---

## Introduction and context

For almost a year now, at I (and a few other contributors) have been working on **FlowG**, a low-code log management software.

{{< cta-link
  url="https://github.com/link-society/flowg/"
  title="GitHub - link-society/flowg: Low Code log management solution"
  description="Low Code log management solution. Contribute to link-society/flowg development by creating an account on GitHub."
>}}

![](img/image-0f39c0df05194955.png)

FlowG’s pipeline editor

![](img/image-1dcdcca701394a79.png)

FlowG’s log viewer

The project first started out of frustration. At the time, we were using [OpenObserve](https://openobserve.ai/). But the pipeline feature it propose was subpar, and did not allow us to do what we needed to.

For some context, we have multiple clusters, each with multiple virtual machines (obviously), running multiple Docker containers, and some of those containers are running multiple processes using another tool I made: [ProcFusion](https://github.com/linkdd/procfusion) (like NGINX and NGINX Prometheus Exporter, and a software that pulls data from [Netbox](https://netboxlabs.com/docs/netbox/en/stable/) and regenerates the NGINX configuration).

We needed to be able to parse the logs, categorize them, refine them, route them to different storages, and trigger some API calls based on their content.

This is exactly what **FlowG** proposes, using the [Vector Remap Language](https://vector.dev/docs/reference/vrl/) (from Datadog) for scripting and [React Flow](https://reactflow.dev/) for the pipeline editor. All in OpenSource!

---

## Making it distributed

At the beginning, FlowG was a standalone server. A simple `docker run` in your infrastructure, and voilà.

However, some users expressed the need for replication. They need high availability, and redundancy. And FlowG introduced a **S**ingle **P**oint **O**f **F**ailure in their infrastructure.

On top of that, the Helm chart we provide to deploy FlowG in a Kubernetes cluster, alongside [Fluentd](https://www.fluentd.org/) to gather the logs of every Pod, had to require a node name to properly configure node affinity (so that FlowG’s pod is always scheduled on the host that has the Persistent Volume, which is by default a path on the node’s host).

We need to add **replication** to FlowG, and the first step is to make it **distributed**. FlowG instances in a cluster need to know about eachother.

In that space, I heard about 3 techniques:

- [Paxos](https://en.wikipedia.org/wiki/Paxos_%28computer_science%29) protocols
- [Raft](https://raft.github.io/) protocol
- [Gossip](https://en.wikipedia.org/wiki/Gossip_protocol) protocols

But I have to admit, I was biased towards *Raft*, as I’ve heard quite a lot about it over the years. Emphasis on “was”.

---

## Reminder: The CAP theorem

When talking about distributed systems, we need to talk about **consensus**. How do nodes agree with each other on what the current state is?

This is similar to the [2 generals problem](https://en.wikipedia.org/wiki/Two_Generals%27_Problem). How to coordinate actions by communicating over an unreliable link. The link here being the network.

The network can (will?) fail, or can be slow (high latency). Yet, that’s the medium we use to communicate between nodes in a cluster.

The [CAP Theorem](https://en.wikipedia.org/wiki/CAP_theorem) says that every distributed system can only choose 2 options out of those 3:

- (Strong) **Consistency**: every read receives the most recent write, or an error
- **Availability:** every request must succeed
- (Network) **Partitioning Tolerance:** the system continues to work despite potential (hopefully temporary) network failures

A system can then be describes as:

- **CP:** strongly consistent, and resistant to network failures, but can result in unavailability
- **AP:** always available, and resistant to network failures, but can result in inconsistent responses
- **CA:** strongly consistent, always available, but can break due to network failures

In reality, one **has** to deal with network failures. We simply can’t escape them. So the choice remain between “C” (strong consistency) and “A” (availability).

---

## About Raft and Paxos?

[Raft](https://raft.github.io/) is a distributed consensus protocol. In terms of CAP theorem, it is “CP”, it favors **strong consistency** and **partitioning tolerance** over **availability**.

*Raft* is a leader/follower protocol, it tries to elect a leader node. All writes must be done on the leader node, which means:

- when leader election fails, the cluster becomes unavailable
- when a write is sent to a node, it must redirect it to the leader node, increasing latency
- since all writes must be directed to a single node, its load increase
- since all writes must have a quorum to be accepted (which implies back-and-forth between the leader and followers), performance can be degraded

> EDIT: One thing I forgot to mention, Raft ensures that all write operation are “durable”. If a leader node accepted a write, there is a guarantee that this write will not disappear after the consensus is achieved.

High Availability and Performance are key concerns for **FlowG**, which makes *Raft* not suited for our use case.

Similarly to *Raft*, [Paxos](https://en.wikipedia.org/wiki/Paxos_%28computer_science%29) protocols are (in terms of CAP theorem) “CP”. Which means the cluster can also become unavailable.

> EDIT: I should clarify, the cluster can always become unavailable no matter what the software does. What I meant here is that, during a network partition, the cluster is split in 2 (or more) parts. “AP” systems will allow writes on all partitions, while “CP” systems will only allow writes on the partition that has a majority of nodes.

---

## SWIM entered the chat

Our last candidate is a Gossip protocol. One that caught my eye is [SWIM](https://en.wikipedia.org/wiki/SWIM_Protocol), it stands for *Scalable Weakly Consistent Infection-style Process Group Membership.*

And there is even a Go library for it:

{{< cta-link
  url="https://github.com/hashicorp/memberlist"
  title="GitHub - hashicorp/memberlist: Golang package for gossip based membership and failure detection"
  description="Golang package for gossip based membership and failure detection - hashicorp/memberlist"
>}}

This protocol is (in terms of the CAP theorem) “AP”. Which is what we want. We will achieve **Eventual Consistency** using [**C**onflict-free **R**eplicated **D**ata **T**ypes](https://en.wikipedia.org/wiki/Conflict-free_replicated_data_type).

In FlowG, we use the library above over HTTP, for 3 reasons:

- HTTP can easily be put behind Reverse Proxies who will handle TLS for you (though we do support it natively as well)
- Authentication is easy to add, we support a secret key to be passed as a header but a Reverse Proxy could add client TLS certificate authentication, Basic/Digest authentication, or anything else
- Many edge infrastructures only allow HTTP(S) traffic

For now it’s HTTP/1.1 for simplicity, but migration to HTTP/2 or even HTTP/3 (there are Golang packages that have feature parity with the standard library) should be smooth.

By default, this library uses a TCP (and UDP) network transport, with no TLS, and expects the exact same configuration on all nodes. We wanted flexibility (maybe some nodes use HTTPS, maybe some are on a non standard port, maybe some are behind a Reverse Proxy, etc…).

This had the consequence that all the information provided by the library to identify a node was:

- a name
- a host \& port

We need the actual scheme ( `http://` or `https://` ? ) as well, and the opportunity to add a path at the end if some Reverse Proxy are doing URL rewriting.

Thankfully, the library provides hooks to the join/leave events. In the node’s metadata (exchanged during the protocol’s `ping`, `alive` and other broadcasted events), we put the actual endpoint. This endpoint is then communicated to each node (alongside with the node’s name). And we hook into the join/leave events to maintain a local, in-memory, cluster mesh, a simple map of strings (node name) to URLs (node endpoint).

The bootstrap of a cluster then looks like this:

![](img/image-d908babe8204a587.png)

---

## The Actual Replication layer

…is not done yet!

But, I can already share some ideas. First things first, FlowG will be **eventually consistent**. Every node in the cluster can receive writes (unlike *Raft*), and the replication will happen behind the scene. The cluster will converge to the same state. Examples of database that use eventual consistency are:

- Cassandra
- DynamoDB
- CouchDB

> EDIT: DynamoDB is actually strongly consistent, but will allow readers to opt in to eventually consistent reads as a latency optimization.

To achieve eventual consistency, we will rely on [**C**onflict-free **R**eplicated **D**ata **T**ypes](https://en.wikipedia.org/wiki/Conflict-free_replicated_data_type) (CRDTs).

### What are CRDTs?

A simple example is a “counter”. A counter is a datatype with 2 operations:

- increment by X
- decrement by X

Two nodes receives many of those operations:

- Node A receives: \+1, -1, \+2, -1 \= 1
- Node B receives: -4, \+2, \+1, \+3 \= 2

Then, they share with each other the history of those operations, apply them to their local state and they both **converge** to the same result: **3**.

In FlowG, are main CRDT will be an “oplog”. It has only one operation:

- append a timestamped write operation

To converge 2 nodes, we will merge both oplog’s together and sort by timestamp.

The “oplog” describe a sequence of write operation to be done in order. Those write operations will be done on CRDTs as well, we have 3 kinds:

- CRUD operations in the Authentication database (where users and roles are stored)
- CRUD operations in the Configuration storage (where pipelines, transformers, etc. are stored)
- Append \& Purge operations in the Log database

For the first 2, we will use the “Last Write Wins” strategy.  
For the third one, they already converge naturally.

Once all the CRDTs are synchronized, a “diff” with the local storage will be made, and applied.

### Some considerations:

Again, this part is not yet implemented. A lot of testing will have to be done. And some potential problems can already arise from this design, like:

- how big will the “oplog” be?
- how fast will it be to exchange the “oplog” with other nodes?
- will we need to make “snapshots” in order to reduce the “oplog” size?
- …

Feedback and contributions to discussions on the Github will be appreciated 🙂

---

## Conclusion

This concludes the first part of this series of articles.

Stay tuned and follow me if you want to read more about FlowG and distributed systems in Go.

Don’t hesitate to clap for this article to give it more visibility, and star the repository on Github:

{{< cta-link
  url="https://github.com/link-society/flowg"
  title="GitHub - link-society/flowg: Low Code log management solution"
  description="Low Code log management solution. Contribute to link-society/flowg development by creating an account on GitHub."
>}}

Any contribution are welcome, be it for documentation, bug reports, feature requests, or simply discussions.

Thank you for reading me, and have a great day (or night depending on when you read this article).
