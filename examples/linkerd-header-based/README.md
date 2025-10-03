# Using Linkerd with Argo Rollouts for header based traffic split

[Linkerd](https://linkerd.io/) is a service mesh for Kubernetes. It makes running services easier and safer by giving you runtime debugging, observability, reliability, and security—all without requiring any changes to your code.

## Prerequisites

A Kubernetes cluster. If you do not have one, you can create one using [kind](https://kind.sigs.k8s.io/), [minikube](https://minikube.sigs.k8s.io/), or any other Kubernetes cluster. This guide will use Kind.

## Step 1 - Create a Kind cluster by running the following command

```shell
kind create cluster --config ./kind-cluster.yaml
```

## Step 2 - Install Linkerd and Linkerd Viz by running the following commands

I will use the Linkerd CLI to install Linkerd in the cluster. You can also install Linkerd using Helm or kubectl.
I tested this guide with Linkerd version `edge-25.9.4`.

> [!IMPORTANT]
> Linkerd version `edge-25.9.4` uses `v1` GatewayAPI apiVersion and the [plugin](https://github.com/argoproj-labs/rollouts-plugin-trafficrouter-gatewayapi) `v0.8.0` expects that. It wouldn't work if `v1beta1` GatewayAPI apiVersion CRDs would be installed (like in the case of an older Linkerd `stable-2.14.10`)

```shell
export LINKERD2_VERSION=edge-25.9.4; curl --proto '=https' --tlsv1.2 -sSfL https://run.linkerd.io/install-edge | sh
export PATH=$PATH:$HOME/.linkerd2/bin
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.1/standard-install.yaml # Gateway API CRDs must be installed prior to installing Linkerd
linkerd install --crds | kubectl apply -f -
linkerd install | kubectl apply -f - && linkerd check
```

## Step 3 - Install Argo Rollouts and Argo Rollouts plugin to allow Linkerd to manage the traffic

```shell
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
helm install argo-rollouts argo/argo-rollouts --version 2.40.4 \
  --namespace argo-rollouts \
  --create-namespace \
  --set 'controller.trafficRouterPlugins[0].location=https://github.com/argoproj-labs/rollouts-plugin-trafficrouter-gatewayapi/releases/download/v0.8.0/gatewayapi-plugin-linux-amd64' \
  --set 'controller.trafficRouterPlugins[0].name=argoproj-labs/gatewayAPI'
```

## Step 4 - Create the services required for traffic split

Create three Services required for canary based rollout strategy

```shell
kubectl apply -f service.yaml
```

## Step 5 - Create HTTPRoute that defines a traffic split between two services

Create a GAMMA [producer `HTTPRoute`](https://gateway-api.sigs.k8s.io/concepts/glossary/#producer-route) resource and connect it to a parent K8s service (using a canary and stable K8s services as backends)

```shell
kubectl apply -f httproute.yaml
```

## Step 6 - Create an example Rollout whose first step is `setHeaderRoute`

First add the Linkerd annotation to the namespace where the pods are deployed to enable [Automatic Proxy Injection](https://linkerd.io/2-edge/features/proxy-injection/)

```shell
kubectl annotate namespace default linkerd.io/inject=enabled
```

Deploy a rollout to get the initial version

```shell
kubectl apply -f rollout.yaml
```

## Step 7 - Patch the rollout to see the canary deployment
```shell
kubectl patch rollout rollouts-demo --type='json' -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/env/0/value", "value": "1.1.0"}]'
```

## Step 8 - Observe the rollout and HTTPRoute rule addition of [canary header matching rule](https://gateway-api.sigs.k8s.io/guides/traffic-splitting/#canary-traffic-rollout) being dropped

Rollout is in an intermediary step (it is not completed):
```shell
$ kubectl argo rollouts get rollout rollouts-demo
Name:            rollouts-demo
Namespace:       default
Status:          ॥ Paused
Message:         CanaryPauseStep
Strategy:        Canary
  Step:          2/5
  SetWeight:     50
  ActualWeight:  50
Images:          hashicorp/http-echo:1.0 (canary, stable)
Replicas:
  Desired:       5
  Current:       8
  Updated:       3
  Ready:         8
  Available:     8

NAME                                       KIND        STATUS     AGE    INFO
⟳ rollouts-demo                            Rollout     ॥ Paused   2m48s
├──# revision:2
│  └──⧉ rollouts-demo-7bd564d79f           ReplicaSet  ✔ Healthy  38s    canary
│     ├──□ rollouts-demo-7bd564d79f-8kfq7  Pod         ✔ Running  38s    ready:2/2
│     ├──□ rollouts-demo-7bd564d79f-rt2zn  Pod         ✔ Running  38s    ready:2/2
│     └──□ rollouts-demo-7bd564d79f-s52t5  Pod         ✔ Running  38s    ready:2/2
└──# revision:1
   └──⧉ rollouts-demo-6f45555c56           ReplicaSet  ✔ Healthy  2m48s  stable
      ├──□ rollouts-demo-6f45555c56-fxlrc  Pod         ✔ Running  2m48s  ready:2/2
      ├──□ rollouts-demo-6f45555c56-hc8zs  Pod         ✔ Running  2m48s  ready:2/2
      ├──□ rollouts-demo-6f45555c56-lj5zr  Pod         ✔ Running  2m48s  ready:2/2
      ├──□ rollouts-demo-6f45555c56-qnjfr  Pod         ✔ Running  2m48s  ready:2/2
      └──□ rollouts-demo-6f45555c56-xddxd  Pod         ✔ Running  2m48s  ready:2/2
$
```
but HTTPRoute header matching rule is not there:
```shell
$ kubectl get httproute argo-rollouts-http-route -o yaml | yq .spec.rules
- backendRefs:
    - group: core
      kind: Service
      name: argo-rollouts-stable-service
      port: 80
      weight: 50
    - group: core
      kind: Service
      name: argo-rollouts-canary-service
      port: 80
      weight: 50
  matches:
    - path:
        type: PathPrefix
        value: /
```

It seems HTTPRoute rule was added and then removed:
```shell
$ kubectl get httproute argo-rollouts-http-route -o yaml --watch
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"gateway.networking.k8s.io/v1beta1","kind":"HTTPRoute","metadata":{"annotations":{},"name":"argo-rollouts-http-route","namespace":"default"},"spec":{"parentRefs":[{"group":"core","kind":"Service","name":"argo-rollouts-service","port":80}],"rules":[{"backendRefs":[{"group":"core","kind":"Service","name":"argo-rollouts-stable-service","port":80},{"group":"core","kind":"Service","name":"argo-rollouts-canary-service","port":80}]}]}}
  creationTimestamp: "2025-10-03T12:38:34Z"
  generation: 2
  name: argo-rollouts-http-route
  namespace: default
  resourceVersion: "1545"
  uid: af2c7847-5477-4534-b570-3851cd07c19e
spec:
  parentRefs:
  - group: core
    kind: Service
    name: argo-rollouts-service
    port: 80
  rules:
  - backendRefs:
    - group: core
      kind: Service
      name: argo-rollouts-stable-service
      port: 80
      weight: 100
    - group: core
      kind: Service
      name: argo-rollouts-canary-service
      port: 80
      weight: 0
    matches:
    - path:
        type: PathPrefix
        value: /
status:
  parents:
  - conditions:
    - lastTransitionTime: "2025-10-03T12:38:34Z"
      message: ""
      reason: Accepted
      status: "True"
      type: Accepted
    - lastTransitionTime: "2025-10-03T12:38:34Z"
      message: ""
      reason: ResolvedRefs
      status: "True"
      type: ResolvedRefs
    controllerName: linkerd.io/policy-controller
    parentRef:
      group: core
      kind: Service
      name: argo-rollouts-service
      namespace: default
      port: 80
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"gateway.networking.k8s.io/v1beta1","kind":"HTTPRoute","metadata":{"annotations":{},"name":"argo-rollouts-http-route","namespace":"default"},"spec":{"parentRefs":[{"group":"core","kind":"Service","name":"argo-rollouts-service","port":80}],"rules":[{"backendRefs":[{"group":"core","kind":"Service","name":"argo-rollouts-stable-service","port":80},{"group":"core","kind":"Service","name":"argo-rollouts-canary-service","port":80}]}]}}
  creationTimestamp: "2025-10-03T12:38:34Z"
  generation: 3
  name: argo-rollouts-http-route
  namespace: default
  resourceVersion: "1803"
  uid: af2c7847-5477-4534-b570-3851cd07c19e
spec:
  parentRefs:
  - group: core
    kind: Service
    name: argo-rollouts-service
    port: 80
  rules:
  - backendRefs:
    - group: core
      kind: Service
      name: argo-rollouts-stable-service
      port: 80
      weight: 100
    - group: core
      kind: Service
      name: argo-rollouts-canary-service
      port: 80
      weight: 0
    matches:
    - path:
        type: PathPrefix
        value: /
  - backendRefs:
    - group: ""
      kind: Service
      name: argo-rollouts-canary-service
      port: 80
      weight: 1
    matches:
    - headers:
      - name: X-Test
        type: Exact
        value: test
      path:
        type: PathPrefix
        value: /
status:
  parents:
  - conditions:
    - lastTransitionTime: "2025-10-03T12:38:34Z"
      message: ""
      reason: Accepted
      status: "True"
      type: Accepted
    - lastTransitionTime: "2025-10-03T12:38:34Z"
      message: ""
      reason: ResolvedRefs
      status: "True"
      type: ResolvedRefs
    controllerName: linkerd.io/policy-controller
    parentRef:
      group: core
      kind: Service
      name: argo-rollouts-service
      namespace: default
      port: 80
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"gateway.networking.k8s.io/v1beta1","kind":"HTTPRoute","metadata":{"annotations":{},"name":"argo-rollouts-http-route","namespace":"default"},"spec":{"parentRefs":[{"group":"core","kind":"Service","name":"argo-rollouts-service","port":80}],"rules":[{"backendRefs":[{"group":"core","kind":"Service","name":"argo-rollouts-stable-service","port":80},{"group":"core","kind":"Service","name":"argo-rollouts-canary-service","port":80}]}]}}
  creationTimestamp: "2025-10-03T12:38:34Z"
  generation: 4
  name: argo-rollouts-http-route
  namespace: default
  resourceVersion: "1805"
  uid: af2c7847-5477-4534-b570-3851cd07c19e
spec:
  parentRefs:
  - group: core
    kind: Service
    name: argo-rollouts-service
    port: 80
  rules:
  - backendRefs:
    - group: core
      kind: Service
      name: argo-rollouts-stable-service
      port: 80
      weight: 100
    - group: core
      kind: Service
      name: argo-rollouts-canary-service
      port: 80
      weight: 0
    matches:
    - path:
        type: PathPrefix
        value: /
  - backendRefs:
    - group: ""
      kind: Service
      name: argo-rollouts-canary-service
      port: 80
      weight: 0
    matches:
    - headers:
      - name: X-Test
        type: Exact
        value: test
      path:
        type: PathPrefix
        value: /
status:
  parents:
  - conditions:
    - lastTransitionTime: "2025-10-03T12:38:34Z"
      message: ""
      reason: Accepted
      status: "True"
      type: Accepted
    - lastTransitionTime: "2025-10-03T12:38:34Z"
      message: ""
      reason: ResolvedRefs
      status: "True"
      type: ResolvedRefs
    controllerName: linkerd.io/policy-controller
    parentRef:
      group: core
      kind: Service
      name: argo-rollouts-service
      namespace: default
      port: 80
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"gateway.networking.k8s.io/v1beta1","kind":"HTTPRoute","metadata":{"annotations":{},"name":"argo-rollouts-http-route","namespace":"default"},"spec":{"parentRefs":[{"group":"core","kind":"Service","name":"argo-rollouts-service","port":80}],"rules":[{"backendRefs":[{"group":"core","kind":"Service","name":"argo-rollouts-stable-service","port":80},{"group":"core","kind":"Service","name":"argo-rollouts-canary-service","port":80}]}]}}
  creationTimestamp: "2025-10-03T12:38:34Z"
  generation: 5
  name: argo-rollouts-http-route
  namespace: default
  resourceVersion: "1809"
  uid: af2c7847-5477-4534-b570-3851cd07c19e
spec:
  parentRefs:
  - group: core
    kind: Service
    name: argo-rollouts-service
    port: 80
  rules:
  - backendRefs:
    - group: core
      kind: Service
      name: argo-rollouts-stable-service
      port: 80
      weight: 100
    - group: core
      kind: Service
      name: argo-rollouts-canary-service
      port: 80
      weight: 0
    matches:
    - path:
        type: PathPrefix
        value: /
status:
  parents:
  - conditions:
    - lastTransitionTime: "2025-10-03T12:38:34Z"
      message: ""
      reason: Accepted
      status: "True"
      type: Accepted
    - lastTransitionTime: "2025-10-03T12:38:34Z"
      message: ""
      reason: ResolvedRefs
      status: "True"
      type: ResolvedRefs
    controllerName: linkerd.io/policy-controller
    parentRef:
      group: core
      kind: Service
      name: argo-rollouts-service
      namespace: default
      port: 80
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"gateway.networking.k8s.io/v1beta1","kind":"HTTPRoute","metadata":{"annotations":{},"name":"argo-rollouts-http-route","namespace":"default"},"spec":{"parentRefs":[{"group":"core","kind":"Service","name":"argo-rollouts-service","port":80}],"rules":[{"backendRefs":[{"group":"core","kind":"Service","name":"argo-rollouts-stable-service","port":80},{"group":"core","kind":"Service","name":"argo-rollouts-canary-service","port":80}]}]}}
  creationTimestamp: "2025-10-03T12:38:34Z"
  generation: 6
  name: argo-rollouts-http-route
  namespace: default
  resourceVersion: "1900"
  uid: af2c7847-5477-4534-b570-3851cd07c19e
spec:
  parentRefs:
  - group: core
    kind: Service
    name: argo-rollouts-service
    port: 80
  rules:
  - backendRefs:
    - group: core
      kind: Service
      name: argo-rollouts-stable-service
      port: 80
      weight: 50
    - group: core
      kind: Service
      name: argo-rollouts-canary-service
      port: 80
      weight: 50
    matches:
    - path:
        type: PathPrefix
        value: /
status:
  parents:
  - conditions:
    - lastTransitionTime: "2025-10-03T12:38:34Z"
      message: ""
      reason: Accepted
      status: "True"
      type: Accepted
    - lastTransitionTime: "2025-10-03T12:38:34Z"
      message: ""
      reason: ResolvedRefs
      status: "True"
      type: ResolvedRefs
    controllerName: linkerd.io/policy-controller
    parentRef:
      group: core
      kind: Service
      name: argo-rollouts-service
      namespace: default
      port: 80
```

And we can see it also in the managed routes configmap the plugin writes to:
```shell
$ kubectl get cm argo-gatewayapi-configmap -o yaml --watch
apiVersion: v1
kind: ConfigMap
metadata:
  creationTimestamp: "2025-10-03T12:39:33Z"
  name: argo-gatewayapi-configmap
  namespace: default
  resourceVersion: "1335"
  uid: 0a7cd183-1599-4309-affe-3cf53690ad1f
---
apiVersion: v1
data:
  httpManagedRoutes: '{"argo-rollouts":{"argo-rollouts-http-route":1}}'
kind: ConfigMap
metadata:
  creationTimestamp: "2025-10-03T12:39:33Z"
  name: argo-gatewayapi-configmap
  namespace: default
  resourceVersion: "1804"
  uid: 0a7cd183-1599-4309-affe-3cf53690ad1f
---
apiVersion: v1
data:
  httpManagedRoutes: '{}'
kind: ConfigMap
metadata:
  creationTimestamp: "2025-10-03T12:39:33Z"
  name: argo-gatewayapi-configmap
  namespace: default
  resourceVersion: "1810"
  uid: 0a7cd183-1599-4309-affe-3cf53690ad1f
```
