---
title: "Kubevisor becomes Kubirds"
date: "2021-04-16T19:29:03Z"
description: "I little while ago, we announced the first release of Kubevisor, a fully automated Cloud-native supervision software. We were pleased with…"
tags: ["kubirds", "kubernetes", "monitoring", "cloud", "automation"]
author: "David Delassus"
images: ["img/image-4f0590631f6bd62c.png"]
---

![](img/image-4f0590631f6bd62c.png)

I little while ago, we announced the first [release of Kubevisor](/posts/2021-03-24-dockerized-supervision-on-kubernetes-with-kubevisor), a **fully automated Cloud-native supervision software**. We were pleased with the feedback, the questions and advices we received.

While “*Kubevisor*” is a perfectly fine name, it doesn’t reflect the key strength of our solution. Hence the rename!

But **Kubirds** does! By automating your supervision “on the fly”, **Kubirds** is the first alert-based supervisor that took off to the cloud(s) 🚀

---

## 📊 Metrics are complicated

For the past few years, the monitoring landscape in the cloud has been all about **metrics**. But humans do not deal well with numbers. They usually prefer simple concepts and visualizations.

That is why, on top the **metric collection**, engineers add many layers of technology to support:

- metric history, with tools like *InfluxDB* or *Prometheus*
- metric analysis, with statistical algorithms and aggregations, and/or machine learning models
- reporting, with tools like *Grafana*
- alerting: with tools like *AlertManager* to notify people when metrics exceed some predefined thresholds

This creates a heavy stack, often hard to maintain and integrate with the rest of your system 😫

---

## 🚨 Alerts are simple

Sometimes, you just want to know if your website is up, or if your router is behaving as intended, or if the data in your database respects some rules.

You may have a complex algorithm, that you just want to plug in a system that will notify you, or you want to schedule your backups easily.

This is where **Kubirds** comes into play! You just need to plug your business logic, and everything will be **automatically** taken care of, for you.

First, interoperability is achieved with Docker containers:

- put your validation logic in a Docker container 🐋
- put your notification logic in a Docker container 🐋
- plug them together with **Kubirds 🔌**

Then, the scheduling is achieved with a natural description:

> every 5 minutes

> every 2 hours on Saturday and Sunday, in December

And if you were expecting a third point, you’d be surprised that you don’t need to do anything else 🙂

---

## 🦅 Kubirds, supervision automation for the Cloud era

By relying on *Kubernetes*, we gain the following features:

- **highly scalable**, you’ll never spend any more seconds on migrating to a new infrastructure
- **nothing new to learn**, your supervision is now done the “*Kubernetes way*”, with [custom resources](http://kubirds.com/docs/concepts/)
- **GitOps compatibility**, use any tool you’d like, *FluxCD*, *ArgoCD* or *Helm*, to **deploy**, **reproduce**, and **reuse** your supervision specifications

So feel free to:

- visit our website <https://kubirds.com> (previously: <https://kubevisor.io>)
- [ask for a demo](mailto:contact@link-society.com)
