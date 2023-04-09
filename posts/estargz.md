---
title: Using eStargz to reduce container startup time on Amazon EKS
description: This post explains how to use eStargz snapshotter with containerd
date: 2022-10-25
tags:
- containerd
- amazon-eks
- Kubernetes
layout: layouts/post.njk
---

A container image bundles executable code, library, and configuration. Images contain everything an application needs to run. It is a best practice to exclude any file that’s unnecessary for the application packaged in the image. A smaller image means that when you create a container, the container runtime (dockerd or containerd) will have to download fewer bits from the container registry, which will result in faster startup time.

There are cases when the container image becomes significantly large (>500 MB). A common scenario is machine learning workloads. ML workload containers usually package model data necessary for the application. The image size for these containers can easily span in to multiple GBs. As a result, these applications have a slower startup time when the runtime has to pull the image. In fact, [research](https://www.usenix.org/conference/fast16/technical-sessions/presentation/harter) shows that pulling packages accounts for 76% of container start time, but only 6.4% of that data is read.

## Lazy-pulling

There are two open source projects designed to improve container startup time. [Stargz](https://github.com/containerd/stargz-snapshotter) and [SOCI](https://github.com/awslabs/soci-snapshotter) are containerd plugins that reduce the cold start time by providing a way to run containers without downloading the entire image. They introduce the concept of *lazy pulling*, a technique that allows the runtime to download the bits from the container registry as needed.

Using lazy pulling significantly reduces the application startup time. eStargz is a lazily-pullable image format that is compatible with [OCI runtimes](https://github.com/opencontainers/runtime-spec) and standard container registries like DockerHub, GitHub Container Registry.

## eStargz

The eStargz image format is based on stargz image format by Container Registry Filesystem (CRFS) open source project. **CRFS** is a read-only FUSE filesystem that lets you mount a container image, served directly from a container registry, without pulling it all locally first. The project introduces **S**eekable tar.gz format, which makes tar.gz files seekable using an index.

## Stargz Snapshotter plugin

Stargz snapshotter is implemented as a [proxy plugin](https://github.com/containerd/containerd/blob/04985039cede6aafbb7dfb3206c9c4d04e2f924d/PLUGINS.md#proxy-plugins) daemon (`containerd-stargz-grpc`) for containerd. When containerd starts a container, it queries the rootfs snapshots to stargz snapshotter daemon through a unix socket. This snapshotter remotely mounts queried eStargz layers from registries on the node and provides these mount points as remote snapshots to containerd. The plugin uses FUSE to mount eStargz layers directly from the container registry.

![Image.tiff](https://res.craft.do/user/full/9d54cc03-adfe-f72f-3389-565eb7356d1d/doc/28146E5F-3F27-4D41-94B7-CF6EF7615505/C70CF5ED-93B1-43E2-B0C2-41D09499BBC8_2/kxUcYlMPhkl9lEW9PWLWDQpsZcldhni1S81OJseSGZgz/Image.tiff)

## Running eStargz images on Amazon EKS

eStargz images are a little different from the images you’d build using `docker build`. In order to create an image that supports lazy pulling, you'll need an eStargz-aware image builder or a converter.

I am going to build my eStargz image using nerdctl, which is an eStargz-aware image builder. Since image size is not an issue, I will use Debian Jessie as the base image to demo. To make the image size artificially large, I will include twenty 50 MB files.

Lets generate large files containing random text:

```python
mkdir files
for i in {0..20}; do base64 /dev/urandom | head -c 50000000 > files/file${i}.txt; done
```

Create a DockerFile:

```python
cat > Dockerfile <<EOF
FROM debian:jessie
RUN apt-get update && apt-get install -y \
    vim
COPY files .

EOF
```

I am going to store my image in Amazon ECR. I’ll create an ECR repository:

```python
ECR_URI=$(aws ecr create-repository \
  --repository-name estargz-demo \
  --query 'repository.repositoryUri' \
  --output text)
```

Use nerdctl to create container image:

```python
sudo nerdctl build -t ${ECR_URI}:1 .
```

The image is not in eStargz formatted right now, I’ll have to convert it:

```python
nerdctl image convert --estargz --oci \
  ${ECR_URI}:1 ${ECR_URI}:1-esgz
```

The resulting images:

```python
nerdctl images
REPOSITORY                                                   TAG       IMAGE ID        CREATED           PLATFORM       SIZE       BLOB SIZE
account.dkr.ecr.us-west-2.amazonaws.com/estargz-demo    1         798b85a131ed    31 minutes ago    linux/amd64    1.2 GiB    828.8 MiB
account.dkr.ecr.us-west-2.amazonaws.com/estargz-demo    1-esgz    81b0ffd2a4a3    36 seconds ago    linux/amd64    0.0 B      832.3 MiB
```

Login to ECR and push the image to ECR:

```python
aws ecr get-login-password | sudo nerdctl login \
   --username AWS \
   --password-stdin \
   $ECR_URI
nerdctl push ${ECR_URI}:1-esgz
```

The image is now available in ECR and I can start running it in my EKS cluster.

## Create a managed node group that uses containerd

EKS nodes currently don’t enable containerd by default. I’ll create a node group that uses containerd.

Create environment variables for AWS Region and EKS cluster name:

```python
AWS_REGION=<Your AWS Region>
CLUSTER_NAME=<Your EKS cluster's name>
```

First, I need to retrieve the id of the EKS optimized AMI in my region:

```python
AMI_ID=$(aws ssm get-parameter --name /aws/service/eks/optimized-ami/1.23/amazon-linux-2/recommended/image_id --region us-west-2 --query "Parameter.Value" --output text)
```

```python
cat > containerd-mng.yaml <<EOF
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: $CLUSTER_NAME
  region: $AWS_REGION

managedNodeGroups:
  - name: containerd-mng
    minSize: 1
    maxSize: 1
    desiredCapacity: 1
    instanceType: m5.xlarge
    ami: $AMI_ID
    overrideBootstrapCommand: |
      #!/bin/bash
      /etc/eks/bootstrap.sh Socrates --container-runtime containerd
    
EOF
```

## Preparing EKS nodes to use eStargz

Next, I need to install eStargz snapshotter plugin on my worker node. I use AWS Systems Manager on my nodes, so I will connect to the containerd node.

```python
aws ssm start-session --target <Instance ID of the worker node>
```

Connect to the node (using ssh or Systems Manager) and back up the current containerd config file (/etc/containerd/config.toml) and replace it with:

```python
sudo mv /etc/containerd/config.toml /etc/containerd/config.toml.bak
cat > config.toml <<EOF
version = 2
root = "/var/lib/containerd"
state = "/run/containerd"

[grpc]
address = "/run/containerd/containerd.sock"

[plugins."io.containerd.grpc.v1.cri".containerd]
default_runtime_name = "runc"
snapshotter = "stargz"
disable_snapshot_annotations = false

[plugins."io.containerd.grpc.v1.cri"]
sandbox_image = "602401143452.dkr.ecr.us-west-2.amazonaws.com/eks/pause:3.5"

[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
runtime_type = "io.containerd.runc.v2"

[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
SystemdCgroup = true

[plugins."io.containerd.grpc.v1.cri".cni]
bin_dir = "/opt/cni/bin"
conf_dir = "/etc/cni/net.d"

[proxy_plugins]
  [proxy_plugins.stargz]
    type = "snapshot"
    address = "/run/containerd-stargz-grpc/containerd-stargz-grpc.sock"
EOF
sudo mv config.toml /etc/containerd/config.toml
```

Install FUSE:

```python
sudo yum install fuse -y
sudo modprobe fuse
sudo bash -c 'echo "fuse" > /etc/modules-load.d/fuse.conf'
```

Install the snapshotter from its [GitHub](https://github.com/containerd/stargz-snapshotter/releases) repository:

```python
wget https://github.com/containerd/stargz-snapshotter/releases/download/v0.12.1/stargz-snapshotter-v0.12.1-linux-amd64.tar.gz
sudo tar xvzf stargz-snapshotter-v0.12.1-linux-amd64.tar.gz -C /usr/local/bin
sudo wget -O /etc/systemd/system/stargz-snapshotter.service https://raw.githubusercontent.com/containerd/stargz-snapshotter/main/script/config/etc/systemd/system/stargz-snapshotter.service
sudo systemctl enable --now stargz-snapshotter
```

Finally, restart the containerd and kubelet:

```python
sudo systemctl restart containerd
sudo systemctl restart kubelet
```

## Test lazy-pulling

I will now run the pod using my eStargz formatted image. Create a manifest for a new pod and replace the node name with the newly created node:

```python
cat > estargz-pod.yaml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: stargz-demo
spec:
  containers:
  - name: stargz-demo
    image: ${ECR_URI}:1-esgz
    command: [ "/bin/bash", "-c", "--" ]
    args: [ "while true; do sleep 30; done;" ]
  nodeName: ip-192-168-32-236.us-west-2.compute.internal
EOF
```

The pod started in 2 seconds:

```python
k get pods
NAME                            READY   STATUS        RESTARTS      AGE
stargz-demo                     1/1     Running       0             2s
```

To compare, I ran the same pod on another node that didn’t use containerd. It took 45 seconds to start the same image. That’s a lot of improvement in pod startup time!!

Image pull time *without* eStargz:

![Image.jpeg](https://res.craft.do/user/full/9d54cc03-adfe-f72f-3389-565eb7356d1d/doc/28146E5F-3F27-4D41-94B7-CF6EF7615505/46004B64-228E-4F69-B607-D0FBBF391149_2/AbByT341edypWUhHSsEY1F3qSAmSIf9KuHJwYVgzJQ0z/Image.jpeg)

Image pull with eStargz:

![Image.jpeg](https://res.craft.do/user/full/9d54cc03-adfe-f72f-3389-565eb7356d1d/doc/28146E5F-3F27-4D41-94B7-CF6EF7615505/495F26BE-BA38-48DF-B3F6-9915FDF52E5C_2/N7tCcNUyi2GGlpNmWfwIzNnBzbdmKsxhWnhSRUVkw7Mz/Image.jpeg)

## Prefetching files

What if you wanted the runtime to always download a file and disable lazy-pulling?

eStargz supports prefetching of files. This mitigates runtime performance drawbacks caused by the on-demand fetching of each file.

The example below always pulls `ls` and `bash` files before starting the container:

```python
$ cat <<EOF > /tmp/record.json
{ "path" : "/usr/bin/bash" }
{ "path" : "/usr/bin/ls" }
EOF
$ nerdctl image convert --estargz --oci \
    --estargz-record-in=/tmp/record.json \
    ubuntu:21.04 ubuntu:21.04-ls
```

## Seekable OCI (SOCI)

Seekable OCI (SOCI) is a technology open sourced by AWS that enables containers to launch faster by lazily loading the container image. SOCI works by creating an index (SOCI Index) of the files within an existing container image. This index is a key enabler to launching containers faster, providing the capability of extracting an individual file from a container image before downloading the entire archive.

SOCI borrows some of the design principles from stargz-snapshotter, but takes a different approach.

A SOCI index is generated separately from the container image, and is stored in the registry as an [OCI Artifact](https://github.com/opencontainers/artifacts) and linked back to the container image by [OCI Reference Types](https://github.com/opencontainers/tob/blob/main/proposals/wg-reference-types.md). This means that the container images do not need to be converted, image digests do not change, and image signatures remain valid.

Most OCI registries like DockerHub and ECR do not currently support the "referrers" feature. So you cannot use SOCI unless you run a local [ORAS](https://oras.land) registry.

## Conclusion

If you are looking to reduce container start time for your workloads, eStargz snapshotter is definitely worth a look. You’ll have to change your existing container build pipelines to add a step to convert images though. When SOCI support is available in OCI registries, you’ll be able to lazily-pull images without converting them first.

You’ll also have to install eStargz snapshotter on your nodes and configure containerd to use the plugin. Creating a custom AMI or using AWS Systems manager will be the best way to do that. You could also use a DaemonSet to configure the system, but keep in mind that you’ll need to account for restarting the containerd and kubelet.

