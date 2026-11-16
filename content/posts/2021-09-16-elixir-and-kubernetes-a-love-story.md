---
title: "Elixir and Kubernetes: A love story — Part 1: Setting up an Elixir Cluster"
date: "2021-09-16T14:26:32Z"
description: "Part 1: Setting up an Elixir Cluster"
tags: ["elixir", "kubernetes", "datapio", "erlang"]
author: "David Delassus"
images: ["img/image-46f1674ff9a00dd9.png"]
---

![](img/image-46f1674ff9a00dd9.png)

In the last few years, **Kubernetes** has been heavily adopted in many companies as the deployment and application orchestration solution.

This adoption birthed many **Cloud-Native** solutions relying on Kubernetes to automate diverse workflows, to quote a few:

- [Tekton](https://tekton.dev): Continuous Integration
- [KubeDB](https://kubedb.com/) \& [KubeVault](https://kubevault.com/): Database \& Secret management
- [RabbitMQ Cluster Operator](https://www.rabbitmq.com/kubernetes/operator/operator-overview.html): Message queues and exchanges
- …

Unfortunately, most of those solutions, often implemented in [**Go**](https://golang.org/), do not support more than one instance running (“replicas” in a Deployment, in Kubernetes terms), creating SPOFs (Single Point of Failure) in your infrastructure.

This is because “leader election” or “distributed consensus” are tricky to implement.

But in the [**Erlang**](http://erlang.org/)**/**[**Elixir**](https://elixir-lang.org/) ecosystem, those problems look like they are already solved. Yet, most tutorials about Elixir only talk about building a webapp with the Phoenix framework.

This article is the first of a series of article that will cover:

- how to build a Kubernetes Operator in Elixir
- why you don’t need l**eader election**
- out of the box **distributed consensus**
- how to rely on other technologies when it’s needed

Hopefully, by the end of this series, you’ll have a better idea of what Elixir can do for you 🙂

---

**📃 Table of Content:**

- **Part 1: Setting up an Elixir Cluster**
- 🚧 Part 2: Build a Kubernetes operator
- 🚧 Part 3: Extend the Kubernetes API
- 🚧 Part 4: Orchestrate other programs
- 🚧 Part 5: Integrate kubectl plugins
- 🚧 Part 6: Complex application monitoring with Kubirds

*Code examples can be found on Github at:* [linkdd/elixir-k8s-love-story](https://github.com/linkdd/elixir-k8s-love-story)

---

In this first article, we’ll see how to setup an Erlang/Elixir cluster on Kubernetes.

But first, let’s recap things a bit…

---

## What is Elixir? 🤔

[Elixir](https://elixir-lang.org/) 1.0 came out in 2015\. At the time of writing, Elixir 1.12 is the latest version.

Elixir is a functional language that compiles to [Erlang](https://www.erlang.org/) bytecode and is run on the [BEAM](https://en.wikipedia.org/wiki/BEAM_(Erlang_virtual_machine)) (the Erlang virtual machine).

Erlang is also a functional language, that came out in the 1980s from Ericsson.

Their key features are:

- **fault tolerance:** you get resilience at the **application level**
- **immutability:** you don’t have to worry about your application state changing
- **pattern matching:** a powerful way to branch based on your values
- **recursion:** Tail Call Optimization happens at the bytecode level
- **concurrency:** done with the actor model and message passing pattern, you have lightweight processes sending messages to each other
- **the OTP (Open Telecom Platform) framework:** a set of tools to build scalable, resilient and fault tolerant distributed applications
- **timeouts, monitoring, and linking:** handle timeouts, watch other processes, and handle their potential crashes
- **single point of return:** your functions returns only once, at the end, simplifying the flow of instructions (with the exception of throw and raise)
- **atoms:** a literal, a constant with a name, they are custom symbols (or keywords) that the developer can use in pattern matching, making the code easier to read
- **compile time configuration:** giving the developer the ability to produce different releases for different environments
- **hot code reloading:** upgrade your application without restarting it (useful if you want to upgrade your drone’s software while it’s flying)
- **language stability:** the language evolve slowly and carefully, your code is very likely to keep working in the next 10–20 years

Elixir adds to those:

- **the pipeline operator:** a simple and effective way to chain computations
- **escript:** to compile and run your code at runtime
- **Mix:** a modern project manager to manage your dependencies and configuration and custom tasks
- **runtime configuration:** will be merged with the compile time configuration at startup
- **exceptions:** Erlang has throw/catch, Elixir adds raise/rescue as a better alternative
- …

The learning curve for Erlang is steeper than Elixir, **BUT** I’d still recommend starting with the amazing book “[Learn you some Erlang for great good](https://learnyousomeerlang.com/)”. It will give you an amazing introduction to **Erlang**, **OTP** and **Mnesia** (a distributed database, native to Erlang, with ACID transactions).

**NB:** I like to use Mnesia without disk persistence to build a distributed cache for my distributed applications. If I need persistence, I would still rely on PostgreSQL or another DBMS (thus, keeping my application “stateless”)

---

## What is Kubernetes? 🤔

[Kubernetes](http://kubernetes.io/) first came out in 2014 from Google. It is a container orchestrator, meaning its main objective is to manage container-based workload.

This includes:

- scheduling and running containers across the nodes of your cluster
- configuration and secret management
- network policies
- volumes (persistent of temporary) handling

It relies on the following standards:

- **OCI (**[**Open Container Initiative**](https://opencontainers.org/)**):** to package and distribute container images
- **CRI (Container Runtime Interface):** to run containers (example: [Docker](https://www.docker.com/), [Podman](https://podman.io/))
- **CNI (Container Network Interface):** to handle networking between containers (example: [Calico](https://www.tigera.io/project-calico/))
- **CSI (Container Storage Interface):** to provide volumes to containers (example: [GlusterFS](https://www.gluster.org/), Google Cloud Storage, …)

With Kubernetes, you declare your **desired state**, and through the control loop, it will modify the **observed state** towards the desired state.

The desired state is defined with a bunch of **resources** (for example: a Pod, a ConfigMap, a Service, …), and operators will watch those resources to decide what to do next. Example:

- the user defines a **Deployment** resource with 3 replicas
- the builtin deployment operator will create 3 **Pod** resources
- the builtin pod operator will start the containers defined in the pod resources

If a container crash, the pod operator will notice it (the observed state has diverged from the desired state), and restart it.

This is called the “control loop”, giving you resilience at the **infrastructure level**.

---

To summarize previous sections:

- Elixir gives you resilience at the **application level**.
- Kubernetes gives you resilience at the **infrastructure level**.

If you want to build reliable and fault tolerant softwares, you might need both.

To understand how both can work together, let’s explore how to build an Erlang/Elixir cluster.

---

## Erlang Clustering 🌐

![](img/image-1e543d4a5cc826b1.png)

Figure 1: 3 nodes Erlang cluster

In Erlang/Elixir, a node is identified by a **basename** and a **hostname** (or an **IP address**, or a **Fully Qualified Domain Name**).

In the *Figure 1* diagram, we have 3 nodes:

- Node 1: basename \= `foo`, hostname \= `127.0.0.1`
- Node 2: basename \= `bar`, hostname \= `127.0.0.2`
- Node 3: basename \= `baz`, hostname \= `127.0.0.3`

To connect the nodes together, they need to share the same **Erlang Cookie**, a secret value used to authenticate nodes upon join.

This cookie is set via the`--cookie` argument when starting your node.

Then, you would run from Node 1 the following code:

```
Node.connect(:'bar@127.0.0.2')
Node.connect(:'baz@127.0.0.3')
```

Let’s create a sample application with:

```
$ mix new my_app --sup
$ cd my_app
```

**NB:** The `--sup` option will create a supervision tree (an OTP application and a supervisor).

Running `mix release` will create a portable release of your application, containing your compiled code, the full runtime and a script to start the Erlang node with your application.

**NB:** If you configured multiple releases within your `mix.exs` file, be sure to pass the correct name to the command with: `mix release ${RELEASE_NAME}`.

This release will be located in `_build/${MIX_ENV}/rel/my_app` and can be copied as-is to the hosts you want to deploy to.

This startup script (found at `bin/my_app`) uses a few environment variables to configure the Erlang node:

- `RELEASE_DISTRIBUTION` controls the kind of node name that is expected (short hostname, or FQDN)
- `RELEASE_NAME` defines the name of the release
- `RELEASE_NODE` defines the full node name (`<basename>@<hostname>`)
- `RELEASE_COOKIE` sets the Erlang Cookie for node authentication (if not set, the default one generated with the release will be used)

---

## Deploy your Erlang/Elixir release 🚀

Using the following Dockerfile, you’ll be able to create a container packaging your application:

[Example of Dockerfile for an Elixir application](https://gist.github.com/linkdd/cd6b0fa44a50563107b31f43c2369120)

```dockerfile
FROM elixir:1.12-alpine AS builder

ARG BUILD_ENV=prod
ARG BUILD_REL=my_app

# Install system dependencies
RUN mix local.hex --force
RUN mix local.rebar --force

# Add sources
ADD . /workspace/
WORKDIR /workspace

ENV MIX_ENV=${BUILD_ENV}

# Fetch dependencies
RUN mix deps.get

# Build project
RUN mix compile

# Run test-suite
RUN mix test --cover

# Build release
RUN mix release ${BUILD_REL}

FROM alpine:latest AS runner

ARG BUILD_ENV=prod
ARG BUILD_REL=my_app

# Install system dependencies
RUN apk add --no-cache openssl ncurses-libs libgcc libstdc++

# Install release
COPY --from=builder /workspace/_build/${BUILD_ENV}/rel/${BUILD_REL} /opt/myapp

## Configure environment

# We want a FQDN in the nodename
ENV RELEASE_DISTRIBUTION="name"

# This value should be overriden at runtime
ENV RELEASE_IP="127.0.0.1"

# This will be the basename of our node
ENV RELEASE_NAME="${BUILD_REL}"

# This will be the full nodename
ENV RELEASE_NODE="${RELEASE_NAME}@${RELEASE_IP}"

# If empty, the default cookie generated by `mix release` will be used
# OVERRIDE IT!!
ENV RELEASE_COOKIE=""

ENTRYPOINT ["/opt/myapp/bin/${BUILD_REL}"]
CMD ["start"]
```

This container can then be deployed to Kubernetes with the following Deployment resource:

[Example of Elixir deployment for Kubernetes](https://gist.github.com/linkdd/d8fc89aa52ac1320906381811d57a6a3)

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-elixir-app
  namespace: default
  labels:
    app.kubernetes.io/name: my-elixir-app
    app.kubernetes.io/instance: myapp-cluster
spec:
  replicas: 3
  selector:
    matchLabels:
      app.kubernetes.io/name: my-elixir-app
      app.kubernetes.io/instance: myapp-node
  template:
    metadata:
      labels:
        app.kubernetes.io/name: my-elixir-app
        app.kubernetes.io/instance: myapp-node
    spec:
      containers:
        - name: main
          image: myapp:latest
          imagePullPolicy: IfNotPresent
          env:
            - name: RELEASE_NODE_IP
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
            - name: RELEASE_COOKIE
              valueFrom:
                secretKeyRef:
                  name: my-elixir-app-cluster
                  key: ERLANG_COOKIE
          ports:
            - name: epmd
              containerPort: 4369
              protocol: TCP
```

Here, we set the current node’s IP to the address of the Pod it’s running on. For security reasons, we are also setting the Erlang cookie from a Kubernetes Secret.

Finally, we expose the **EPMD** (Erlang Port Mapper Daemon) port. This is how Erlang/Elixir communicate with other nodes. We’ll use this later.

There is a step missing though. We need to know the IP address of the other nodes before we can connect them together.

Then, what if a Pod crashes? What if there is a Deployment rollout starting new Pods and shutting down old ones?

Fortunately, if there is a problem, there is a solution!

---

## Automatic Cluster Formation and Healing 🤖

[libcluster](https://hexdocs.pm/libcluster/readme.html) is a library that provides a mechanism for automatically forming clusters of Erlang nodes, with either static or dynamic node membership.

It provides a pluggable “strategy” system, with a variety of strategies provided out of the box. The one we care about is the **Kubernetes DNS** strategy.

Using this strategy, **libcluster** will perform a DNS query against a **headless** Kubernetes Service, getting the IP address of all Pods running our Erlang cluster:

[Example of headless Service for libcluster (Elixir)](https://gist.github.com/linkdd/cf3bad1d2333180413d996b552e20546)

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: my-elixir-app-svc-headless
  namespace: default
  labels:
    app.kubernetes.io/name: my-elixir-app
    app.kubernetes.io/instance: myapp-svc-headless
spec:
  type: ClusterIP
  clusterIP: None
  ports:
    - port: 4369
      targetPort: epmd
      protocol: TCP
      name: epmd
  selector:
    app.kubernetes.io/name: my-elixir-app
    app.kubernetes.io/instance: myapp-node
```

To start using **libcluster**, add `{:libcluster, “~> 3.2”}` to your dependencies, then in your application module (`lib/my_app/application.ex`), add the following:

[Example of libcluster configuration (Elixir)](https://gist.github.com/linkdd/b121d2a21f5aaec595be907a862bbe05)

```elixir
defmodule MyApp.Application do
  use Application

  def start(_type, _args) do
    topologies = [
      default: [
        strategy: Cluster.Strategy.Kubernetes.DNS,
        config: [
          service: "my-elixir-app-svc-headless",
          application_name: "my_app"
        ]
      ]
    ]
    children = [
      {Cluster.Supervisor, [topologies, [name: MyApp.ClusterSupervisor]]},
      # ..other children..
    ]
    Supervisor.start_link(children, strategy: :one_for_one, name: MyApp.Supervisor)
  end
end
```

That’s it!

Although, you might want to follow the [**12 Factor App**](https://12factor.net/) design principles, and make this configurable:

- what if I want to run the application on my computer (single node)?
- what if I want to configure the Service name, or the application name?
- …

This is where [**Datapio**](https://datapio.co) comes into the picture…

---

## Exploit Kubernetes with Datapio 🔨

[**Datapio**](https://datapio.co) aims to provide a complete platform to build Cloud-Native systems on Kubernetes. It comes with 3 packages:

- [**Datapio OpenCore**](https://opencore.datapio.co)**:** an Open-Source CI/CD platform based on [Tekton](https://tekton.dev)
- 🚧 **Datapio Microservices:** a PaaS (Platform as a Service) to deploy microservices, based on [KubeVela](https://kubevela.io/)
- 🚧 **Datapio Pipelines:** a Complex Event Processing infrastructure, based on [Kubirds](https://kubirds.com) and [KubeVela](https://kubevela.io/)

The **OpenCore** package is distributed as an **umbrella project** containing the following sub-projects:

- `datapio_cluster`: integration of **libcluster** as an OTP application
- `datapio_k8s`: utility library to manipulate Kubernetes resources
- `datapio_controller`: wrapper around the [k8s](https://hexdocs.pm/k8s/readme.html) Elixir library to build Kubernetes operators, with JSON schema validation (if not provided by the Resource Definition)
- `datapio_mq`: very simple distributed queues and consumers for the most basic use-case
- `datapio_play`: very simple task runner inspired by [Ansible](https://ansible.com/) (we use it to implement our E2E test suites)

We will take a closer look to each of those sub-projects in this series. But today, let’s focus on `datapio_cluster`.

First, add the following to your dependencies (and remove **libcluster**):

```
{
  :datapio_cluster,
  github: "datapio/opencore",
  ref: "main",
  sparse: "apps/datapio_cluster"
}
```

**NB:** Make sure **git** is installed when running `mix deps.get`, it is required to clone a Git repository from Github (or another VCS provider).

Then, add the `:datapio_cluster` to the `extra_applications` field. Your `mix.exs`, should look like this:

[Sample Mix project configuration with datapio\_cluster](https://gist.github.com/linkdd/d49c9afce54e66e2a0af5a6b32ade6cd)

```elixir
defmodule MyApp.MixProject do
  use Mix.Project

  def project do
    [
      app: :my_app,
      version: "0.1.0",
      elixir: "~> 1.12",
      start_permanent: Mix.env() == :prod,
      deps: deps()
    ]
  end

  # Run "mix help compile.app" to learn about applications.
  def application do
    [
      extra_applications: [:logger, :datapio_cluster],
      mod: {MyApp.Application, []}
    ]
  end

  # Run "mix help deps" to learn about dependencies.
  defp deps do
    [
      {
        :datapio_cluster,
        github: "datapio/opencore",
        ref: "main",
        sparse: "apps/datapio_cluster"
      }
    ]
  end
end
```

This will ensure that the `datapio_cluster` OTP application is started before yours.

This application will read the following environment variables:

- `DATAPIO_SERVICE_NAME`: the name of the headless Kubernetes Service (in the same namespace as the Deployment/Pod), if not set, no automatic clustering will be done
- `DATAPIO_APP_NAME`: the base name of your node, with the previous Dockerfile, it is the value of `$RELEASE_NAME`, defaults to `datapio-opencore`

Those variables will be used to configure **libcluster**.

Therefore, the modifications done in the previous section to your application module (`lib/my_app/application.ex`) can be reverted.

In addition, this application will take care of configuring the **Mnesia** application to enable replication across nodes, and provide a mechanism to create your RAM-only sets at startup.

To customize this, you can add to your `config/config.exs` (compile time configuration) the following:

[Example of datapio\_cluster\_config](https://gist.github.com/linkdd/5151598a9900465221a63f708a3f0c2e)

```elixir
import Config

config :datapio_cluster,
  service_name: [env: "MYAPP_SERVICE_NAME", default: nil],
  app_name: [env: "MYAPP_APP_NAME", default: "my_app"],
  cache_tables: [
    some_set: [:id, :attr1, :attr2],
    some_other_set: [:id, :attr]
  ]
```

Nothing else is required!

---

## Wrapping up 📦

In this article, we saw how easy it is to setup an Erlang cluster on Kubernetes, allowing us to get resilience at the application level **AND** the infrastructure level.

**NB:** To clarify one point, Erlang/Elixir gives you the tools to have resilience at the application level, but this is not automatic, you have to use them 😉

In the next parts of this series, we will continue this Kubernetes journey by writing a small Kubernetes Operator, discovering how [**Datapio**](https://datapio.co) and [**Kubirds**](https://kubirds.com) can help us setup and monitor Elixir systems.

Stay tuned for Part 2!
