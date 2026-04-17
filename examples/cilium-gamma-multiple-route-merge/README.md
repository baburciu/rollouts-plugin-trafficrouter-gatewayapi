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

This is expected behavior: the Argo Rollouts controller reconciles the HTTPRoute and overrides the weights based on the current rollout step. On initial deploy (revision 1), all traffic goes to the stable service.

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
