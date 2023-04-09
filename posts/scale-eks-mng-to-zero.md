---
title:  Reduce Amazon EKS cost by scaling node groups to zero
description: This post describes how to scale Amazon EKS managed node groups to zero to save on cossts
date: 2022-11-16
tags:
  - Kubernetes
layout: layouts/post.njk
image: https://cdn.pixabay.com/photo/2020/08/30/20/54/rice-field-5530707_1280.jpg
---

Amazon EKS just released the support for [Kubernetes version 1.24](https://aws.amazon.com/blogs/containers/amazon-eks-now-supports-kubernetes-version-1-24/). The new version supports a bunch of cool features. My favorite feature in this release is the ability to scale EKS managed node groups to (and from) zero.

Many customers I engage with have workloads that don’t run continuously. A good example is building software. Software build jobs run when new development teams push new code. Outside of business hours, the supporting infrastructure (like nodes) sits idle. Customers use autoscaling to scale down node groups, but managed node groups required a minimum of 1 node in a node group previously. That’s one node too many, especially when you need beefier and costly nodes with GPUs.

Scaling down to zero results in significant cost savings in such cases. In my opinion, you wouldn’t want to scale your entire cluster to zero. After all, you’d need some nodes to run Cluster Autoscaler and other shared services like Prometheus, AWS Load Balancer, CoreDNS, etc. You can use EKS on Fargate to run some of these services. But keep in mind that Prometheus requires a block storage, and AWS Fargate doesn’t support Amazon EBS yet.

You’d want to run a managed node group for your shared services, like Cluster Autoscaler, that run continuously. You can then add another node group for workloads that spawn periodically, and scale that node group to zero.

![Untitled Diagram.drawio.png](https://res.craft.do/user/full/9d54cc03-adfe-f72f-3389-565eb7356d1d/doc/1CAE5716-0A06-487B-94D9-ED3A26E55B46/F10EE54C-3121-4A95-A3E8-2084AC07ED39_2/0ePoBv7QUOJphRVgz14AABF7CLN8q6hh0yNumolViQMz/Untitled%20Diagram.drawio.png)

## Cluster Autoscaler Managed Node group cache

The Kubernetes Cluster Autoscaler project added support for scaling node groups to and from zero in version 0.6.1. However, it only worked if you added [specific tags](https://github.com/kubernetes/autoscaler/blob/master/cluster-autoscaler/FAQ.md#how-can-i-scale-a-node-group-to-0) to Auto Scaling groups. In other words, after creating a managed node group, you had to find out the associated Auto Scaling group and add Cluster Autoscaler tags yourself.

Starting Kubernetes version 1.24, you can create node groups (or tag existing node groups) with Cluster Autoscaler tags and Cluster Autoscaler will scale that node group to and from zero.

To enable scaling to and from zero, the awesome EKS team contributed a feature to the upstream Cluster Autoscaler project. The new feature adds a manage node group cache that holds labels and taints associated with managed node groups. Cluster Autoscaler now uses the EKS [`DescribeNodegroup`](https://docs.aws.amazon.com/eks/latest/APIReference/API_DescribeNodegroup.html) API to determine a node's label and taints when there are no nodes in the node group. This allows scaling to and from zero and doesn't require adding Auto Scaling group tags.

## Cluster Autoscaler tags for scaling to zero

Before you can start scaling a managed node group to and from zero, you’d need to [add a few tags](https://docs.aws.amazon.com/eks/latest/userguide/autoscaling.html) to your node group. The tags you attach to your node group will help Cluster Autoscaler determine which node group to scale when a Pod is pending. You can add tags that help Cluster Autoscaler taints, labels, and node group’s resources like `WindowsENI`, `PrivateIPv4Address`, etc.

Labels and taints will tell the Kubernetes scheduler to assign Pods to specific nodes. When those Pods don’t have a node to run on (which will be the case when the node group is scaled to zero), Cluster Autoscaler can determine which node group to scale based on the tags. Let’s explore it using an example.

## Scaling a managed node group from zero

You’d need a Kubernetes version 1.24 cluster to follow along. My EKS cluster already has node group, and I have [installed Cluster Autoscaler](https://docs.aws.amazon.com/eks/latest/userguide/autoscaling.html) using EKS documentation.

Cluster Autoscaler needs the permissions to call the EKS `DescribeNodegroup` API to be able to read a node group's tags. The instructions in EKS documentation currently do not add `DescribeNodegroup` permissions to the Cluster Autoscaler IAM role.

The first thing you’d need to do is create an IAM policy that allows the Cluster Autoscaler IAM role to use `DescribeNodegroup` API.

Create a policy to allow EKS `DescribeNodegroup`  API:

```python
cat > describe-nodegroup.json << EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": [
                "eks:DescribeNodegroup"
            ],
            "Effect": "Allow",
            "Resource": "*"
        }
    ]
}
EOF
```

Now you need to add this policy to the Cluster Autoscaler IAM Role. Determine the name of the IAM Role attached to the Cluster Autoscaler service account:

```python
CA_IAM_ROLE=$(kubectl -n kube-system get  sa cluster-autoscaler -o  jsonpath='{.metadata.annotations.eks\.amazonaws\.com/role-arn}' | sed 's|.*/||' )
```

![Image.png](https://res.craft.do/user/full/9d54cc03-adfe-f72f-3389-565eb7356d1d/doc/1CAE5716-0A06-487B-94D9-ED3A26E55B46/6EF2F66E-E9B7-4B3D-9D75-06400FAE0BF2_2/gI025AVUilKKRF9GfkjpTeq3GmtiWypNeTVa4uE6XYYz/Image.png)

Then, add the policy you created to the IAM role that Cluster Autoscaler uses:

```python
aws iam put-role-policy --role-name $CA_IAM_ROLE \
  --policy-name EKSDescribeNodegroup \
  --policy-document file://describe-nodegroup.json
```

Now you can start creating a new node group that you can set to scale to and from zero. Find out the role attached to an existing node group in your cluster. You can use AWS CLI to query that information.

```python
# Store your EKS cluster name in an environment variable
EKS_CLUSTER=<YOUR CLUSTER NAME>
AWS_ACCOUNT=$(aws sts get-caller-identity --query 'Account' --output text)

NODE_ROLE=$(aws eks describe-nodegroup \
  --cluster-name $EKS_CLUSTER \
  --nodegroup-name <YOUR NODE GROUP NAME> \
  --query 'nodegroup.nodeRole' \
  --output text)
```

You’d also need to provide the subnets for the new node group. You can use `describe-nodegroup` to find out the subnets attached to an existing node group.

Create a node group with a label that you will later use to assign Pods to nodes in this node group:

```python
aws eks create-nodegroup \
 --cli-input-json '
{
  "clusterName": "${EKS_CLUSTER}",
  "nodegroupName": "scale-to-zero",
  "scalingConfig": {
     "minSize": 0,
     "maxSize": 5,
     "desiredSize": 0
  },
  "subnets": [
     "<subnet-ID1>",
     "<subnet-ID2>",
     "<subnet-ID3>"
   ],
  "nodeRole": "${NODE_ROLE}",
  "labels": {
     "app": "frontend"
  },
  "tags": {
     "k8s.io/cluster-autoscaler/node-template/label/app": "frontend"
  }
}'
```

*Replace the subnet IDs to match your environment.*

Note that I added a tag `k8s.io/cluster-autoscaler/node-template/label/app` with value `frontend`. This is the same as running `kubectl label nodes <YOUR NODE NAME> app=frontend`. When a node gets created in this node group, it will already have label `app=frontend`.

Now that the node group is created, let’s create a pod with a nodeSelector:

```python
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: nginx-test
spec:
  containers:
  - name: nginx
    image: nginx:latest
  nodeSelector:
    app: frontend
EOF
```

The pod will remain pending for a few minutes. In my case, the pod was in pending state for five minutes.

In the meantime, you can see Cluster Autoscaler logs:

`kubectl -n kube-system logs -f deployment.apps/cluster-autoscaler | grep scale-to-zero`

Once Cluster Autoscaler adds a new node, the pod will start running. You can enable [Auto Scaling group metrics collection](https://aws.amazon.com/blogs/containers/automatically-enable-group-metrics-collection-for-amazon-eks-managed-node-groups/) to see how your node group scales.

![Image.jpeg](https://res.craft.do/user/full/9d54cc03-adfe-f72f-3389-565eb7356d1d/doc/1CAE5716-0A06-487B-94D9-ED3A26E55B46/D344F40B-8692-4F2F-921C-422DA115E1B7_2/qBfnOkqluRY1yVqZBaMsB6K5xJgA1idkKOxkkfmzx8Az/Image.jpeg)

Great! Cluster Autoscaler saw that the test pod was pending, so it scaled the node group from 0.

Now let’s delete the test pod and verify that the node group goes back to 0:

```python
kubectl delete pods nginx-test
```

Cluster Autoscaler will notice that the node with app=frontend is not running any pods and scale down the node group (after the cooldown period).

![Image.jpeg](https://res.craft.do/user/full/9d54cc03-adfe-f72f-3389-565eb7356d1d/doc/1CAE5716-0A06-487B-94D9-ED3A26E55B46/EAD1C890-6DB1-455E-BABB-FAF71E5FD960_2/kDUeT7u0sJs3WqNy7SG8FBvHyHi9ALDkNEsxsuL1jtIz/Image.jpeg)

Perfect, the node group is back to having 0 nodes.

## Conclusion

Scaling down to zero can result in significant cost savings when you have workloads that don’t run 24x7. With Kubernetes 1.24, all you need to do is tag your node groups with labels, taints, or resources, and Cluster Autoscaler will scale your nodes to and from zero.

Happy scaling!
