# Iconicto Kubernetes Config
GitOps Implementation for Iconicto's Kubernetes Cluster Using Weaveworks' [FluxCD](https://fluxcd.io/)
> __Disclaimer :__ This repo only contains configs for open source projects maintained by Iconicto, All the configs and helm charts for private/client's projects are stored in private repository for security and privacy concerns

## How it works
![](https://fluxcd.io/img/flux-cd-diagram.png)

## Prerequisites

- [Helm](https://helm.sh/docs/intro/install/)
- [fluxctl](https://docs.fluxcd.io/en/1.20.2/references/fluxctl/)
- [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
- [kubeseal](https://github.com/bitnami-labs/sealed-secrets#installation)

## Important Links

- <https://www.weave.works/technologies/gitops>
- <https://github.com/fluxcd/helm-operator>
- <https://github.com/fluxcd/helm-operator-get-started>
- <https://helm.workshop.flagger.dev>
- <https://github.com/bitnami-labs/sealed-secrets>


### Install Flux

The first step in automating Helm releases with [Flux](https://github.com/fluxcd/flux) is to create a Git repository with your charts source code.

Add FluxCD repository to Helm repos:

```bash
helm repo add fluxcd https://charts.fluxcd.io
```

Create the `fluxcd` namespace:

```sh
kubectl create ns fluxcd
```

Install Flux:

```bash
helm upgrade -i flux fluxcd/flux --wait \
--namespace fluxcd \
--set registry.pollInterval=1m \
--set git.pollInterval=1m \
--set git.url=git@github.com:Iconicto/Kubernetes-Config.git \
--set syncGarbageCollection.enabled=true
```

Install Flux Helm Operator with __Helm v3__ support:

```bash
helm upgrade -i helm-operator fluxcd/helm-operator --wait \
--namespace fluxcd \
--set git.ssh.secretName=flux-git-deploy \
--set git.pollInterval=1m \
--set chartsSyncInterval=1m \
--set helm.versions=v3 \
--set createCRD=true
```

The Flux Helm operator provides an extension to Flux that automates Helm Chart releases for it.
A Chart release is described through a Kubernetes custom resource named HelmRelease.
The Flux daemon synchronizes these resources from git to the cluster,
and the Flux Helm operator makes sure Helm charts are released as specified in the resources.

_Note that Flux Helm Operator works with Kubernetes 1.11 or newer._

At startup, Flux generates a SSH key and logs the public key. Find the public key with:

```bash
fluxctl identity --k8s-fwd-ns fluxcd
```

In order to sync your cluster state with Git you need to copy the public key and
create a **deploy key** with **write access** on your GitHub repository.

Open GitHub, navigate to your fork, go to _Setting > Deploy keys_ click on _Add deploy key_, check
_Allow write access_, paste the Flux public key and click _Add key_.

## Sealed secrets

A Kubernetes controller and tool for one-way encrypted Secrets

At startup, the sealed-secrets controller generates a RSA key and logs the public key. Using kubeseal you can save your public key as kubeseal-cert.pem, the public key can be safely stored in Git, and can be used to encrypt secrets without direct access to the Kubernetes cluster:

```bash
kubeseal --fetch-cert \
--controller-namespace=fluxcd \
--controller-name=sealed-secrets \
> kubeseal-cert.pem
```

Update the FILE variable with kubernetes secret object you want to encrypt

```bash
FILE=everything-flutter/secrets.yaml; mkdir -p "secrets/$(dirname $FILE)" && kubeseal --format=yaml --cert=kubeseal-cert.pem < decrypted/$FILE > secrets/$FILE
```

Then push to origin and Flux will pull it descrypt and deploy it

```bash
git add $FILE
git commit -m "Added $FILE Secret"
git push $ENV
```
