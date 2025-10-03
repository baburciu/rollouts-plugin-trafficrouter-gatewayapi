# Using Cilium with Argo Rollouts for header based traffic split

## Prerequisites

A Kubernetes cluster. If you do not have one, you can create one using [kind](https://kind.sigs.k8s.io/), [minikube](https://minikube.sigs.k8s.io/), or any other Kubernetes cluster. This guide will use Kind.

## Step 1 - Create a Kind cluster by running the following command

```shell
kind create cluster --config ./kind-cluster.yaml
```

## Step 2 - Install Cilium

I will use helm to install Cilium in the cluster, but before that we'll need to install Gateway API CRDs. You can also install Cilium using [cilium CLI](https://docs.cilium.io/en/stable/gettingstarted/k8s-install-default/#install-the-cilium-cli).

> [!NOTE]
> Cilium `v1.18.2` supports Gateway API v1.2.0, per [docs](https://docs.cilium.io/en/stable/network/servicemesh/gateway-api/gateway-api/).

```shell
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.2.0/config/crd/standard/gateway.networking.k8s.io_gatewayclasses.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.2.0/config/crd/standard/gateway.networking.k8s.io_gateways.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.2.0/config/crd/standard/gateway.networking.k8s.io_httproutes.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.2.0/config/crd/standard/gateway.networking.k8s.io_referencegrants.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.2.0/config/crd/standard/gateway.networking.k8s.io_grpcroutes.yaml
```
```shell
helm repo add cilium https://helm.cilium.io/
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
helm install cilium cilium/cilium --version 1.18.2 \
     --namespace kube-system \
     --set image.pullPolicy=IfNotPresent \
     --set ipam.mode=kubernetes \
     --set cni.exclusive=false \
     --set kubeProxyReplacement=true \
     --set gatewayAPI.enabled=true \
     --wait
cilium status --wait
```

## Step 3 - Install Argo Rollouts and Argo Rollouts plugin to allow Cilium to manage the traffic

```shell
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

NAME                                       KIND        STATUS     AGE  INFO
⟳ rollouts-demo                            Rollout     ॥ Paused   14m
├──# revision:2
│  └──⧉ rollouts-demo-7bd564d79f           ReplicaSet  ✔ Healthy  4s   canary
│     ├──□ rollouts-demo-7bd564d79f-4wwcf  Pod         ✔ Running  4s   ready:1/1
│     ├──□ rollouts-demo-7bd564d79f-8ltts  Pod         ✔ Running  4s   ready:1/1
│     └──□ rollouts-demo-7bd564d79f-g85z6  Pod         ✔ Running  4s   ready:1/1
└──# revision:1
   └──⧉ rollouts-demo-784858d6db           ReplicaSet  ✔ Healthy  14m  stable
      ├──□ rollouts-demo-784858d6db-8bqp6  Pod         ✔ Running  14m  ready:1/1
      ├──□ rollouts-demo-784858d6db-8rx54  Pod         ✔ Running  14m  ready:1/1
      ├──□ rollouts-demo-784858d6db-kvgjx  Pod         ✔ Running  14m  ready:1/1
      ├──□ rollouts-demo-784858d6db-mnx79  Pod         ✔ Running  14m  ready:1/1
      └──□ rollouts-demo-784858d6db-r56wg  Pod         ✔ Running  14m  ready:1/1
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
  creationTimestamp: "2025-10-03T11:10:07Z"
  generation: 2
  name: argo-rollouts-http-route
  namespace: default
  resourceVersion: "5550"
  uid: c60e21fa-1faf-4d45-afd5-15cabc698771
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
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"gateway.networking.k8s.io/v1beta1","kind":"HTTPRoute","metadata":{"annotations":{},"name":"argo-rollouts-http-route","namespace":"default"},"spec":{"parentRefs":[{"group":"core","kind":"Service","name":"argo-rollouts-service","port":80}],"rules":[{"backendRefs":[{"group":"core","kind":"Service","name":"argo-rollouts-stable-service","port":80},{"group":"core","kind":"Service","name":"argo-rollouts-canary-service","port":80}]}]}}
  creationTimestamp: "2025-10-03T11:10:07Z"
  generation: 3
  name: argo-rollouts-http-route
  namespace: default
  resourceVersion: "7504"
  uid: c60e21fa-1faf-4d45-afd5-15cabc698771
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
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"gateway.networking.k8s.io/v1beta1","kind":"HTTPRoute","metadata":{"annotations":{},"name":"argo-rollouts-http-route","namespace":"default"},"spec":{"parentRefs":[{"group":"core","kind":"Service","name":"argo-rollouts-service","port":80}],"rules":[{"backendRefs":[{"group":"core","kind":"Service","name":"argo-rollouts-stable-service","port":80},{"group":"core","kind":"Service","name":"argo-rollouts-canary-service","port":80}]}]}}
  creationTimestamp: "2025-10-03T11:10:07Z"
  generation: 4
  name: argo-rollouts-http-route
  namespace: default
  resourceVersion: "7506"
  uid: c60e21fa-1faf-4d45-afd5-15cabc698771
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
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"gateway.networking.k8s.io/v1beta1","kind":"HTTPRoute","metadata":{"annotations":{},"name":"argo-rollouts-http-route","namespace":"default"},"spec":{"parentRefs":[{"group":"core","kind":"Service","name":"argo-rollouts-service","port":80}],"rules":[{"backendRefs":[{"group":"core","kind":"Service","name":"argo-rollouts-stable-service","port":80},{"group":"core","kind":"Service","name":"argo-rollouts-canary-service","port":80}]}]}}
  creationTimestamp: "2025-10-03T11:10:07Z"
  generation: 5
  name: argo-rollouts-http-route
  namespace: default
  resourceVersion: "7510"
  uid: c60e21fa-1faf-4d45-afd5-15cabc698771
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
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"gateway.networking.k8s.io/v1beta1","kind":"HTTPRoute","metadata":{"annotations":{},"name":"argo-rollouts-http-route","namespace":"default"},"spec":{"parentRefs":[{"group":"core","kind":"Service","name":"argo-rollouts-service","port":80}],"rules":[{"backendRefs":[{"group":"core","kind":"Service","name":"argo-rollouts-stable-service","port":80},{"group":"core","kind":"Service","name":"argo-rollouts-canary-service","port":80}]}]}}
  creationTimestamp: "2025-10-03T11:10:07Z"
  generation: 6
  name: argo-rollouts-http-route
  namespace: default
  resourceVersion: "7568"
  uid: c60e21fa-1faf-4d45-afd5-15cabc698771
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
```

And we can see it also in the managed routes configmap the plugin writes to:
```shell
$ kubectl get cm argo-gatewayapi-configmap -o yaml --watch
apiVersion: v1
kind: ConfigMap
metadata:
  creationTimestamp: "2025-10-03T11:10:20Z"
  name: argo-gatewayapi-configmap
  namespace: default
  resourceVersion: "5420"
  uid: 56eb3613-7843-49f8-8061-16d40c9fbeac
---
apiVersion: v1
data:
  httpManagedRoutes: '{"argo-rollouts":{"argo-rollouts-http-route":1}}'
kind: ConfigMap
metadata:
  creationTimestamp: "2025-10-03T11:10:20Z"
  name: argo-gatewayapi-configmap
  namespace: default
  resourceVersion: "7505"
  uid: 56eb3613-7843-49f8-8061-16d40c9fbeac
---
apiVersion: v1
data:
  httpManagedRoutes: '{}'
kind: ConfigMap
metadata:
  creationTimestamp: "2025-10-03T11:10:20Z"
  name: argo-gatewayapi-configmap
  namespace: default
  resourceVersion: "7511"
  uid: 56eb3613-7843-49f8-8061-16d40c9fbeac
```
