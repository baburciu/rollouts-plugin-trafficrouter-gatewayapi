# Using Cilium with Argo Rollouts for GAMMA route merging

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

## Step 3 - Install Argo Rollouts (`v1.9.0`) and Argo Rollouts plugin to instruct Cilium to manage the traffic

```shell
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
helm install argo-rollouts argo/argo-rollouts --version 2.40.9 \
  --namespace argo-rollouts \
  --create-namespace \
  --set 'controller.trafficRouterPlugins[0].location=https://github.com/argoproj-labs/rollouts-plugin-trafficrouter-gatewayapi/releases/download/v0.13.0/gatewayapi-plugin-linux-amd64' \
  --set 'controller.trafficRouterPlugins[0].name=argoproj-labs/gatewayAPI'
```

## Step 4 - Create the infrastructure for 3 environments, the long-running App and 2 x short-lived PR envs

Each App instance consists of:
- Argo `Rollout` to get the initial version
- 3 x K8s `Services` required for canary based rollout strategy (parent, stable, canary)
- GAMMA [producer `HTTPRoute`](https://gateway-api.sigs.k8s.io/concepts/glossary/#producer-route) connected to a parent K8s service (using a canary and stable K8s services as backends) that defines a traffic split between the two backends

> [!IMPORTANT]
> For Cilium the K8s Services refs need to use `HTTPRoute.parentRefs[].group: ""`. This is different than Linkerd, where `group: "core"` could be used.

## Step 5 - Observe the initial weight ratio set by Argo Rollouts

Even though the GAMMA HTTPRoute is deployed with a 50/50 (or any other) weight split between the stable and canary backend services, Argo Rollouts takes control of the weights during the initial deploy. Since no canary promotion has happened yet, it sets the weight to **100/0** (stable/canary):

```shell
$ kubectl get httproute rollouts-demo-http-route -o yaml | yq .spec.rules
- backendRefs:
    - group: ""
      kind: Service
      name: rollouts-demo-stable-service
      port: 80
      weight: 100
    - group: ""
      kind: Service
      name: rollouts-demo-canary-service
      port: 80
      weight: 0
  matches:
    - path:
        type: PathPrefix
        value: /
```

This is expected behavior: the Argo Rollouts Gateway API plugin [hardcodes the weights to always sum to 100](https://github.com/argoproj-labs/rollouts-plugin-trafficrouter-gatewayapi/blob/v0.13.0/pkg/plugin/httproute.go#L25) (`stable = 100 - canaryWeight`), and on initial deploy (revision 1) all traffic goes to the stable service.

## Step 6 - Test GAMMA route merging with multiple HTTPRoutes on the same parent service

When multiple GAMMA producer HTTPRoutes attach to the same parent K8s service (via `spec.parentRefs`), Cilium merges their backend refs into a single Envoy weighted cluster configuration. This can be verified by inspecting the `CiliumEnvoyConfig` that Cilium creates for the parent service:

```shell
$ kubectl get ciliumenvoyconfig rollouts-demo-service -o yaml | yq '.spec.resources[] | select(.["@type"] == "type.googleapis.com/envoy.config.route.v3.RouteConfiguration") | .virtualHosts[0].routes[0].route.weightedClusters'
clusters:
  - name: default:prod-preview-rollouts-demo-pr-2-stable-service:80
    weight: 100
  - name: default:prod-preview-rollouts-demo-pr-2-canary-service:80
    weight: 0
  - name: default:rollouts-demo-stable-service:80
    weight: 100
  - name: default:rollouts-demo-canary-service:80
    weight: 0
  - name: default:prod-preview-rollouts-demo-pr-1-stable-service:80
    weight: 100
  - name: default:prod-preview-rollouts-demo-pr-1-canary-service:80
    weight: 0
```

All three HTTPRoutes' backends are merged into one Envoy route. Each stable service gets weight **100** out of a total of **300**, meaning each app instance receives ~**33.3%** of the traffic, regardless of its replica count.

Deploy a test pod and send 100 requests to the parent service:

```shell
$ kubectl run -it --image nicolaka/netshoot:v0.13 network-test -- sh  # run a pod to source curl tests
~ #
~ # seq 1 100 | xargs -P 10 -I {} bash -c 'curl -s http://rollouts-demo-service' > pods.txt
~ # sort pods.txt | uniq -c | sort -rn
      8 Hello from prod-preview-rollouts-demo-pr-1-5b9c4cdf76-b9xl8
      7 Hello from prod-preview-rollouts-demo-pr-1-5b9c4cdf76-w6sxr
      6 Hello from rollouts-demo-6f45555c56-cj26m
      6 Hello from prod-preview-rollouts-demo-pr-2-55bb9c94dc-z9gnx
      6 Hello from prod-preview-rollouts-demo-pr-2-55bb9c94dc-j9w94
      6 Hello from prod-preview-rollouts-demo-pr-1-5b9c4cdf76-qlvdd
      5 Hello from rollouts-demo-6f45555c56-t89qm
      5 Hello from rollouts-demo-6f45555c56-bcpsh
      5 Hello from rollouts-demo-6f45555c56-4xc95
      5 Hello from prod-preview-rollouts-demo-pr-2-55bb9c94dc-nnt7j
      5 Hello from prod-preview-rollouts-demo-pr-2-55bb9c94dc-lthwc
      5 Hello from prod-preview-rollouts-demo-pr-1-5b9c4cdf76-w4445
      5 Hello from prod-preview-rollouts-demo-pr-1-5b9c4cdf76-q57d4
      4 Hello from rollouts-demo-6f45555c56-p4pvg
      4 Hello from rollouts-demo-6f45555c56-nxws9
      4 Hello from rollouts-demo-6f45555c56-kvdjg
      4 Hello from rollouts-demo-6f45555c56-bplct
      4 Hello from prod-preview-rollouts-demo-pr-2-55bb9c94dc-9ckxr
      3 Hello from rollouts-demo-6f45555c56-nq2js
      3 Hello from rollouts-demo-6f45555c56-lg59j
```

Aggregating by app instance:

| App instance | Replicas | Requests | Share | Per-pod avg |
|---|---|---|---|---|
| Long-running app (`rollouts-demo`) | 10 | 43 | ~33% | ~4.3 |
| PR env 1 (`prod-preview-...-pr-1`) | 5 | 31 | ~33% | ~6.2 |
| PR env 2 (`prod-preview-...-pr-2`) | 5 | 26 | ~33% | ~5.2 |

> [!NOTE]
> **GAMMA route merging splits traffic equally per route, not per pod.** Envoy assigns each merged route's backends the same weight share. The number of replicas behind a service does not affect the inter-route traffic split — it only affects the intra-route round-robin distribution. Adding more HTTPRoutes to the same parent service will dilute each route's share equally (e.g., 4 routes = 25% each).

## Step 7 - Test with plain Deployments to verify that custom weights are respected

Since Argo Rollouts overrides the HTTPRoute weights to always sum to 100, we can verify that Envoy respects custom relative weights by using plain K8s Deployments instead. Each app instance consists of:
- A `Deployment`
- A single backend `Service`
- A GAMMA producer `HTTPRoute` attached to the shared parent service, with a custom `backendRefs[].weight`

The long-running app HTTPRoute uses **weight: 80**, while each PR env uses **weight: 10**:

```shell
kubectl apply -f deploy-long-running-app.yaml
kubectl apply -f deploy-preview-pr-1.yaml
kubectl apply -f deploy-preview-pr-2.yaml
```

The merged Envoy weighted clusters now reflect the custom weights:

```shell
$ kubectl get ciliumenvoyconfig rollouts-demo-service -o yaml | yq '.spec.resources[] | select(.["@type"] == "type.googleapis.com/envoy.config.route.v3.RouteConfiguration") | .virtualHosts[0].routes[0].route.weightedClusters'
clusters:
  - name: default:rollouts-demo-stable-service:80
    weight: 80
  - name: default:prod-preview-rollouts-demo-pr-1-stable-service:80
    weight: 10
  - name: default:prod-preview-rollouts-demo-pr-2-stable-service:80
    weight: 10
```

Send 100 requests and aggregate by app instance:

```shell
$ kubectl exec -it network-test -- sh
~ # seq 1 100 | xargs -P 10 -I {} bash -c 'curl -s http://rollouts-demo-service' > pods.txt
~ # awk '{print $NF}' pods.txt | sed 's/-[a-z0-9]*-[a-z0-9]*$//' | sort | uniq -c | sort -rn
     75 rollouts-demo
     14 prod-preview-rollouts-demo-pr-1
     11 prod-preview-rollouts-demo-pr-2
```

| App instance | Replicas | Weight | Expected share | Observed |
|---|---|---|---|---|
| Long-running app (`rollouts-demo`) | 10 | 80 | 80% | 75% |
| PR env 1 (`prod-preview-...-pr-1`) | 5 | 10 | 10% | 14% |
| PR env 2 (`prod-preview-...-pr-2`) | 5 | 10 | 10% | 11% |

> [!NOTE]
> **Envoy uses relative weights across merged GAMMA routes.** Setting lower `backendRefs[].weight` values on PR environment HTTPRoutes results in proportionally less traffic. This confirms that a `weightScale` feature in the Argo Rollouts Gateway API plugin could enable the same behavior — by scaling the plugin-managed weights so that PR environments contribute a smaller share to the merged Envoy configuration.

## Step 8 - Hybrid test: Argo Rollout (long-running app) + plain Deployments (PR envs)

In practice, the long-running app would use an Argo `Rollout` (for canary deployments), while the short-lived PR environments could use plain `Deployments` with lower HTTPRoute weights. This creates a hybrid setup:

- **Long-running app**: `Rollout` with HTTPRoute managed by the plugin → weights hardcoded to **stable=100, canary=0**
- **PR env 1**: `Deployment` with HTTPRoute `backendRefs[].weight: 10`
- **PR env 2**: `Deployment` with HTTPRoute `backendRefs[].weight: 10`

The merged Envoy total weight is **120** (100 + 10 + 10), giving an expected split of ~83%/~8%/~8%:

```shell
$ kubectl exec -it network-test -- sh
~ # seq 1 100 | xargs -P 10 -I {} bash -c 'curl -s http://rollouts-demo-service' > pods.txt
~ # awk '{print $NF}' pods.txt | sed 's/-[a-z0-9]*-[a-z0-9]*$//' | sort | uniq -c | sort -rn
     80 rollouts-demo
     14 prod-preview-rollouts-demo-pr-1
      6 prod-preview-rollouts-demo-pr-2
```

| App instance | Type | Weight | Expected share | Observed |
|---|---|---|---|---|
| Long-running app (`rollouts-demo`) | Rollout | 100 | 83.3% | 80% |
| PR env 1 (`prod-preview-...-pr-1`) | Deployment | 10 | 8.3% | 14% |
| PR env 2 (`prod-preview-...-pr-2`) | Deployment | 10 | 8.3% | 6% |

> [!NOTE]
> **Mixing Rollout-managed and Deployment-backed HTTPRoutes works.** The Rollout's plugin-managed weight of 100 and the Deployment HTTPRoutes' weight of 10 are merged by Cilium into a single Envoy weighted cluster with a total weight of 120. The long-running app receives ~83% of traffic, while each PR environment receives ~8%. This hybrid approach can be used today without any plugin changes — the only requirement is that the PR environment HTTPRoutes are not managed by Argo Rollouts.
