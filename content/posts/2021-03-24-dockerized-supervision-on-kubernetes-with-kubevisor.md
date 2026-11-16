---
title: "Dockerized Supervision on Kubernetes with Kubevisor"
date: "2021-03-24T09:21:05Z"
description: "Link Society started 4 years ago. Jonathan Labéjof and myself have been working together for at least 3 years at the time."
tags: ["kubernetes", "monitoring", "cloud"]
author: "David Delassus"
images: ["img/image-ee9c2e6dd3e0435b.png"]
---

[Link Society](https://link-society.com) started 4 years ago. [Jonathan Labéjof](https://www.linkedin.com/in/b3j0f) and [myself](https://www.linkedin.com/in/linkdd) have been working together for at least 3 years at the time.

We started the company with no precise goal, we just wanted a legal structure if any of our work could generate revenue.

Today, I’m proud to announce that, after 1 year of development, our first product, [Kubevisor](https://kubevisor.io), is ready!

![](img/image-ee9c2e6dd3e0435b.png)

As engineers (full stack devops \[insert more buzzwords here\]), one of our main pain points was the monitoring of our applications and infrastructure.

We identified two ways of dealing with monitoring:

## Collecting Metrics

The workflow is simple:

- make your application produce metrics about its business logic (number of orders, user activity, …)
- collect metrics from your infrastructure (CPU/RAM usage, storage space, latency, …)
- push those metrics to a dedicated service (Prometheus, InfluxDB, ELK, …)
- configure thresholds for your metrics
- visualize everything in dashboards like Grafana

This workflow gives you plenty of insights about “why” problems happen, and with the help of Machine Learning, you can even predict changes in your metrics.

Infrastructure Management Systems like Kubernetes often integrate with some of those monitoring solutions to facilitate the developer’s work.

## Periodically check your infrastructure

The workflow is as follows:

- identify the properties you want to check about your application or infrastructure (responses to HTTP requests, content of your Database, …)
- configure a check scheduling tool (Nagios, Icinga, Shinken, Zabbix, …) to verify those properties
- notify your teams as soon as something wrong happen

This workflow enables you to react quickly “when” problems happen.

So far, none of the tools mentioned above integrates well with Kubernetes, whose main job is to schedule and scale workload.

This is how **Kubevisor** came to be.

## Kubevisor, the Kubernetes Supervisor, to the rescue

![](img/image-e344479a394384c4.png)

As a *Kubernetes Operator*, **Kubevisor** takes care of the configuration and the scheduling, and then delegates the monitoring workload (checks and notifications) to [TektonCD](https://tekton.dev) and Kubernetes.

This allows your monitoring infrastructure to grow and scale alongside your application.

**Kubevisor** introduces 3 concepts:

- **a Unit:** the most basic element of your monitoring, this describes the check that needs to be performed, and when it should be performed
- **a Reactor:** this is the notification system that needs to be triggered whenever a **Unit** has been executed
- **an Inhibitor:** this completes the scheduling model of **Kubevisor**, it allows you to skip your **Unit**’s execution during recurrent time periods.

By presenting itself as a *Kubernetes Operator*, **Kubevisor** provides those concepts as *Kubernetes Custom Resources*, integrating itself by default with tools like [Helm](https://helm.sh), [FluxCD](https://fluxcd.io), or [ArgoCD](https://argo-cd.readthedocs.io/en/stable/) (often found in businesses using Kubernetes).

## Simplicity \& Interoperability is the key

![](img/image-5609de72d6edc5ea.png)

**Kubevisor** aims to be as simple as possible, this is why most of its features come from its dependencies:

- [TektonCD](https://tekton.dev) to bring your units and reactors in a pipeline
- [RabbitMQ](https://rabbitmq.com) to schedule the execution of said pipelines
- [Kubernetes](https://kubernetes.io) to run the workload of said pipelines
- [GraphQL API](https://graphql.org/) to expose your configuration to any authorized client and/or frontends

The overhead of running your checks and notifications as Docker containers is compensated by the interoperability it brings: **any** Docker image can be used as a check or a notification, therefore **any** system can be interrogated and/or notified.

## Getting Started

![](img/image-6cc11be2e7a8955f.png)

**Kubevisor** is distributed as an [*Helm Chart*](https://charts.link-society.com). After using it internally during our *Quality Assurance* phase (4 months), we are eager to get feedback and run it in the real world!

Feel free to:

- visit **Kubevisor**’s website at <https://kubevisor.io>
- visit our website (in french) at <https://link-society.com>
- [contact us](mailto:contact@link-society.com) to schedule a **free** demo
