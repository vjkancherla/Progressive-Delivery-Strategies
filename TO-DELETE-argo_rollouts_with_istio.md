# Progressive Delivery with Argo Rollouts and Istio
---------

This guide covers how Argo Rollouts integrates with the Istio Service Mesh for traffic shaping whilst performing Canary Deployments 


Source: https://argo-rollouts.readthedocs.io/en/stable/getting-started/istio/


## Prerequisites Installation

### 0. Create a K3D Cluster
```bash 
k3d cluster create mycluster -a 1 --subnet 172.19.0.0/16
```

### 1. Install Istio in SIDECAR MODE
```bash
helm repo add istio https://istio-release.storage.googleapis.com/charts

helm repo update

# The base chart contains the basic CRDs and cluster roles required to set up Istio. This should be installed prior to any other Istio component.
helm search repo istio/base --versions -l | more

helm install istio-base istio/base \
--version 1.26.2 \
-n istio-system \
--create-namespace \
--wait

# istiod is the CONTROL PLANE component that manages and configures the proxies to route traffic within the mesh.
helm search repo istio/istiod --versions -l | more

helm install istiod istio/istiod \
--version 1.26.2 \
--namespace istio-system \
--wait

k get pods -n istio-system
```

### 2. Install Istio Ingress Gateway
```bash

helm search repo istio/gateway --versions -l | more

helm install istio-ingressgateway \
istio/gateway \
--version 1.26.2 \
--namespace istio-ingress \
--create-namespace \
--wait

k get pods -n istio-ingress

```


### 3. Install Argo Rollouts
```bash
helm repo add argo https://argoproj.github.io/argo-helm

helm repo update

helm search repo argo/argo-rollouts --versions -l | more

helm install dev-argo-rollouts \
argo/argo-rollouts \
--version 2.39.6 \
--namespace argo \
--create-namespace \
--set dashboard.enabled=true

k get pods -n argo

[Optional] 
k port-forward -n argo service/dev-argo-rollouts-dashboard 3100:3100
browse http://localhost:3100
```


### 4. Install MetalLB for LoadBalancer support for the ISTIO Gateway
``` bash
# Install MetalLB to create "LoadBalancer" services on K3D. This is required for Kubernetes Gateway
helm repo add metallb https://metallb.github.io/metallb

helm install my-metallb metallb/metallb \
--version 0.15.2 \
--namespace metallb-system \
--create-namespace

k get pods -n metallb-system
```

## 5. Configure MetalLB IP Pool

```bash
cat <<EOF | kubectl apply -f -
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default-pool
  namespace: metallb-system
spec:
  addresses:
  - 172.19.255.200-172.19.255.250
EOF

cat <<EOF | kubectl apply -f -
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: default-advertise
  namespace: metallb-system
spec:
  ipAddressPools:
  - default-pool
EOF
```

### 6 . Install Argo Rollouts Plugin
```bash
curl -LO https://github.com/argoproj/argo-rollouts/releases/latest/download/kubectl-argo-rollouts-darwin-amd64

chmod +x ./kubectl-argo-rollouts-darwin-amd64

sudo mv ./kubectl-argo-rollouts-darwin-amd64 /usr/local/bin/kubectl-argo-rollouts

kubectl argo rollouts version

```


## Deploy the Demo Application

### Create a namespace for the demo

k create ns my-demo


### Enable Istio SideCar injection

k label namespace my-demo istio-injection=enabled


### Deploy the Rollout resource
k apply -n my-demo -f example/rollout.yaml

Note: The Rollout resource (and the Pods) will not be created until all the below objects are created.

The rollout resource is configured to use the Canary Strategy with the following rollout steps:

steps:
    - setWeight: 20 # Sets the ratio of canary ReplicaSet to 20%
    - pause: {} # Pauses indefinitely until manually resumed
    - setWeight: 40
    - pause: {duration: 1m} # Pauses the rollout for 1 minute
    - setWeight: 60
    - pause: {duration: 1m} # Pauses the rollout for 1 minute
    - setWeight: 80
    - pause: {duration: 1m} # Pauses the rollout for 1 minute
    - setWeight: 100


### Deploy stable and canary services
k apply -n my-demo -f example/services.yaml


### Deploy Istio Gateway
k apply -n my-demo -f example/gateway.yaml


### Deploy Istio VirtualService
k apply -n my-demo -f example/virtualsvc.yaml


### Verify Application Pods
k get pods -n my-demo


### Verify external access to the Application, via the Istio-Ingress-Gateway
Internet -> Istio-Ingress-Gateway -> Gateway -> VirtualService -> Service -> Pod

k port-forward -n istio-ingress svc/istio-ingressgateway 8080:80

http://localhost:8080



### Check the inital state of the Rollout
```
k argo rollouts -n my-demo get rollout rollouts-demo

Name:            rollouts-demo
Namespace:       my-demo
Status:          ✔ Healthy
Strategy:        Canary
  Step:          9/9
  SetWeight:     100
  ActualWeight:  100
Images:          argoproj/rollouts-demo:blue (stable)
Replicas:
  Desired:       4
  Current:       4
  Updated:       4
  Ready:         4
  Available:     4

NAME                                      KIND        STATUS     AGE    INFO
⟳ rollouts-demo                           Rollout     ✔ Healthy  5m32s
└──# revision:1
   └──⧉ rollouts-demo-76c569d8b           ReplicaSet  ✔ Healthy  36s    stable
      ├──□ rollouts-demo-76c569d8b-24gls  Pod         ✔ Running  25s    ready:2/2
      ├──□ rollouts-demo-76c569d8b-ck4rq  Pod         ✔ Running  25s    ready:2/2
      ├──□ rollouts-demo-76c569d8b-kxb56  Pod         ✔ Running  25s    ready:2/2
      └──□ rollouts-demo-76c569d8b-wmgnm  Pod         ✔ Running  25s    ready:2/2
```

## Performing a Progressive Update

### Update the image version
k apply -n my-demo -f example/rollout-update.yaml


### Check the state of the Canary Rollout
Argo Rollout does the following:

1. 
Canary-Rollout-Step:
  - setWeight: 20 # Sets the ratio of canary ReplicaSet to 20%
  - pause: {} # Pauses indefinitely until manually resumed

Note: the rollout controller will scale the canary to match the current trafficWeight of the current step. For example, if the current weight is 25%, and there are four replicas, then the canary will be scaled to 1, to match the traffic weight.

Note: the stable ReplicaSet is left scaled to 100% during the update. This has the advantage that if an abort occurs, traffic can be immediately shifted back to the stable ReplicaSet without delay. 

- created a new replica-set: rollouts-demo-5cf4fb69f8
- created one new pod: rollouts-demo-5cf4fb69f8-pc986
- updated rollouts-demo-canary service's selector: 
    Selector: app=rollouts-demo,rollouts-pod-template-hash=5cf4fb69f8
- updated Istio VirtualService - rollouts-demo-vsvc - and set the weights to correspond to the rollout:
    Route:
      Destination:
        Host:  rollouts-demo-stable
      Weight:  80
      Destination:
        Host:  rollouts-demo-canary
      Weight:  20


```
k argo rollouts -n my-demo get rollout rollouts-demo 



Name:            rollouts-demo
Namespace:       my-demo
Status:          ॥ Paused
Message:         CanaryPauseStep
Strategy:        Canary
  Step:          1/9
  SetWeight:     20
  ActualWeight:  20
Images:          argoproj/rollouts-demo:blue (stable)
                 argoproj/rollouts-demo:yellow (canary)
Replicas:
  Desired:       4
  Current:       5
  Updated:       1
  Ready:         5
  Available:     5

NAME                                       KIND        STATUS     AGE    INFO
⟳ rollouts-demo                            Rollout     ॥ Paused   38m
├──# revision:2
│  └──⧉ rollouts-demo-5cf4fb69f8           ReplicaSet  ✔ Healthy  3m13s  canary
│     └──□ rollouts-demo-5cf4fb69f8-5hlv9  Pod         ✔ Running  3m13s  ready:2/2
└──# revision:1
   └──⧉ rollouts-demo-76c569d8b            ReplicaSet  ✔ Healthy  33m    stable
      ├──□ rollouts-demo-76c569d8b-24gls   Pod         ✔ Running  33m    ready:2/2
      ├──□ rollouts-demo-76c569d8b-ck4rq   Pod         ✔ Running  33m    ready:2/2
      ├──□ rollouts-demo-76c569d8b-kxb56   Pod         ✔ Running  33m    ready:2/2
      └──□ rollouts-demo-76c569d8b-wmgnm   Pod         ✔ Running  33m    ready:2/2


k describe -n my-demo svc/rollouts-demo-canary                                            
Name:              rollouts-demo-canary
Namespace:         my-demo
Labels:            <none>
Annotations:       argo-rollouts.argoproj.io/managed-by-rollouts: rollouts-demo
Selector:          app=rollouts-demo,rollouts-pod-template-hash=5cf4fb69f8
Type:              ClusterIP
IP Family Policy:  SingleStack
IP Families:       IPv4
IP:                10.43.8.112
IPs:               10.43.8.112
Port:              http  80/TCP
TargetPort:        http/TCP
Endpoints:         10.42.0.40:8080
Session Affinity:  None
Events:            <none>


k describe -n my-demo virtualservices/rollouts-demo-vsvc                                   
Name:         rollouts-demo-vsvc
Namespace:    my-demo
Labels:       <none>
Annotations:  <none>
API Version:  networking.istio.io/v1
Kind:         VirtualService
Metadata:
  Creation Timestamp:  2025-07-16T13:14:27Z
  Generation:          2
  Managed Fields:
    API Version:  networking.istio.io/v1alpha3
    Fields Type:  FieldsV1
    fieldsV1:
      f:metadata:
        f:annotations:
          .:
          f:kubectl.kubernetes.io/last-applied-configuration:
      f:spec:
        .:
        f:gateways:
        f:hosts:
    Manager:      kubectl-client-side-apply
    Operation:    Update
    Time:         2025-07-16T13:14:27Z
    API Version:  networking.istio.io/v1alpha3
    Fields Type:  FieldsV1
    fieldsV1:
      f:spec:
        f:http:
    Manager:         rollouts-controller
    Operation:       Update
    Time:            2025-07-16T13:45:06Z
  Resource Version:  106554
  UID:               fc66bfde-fc1f-4eed-a76c-9bf5054f18e9
Spec:
  Gateways:
    rollouts-demo-gateway
  Hosts:
    rollouts-demo.local
  Http:
    Name:  primary
    Route:
      Destination:
        Host:  rollouts-demo-stable
      Weight:  80
      Destination:
        Host:  rollouts-demo-canary
      Weight:  20
Events:        <none>
```

At this point the Canary deployment is paused indefinitely due to - pause: {}.

To move forward with the deployment, run

k argo rollouts -n my-demo promote rollouts-demo

Argo Rollouts then:
- created one new pod (2 in total)
- updated Istio VirtualService - rollouts-demo-vsvc - and set the weights to correspond to the rollout:
    Route:
      Destination:
        Host:  rollouts-demo-stable
      Weight:  60
      Destination:
        Host:  rollouts-demo-canary
      Weight:  40

```
k argo rollouts -n my-demo get rollout rollouts-demo                                       
Name:            rollouts-demo
Namespace:       my-demo
Status:          ॥ Paused
Message:         CanaryPauseStep
Strategy:        Canary
  Step:          3/9
  SetWeight:     40
  ActualWeight:  40
Images:          argoproj/rollouts-demo:blue (stable)
                 argoproj/rollouts-demo:yellow (canary)
Replicas:
  Desired:       4
  Current:       6
  Updated:       2
  Ready:         6
  Available:     6

NAME                                       KIND        STATUS     AGE  INFO
⟳ rollouts-demo                            Rollout     ॥ Paused   49m
├──# revision:2
│  └──⧉ rollouts-demo-5cf4fb69f8           ReplicaSet  ✔ Healthy  12m  canary
│     ├──□ rollouts-demo-5cf4fb69f8-pc986  Pod         ✔ Running  12m  ready:2/2
│     └──□ rollouts-demo-5cf4fb69f8-f895m  Pod         ✔ Running  27s  ready:2/2
└──# revision:1
   └──⧉ rollouts-demo-76c569d8b            ReplicaSet  ✔ Healthy  49m  stable
      ├──□ rollouts-demo-76c569d8b-4qkd7   Pod         ✔ Running  49m  ready:2/2
      ├──□ rollouts-demo-76c569d8b-8c4j7   Pod         ✔ Running  49m  ready:2/2
      ├──□ rollouts-demo-76c569d8b-dx4df   Pod         ✔ Running  49m  ready:2/2
      └──□ rollouts-demo-76c569d8b-hfxzs   Pod         ✔ Running  49m  ready:2/2


k describe -n my-demo virtualservices/rollouts-demo-vsvc                                  
Name:         rollouts-demo-vsvc
Namespace:    my-demo
Labels:       <none>
Annotations:  <none>
API Version:  networking.istio.io/v1
Kind:         VirtualService
Metadata:
  Creation Timestamp:  2025-07-17T16:25:34Z
  Generation:          6
  Managed Fields:
    API Version:  networking.istio.io/v1alpha3
    Fields Type:  FieldsV1
    fieldsV1:
      f:metadata:
        f:annotations:
          .:
          f:kubectl.kubernetes.io/last-applied-configuration:
      f:spec:
        .:
        f:gateways:
        f:hosts:
    Manager:      kubectl-client-side-apply
    Operation:    Update
    Time:         2025-07-17T16:58:32Z
    API Version:  networking.istio.io/v1alpha3
    Fields Type:  FieldsV1
    fieldsV1:
      f:spec:
        f:http:
    Manager:         rollouts-controller
    Operation:       Update
    Time:            2025-07-17T17:17:44Z
  Resource Version:  11881
  UID:               645f34e7-262e-49ef-b577-42d648d0c7de
Spec:
  Gateways:
    rollouts-demo-gateway
  Hosts:
    *
  Http:
    Name:  primary
    Route:
      Destination:
        Host:  rollouts-demo-stable
      Weight:  60
      Destination:
        Host:  rollouts-demo-canary
      Weight:  40
```

The above process continues until:

- There are 4 new pods that are serviced by the Canary Serive

- updated Istio VirtualService - rollouts-demo-vsvc - and set the weights to correspond to the rollout:
    Route:
      Destination:
        Host:  rollouts-demo-stable
      Weight:  0
      Destination:
        Host:  rollouts-demo-canary
      Weight:  100
```
k argo rollouts -n my-demo get rollout rollouts-demo                                     
Name:            rollouts-demo
Namespace:       my-demo
Status:          ✔ Healthy
Strategy:        Canary
  Step:          9/9
  SetWeight:     100
  ActualWeight:  100
Images:          argoproj/rollouts-demo:blue
                 argoproj/rollouts-demo:yellow (stable)
Replicas:
  Desired:       4
  Current:       8
  Updated:       4
  Ready:         8
  Available:     8

NAME                                       KIND        STATUS     AGE    INFO
⟳ rollouts-demo                            Rollout     ✔ Healthy  53m
├──# revision:2
│  └──⧉ rollouts-demo-5cf4fb69f8           ReplicaSet  ✔ Healthy  16m    stable
│     ├──□ rollouts-demo-5cf4fb69f8-pc986  Pod         ✔ Running  16m    ready:2/2
│     ├──□ rollouts-demo-5cf4fb69f8-f895m  Pod         ✔ Running  4m15s  ready:2/2
│     ├──□ rollouts-demo-5cf4fb69f8-fp5h8  Pod         ✔ Running  2m51s  ready:2/2
│     └──□ rollouts-demo-5cf4fb69f8-x7v4s  Pod         ✔ Running  93s    ready:2/2
└──# revision:1
   └──⧉ rollouts-demo-76c569d8b            ReplicaSet  ✔ Healthy  53m    delay:7s
      ├──□ rollouts-demo-76c569d8b-4qkd7   Pod         ✔ Running  53m    ready:2/2
      ├──□ rollouts-demo-76c569d8b-8c4j7   Pod         ✔ Running  53m    ready:2/2
      ├──□ rollouts-demo-76c569d8b-dx4df   Pod         ✔ Running  53m    ready:2/2
      └──□ rollouts-demo-76c569d8b-hfxzs   Pod         ✔ Running  53m    ready:2/2

k describe -n my-demo virtualservices/rollouts-demo-vsvc                                  
Name:         rollouts-demo-vsvc
Namespace:    my-demo
Labels:       <none>
Annotations:  <none>
API Version:  networking.istio.io/v1
Kind:         VirtualService
Metadata:
  Creation Timestamp:  2025-07-17T16:25:34Z
  Generation:          8
  Managed Fields:
    API Version:  networking.istio.io/v1alpha3
    Fields Type:  FieldsV1
    fieldsV1:
      f:metadata:
        f:annotations:
          .:
          f:kubectl.kubernetes.io/last-applied-configuration:
      f:spec:
        .:
        f:gateways:
        f:hosts:
    Manager:      kubectl-client-side-apply
    Operation:    Update
    Time:         2025-07-17T16:58:32Z
    API Version:  networking.istio.io/v1alpha3
    Fields Type:  FieldsV1
    fieldsV1:
      f:spec:
        f:http:
    Manager:         rollouts-controller
    Operation:       Update
    Time:            2025-07-17T17:18:45Z
  Resource Version:  11989
  UID:               645f34e7-262e-49ef-b577-42d648d0c7de
Spec:
  Gateways:
    rollouts-demo-gateway
  Hosts:
    *
  Http:
    Name:  primary
    Route:
      Destination:
        Host:  rollouts-demo-stable
      Weight:  100
      Destination:
        Host:  rollouts-demo-canary
      Weight:  0
Events:        <none>
```

Once the depoyment in complete, Argo Rollouts will:

- Scale-Down the previous Stable ReplicaSet to 0
- Marks the previous Canary ReplicaSet as Stable
- Updates Stable Service - rollouts-demo-stable - to use the pods from the new replica set
- Now, both Services, rollouts-demo-stable and rollouts-demo-canary are sending traffic to the same, new, pods

```
k argo rollouts -n my-demo get rollout rollouts-demo                                      
Name:            rollouts-demo
Namespace:       my-demo
Status:          ✔ Healthy
Strategy:        Canary
  Step:          9/9
  SetWeight:     100
  ActualWeight:  100
Images:          argoproj/rollouts-demo:yellow (stable)
Replicas:
  Desired:       4
  Current:       4
  Updated:       4
  Ready:         4
  Available:     4

NAME                                       KIND        STATUS        AGE    INFO
⟳ rollouts-demo                            Rollout     ✔ Healthy     53m
├──# revision:2
│  └──⧉ rollouts-demo-5cf4fb69f8           ReplicaSet  ✔ Healthy     16m    stable
│     ├──□ rollouts-demo-5cf4fb69f8-pc986  Pod         ✔ Running     16m    ready:2/2
│     ├──□ rollouts-demo-5cf4fb69f8-f895m  Pod         ✔ Running     4m44s  ready:2/2
│     ├──□ rollouts-demo-5cf4fb69f8-fp5h8  Pod         ✔ Running     3m20s  ready:2/2
│     └──□ rollouts-demo-5cf4fb69f8-x7v4s  Pod         ✔ Running     2m2s   ready:2/2
└──# revision:1
   └──⧉ rollouts-demo-76c569d8b            ReplicaSet  • ScaledDown  53m
```