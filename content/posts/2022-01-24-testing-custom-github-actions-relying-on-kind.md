---
title: "Testing custom Github Actions relying on KinD"
date: "2022-01-24T09:33:26Z"
description: "Docker-based Github Actions interacting with Kubernetes are tricky to test."
tags: ["github-actions", "kubernetes", "docker", "testing", "ci-cd"]
author: "David Delassus"
images: ["img/image-38f8facb3fabfe15.png"]
---

![](img/image-38f8facb3fabfe15.png)

---

It has been a while since I’ve written an article here. Last few months have been busy.

I’m still working on my “[Elixir and Kubernetes : A love story](/posts/2021-09-16-elixir-and-kubernetes-a-love-story)” series, but recently, I have been prioritizing automating the deployment of [Kubirds](https://kubirds.com) on our infrastructure.

---

## 🚀 Deploying, the GitOps way

We have a few repositories to do this:

- `cloud-infra` : containing our [Terraform](https://www.terraform.io/) manifests to create the Kubernetes cluster and DNS records
- `k8s-cluster-state` : containing our [klifter](https://klifter.datapio.co) project to deploy our Helm charts to the cluster
- `charts` : containing our Helm repository

Everything is automated with [Github Actions](https://github.com/features/actions).

[**klifter**](https://klifter.datapio.co) and [**klander**](https://klander.datapio.co) are homemade tools to provide Continuous Deployment and Reconciliation of our Kubernetes cluster. But, using them within a Github workflow was quite tricky.

This is why we developed [custom Github Actions](https://github.com/datapio/actions) to simplify this:

```
- uses: datapio/actions/klifter@main
  with:
    path: path/to/klifter-project
    kubeconfig: path/to/kubeconfig.yml
```

```
- uses: datapio/actions/klander@main
  with:
    kubeconfig: path/to/kubeconfig.yml
```

---

## 🧪 Testing our Github Actions

The next tricky part was testing those actions.

We started by creating a Kubernetes cluster with the [engineerd/setup-kind](https://github.com/engineerd/setup-kind) Github Action, which creates the cluster the [KinD](https://kind.sigs.k8s.io/) (short for “Kubernetes in Docker”). It edits the current user’s `kubeconfig` automatically.

**The problem:**

The `kubeconfig` points to `127.0.0.1:<random port>`, which in turns route trafic to the API Server within the Docker container deployed by KinD.

Because our custom Github Action is based on a Docker container as well, we can’t just use the `kubeconfig` as-is. Inside the Docker container, there is no API Server on `127.0.0.1`.

**The solution:**

We need to edit the `kubeconfig` in order to point to the Github’s Docker host instead. The following script allows us to get the IP address of this host:

[Get Docker Host IP Address](https://gist.github.com/linkdd/4bbd9fc4966e00e067305f1fb8c2588c)

```bash
#!/bin/sh

cat <<EOF | docker run --rm -i alpine:latest sh
apk add --no-cache iproute2 >/dev/null
ip -4 route show default | cut -d' ' -f3
EOF
```

Then, in our workflow, before running the **klifter** or **klander** actions, we modify the `kubeconfig` :

[Github Workflow Step to edit the KinD generated kubeconfig for use inside a Docker container](https://gist.github.com/linkdd/34770dbd201d7ec1e2c68b00731e0384)

```yaml
# uses script from https://gist.github.com/linkdd/4bbd9fc4966e00e067305f1fb8c2588c
- name: setup@kubeconfig
  run: |
    hostip=$(sh .github/scripts/get-docker-host.sh)
    sed "s/127.0.0.1/$hostip/g" $HOME/.kube/config > ${{ github.workspace }}/kubeconfig.yml
```

Now, our actions can use the new `kubeconfig` to connect to the KinD cluster.

**The next problem:**

We are now getting a `Connection Refused` error when running `kubectl` commands.

This is because the KinD cluster’s API Server is configured to listen on `127.0.0.1`, not the Docker host IP address.

**The solution:**

Fortunately, KinD can take a configuration file as input before creating the cluster, allowing us to configure the IP address of the API Server:

[Example of KinD cluster configuration](https://gist.github.com/linkdd/97c7610877792b79a5f3a4b7ed451f13)

```yaml
---
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  apiServerAddress: "127.0.0.1"
```

We save this configuration to `.github/config/kind.yml.in` and then generate the final configuration using our `get-docker-host.sh` script:

[Example of KinD cluster creation in a Github Workflow](https://gist.github.com/linkdd/5b341631399651e7000799874e31d838)

```yaml
- name: setup@kindconfig
  run: |
    kind_in="${{ github.workspace }}/.github/config/kind.yml.in"
    kind_out="${{ github.workspace }}/.github/config/kind.yml"
    hostip=$(sh .github/scripts/get-docker-host.sh)
    sed "s/127.0.0.1/$hostip/g" $kind_in > $kind_out

- name: setup@kubernetes
  uses: engineerd/setup-kind@v0.5.0
  with:
    config: .github/config/kind.yml
```

---

## Conclusion

Using this method, we can now create a new empty Kubernetes cluster for our test suite, and run Docker images (either with `docker run`, or as a custom Github Action) interacting with this cluster.

To see a complete example, you can see the workflow we use [here](https://github.com/datapio/actions/blob/main/.github/workflows/test.yml).
