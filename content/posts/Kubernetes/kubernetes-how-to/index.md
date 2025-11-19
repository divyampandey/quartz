---
title: Kubernetes-how-to
description: Small nuggets of how-to's for the k8
tags:
  - notes
draft: false
---
# How to configure `kubectl` to talk to a remote Kubernetes cluster

`kubectl` is the CLI we use to talk to a Kubernetes cluster. It reads connection details from a **kubeconfig** file (usually `~/.kube/config`).



**My** `~/.kube` **folder**

![Kube config directory](https://divyampandey.github.io/quartz/assets/kube-config.png)

### What’s inside kubeconfig

A kubeconfig is just YAML that tells `kubectl`:

- which **clusters** exist (API server URL + CA cert),
- which **users/credentials** to use (client cert/key or token),
- which **contexts** tie a _user_ to a _cluster_ (plus an optional namespace).

The important bits you’ll see:
- **Cluster**
    - `name`: e.g. `docker-desktop`, `minikube`, `rancher-desktop`, or a KIND cluster like `kind-argo-demo`
    - `server`: API server address, e.g. `https://127.0.0.1:6443` or a cloud endpoint
    - `certificate-authority-data`: the cluster CA (base64)
    
- **User**
    - `client-certificate-data` and `client-key-data` (for client-cert auth), or
    - a bearer token or exec plugin (for cloud auth providers)
    
- **Context**
    - binds `cluster` + `user` (and optionally `namespace`)
    - you “use” a context to point `kubectl` at one cluster with one identity

A minimal shape looks like this:
```
apiVersion: v1
kind: Config
clusters:
  - name: docker-desktop
    cluster:
      server: https://127.0.0.1:6443
      certificate-authority-data: <base64>
users:
  - name: docker-desktop
    user:
      client-certificate-data: <base64>
      client-key-data: <base64>
contexts:
  - name: docker-desktop
    context:
      cluster: docker-desktop
      user: docker-desktop
current-context: docker-desktop
preferences: {}

```


## Add a remote cluster (without breaking your local ones)

 **My remote** `kubeconfig` **file:**
![Remote kubeconfig](https://divyampandey.github.io/quartz/assets/remote-kubeconfig.png)


If your cloud provider gave you a separate kubeconfig file (say `~/Downloads/do.kubeconfig`), you don’t have to overwrite `~/.kube/config`. Merge them and keep everything:

```
# Temporarily point KUBECONFIG at both files
KUBECONFIG=$HOME/.kube/config:~/Downloads/do.kubeconfig \
kubectl config view --merge --flatten > /tmp/merged.kubeconfig

# Replace your main config with the merged one (optional but convenient)
mv /tmp/merged.kubeconfig $HOME/.kube/config

```

**My merged** `config` **file**:

![Merged config](https://divyampandey.github.io/quartz/assets/merged-kubeconfig.png)



## See what you’ve got and switch safely

```
# list contexts (which cluster+user pairs are available)
kubectl config get-contexts

# show the current context
kubectl config current-context

# switch to the remote context (example)
kubectl config use-context do-sgp1-my-prod

# verify you’re talking to the right cluster
kubectl cluster-info
kubectl get nodes

```


That’s it. Merge your remote kubeconfig, switch context, and you’re in.