---
title: Use containerd to handle k8s.gcr.io deprecation
description: How to use containerd mirror feature 
date: 2023-04-04
layout: layouts/post.njk
---


The Kubernetes community is getting ready for yet another major change. Until fall of 2022, `k8s.gcr.io` container registry hosted many Kubernetes community-managed containers images like Cluster Autoscaler, metrics-server, cluster-proportional-autoscaler. The “gcr.io” part in the registry is Google Cloud Registry. In order to be vendor neutral, the community is moving away from using Google Cloud Registry to host container images.

As a result, starting March 20th, traffic from the old `k8s.gcr.io` registry is being redirected to `registry.k8s.io`. The older `k8s.gcr.io` will remain functioning for sometime but it is eventually getting deprecated.

![Image.jpeg](https://res.craft.do/user/full/9d54cc03-adfe-f72f-3389-565eb7356d1d/doc/13FE78CC-1DDD-47D9-B875-EE3FB15F916E/FC421666-8104-4CD9-8C7F-8E4F4BCE2B83_2/6olkWYEK1kl10toAonqYL40P1ZSQIUY2N6iHRwId78kz/Image.jpeg)

## What’s the impact?

The change that occurred six days after Pi day is unlikely to cause major problems. There are some edge cases. But unless you operate in an airgapped or highly restrictive environment that applies strict domain name access controls, you won’t notice the change.

This doesn’t mean that there’s no action required. Now is the time to scan code repositories and clusters for usage of the old registry. Failing to act will result in cluster components failing.

Once the old registry goes away, **Kubernetes will not be able to create new Pods** (unless image is cached) if the container uses an image hosted on `k8s.gcr.io`.

## What do you need to change?

Cluster owners and development teams have to ensure they are not using any images stored in the old registry. The change is fairly simple. It's pretty simple, really.

You need to change your manifests to use `registry.k8s.io` container registry.

![Image.png](https://res.craft.do/user/full/9d54cc03-adfe-f72f-3389-565eb7356d1d/doc/13FE78CC-1DDD-47D9-B875-EE3FB15F916E/EB0981CC-3DC8-49EE-A6F5-A857A69CC949_2/D0F0mQ2JMAl55JNCuS66lUylRznVZywEMdojiRqkc0sz/Image.png)

You can find out which Pods use the old registry using `kubectl`:

```bash
kubectl get pods --all-namespaces -o jsonpath="{.items[*].spec.containers[*].image}" |\0;31;37M0;31;38m
tr -s '[[:space:]]' '\n' |\
sort |\
uniq -c | grep -i gcr.io
```

Here are the Pods in my test cluster that use the old registry:

![Image.jpeg](https://res.craft.do/user/full/9d54cc03-adfe-f72f-3389-565eb7356d1d/doc/13FE78CC-1DDD-47D9-B875-EE3FB15F916E/ED8BD697-7BD9-4D58-83FC-833C66606E97_2/dcAQoH7ADPaoupccwi3WFMxt1PkS5ME4zXHHyxjAmdMz/Image.jpeg)

These are the at-risk Pods. I’ll have to update the container registry used in the Pods.

When hunting for references to old registry, be sure to include containers that may not be currently running in your cluster.

## What if I don’t control the workloads?

One of my colleagues raised an intriguing question. Is there’s a way to handle this change at a cluster level? He had a valid concern. Many large enterprises might not be able to implement this change in time before the community sunsets `k8s.gcr.io`.

I work with many customers that manage large Kubernetes clusters, but have little control over the workloads that get deployed into the cluster. Some of these clusters are shared by hundreds of development teams. The burden is on Central Platform Engineering teams to dissipate this information to individual dev teams (who are busy writing code and not checking Kubernetes news!).

So, what can these teams do to make sure when the old registry finally croaks, they don’t get paged for in the middle of the night for `ErrImagePull` and `ImagePullBackOff`errors?

Turns out you can use containerd to handle this redirection at node level. Let’s find out how.

## Using mirrors in containerd

Ever since Dockerhub started rate limiting image pulls, many have opted to store images in local registries. Mirrors save network bandwidth, reduce image pull time, and don’t rate-limit.

You can configure `registry.k8s.io` as a mirror to `k8s.gcr.io` in containerd. This configuration will **automatically pull images from `registry.k8s.io`** whenever a Pod uses an image stored in `k8s.gcr.io`.

On your worker node, append these lines in the containerd config file at `/etc/containerd/config.toml`:

```bash
[plugins."io.containerd.grpc.v1.cri".registry]
   config_path = "/etc/containerd/certs.d"
```

The final file on an Amazon EKS cluster looks like this:

```yaml
version = 2
root = "/var/lib/containerd"
state = "/run/containerd"

[grpc]
address = "/run/containerd/containerd.sock"

[plugins."io.containerd.grpc.v1.cri".containerd]
default_runtime_name = "runc"

[plugins."io.containerd.grpc.v1.cri"]
sandbox_image = "602401143452.dkr.ecr.us-west-2.amazonaws.com/eks/pause:3.5"

[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
runtime_type = "io.containerd.runc.v2"

[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
SystemdCgroup = true

[plugins."io.containerd.grpc.v1.cri".cni]
bin_dir = "/opt/cni/bin"
conf_dir = "/etc/cni/net.d"

[plugins."io.containerd.grpc.v1.cri".registry]
config_path = "/etc/containerd/certs.d"
```

Next, create a directory called `k8s.gcr.io` and a `hosts.toml` file inside it:

```bash
mkdir -p /etc/containerd/certs.d/k8s.gcr.io

cat << EOF > /etc/containerd/certs.d/k8s.gcr.io/hosts.toml
server = "https://k8s.gcr.io"

[host."https://registry.k8s.io"]
capabilities = ["pull", "resolve"]
EOF
```

Image pull requests to `k8s.gcr.io` will now be sent to `registry.k8s.io`.

Restart containerd and kubelet for the change to take effect.

```bash
systemctl restart containerd kubelet
```

Let’s validate that images are indeed getting pulled from the new registry. I added an entry to my `/etc/hosts` file to break `k8s.gcr.io`:

![Image.png](https://res.craft.do/user/full/9d54cc03-adfe-f72f-3389-565eb7356d1d/doc/13FE78CC-1DDD-47D9-B875-EE3FB15F916E/8AEBA792-CA42-4C8B-9C8D-E34AC55A9847_2/7A4cAKtUBNGVqmUysXRwpW0WGy3cNGz14xxujKlsmc8z/Image.png)

Containerd can no longer pull an image from `k8s.gcr.io`

![Image.jpeg](https://res.craft.do/user/full/9d54cc03-adfe-f72f-3389-565eb7356d1d/doc/13FE78CC-1DDD-47D9-B875-EE3FB15F916E/D382C5B0-437D-44E3-BAA2-E32032FF7E29_2/zmbCw5wl0LyV1TNJdiEQxCnSqKBS3s4IDZWYljKaLvkz/Image.jpeg)

I can tell `ctr` to use the mirror by specifying the `—hosts-dir` parameter:

```bash
ctr images pull --hosts-dir "/etc/containerd/certs.d" k8s.gcr.io/pause:latest
```

This time the operation succeeds.

![Image.jpeg](https://res.craft.do/user/full/9d54cc03-adfe-f72f-3389-565eb7356d1d/doc/13FE78CC-1DDD-47D9-B875-EE3FB15F916E/94C5856D-E54D-414D-86E6-5B7C70A7989B_2/2bTbUm39iI52DAfS4D9GgH1T1LW5pBbWxTHNIjgS2v4z/Image.jpeg)

Any Pods I create now onwards will use the new registry even though the manifests reference old registry. Here’s a test using a pause container.

```bash
kubectl create deployment pause --image k8s.gcr.io/pause
```

![Image.jpeg](https://res.craft.do/user/full/9d54cc03-adfe-f72f-3389-565eb7356d1d/doc/13FE78CC-1DDD-47D9-B875-EE3FB15F916E/88014DF3-E73E-4A60-B881-64C31B8210B6_2/lnuzXKg7G4RwURvQUxrZwRFuBDljKWpX6m01nVydnxoz/Image.jpeg)

Perfect! Kubernetes could create Pods even though I blocked `k8s.gcr.io` on the node.

## What’s the best way to implement this in production?

In my little demo, I changed a single node in the cluster. What about the rest of the nodes?

There are three ways you can use to implement this change on every node in your cluster:

1. The easiest way is to use a daemonset to change to containerd config.toml and add hosts.toml file. IBM cloud has shared [this](https://raw.githubusercontent.com/IBM-Cloud/kube-samples/master/containerd-registry-daemonset-example) on Github
2. You can use [EC2 user data](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/user-data.html) or [AWS Systems Manager](https://aws.amazon.com/systems-manager/) to make this change when a node gets created
3. You can [create your own AM](https://docs.aws.amazon.com/eks/latest/userguide/eks-ami-build-scripts.html)I

## What if I use Docker as runtime?

Starting Kubernetes version `1.24`, containerd is the only runtime available in Amazon EKS AMIs. If you have an edge case that requires using Docker, there's still hope.

Docker also has support for registry mirrors. [Here’s](https://docs.docker.com/registry/recipes/mirror/#configure-the-docker-daemon) the documentation page you need.

## Don’t rely on stop gaps

While the solution included in this post works, I recommend only using as a safety measure. The main reason is that you’ll need to customize the Amazon EKS AMI or create your own AMI to use it.

You’ll have less operational overhead if you can simply use EKS AMIs as is. The best way to handle this registry deprecation is to update manifests.

Oh, and by the way, you can also use mirrors to enforce pull through cache.

