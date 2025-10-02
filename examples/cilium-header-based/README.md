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
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml
kubectl apply -k https://github.com/argoproj/argo-rollouts/manifests/crds\?ref\=stable
kubectl apply -f argo-rollouts-plugin.yaml
kubectl rollout restart deploy -n argo-rollouts
```

## Step 4 - Grant Argo Rollouts SA access to the HTTPRoute

```shell
kubectl apply -f cluster-role.yaml
```
__Note:__ These permission are very permissive. You should lock them down according to your needs.

With the following role we allow Argo Rollouts to have Admin access to HTTPRoutes.

```shell
kubectl apply -f cluster-role-binding.yaml
```

## Step 5 - Create HTTPRoute that defines a traffic split between two services

Create a GAMMA [producer `HTTPRoute`](https://gateway-api.sigs.k8s.io/concepts/glossary/#producer-route) resource and connect it to a parent K8s service (using a canary and stable K8s services as backends)

```shell
kubectl apply -f httproute.yaml
```

## Step 6 - Create the services required for traffic split

Create three Services required for canary based rollout strategy

```shell
kubectl apply -f service.yaml
```

## Step 8 - Create an example Rollout

Deploy a rollout to get the initial version

```shell
kubectl apply -f rollout.yaml
```

## Step 9 - Watch the rollout

Monitor the HTTPRoute configuration to see how traffic is split and header-based routing is configured

```shell
watch "kubectl get httproute.gateway.networking.k8s.io/argo-rollouts-http-route -o jsonpath='{\" HEADERS: \"}{.spec.rules[*].matches[*]}'"
```

## Step 10 - Patch the rollout to see the canary deployment
```shell
kubectl patch rollout rollouts-demo --type='json' -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/env/0/value", "value": "1.1.0"}]'
```

## Step 11 - Observe the rollout and HTTPRoute rule addition of [canary header matching rule](https://gateway-api.sigs.k8s.io/guides/traffic-splitting/#canary-traffic-rollout)

```shell
$ kubectl argo rollouts promote rollouts-demo  # promote to Rollout step 1
$ kubectl argo rollouts get rollout rollouts-demo
Name:            rollouts-demo
Namespace:       default
Status:          ॥ Paused
Message:         CanaryPauseStep
Strategy:        Canary
  Step:          3/5
  SetWeight:     0
  ActualWeight:  0
Images:          hashicorp/http-echo:1.0 (canary, stable)
Replicas:
  Desired:       5
  Current:       6
  Updated:       1
  Ready:         6
  Available:     6

NAME                                       KIND        STATUS     AGE    INFO
⟳ rollouts-demo                            Rollout     ॥ Paused   9m9s
├──# revision:2
│  └──⧉ rollouts-demo-7bd564d79f           ReplicaSet  ✔ Healthy  8m46s  canary
│     └──□ rollouts-demo-7bd564d79f-h546k  Pod         ✔ Running  8m25s  ready:1/1
└──# revision:1
   └──⧉ rollouts-demo-784858d6db           ReplicaSet  ✔ Healthy  9m9s   stable
      ├──□ rollouts-demo-784858d6db-2bvqf  Pod         ✔ Running  9m9s   ready:1/1
      ├──□ rollouts-demo-784858d6db-48qsb  Pod         ✔ Running  9m9s   ready:1/1
      ├──□ rollouts-demo-784858d6db-fmsrn  Pod         ✔ Running  9m9s   ready:1/1
      ├──□ rollouts-demo-784858d6db-svp5m  Pod         ✔ Running  9m9s   ready:1/1
      └──□ rollouts-demo-784858d6db-wv7jr  Pod         ✔ Running  9m9s   ready:1/1
$
$ kubectl get httproute argo-rollouts-http-route -o yaml | yq .spec.rules
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
```
```shell
$ kubectl run -it --image nicolaka/netshoot:v0.13 network-test -- sh  # run a pod to source curl tests
~ # # stable K8s service targets any of the 5 stable pods
~ # curl http://argo-rollouts-stable-service/
Hello from rollouts-demo-5d78c448f9-l757w
~ #
~ # # canary K8s service targets the one canary pod, created for the `setCanaryScale` step in the Rollout
~ # curl http://argo-rollouts-canary-service/
Hello from rollouts-demo-8598f766fd-2cvt4
~ #
~ # # GAMMA-type HTTPRoute's `.spec.parentRefs` K8s service should only target stable pods since no `setWeight` step is used in the Rollout, but it's not the case in our testing
~ # seq 1 100 | xargs -P 10 -I {} bash -c 'curl -s http://argo-rollouts-service' > pods.txt
~ # sort pods.txt | uniq -c | sort -rn
     23 Hello from rollouts-demo-7bd564d79f-h546k  # <== canary
     18 Hello from rollouts-demo-784858d6db-wv7jr
     18 Hello from rollouts-demo-784858d6db-48qsb
     17 Hello from rollouts-demo-784858d6db-svp5m
     12 Hello from rollouts-demo-784858d6db-fmsrn
     12 Hello from rollouts-demo-784858d6db-2bvqf
~ #
```

## Step 12 - Test the header-based routing with curl (not working with Cilium v1.8.2 by the look of it)

You can test the header-based routing by sending requests with the specified header.
With header it should always go to canary only, but we still see stable pods:

```shell
$ kubectl exec -it network-test -- sh
~ #
~ # seq 1 100 | xargs -P 10 -I {} bash -c 'curl -s -H "X-Test: test" http://argo-rollouts-service' > pods.txt
~ # sort pods.txt | uniq -c | sort -rn
     21 Hello from rollouts-demo-784858d6db-2bvqf
     19 Hello from rollouts-demo-7bd564d79f-h546k
     18 Hello from rollouts-demo-784858d6db-fmsrn
     16 Hello from rollouts-demo-784858d6db-svp5m
     15 Hello from rollouts-demo-784858d6db-48qsb
     11 Hello from rollouts-demo-784858d6db-wv7jr
~ #
~ # seq 1 100 | xargs -P 10 -I {} bash -c 'curl -s -H "X-Test: test" http://argo-rollouts-service' > pods.txt
~ # sort pods.txt | uniq -c | sort -rn
     19 Hello from rollouts-demo-7bd564d79f-h546k
     19 Hello from rollouts-demo-784858d6db-wv7jr
     19 Hello from rollouts-demo-784858d6db-fmsrn
     15 Hello from rollouts-demo-784858d6db-2bvqf
     14 Hello from rollouts-demo-784858d6db-svp5m
     14 Hello from rollouts-demo-784858d6db-48qsb
~ #
```
