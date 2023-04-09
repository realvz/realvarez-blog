---
title: Autoscale Kubernetes Metrics Server
description: This is a post on My Blog about touchpoints and circling wagons.
date: 2022-11-03
tags:
- Kubernetes
- amazon-eks
layout: layouts/post.njk
---

Many organizations are happy to standardize their infrastructure platform on Kubernetes. Kubernetes gives engineers a consistent platform across cloud providers and on premises. It abstracts underlying infrastructure so engineers can focus on writing code without having tight-coupling with methods for load balancing, observability, configuration, secrets management, etc. 

I frequently speak with organizations that run their entire workloads in one or two Kubernetes clusters. Effectively, they have moved their entire data centers into Kubernetes clusters. 

Now, Kubernetes is not without its flaws. Especially when operating at scale. Once clusters go beyond hundreds of nodes, nuanced behavior starts showing up. Besides scaling the Kubernetes control plane and data plane, platform teams have to scale Kubernetes components like CoreDNS, core components, and add-ons. 

In this post, I am going to show how to scale the metrics server add-on, so when your cluster scales the Horizontal Pod Autoscaler can reliably scale your workload. 

## Scaling Kubernetes add-ons

Kubernetes add-ons are software packages that extend the functionality of Kubernetes. Vanilla Kubernetes clusters lack capabilities that most production clusters require. For example, data plane scaling functionality in Kubernetes is provided by the [Kubernetes Cluster Autoscaler](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler). Metrics server collects Node and Pod metrics. Log aggregators like Fluent Bit and Fluentd allow you to collect Kubernetes application and system logs. 

It is a best practice to deploy these add-ons with resource limits to account for bugs and memory leaks. Requests and limits allow us to control system resource allocated to each Pod. This feature makes it safer to run multiple Pods on a node without worrying about resource contention or oversubscription.  

Add-ons are deployed either as DaemonSets or Deployments. As a cluster scales, DaemonSets scale automatically as they run once per node. However, add-ons deployed as Deployments do not scale automatically because they are unaware of the size of the cluster’s scale. 

As the cluster scales, add-ons such as [metrics-server](https://github.com/kubernetes-sigs/metrics-server) and [kube-state-metrics](https://github.com/kubernetes/kube-state-metrics) have to hold more data in memory. The default resource requests on the metrics-server are sized for clusters of up to 100 nodes. In clusters larger than that, the metrics-server can run out of memory, which breaks the Horizontal Pod Autoscaler. 

As a result, operators have to scale add-ons, such as the metrics server, vertically as the cluster scales. [Addon-resizer](https://github.com/kubernetes/autoscaler/tree/master/addon-resizer) is an open source tool you can use to scale Deployments in proportion to the data plane. While the Kubernetes [Cluster Proportional Autoscaler](https://github.com/kubernetes-sigs/cluster-proportional-autoscaler) scales Deployments horizontally, the addon-resizer scales Deployments vertically. 

Some cloud providers use addon-resizer to scale the metrics-server add-on. Amazon EKS currently doesn't automatically scale metrics-server. I am going to run addon-resizer to autoscale the metrics-server Deployment in an EKS cluster.

## Addon-resizer

Addon-resizer is a container that vertically scales a Deployment based on the number of nodes in your cluster. It scales Deployments linearly as the cluster’s data plane grows and shrinks. 

The container monitors your cluster periodically and increases or decreases the requests and limits of a Deployment in proportion to the number of nodes. Vertical scaling implies that **addon-resizer will recreate the Pods** with newer resource limits.

At the core of addon-resizer lies the *nanny *program.

```
Usage of ./pod_nanny:
      --acceptance-offset=20: A number from range 0-100. The dependent's resources are rewritten when they deviate from expected by a percentage that is higher than this threshold. Can't be lower than recommendation-offset.
      --alsologtostderr[=false]: log to standard error as well as files
      --container="pod-nanny": The name of the container to watch. This defaults to the nanny itself.
      --cpu="MISSING": The base CPU resource requirement.
      --deployment="": The name of the deployment being monitored. This is required.
      --extra-cpu="0": The amount of CPU to add per node.
      --extra-memory="0Mi": The amount of memory to add per node.
      --extra-storage="0Gi": The amount of storage to add per node.
      --log-flush-frequency=5s: Maximum number of seconds between log flushes
      --log_backtrace_at=:0: when logging hits line file:N, emit a stack trace
      --log_dir="": If non-empty, write log files in this directory
      --logtostderr[=true]: log to standard error instead of files
      --memory="MISSING": The base memory resource requirement.
      --namespace="": The namespace of the ward. This defaults to the nanny pod's own namespace.
      --pod="": The name of the pod to watch. This defaults to the nanny's own pod.
      --poll-period=10000: The time, in milliseconds, to poll the dependent container.
      --recommendation-offset=10: A number from range 0-100. When the dependent's resources are rewritten, they are set to the closer end of the range defined by this percentage threshold.
      --stderrthreshold=2: logs at or above this threshold go to stderr
      --storage="MISSING": The base storage resource requirement.
      --v=0: log level for V logs
      --vmodule=: comma-separated list of pattern=N settings for file-filtered logging
```

The nanny program takes the base CPU and memory and adds extra resources per node. Here’s the formula it uses:

```
Base  CPU + (Extra CPU * Nodes)
```

Let’s say we allocate 100m CPU and 200Mi memory to a container in our cluster. We configure addon-resizer to add 1m CPU and 2Mi memory per node. When the cluster scales to 75 nodes, addon-resizer will scale the target container using the formula below:

```
100m+(1m*75) = 175m
```

It will also increase memory:

```
200Mi + (2Mi*75)= 350Mi
```

## Scale metrics-server

The first question you may have is when should you scale metrics-server. The default resource configuration in metrics-server Deployment is recommended for clusters of up to 100 nodes. Beyond that you may notice the metrics-server restarting frequently (as it gets killed by Kubernetes Out of Memory killer). 

When metrics-server needs more resources than allocated `kubectl top nodes` and `kubectl top pods` will fail. You may get the following error message:

`unable to get metrics for resource cpu: unable to fetch metrics from resource metrics API: the server is currently unable to handle the request (get pods.metrics.k8s.io)` 

Also, the Horizontal Pod Autoscaler may stop working. If you run `kubectl get apiservices v1beta1.metrics.k8s.io`, you may get the following message:

```
NAME                     SERVICE                      AVAILABLE                      AGE
v1beta1.metrics.k8s.io   kube-system/metrics-server   False (FailedDiscoveryCheck)   12m
```

## Deploy addon-resizer

The addon-resizer container can run in its own Pod or as a sidecar. We’re going to deploy the container as a sidecar in the metrics-server Deployment. 

The metrics server defaults to 100m CPU and 200Mi memory. Get the current limits: 

```
kubectl -n kube-system get \
  deployments metrics-server \
  -o jsonpath='{.spec.template.spec.containers[].resources}'
```

Here’s the output from my cluster:

```
{"requests":{"cpu":"100m","memory":"200Mi"}}
```

My cluster currently has 5 nodes. I’ll configure the addon-resizer to scale the metrics-server Deployment vertically by adding 1m CPU per node in addition to the base CPU which is set to 20m. The base memory is 15Mi, and addon-resizer will increase metrics-server memory by 2Mi per node. I took these values from addon-resizer recommendations.

Deploy the manifest to create a ClusterRole, Role, and ClusterRoleBinding that gives the metrics-server service account the permissions to patch the metrics-server Deployment:

```
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: eks:metrics-server-nanny
  labels:
    k8s-app: metrics-server
rules:
- nonResourceURLs:
  - /metrics
  verbs:
  - get
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: eks:metrics-server-nanny
  labels:
    k8s-app: metrics-server
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: eks:metrics-server-nanny
subjects:
  - kind: ServiceAccount
    name: metrics-server
    namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: metrics-server-nanny
  namespace: kube-system
  labels:
    k8s-app: metrics-server
rules:
- apiGroups:
  - ""
  resources:
  - pods
  verbs:
  - get
- apiGroups:
  - apps
  resources:
  - deployments
  resourceNames:
  - metrics-server
  verbs:
  - get
  - patch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: metrics-server-nanny
  namespace: kube-system
  labels:
    k8s-app: metrics-server
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: metrics-server-nanny
subjects:
  - kind: ServiceAccount
    name: metrics-server
    namespace: kube-system
EOF
```

Create a patch file to add the nanny container to the metrics-server Deployment.

```
cat > metrics-server-addon-patch.yaml << EOF
spec:
  template:
    spec:
      containers:
      - name: metrics-server-nanny
          image: registry.k8s.io/autoscaling/addon-resizer:1.8.14
          resources:
            limits:
              cpu: 40m
              memory: 25Mi
            requests:
              cpu: 40m
              memory: 25Mi
          env:
            - name: MY_POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: MY_POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          command:
            - /pod_nanny
            - --cpu=20m
            - --extra-cpu=1m
            - --memory=15Mi
            - --extra-memory=2Mi
            - --threshold=5
            - --deployment=metrics-server
            - --container=metrics-server
            - --poll-period=30000
            - --estimator=exponential
            - --minClusterSize=10
            - --use-metrics=true
EOF

kubectl -n kube-system patch deployments metrics-server --patch-file metrics-server-addon-patch.yaml
```

If you install the metrics-server as documented in Amazon EKS documentation, it requests 100m CPU and 200Mi memory. After deploying the patch above, the metrics server requests set to 40m CPU and 15Mi memory. As you add more nodes, the nanny will automatically adjust the requests and limits for the metrics-server container.

I scaled my cluster to 15 nodes and the addon-resizer configured metrics-server requests to 35m CPU and 45Mi memory. 

```
20baseCPU+(15nodes*1extraCPU) = 35m

```

Memory calculation

```
15baseMemory+(15nodes*2extraMemory) = 45Mi

```

Addon-resizer calculates the resources reservation for the metrics-server container and  restarts the container automatically.

## What about scaling metrics server horizontally?

While you can run metrics server in high-availability mode, its main purpose is ensuring that if one of the metrics server Pods terminate, the other one can still serve requests.

Running two instances of metrics server doesn’t provide any further benefits. Both instances will scrape all nodes to collect metrics, but only one instance will be actively serving metrics API.

## Conclusion

The Amazon EKS documentation currently documents steps to deploy metrics-server in static configuration. You can use addon-resizer to autoscale the metrics-server based on the number of nodes in your cluster. 