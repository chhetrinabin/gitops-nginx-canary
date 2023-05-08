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

## Bootstrap

### Create a test namespace, deployment and horizonatl pod autoscaler

- Using Helm v3 and imperative approach

```bash
kubectl create ns test

kubectl apply -k https://github.com/fluxcd/flagger//kustomize/podinfo?ref=main
```

- GitOps using declarative approach

```bash
apiVersion: v1
kind: Namespace
metadata:
  name: test
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: podinfo
  namespace: test
  labels:
    app: podinfo
spec:
  minReadySeconds: 5
  revisionHistoryLimit: 5
  progressDeadlineSeconds: 60
  strategy:
    rollingUpdate:
      maxUnavailable: 1
    type: RollingUpdate
  selector:
    matchLabels:
      app: podinfo
  template:
    metadata:
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9797"
      labels:
        app: podinfo
    spec:
      containers:
        - name: podinfod
          image: ghcr.io/stefanprodan/podinfo:6.0.0
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 9898
              protocol: TCP
            - name: http-metrics
              containerPort: 9797
              protocol: TCP
            - name: grpc
              containerPort: 9999
              protocol: TCP
          command:
            - ./podinfo
            - --port=9898
            - --port-metrics=9797
            - --grpc-port=9999
            - --grpc-service-name=podinfo
            - --level=info
            - --random-delay=false
            - --random-error=false
          env:
            - name: PODINFO_UI_COLOR
              value: "#34577c"
          livenessProbe:
            exec:
              command:
                - podcli
                - check
                - http
                - localhost:9898/healthz
            initialDelaySeconds: 5
            timeoutSeconds: 5
          readinessProbe:
            exec:
              command:
                - podcli
                - check
                - http
                - localhost:9898/readyz
            initialDelaySeconds: 5
            timeoutSeconds: 5
          resources:
            limits:
              cpu: 2000m
              memory: 512Mi
            requests:
              cpu: 100m
              memory: 64Mi
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: podinfo
  namespace: test
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: podinfo
  minReplicas: 2
  maxReplicas: 4
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          # scale up if usage is above
          # 99% of the requested CPU (100m)
          averageUtilization: 99
```

### Deploy the load testing service to generate traffic during the canary analysis

- Using Helm v3 imperative approach

```bash
helm upgrade -i flagger-loadtester flagger/loadtester \
--namespace=test
```

- GitOps using declarative approach

```bash
---
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: OCIRepository
metadata:
  name: flagger-loadtester
  namespace: test
spec:
  interval: 6h # scan for new versions every six hours
  url: oci://ghcr.io/fluxcd/flagger-manifests
  ref:
    semver: 1.x # update to the latest version
  verify: # verify the artifact signature with Cosign keyless
    provider: cosign
---
apiVersion: kustomize.toolkit.fluxcd.io/v1beta2
kind: Kustomization
metadata:
  name: flagger-loadtester
  namespace: test
spec:
  interval: 6h
  wait: true
  timeout: 5m
  prune: true
  sourceRef:
    kind: OCIRepository
    name: flagger-loadtester
  path: ./tester
  targetNamespace: test
```

### Create an ingress definition (replace app.example.com with your own domain)

```bash
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: podinfo
  namespace: test
  labels:
    app: podinfo
  annotations:
    kubernetes.io/ingress.class: "nginx"
spec:
  rules:
    - host: "app.example.com"
      http:
        paths:
          - pathType: Prefix
            path: "/"
            backend:
              service:
                name: podinfo
                port:
                  number: 80
````

### Create a canary custom resource (replace app.example.com with your own domain)

```bash
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: podinfo
  namespace: test
spec:
  provider: nginx
  # deployment reference
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: podinfo
  # ingress reference
  ingressRef:
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    name: podinfo
  # HPA reference (optional)
  autoscalerRef:
    apiVersion: autoscaling/v2
    kind: HorizontalPodAutoscaler
    name: podinfo
  # the maximum time in seconds for the canary deployment
  # to make progress before it is rollback (default 600s)
  progressDeadlineSeconds: 60
  service:
    # ClusterIP port number
    port: 80
    # container port number or name
    targetPort: 9898
  analysis:
    # schedule interval (default 60s)
    interval: 10s
    # max number of failed metric checks before rollback
    threshold: 10
    # max traffic percentage routed to canary
    # percentage (0-100)
    maxWeight: 50
    # canary increment step
    # percentage (0-100)
    stepWeight: 5
    # NGINX Prometheus checks
    metrics:
    - name: request-success-rate
      # minimum req success rate (non 5xx responses)
      # percentage (0-100)
      thresholdRange:
        min: 99
      interval: 1m
    # testing (optional)
    webhooks:
      - name: acceptance-test
        type: pre-rollout
        url: http://flagger-loadtester.test/
        timeout: 30s
        metadata:
          type: bash
          cmd: "curl -sd 'test' http://podinfo-canary/token | grep token"
      - name: load-test
        url: http://flagger-loadtester.test/
        timeout: 5s
        metadata:
          cmd: "hey -z 1m -q 10 -c 2 -host 'app.example.com' 'http://ingress-nginx-controller.ingress-nginx'"
```
