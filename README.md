# Nginx Canary Deployments via GitOps using Flux and Flagger

## Prerequisites

- A Kubernetes Cluster
- A GitHub personal Access Token with repo permissions
- Flux

## Install the Flux CLI

Use the following command to install flux-cli in linux.

```bash
curl -S https://fluxcd.io/install.sh | sudo bash
```

## Export your credentials

Export your GitHub personal access token and username:

```bash
export GITHUB_TOKEN=<your-token>
export GITHUB_USER=<your-username>
```

## Check your Kubernetes cluster

Check you have everything needed to run Flux by running the following command:

```bash
flux check --pre
```

## Install Flux onto your cluster 

Run the following bootstrap command:

```bash
flux bootstrap github \
  --owner=<github-username> \
  --repository=<repo-name> \
  --branch=main \
  --path=./clusters/my-cluster \
  --personal
```

The above bootstrap command does the following:

- Creates a git repository ``` <repo-name> ``` on your GitHub Account.
- Adds Flux components manifests to the repository.
- Deploy Flux components to your Kubernetes Cluster.
- Configures Flux components to track the path ``` /clusters/my-cluster/ ``` in the repository.

## Install the NGINX ingress controller

- Helm v3 Approach
  
```bash

# Install the NGINX ingress controller with Helm v3
kubectl create ns ingress-nginx

helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx

helm upgrade -i ingress-nginx ingress-nginx/ingress-nginx \
--namespace ingress-nginx \
--set controller.metrics.enabled=true \
--set controller.podAnnotations."prometheus\.io/scrape"=true \
--set controller.podAnnotations."prometheus\.io/port"=10254
```

- GitOps using Declarative approach

```bash
---
apiVersion: v1
kind: Namespace
metadata:
  name: ingress-nginx
  labels:
    toolkit.fluxcd.io/tenant: sre-team
---
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: HelmRepository
metadata:
  name: ingress-nginx
  namespace: ingress-nginx
spec:
  interval: 1h
  url: https://kubernetes.github.io/ingress-nginx
---
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: ingress-nginx
  namespace: ingress-nginx
spec:
  interval: 1h
  releaseName: ingress-nginx
  chart:
    spec:
      chart: ingress-nginx
      interval: 1h
      sourceRef:
        kind: HelmRepository
        name: ingress-nginx
  values:
    controller:
      metrics:
        enabled: true
      podAnnotations:
        "prometheus.io/scrape": "true"
        "prometheus.io/port": "10254"
```

## Install Flagger and the Prometheus add-on in the same namespace as the ingress controller

- Helm v3 Approach
  
```bash

# Installs the Flagger and the Prometheus add-on with Helm v3
helm repo add flagger https://flagger.app

helm upgrade -i flagger flagger/flagger \
--namespace ingress-nginx \
--set prometheus.install=true \
--set meshProvider=nginx
```

- GitOps using Declarative approach

```bash
---
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: HelmRepository
metadata:
  name: flagger
  namespace: ingress-nginx
spec:
  interval: 1h
  url: https://flagger.app
---
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: flagger
  namespace: ingress-nginx
spec:
  interval: 1h
  releaseName: flagger
  install: # override existing Flagger CRDs
    crds: CreateReplace
  upgrade: # update Flagger CRDs
    crds: CreateReplace
  chart:
    spec:
      chart: flagger
      version: 1.30.0 # update Flagger to the latest minor version
      interval: 6h # scan for new versions every six hours
      sourceRef:
        kind: HelmRepository
        name: flagger
  values:
    prometheus:
      install: true
    meshProvider: nginx
    metricsServer: http://prometheus.monitoring:9090
```
