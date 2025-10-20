---
title: Kubernetes-Pods
description: Everything about kubernetes pod
tags:
  - notes
draft: false
---
# Pods
Kubernetes Pods are very similar to the docker container. 
The main difference is Pod allows to run 1 or more containers within it - e.g sidecar pattern.

Inside the pod container share the same network and resources and can communicate with each other using the localhost.


**Table comparison: Docker vs Kubernetes:**


![Comparison table](https://divyampandey.github.io/quartz/assets/pod-comparison.png)


### Points to Remember 

* A`Pod` always runs on a `Node`  (usually on a worker node)
*  A `Node` is a worker machine in kubernetes.
* Each `Node` is managed by the control plane in kubernetes.
* A `Node` can have multiple pods.

![Get nodes](https://divyampandey.github.io/quartz/assets/get-nodes.png)
* `Pod` running in the `Node` has different IP addresses. 
* Below you can see the ip address of the node is different than the ip address of the pod 

![get-nodes-wide](https://divyampandey.github.io/quartz/assets/get-nodes-wide.png)


![pods-in-node](https://divyampandey.github.io/quartz/assets/pods-in-node.png)