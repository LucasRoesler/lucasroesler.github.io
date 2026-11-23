---
title: "Customizing KinD for easy local development"
date: 2021-01-30T00:00:00+02:00
tags:
  - programming
  - kubernetes
  - kind
  - ingress
draft: true
# coverImage: images/cover1.jpg
---

I have always had an allergic reaction to network dependencies for local development. This is why I love projects like KinD, minikube, and microk8s (just to name a few).

<!-- more -->

Create a file `cluster.yaml` with
```yaml
# cluster.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
  - containerPort: 31112 # this is the NodePort created by the helm chart
    hostPort: 8080 # this is your port on localhost
    protocol: TCP
```

then, run

```
kind create cluster --config=cluster.yaml
```