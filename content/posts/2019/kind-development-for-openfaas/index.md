---
title: "Developing OpenFaaS KinD-ly"
summary: "Creating an isolated OpenFaaS Development environment with KinD"
date: 2019-04-14T00:00:00+02:00
tags:
  - programming
  - openfaas
  - development
  - kubernetes
  - kind
coverImage: tools-498202_1280.jpg
---

There are several ways to run small Kubernetes clusters: [k3s and k3d](https://github.com/rancher/k3d), [minikube](https://kubernetes.io/docs/setup/minikube/), and [microk8s](https://microk8s.io/) to name a few. Currently, my favorite is the Kubernetes provided by [`Docker Desktop for Mac`](https://www.docker.com/products/docker-desktop). But this doesn't help my friends (or myself) when I am on a Linux laptop. It would be nice to have one lightweight and shared environment for Kubernetes.

Luckily, one of the Kubernetes groups has released [KinD - Kubernetes in Docker](https://kind.sigs.k8s.io/). In this post I will show how I use KinD for local development of OpenFaaS. I like KinD because, similar to Docker Desktop, it creates a simple single node Kubernetes environment in a Docker container and has a simple `kind load` command that allows us avoid the push/pull of local development builds of our project.

---

## Creating a dev environment
### Start a KinD cluster
Because we are starting a cluster for development of OpenFaaS, I assume that you have [go](https://golang.org/doc/install) and [docker](https://docs.docker.com/install/) installed already.

```sh
# install/upgrade kind
go get -u sigs.k8s.io/kind
# create a cluster
kind create cluster --name dev-openfaas

# verify everything started
export KUBECONFIG="$(kind get kubeconfig-path --name="dev-openfaas")"
kubectl rollout status deploy coredns --watch -n kube-system
```


### Installing Helm
You must have the [Helm client installed](https://helm.sh/docs/using_helm/#installing-the-helm-client).

Then, setup the RBAC permissions to make the Helm tiller a cluster admin
```sh
kubectl -n kube-system create sa tiller
kubectl create clusterrolebinding tiller \
  --clusterrole cluster-admin \
  --serviceaccount=kube-system:tiller
```

Then, install Helm into the cluster
```sh
# export is not needed if you are in the original terminal used to setup KinD
export KUBECONFIG="$(kind get kubeconfig-path --name="dev-openfaas")"
helm init --skip-refresh --upgrade --service-account tiller
```

### Installing OpenFaaS
First we need to install a vanilla release of OpenFaaS

1. Setup the default namespaces

    ```sh
    kubectl apply -f https://raw.githubusercontent.com/openfaas/faas-netes/master/namespaces.yml
    ```

2. Setup Gateway basic auth credentials

    ```sh
    export PASSWORD="tester"
    kubectl -n openfaas create secret generic basic-auth \
    --from-literal=basic-auth-user=admin \
    --from-literal=basic-auth-password="$PASSWORD"
    ```

3. Instll OpenFaaS via Helm

    ```sh
    helm repo add openfaas https://openfaas.github.io/faas-netes/
    helm repo update
    helm upgrade openfaas --install openfaas/openfaas \
        --namespace openfaas  \
        --set basic_auth=true \
        --set functionNamespace=openfaas-fn
    kubectl rollout status deploy gateway --watch -n openfaas
    ```

    At this point, we have a default OpenFaaS installation isolated in a Docker container. Normally, this would be accessible at http://localhost:31112/ui but this is not exposed yet. Unlike Docker Desktop, NodePort services are not automatically bound to your localhost.  Instead, we must manually expose it.


4. In a separate terminal, we need to expose the

    ```sh
    kubectl -n openfaas port-forward deploy/gateway 31112:8080
    ```

5. Deploy a test function
Now that we can access the gateway, we can deploy a test function with the `faas-cli`

    ```sh
    export GATEWAY=http://localhost:31112
    faas-cli login --gateway=$GATEWAY -u admin -p tester
    faas-cli store deploy nodeinfo --gateway=$GATEWAY
    echo "" | faas-cli invoke nodeinfo --gateway=$GATEWAY
    ```

### Deploy a local development version of the kubernetes provider
Now that we have this environment created, we can start testing our local development builds.

If I want to test a local branch of the faas-netes project, I use the `kind load` command to copy the docker image from my local docker into kind cluster.  This a lot faster than pushing it to a registry and then pulling it back to my laptop!

```sh
make build
kind load docker-image --name dev-openfaas openfaas/faas-netes:latest
export KUBECONFIG="$(kind get kubeconfig-path --name="dev-openfaas")"
helm upgrade openfaas --install openfaas/openfaas \
    --namespace openfaas  \
    --set basic_auth=true \
    --set openfaasImagePullPolicy=IfNotPresent \
    --set faasnetes.imagePullPolicy=IfNotPresent \
    --set faasnetes.image=openfaas/faas-netes:latest \
    --set functionNamespace=openfaas-fn
```

### Cleaning up
If I ever get in trouble and want to start from scratch, I can always recreate the cluster in a minute

```sh
kind delete cluster --name dev-openfaas
unset KUBECONFIG
```
And start again!  I have moved most of this into a simple bash script that I have shared as [a gist.](https://gist.github.com/LucasRoesler/c029f7b7446f7c051522b8edb456b703)

---

KinD provides a lightweight, disposable, and consistent method to test OpenFaaS on Kubernetes without needing a cloud provider or pushing images around the cloud. Going forward, all of my OpenFaaS pull requests will include test instructions that start from a clean cluster using KinD.
