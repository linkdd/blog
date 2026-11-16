---
title: "How we avoided Redis thanks to Mnesia"
date: "2022-08-05T07:11:49Z"
description: "Discover how we reduced the operational complexity of Kubirds by getting rid of Redis in favor of Erlang’s distributed database: Mnesia."
tags: ["elixir", "redis", "kubernetes", "mnesia", "kubirds"]
author: "David Delassus"
images: ["img/image-860e942b9ba37ae4.png"]
---

![](img/image-860e942b9ba37ae4.png)

---

[Redis](https://redis.io) is an open-source in-memory store. You can use it as a cache for your web workloads, as message broker for your batch jobs, and many more.

Over the years, it became a very prominent brick in a lot of technical stacks, it is not really a surprise, Redis is a great technology.

Why should we want to avoid it then?

---

At [Link Society](https://link-society.com), we built a Cloud Native Supervision Engine, think Nagios or Icinga’s alerting but with containers. [Kubirds](https://kubirds.com) is a [Kubernetes](https://kubernetes.io) Operator written in [Elixir](https://elixir-lang.org/), allowing you to define your supervision workflows with custom resources.

![](img/image-4f0590631f6bd62c.png)

https://kubirds.com

Thanks to Elixir’s distributed computing tools, we made Kubirds highly available quite easily. If you want more informations on how we achieved this, check out this [story](/posts/2021-09-16-elixir-and-kubernetes-a-love-story) I’ve written a while ago (I should continue this series by the way…).

Long story short, Kubirds uses [TektonCD](https://tekton.dev) to schedule the supervision workloads. **Units** are executed periodically and **Reactors** are executed in response to the Units results. Those custom resources are “compiled” to a TektonCD pipeline.

Due to a limitation of TektonCD and Kubernetes, a pipeline task can only send so much data to other tasks (about 2kb). Which means we could not transfer the Unit’s logs to the Reactors.

Enter Redis. At the end of the Unit’s execution, its logs are temporarily stored in Redis, where the Reactors can fetch it from. Logs are then removed once the pipeline is done. The only limit remaining is how much RAM is available.

---

Introducing Redis as a dependency implies deploying it on Kubernetes alongside Kubirds.

We have now 2 choices:

- let Redis be a SPOF (**S**ingle **P**oint **O**f **F**ailure)
- deploy a Redis cluster (via another Kubernetes Operator, or ourselves)

The first choice is not desirable as we want Kubirds to be a highly available operator.

The second choice brings in its fair share of operational complexity. Is there a better solution?

---

The epiphany came by reading this article: [Build an Elixir Redis API that’s 100x faster than HTTP](https://docs.statetrace.com/blog/redis-server/).

Nevermind the clickbaity title, it’s a quite interesting content anyway. The idea is to write a very simple TCP Server using the Redis Protocol.

By implementing a subset of Redis commands (`GET`, `SET`, `DEL`, only the ones we need), we can still rely on [redis-cli](https://redis.io/docs/manual/cli/) to interact with our server.

---

> Mnesia joins the chat

[Mnesia](https://www.erlang.org/doc/man/mnesia.html) is a native Erlang DBMS with ACID transactions. It can be fully in-memory, or have on-disk copies (which requires a schema). For our purpose, we only need in-memory copies.

What’s great about Mnesia is that it’s built in the Erlang/Elixir ecosystem. But it’s also natively distributed over your Erlang/Elixir nodes that compose your cluster.

Mapping the `GET`, `SET` and `DEL` Redis commands of our custom TCP server to Mnesia transaction was the last nail in the Redis coffin. It allowed us to completely remove Redis from our infrastructure, which greatly reduced the operational complexity of Kubirds:

- a `Deployment` for the operator
- a headless `Service` for automatic clustering
- a `Service` to expose the APIs
- some other details not relevant here (an `APIService`, some `ConfigMap`, some RBAC related configuration, …)

---

Mnesia is a great tech to have in your toolbelt, especially if you work with Elixir/[Phoenix](https://www.phoenixframework.org/).

It’s also worth to note that the Redis Protocol is a perfectly fine alternative to an HTTP API. It is relatively easy to implement, and the client-side tooling already exists.

If this article made you want to dip your toes into Elixir, I would recommend going through [Learn you some Erlang for great good!](https://learnyousomeerlang.com/)

This book (available freely online) is a great introduction to the ecosystem, the OTP framework, and how things should be done in Erlang. Learning Elixir after that will be quite easy.
