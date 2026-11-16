---
title: "Easy GitOps with klifter 🚀"
date: "2021-05-31T08:24:26Z"
description: "klifter is a DevOps tool from the Datapio ecosystem. In a nutshell, its purpose is to deploy your infrastructure on Kubernetes. In details…"
tags: ["kubernetes", "gitops", "datapio", "deployment"]
author: "David Delassus"
images: ["img/image-01d8ca3b4ce2d55a.png"]
---

![](img/image-01d8ca3b4ce2d55a.png)

[**klifter**](https://klifter.datapio.co) is a DevOps tool from the [**Datapio**](https://datapio.co) ecosystem. In a nutshell, its purpose is to deploy your infrastructure on *Kubernetes*. In details, we’ll see in the next part of this article.

---

## About Datapio

As said in the introduction, **Datapio** is an ecosystem of tools for Kubernetes and Cloud based technologies. Currently, it’s composed of:

- [*klifter*](https://klifter.datapio.co), for easy **GitOps** deployments
- [*klander*](https://klander.datapio.co), for **Compliance as Code**
- [*Kubirds*](https://kubirds.com), a **Cloud Native** supervision engine we already discussed [here](/posts/2021-03-24-dockerized-supervision-on-kubernetes-with-kubevisor) and [here](/posts/2021-04-16-kubevisor-becomes-kubirds)
- [*Datapio OpenCore*](https://opencore.datapio.co), a **CI/CD** platform based on **klifter** and [Tekton](https://tekton.dev) (still a work in progress)
- [*Datapio SDK*](https://sdk.datapio.co), a set of libraries for Kubernetes based development

**Datapio** aims to simplify Kubernetes cluster management, and development.

It is to Kubernetes what Debian is to Linux.

---

## Discover klifter

Simply put, it’s an [Ansible](https://ansible.com) playbook used to deploy a complete [Kubernetes](https://kubernetes.io/)-based infrastructure from a [Git](https://git-scm.com/) repository.

It supports:

- executing Bash scripts
- building Docker images with [docker](https://docker.com/) or [kbld](https://carvel.dev/kbld)
- applying Kubernetes manifests (through `kubectl apply` or with [kustomize](https://kustomize.io/))
- deploying Kubernetes applications with [Helm](https://helm.sh/) or [kapp](https://carvel.dev/kapp)
- generating Kubernetes applications with [ytt](https://carvel.dev/ytt) or `helm template`

**klifter** is distributed as a *Docker* image, deploying from a Git repository is as simple as:

```
$ docker run --rm -it \
      -v $HOME/.kube:/workspace/.kube \
      -e K8S_STATE_SOURCE_KIND=git \
      -e K8S_STATE_SOURCE_GIT_URL=https://github.com/USER/REPO.git \
      -e K8S_STATE_SOURCE_GIT_REF=main \
      ghcr.io/datapio/klifter:latest
```

It expects the Git repository to contain a file named `vars.yml` containing a listing of packages to deploy, for example:

```
---
packages:
  - name: rbac
    kind: bundle
  - name: rabbitmq-cluster-operator
    kind: application
```

As you can see, there are 2 kinds of packages:

**Bundles:** located at `bundles/<bundle name>` and consisting of a set of manifests:

- Kubernetes YAML files deployed with `kubectl apply`
- Shell scripts executed with `bash`
- more to come…

**Applications:** located at `applications/<app name>` and consisting of a set of Kubernetes YAML files deployed with [kapp](https://carvel.dev/kapp) in a dedicated namespace.

**klifter** also supports a number of tools that can be installed automatically and used by your Bash manifests, like `helm` , `docker`, `kbld` , `vault` and many others. Enabling them is done by adding the following to your `vars.yml`:

```
---
tools:
  - helm
  - docker
  - kbld
  - vault 
```

You can find out more about how to write a pipeline for **klifter** in its documentation : <https://klifter.datapio.co/docs/>

---

## Automate deployment with Kubirds

![](img/image-4f0590631f6bd62c.png)

[**Kubirds**](https://kubirds.com) is a Cloud-Native supervision engine for *Kubernetes*. It embeds into *Docker* images the business-logic for periodic jobs (**Units**) and notifications (**Reactors**).

Thus, it provides a complete **easy to use** pipeline to implement alerting, monitoring, and so much more.

Units and Reactors are Kubernetes Custom Resources, and can be deployed with the usual tooling (`kubectl` or `helm` or …).

An *Helm Chart* for a **klifter** Unit is provided out-of-the-box:

```
$ helm repo add \
    link-society-incubator \
    https://charts.link-society.com/incubator
$ helm repo update
$ helm install \
    deploy-my-infra \
    link-society-incubator/kubirds-klifter-unit \
    --set git_url=https://github.com/USER/REPO.git \
    --set git_ref=main \
    --set schedule="every 5 minutes"
```

## Want to know more?

You can send us a mail at any time at [contact@link-society.com](mailto:contact@link-society.com) or visit our websites:

- **Datapio :** <https://datapio.co>
- **Kubirds :** <https://kubirds.com>
- **Link Society (fr) :** <https://link-society.com>

We’ll be happy to answer any questions 🙂
