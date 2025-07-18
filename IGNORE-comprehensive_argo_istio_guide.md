# Comprehensive Argo Rollouts + Istio Guide

## Prerequisites Installation

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

# The Kubernetes Gateway API CRDs do not come installed by default on most Kubernetes clusters, so make sure they are installed before using the Gateway API.
kubectl get crd gateways.gateway.networking.k8s.io 

kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.0.0/standard-install.yaml


```

### 2. Install Argo Rollouts
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

>> k port-forward -n argo service/dev-argo-rollouts-dashboard 3100:3100

Install Argo Rollouts Plugin
>> brew install argoproj/tap/kubectl-argo-rollouts

```

---

## Core Application Configuration

### 1. Application Rollout

```yaml
# rollout.yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: demo-app
spec:
  replicas: 5
  strategy:
    canary:
      # Istio traffic splitting configuration
      trafficRouting:
        istio:
          virtualService: 
            name: demo-app-vs
            routes:
            - primary
          destinationRule:
            name: demo-app-dr
            canarySubsetName: canary
            stableSubsetName: stable
      
      # Analysis configuration
      analysis:
        templates:
        - templateName: success-rate
        - templateName: latency
        startingStep: 2                 # Start analysis after 25% traffic
        args:
        - name: service-name
          value: demo-app-service
        - name: canary-hash
          valueFrom:
            podTemplateHashValue: Latest
        - name: stable-hash
          valueFrom:
            podTemplateHashValue: Stable
      
      # Progressive rollout steps with analysis
      steps:
      - setWeight: 10                   # 10% canary traffic
      - pause: {duration: 2m}
      
      - setWeight: 25                   # 25% canary traffic  
      - pause: {duration: 1m}
      - analysis:
          templates:
          - templateName: success-rate
          args:
          - name: service-name
            value: demo-app-service
      
      - setWeight: 50                   # 50% canary traffic
      - pause: {duration: 2m}
      - analysis:
          templates:
          - templateName: success-rate
          - templateName: latency
          args:
          - name: service-name
            value: demo-app-service
      
      - setWeight: 75                   # 75% canary traffic
      - pause: {duration: 1m}
      - analysis:
          templates:
          - templateName: success-rate
          - templateName: latency
          args:
          - name: service-name
            value: demo-app-service
      
      # 100% happens automatically after final analysis passes
      
  selector:
    matchLabels:
      app: demo-app
  template:
    metadata:
      labels:
        app: demo-app
    spec:
      containers:
      - name: demo-app
        image: nginx:1.20
        ports:
        - containerPort: 80
        env:
        - name: VERSION
          value: "v1.0"
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "128Mi"
            cpu: "100m"
```

### 2. Kubernetes Service

```yaml
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: demo-app-service
spec:
  selector:
    app: demo-app
  ports:
  - port: 80
    targetPort: 80
    name: http
  type: ClusterIP
```

### 3. Istio DestinationRule

```yaml
# destination-rule.yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: demo-app-dr
spec:
  host: demo-app-service
  trafficPolicy:
    loadBalancer:
      simple: LEAST_CONN
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http1MaxPendingRequests: 50
        maxRequestsPerConnection: 10
    outlierDetection:
      consecutiveErrors: 3
      interval: 30s
      baseEjectionTime: 30s
  subsets:
  - name: stable
    labels:
      app: demo-app
  - name: canary
    labels:
      app: demo-app
```

### 4. Istio VirtualService with Header Routing

```yaml
# virtual-service.yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: demo-app-vs
spec:
  hosts:
  - demo-app-service
  http:
  # Beta users always get canary version
  - match:
    - headers:
        x-user-type:
          exact: "beta"
    route:
    - destination:
        host: demo-app-service
        subset: canary
      weight: 100
    timeout: 10s
    headers:
      response:
        add:
          x-canary-served: "beta-user"
  
  # Internal services get canary during rollout
  - match:
    - headers:
        x-service-type:
          exact: "internal"
    route:
    - destination:
        host: demo-app-service
        subset: canary
      weight: 100
    timeout: 10s
    headers:
      response:
        add:
          x-canary-served: "internal-service"
  
  # Specific canary users (for testing)
  - match:
    - headers:
        x-canary-user:
          exact: "true"
    route:
    - destination:
        host: demo-app-service
        subset: canary
      weight: 100
    timeout: 10s
    headers:
      response:
        add:
          x-canary-served: "canary-user"
  
  # Primary route - managed by Argo Rollouts for traffic splitting
  - name: primary
    route:
    - destination:
        host: demo-app-service
        subset: stable
      weight: 100                       # Argo Rollouts modifies this
    - destination:
        host: demo-app-service
        subset: canary  
      weight: 0                         # Argo Rollouts modifies this
    timeout: 15s
    retries:
      attempts: 3
      perTryTimeout: 5s
      retryOn: gateway-error,connect-failure,refused-stream
```

---

## Analysis Templates for Automated Safety

### 1. Success Rate Analysis

```yaml
# analysis-success-rate.yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
spec:
  args:
  - name: service-name
  - name: canary-hash
    valueFrom:
      podTemplateHashValue: Latest
  metrics:
  - name: success-rate
    interval: 30s
    successCondition: result[0] >= 0.95   # 95% success rate required
    failureLimit: 2                       # Allow 2 failures before rollback
    count: 5                              # Run 5 checks
    provider:
      prometheus:
        address: http://prometheus.istio-system:9090
        query: |
          sum(
            rate(
              istio_requests_total{
                destination_service_name="{{args.service-name}}",
                destination_service_namespace="default",
                response_code!~"5.*"
              }[2m]
            )
          ) / 
          sum(
            rate(
              istio_requests_total{
                destination_service_name="{{args.service-name}}",
                destination_service_namespace="default"
              }[2m]
            )
          )
```

### 2. Latency Analysis

```yaml
# analysis-latency.yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: latency
spec:
  args:
  - name: service-name
  metrics:
  - name: latency-p95
    interval: 30s
    successCondition: result[0] <= 500    # P95 must be under 500ms
    failureLimit: 2
    count: 5
    provider:
      prometheus:
        address: http://prometheus.istio-system:9090
        query: |
          histogram_quantile(0.95,
            sum(
              rate(
                istio_request_duration_milliseconds_bucket{
                  destination_service_name="{{args.service-name}}",
                  destination_service_namespace="default"
                }[2m]
              )
            ) by (le)
          )
  - name: latency-p99
    interval: 30s
    successCondition: result[0] <= 1000   # P99 must be under 1000ms
    failureLimit: 2
    count: 5
    provider:
      prometheus:
        address: http://prometheus.istio-system:9090
        query: |
          histogram_quantile(0.99,
            sum(
              rate(
                istio_request_duration_milliseconds_bucket{
                  destination_service_name="{{args.service-name}}",
                  destination_service_namespace="default"
                }[2m]
              )
            ) by (le)
          )
```

### 3. Error Rate Analysis

```yaml
# analysis-error-rate.yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: error-rate
spec:
  args:
  - name: service-name
  metrics:
  - name: error-rate
    interval: 60s
    successCondition: result[0] <= 0.05   # Error rate under 5%
    failureLimit: 1                       # Fail fast on errors
    count: 3
    provider:
      prometheus:
        address: http://prometheus.istio-system:9090
        query: |
          sum(
            rate(
              istio_requests_total{
                destination_service_name="{{args.service-name}}",
                destination_service_namespace="default",
                response_code=~"5.*"
              }[2m]
            )
          ) / 
          sum(
            rate(
              istio_requests_total{
                destination_service_name="{{args.service-name}}",
                destination_service_namespace="default"
              }[2m]
            )
          )
```

---

## External Access Configuration

### 1. Istio Gateway

```yaml
# gateway.yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: demo-app-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - demo-app.local
    - beta.demo-app.local
  - port:
      number: 443
      name: https
      protocol: HTTPS
    tls:
      mode: SIMPLE
      credentialName: demo-app-tls
    hosts:
    - demo-app.local
    - beta.demo-app.local
```

### 2. External VirtualService

```yaml
# external-virtual-service.yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: demo-app-external-vs
spec:
  hosts:
  - demo-app.local
  - beta.demo-app.local
  gateways:
  - demo-app-gateway
  http:
  # Beta subdomain always gets canary
  - match:
    - uri:
        prefix: /
      headers:
        host:
          exact: beta.demo-app.local
    route:
    - destination:
        host: demo-app-service
        subset: canary
      weight: 100
    headers:
      request:
        add:
          x-user-type: "beta"
      response:
        add:
          x-served-by: "beta-gateway"
  
  # Regular external traffic with controlled canary percentage
  - match:
    - uri:
        prefix: /
      headers:
        host:
          exact: demo-app.local
    route:
    - destination:
        host: demo-app-service
        subset: stable
      weight: 95                          # 95% stable for external users
    - destination:
        host: demo-app-service
        subset: canary
      weight: 5                           # 5% canary for external users
    headers:
      response:
        add:
          x-served-by: "main-gateway"
```

---

## Consumer Applications

### 1. Regular Consumer (No Headers)

```yaml
# consumer-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: consumer-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: consumer-app
  template:
    metadata:
      labels:
        app: consumer-app
      annotations:
        sidecar.istio.io/inject: "true"
    spec:
      containers:
      - name: consumer
        image: curlimages/curl:latest
        command: ["/bin/sh"]
        args:
        - -c
        - |
          while true; do
            echo "=== Regular Request (follows traffic split) ==="
            curl -s -w "Status: %{http_code}, Time: %{time_total}s\n" \
              http://demo-app-service/ || echo "Request failed"
            
            echo "Sleeping for 5 seconds..."
            sleep 5
          done
        resources:
          requests:
            memory: "32Mi"
            cpu: "10m"
          limits:
            memory: "64Mi"
            cpu: "50m"
```

### 2. Internal Service Consumer

```yaml
# internal-service.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: internal-service
spec:
  replicas: 1
  selector:
    matchLabels:
      app: internal-service
  template:
    metadata:
      labels:
        app: internal-service
      annotations:
        sidecar.istio.io/inject: "true"
    spec:
      containers:
      - name: internal-service
        image: curlimages/curl:latest
        command: ["/bin/sh"]
        args:
        - -c
        - |
          while true; do
            echo "=== Internal Service Request (always canary) ==="
            curl -s -w "Status: %{http_code}, Time: %{time_total}s\n" \
              -H "x-service-type: internal" \
              http://demo-app-service/ || echo "Request failed"
            
            echo "Sleeping for 8 seconds..."
            sleep 8
          done
```

### 3. Beta User Simulator

```yaml
# beta-user.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: beta-user
spec:
  replicas: 1
  selector:
    matchLabels:
      app: beta-user
  template:
    metadata:
      labels:
        app: beta-user
      annotations:
        sidecar.istio.io/inject: "true"
    spec:
      containers:
      - name: beta-user
        image: curlimages/curl:latest
        command: ["/bin/sh"]
        args:
        - -c
        - |
          while true; do
            echo "=== Beta User Request (always canary) ==="
            curl -s -w "Status: %{http_code}, Time: %{time_total}s\n" \
              -H "x-user-type: beta" \
              http://demo-app-service/ || echo "Request failed"
            
            echo "=== Manual Canary Test ==="
            curl -s -w "Status: %{http_code}, Time: %{time_total}s\n" \
              -H "x-canary-user: true" \
              http://demo-app-service/ || echo "Request failed"
            
            echo "Sleeping for 10 seconds..."
            sleep 10
          done
```

---

## Complete Deployment Guide

### Deploy All Components
```bash
# Deploy analysis templates first
kubectl apply -f analysis-success-rate.yaml
kubectl apply -f analysis-latency.yaml
kubectl apply -f analysis-error-rate.yaml

# Deploy core application components
kubectl apply -f service.yaml
kubectl apply -f destination-rule.yaml  
kubectl apply -f virtual-service.yaml
kubectl apply -f rollout.yaml

# Deploy consumer applications
kubectl apply -f consumer-app.yaml
kubectl apply -f internal-service.yaml
kubectl apply -f beta-user.yaml

# Optional: Deploy external access
kubectl apply -f gateway.yaml
kubectl apply -f external-virtual-service.yaml

# Verify deployment
kubectl get pods
kubectl get rollout
kubectl get analysistemplates
kubectl get virtualservices
kubectl get destinationrules
```

### Trigger Canary Deployment
```bash
# Update to new version (triggers canary deployment)
kubectl argo rollouts set image demo-app demo-app=nginx:1.21

# Watch rollout progress with analysis
kubectl argo rollouts get rollout demo-app --watch

# Check traffic distribution in real-time
watch 'kubectl get virtualservice demo-app-vs -o yaml | grep -A 10 weight'
```

### Monitor Throughout Deployment
```bash
# Monitor analysis runs
kubectl get analysisruns
kubectl describe analysisrun <analysis-run-name>

# Check pod distribution across versions
kubectl get pods -l app=demo-app --show-labels

# Watch consumer application logs
kubectl logs -f deployment/consumer-app
kubectl logs -f deployment/internal-service  
kubectl logs -f deployment/beta-user

# Check Istio proxy configuration
istioctl proxy-config routes deployment/demo-app
```

---

## Deployment Lifecycle with Header Routing

### Initial State: 100% Stable
```
Pods: [v1.0] [v1.0] [v1.0] [v1.0] [v1.0]

Traffic Distribution:
├─ Regular requests: 100% → stable (v1.0)
├─ Beta users (x-user-type: beta): 100% → stable (v1.0) *
├─ Internal services (x-service-type: internal): 100% → stable (v1.0) *
└─ Canary testers (x-canary-user: true): 100% → stable (v1.0) *

* Headers route to "canary" subset, but no canary pods exist yet
```

### Step 1: 10% Canary Deployed
```
Pods: [v1.0] [v1.0] [v1.0] [v1.0] [v1.21]

Traffic Distribution:
├─ Regular requests: 90% → stable (v1.0), 10% → canary (v1.21)
├─ Beta users: 100% → canary (v1.21) ← Now gets new version!
├─ Internal services: 100% → canary (v1.21) ← Testing new version first
└─ Canary testers: 100% → canary (v1.21) ← Explicit canary access

Analysis Running:
✓ No analysis yet (starts at step 2)
```

### Step 2: 25% Canary + Analysis
```
Pods: [v1.0] [v1.0] [v1.0] [v1.21] [v1.21]

Traffic Distribution:
├─ Regular requests: 75% → stable (v1.0), 25% → canary (v1.21)
├─ Beta users: 100% → canary (load balanced across 2 v1.21 pods)
├─ Internal services: 100% → canary (load balanced across 2 v1.21 pods)
└─ Canary testers: 100% → canary (load balanced across 2 v1.21 pods)

Analysis Running:
✓ Success rate: 96% (>= 95% required) ← Passes, continues
```

### Step 3: 50% Canary + Analysis
```
Pods: [v1.0] [v1.0] [v1.21] [v1.21] [v1.21]

Traffic Distribution:
├─ Regular requests: 50% → stable (v1.0), 50% → canary (v1.21)
├─ Beta users: 100% → canary (load balanced across 3 v1.21 pods)
├─ Internal services: 100% → canary (load balanced across 3 v1.21 pods)
└─ Canary testers: 100% → canary (load balanced across 3 v1.21 pods)

Analysis Running:
✓ Success rate: 97% (>= 95% required)
✓ P95 latency: 420ms (<= 500ms required)
✓ P99 latency: 980ms (<= 1000ms required)
✓ All metrics pass, continues to next step
```

### Step 4: 75% Canary + Analysis
```
Pods: [v1.0] [v1.21] [v1.21] [v1.21] [v1.21]

Traffic Distribution:
├─ Regular requests: 25% → stable (v1.0), 75% → canary (v1.21)
├─ Beta users: 100% → canary (load balanced across 4 v1.21 pods)
├─ Internal services: 100% → canary (load balanced across 4 v1.21 pods)
└─ Canary testers: 100% → canary (load balanced across 4 v1.21 pods)

Analysis Running:
✓ Success rate: 95.5% (>= 95% required)
✓ P95 latency: 450ms (<= 500ms required)
✓ P99 latency: 900ms (<= 1000ms required)
✓ Final analysis passes, promoting to 100%
```

### Final: 100% Promotion
```
Pods: [v1.21] [v1.21] [v1.21] [v1.21] [v1.21]

Traffic Distribution:
├─ Regular requests: 100% → stable (now v1.21)
├─ Beta users: 100% → canary (now v1.21) ← Same pods as stable
├─ Internal services: 100% → canary (now v1.21) ← Same pods as stable  
└─ Canary testers: 100% → canary (now v1.21) ← Same pods as stable

Result: All traffic goes to v1.21, headers become redundant until next deployment
```

---

## Manual Control Commands

### Rollout Control
```bash
# Promote to next step
kubectl argo rollouts promote demo-app

# Promote immediately to 100%
kubectl argo rollouts promote demo-app --full

# Abort and rollback to stable
kubectl argo rollouts abort demo-app

# Restart the rollout
kubectl argo rollouts restart demo-app

# Check rollout history
kubectl argo rollouts history demo-app

# Rollback to previous version
kubectl argo rollouts undo demo-app
```

### Analysis and Monitoring
```bash
# Check analysis results
kubectl get analysisruns
kubectl describe analysisrun <analysis-run-name>

# Monitor real-time metrics
kubectl argo rollouts get rollout demo-app --watch

# Check Prometheus connectivity
kubectl exec -it deployment/demo-app -- curl http://prometheus.istio-system:9090/api/v1/query?query=up

# Verify Istio metrics
kubectl exec -it deployment/demo-app -- curl "http://prometheus.istio-system:9090/api/v1/query?query=istio_requests_total"
```

### Traffic Analysis
```bash
# Test traffic distribution
kubectl exec -it deployment/consumer-app -- sh -c '
for i in {1..100}; do
  curl -s http://demo-app-service/ 2>/dev/null | grep -o "Server.*" || echo "no-server-header"
  sleep 0.1
done | sort | uniq -c
'

# Test header routing
kubectl exec -it deployment/consumer-app -- curl -H "x-user-type: beta" http://demo-app-service/

# Check Istio proxy configuration
istioctl proxy-config routes deployment/demo-app
istioctl proxy-config cluster deployment/demo-app --fqdn demo-app-service.default.svc.cluster.local
```

---

## Key Benefits

### 1. **Zero-Impact Deployments**
- Consumer applications require no changes
- Progressive traffic shifting with safety analysis
- Automatic rollback on performance degradation

### 2. **Advanced Traffic Control**
- Precise percentage-based splitting (not pod-ratio based)
- Header-based routing for specific user types
- Different canary percentages for internal vs external traffic

### 3. **Production Safety**
- Automated health monitoring with Prometheus metrics
- Multi-step analysis (success rate, latency, error rate)
- Circuit breaking and retry policies

### 4. **Flexible Routing Options**
- Beta users always get latest features first
- Internal services test canary versions before general rollout
- Manual canary testing capabilities
- External traffic gets controlled exposure

This comprehensive setup provides enterprise-grade canary deployments with maximum safety, flexibility, and minimal consumer impact!