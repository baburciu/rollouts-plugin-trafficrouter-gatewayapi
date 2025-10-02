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

## Step 3 - Create the services required for traffic split

Create three Services required for canary based rollout strategy

```shell
kubectl apply -f service.yaml
```

## Step 5 - Create HTTPRoute that defines a traffic split between two services

> [!IMPORTANT]
> For Cilium the K8s Services refs need to use `group: ""`. This is different than Linkerd, where `group: "core"` could be used.
  ```yaml
  apiVersion: gateway.networking.k8s.io/v1beta1
  kind: HTTPRoute
  spec:
    hostnames:
    - dummy  # does not count in GAMMA GatewayAPI yet, see https://github.com/kubernetes-sigs/gateway-api/issues/2885
    parentRefs:
      - group: ""
        name:
        kind: Service
        port:
    rules:
      - backendRefs:
          - group: ""
            name:
            kind: Service
            port:
  ```

Create a GAMMA [producer `HTTPRoute`](https://gateway-api.sigs.k8s.io/concepts/glossary/#producer-route) resource and connect it to a parent K8s service (using a canary and stable K8s services as backends)

```shell
kubectl apply -f httproute.yaml
```

## Step 6 - Create 2 Deployments for canary and stable services to target

Deploy a rollout to get the initial version

```shell
kubectl apply -f deployment
```

## Step 7 - Observe Cilium managing traffic split for GAMMA HTTPRoute

```shell
$ kubectl run -it --image nicolaka/netshoot:v0.13 network-test -- sh
If you don't see a command prompt, try pressing enter.
~ #
~ # seq 1 100 | xargs -P 10 -I {} bash -c 'curl -s http://parent-service' > pods.txt
~ # sort pods.txt | uniq -c | sort -rn
     50 Hello from stable-service-55b9db664b-lgk46
     50 Hello from stable-service-55b9db664b-k9ztk
~ #
~ #
~ # seq 1 100 | xargs -P 10 -I {} bash -c 'curl -s -H "X-Canary: test" http://parent-service' > pods.txt
~ # sort pods.txt | uniq -c | sort -rn
     50 Hello from canary-service-5bd55f8f7f-rkq5z
     50 Hello from canary-service-5bd55f8f7f-46vc4
~ #
```
```shell
$ kubectl get gamma-http-route -o yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"gateway.networking.k8s.io/v1beta1","kind":"HTTPRoute","metadata":{"annotations":{},"name":"gamma-http-route","namespace":"default"},"spec":{"hostnames":["dummy"],"parentRefs":[{"group":"","kind":"Service","name":"parent-service","port":80}],"rules":[{"backendRefs":[{"group":"","kind":"Service","name":"stable-service","port":80,"weight":100},{"group":"","kind":"Service","name":"canary-service","port":80,"weight":0}],"matches":[{"path":{"type":"PathPrefix","value":"/"}}]},{"backendRefs":[{"group":"","kind":"Service","name":"canary-service","port":80,"weight":0}],"matches":[{"headers":[{"name":"X-Canary","type":"Exact","value":"test"}],"path":{"type":"PathPrefix","value":"/"}}]}]}}
  creationTimestamp: "2025-10-03T14:10:05Z"
  generation: 4
  name: gamma-http-route
  namespace: default
  resourceVersion: "6929"
  uid: ed78c53e-2a59-456b-9bc7-2403d4599c1d
spec:
  hostnames:
  - dummy
  parentRefs:
  - group: ""
    kind: Service
    name: parent-service
    port: 80
  rules:
  - backendRefs:
    - group: ""
      kind: Service
      name: stable-service
      port: 80
      weight: 100
    - group: ""
      kind: Service
      name: canary-service
      port: 80
      weight: 0
    matches:
    - path:
        type: PathPrefix
        value: /
  - backendRefs:
    - group: ""
      kind: Service
      name: canary-service
      port: 80
      weight: 0
    matches:
    - headers:
      - name: X-Canary
        type: Exact
        value: test
      path:
        type: PathPrefix
        value: /
status:
  parents:
  - conditions:
    - lastTransitionTime: "2025-10-03T14:52:13Z"
      message: Accepted HTTPRoute
      observedGeneration: 4
      reason: Accepted
      status: "True"
      type: Accepted
    - lastTransitionTime: "2025-10-03T14:52:13Z"
      message: Service reference is valid
      observedGeneration: 4
      reason: ResolvedRefs
      status: "True"
      type: ResolvedRefs
    controllerName: io.cilium/gateway-controller
    parentRef:
      group: ""
      kind: Service
      name: parent-service
      port: 80
```
