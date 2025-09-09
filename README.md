# Istio Service Mesh Cheat Sheet

## Installation
```bash
# Download Istio
curl -L https://istio.io/downloadIstio | sh -
cd istio-*
export PATH=$PWD/bin:$PATH

# Install Istio on Kubernetes
istioctl install --set values.defaultRevision=default

# Enable sidecar injection
kubectl label namespace default istio-injection=enabled

# Verify installation
istioctl verify-install
kubectl get pods -n istio-system
```

## Traffic Management

### Virtual Services
```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: reviews
spec:
  http:
  - match:
    - headers:
        end-user:
          exact: jason
    route:
    - destination:
        host: reviews
        subset: v2
  - route:
    - destination:
        host: reviews
        subset: v1
      weight: 75
    - destination:
        host: reviews
        subset: v2
      weight: 25

# Fault injection
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: ratings
spec:
  http:
  - fault:
      delay:
        percentage:
          value: 0.1
        fixedDelay: 5s
      abort:
        percentage:
          value: 0.1
        httpStatus: 400
    route:
    - destination:
        host: ratings
```

### Destination Rules
```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: reviews
spec:
  host: reviews
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 10
      http:
        http1MaxPendingRequests: 10
        maxRequestsPerConnection: 2
    circuitBreaker:
      consecutiveErrors: 3
      interval: 30s
      baseEjectionTime: 30s
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
    trafficPolicy:
      portLevelSettings:
      - port:
          number: 80
        connectionPool:
          tcp:
            maxConnections: 5
```

### Gateways
```yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: bookinfo-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - bookinfo.example.com
  - port:
      number: 443
      name: https
      protocol: HTTPS
    tls:
      mode: SIMPLE
      credentialName: bookinfo-secret
    hosts:
    - bookinfo.example.com

---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: bookinfo
spec:
  hosts:
  - bookinfo.example.com
  gateways:
  - bookinfo-gateway
  http:
  - route:
    - destination:
        host: productpage
        port:
          number: 9080
```

### Service Entries
```yaml
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: external-svc-https
spec:
  hosts:
  - api.external.com
  ports:
  - number: 443
    name: https
    protocol: HTTPS
  location: MESH_EXTERNAL
  resolution: DNS

---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: external-svc
spec:
  hosts:
  - api.external.com
  http:
  - timeout: 10s
    route:
    - destination:
        host: api.external.com
```

## Security

### PeerAuthentication
```yaml
# Cluster-wide mTLS
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-system
spec:
  mtls:
    mode: STRICT

# Namespace-level mTLS
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT

# Service-specific mTLS
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: database
spec:
  selector:
    matchLabels:
      app: database
  mtls:
    mode: STRICT
  portLevelMtls:
    5432:
      mode: DISABLE
```

### AuthorizationPolicy
```yaml
# Deny all policy
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: deny-all
spec: {}

# Allow specific service
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-productpage
spec:
  selector:
    matchLabels:
      app: productpage
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/default/sa/bookinfo-gateway"]
  - operation:
      methods: ["GET"]
    when:
    - key: request.headers[end-user]
      values: ["alice", "bob"]

# JWT validation
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: require-jwt
spec:
  selector:
    matchLabels:
      app: httpbin
  rules:
  - from:
    - source:
        requestPrincipals: ["testing@secure.istio.io/testing@secure.istio.io"]
```

### RequestAuthentication
```yaml
apiVersion: security.istio.io/v1beta1
kind: RequestAuthentication
metadata:
  name: jwt-example
spec:
  selector:
    matchLabels:
      app: httpbin
  jwtRules:
  - issuer: "testing@secure.istio.io"
    jwksUri: "https://raw.githubusercontent.com/istio/istio/release-1.16/security/tools/jwt/samples/jwks.json"
    audiences:
    - "testing@secure.istio.io"
```

## Observability

### Enable Observability Add-ons
```bash
# Install Prometheus, Grafana, Jaeger, Kiali
kubectl apply -f samples/addons/prometheus.yaml
kubectl apply -f samples/addons/grafana.yaml
kubectl apply -f samples/addons/jaeger.yaml
kubectl apply -f samples/addons/kiali.yaml

# Access dashboards
istioctl dashboard kiali
istioctl dashboard grafana
istioctl dashboard jaeger
istioctl dashboard prometheus
```

### Telemetry Configuration
```yaml
apiVersion: telemetry.istio.io/v1alpha1
kind: Telemetry
metadata:
  name: metrics
spec:
  metrics:
  - providers:
    - name: prometheus
  - overrides:
    - match:
        metric: ALL_METRICS
      tagOverrides:
        request_id:
          value: "%{REQUEST_ID}"

---
apiVersion: telemetry.istio.io/v1alpha1
kind: Telemetry
metadata:
  name: tracing
spec:
  tracing:
  - providers:
    - name: jaeger

---
apiVersion: telemetry.istio.io/v1alpha1
kind: Telemetry
metadata:
  name: access-logs
spec:
  accessLogging:
  - providers:
    - name: otel
```

### Custom Metrics
```yaml
apiVersion: telemetry.istio.io/v1alpha1
kind: Telemetry
metadata:
  name: custom-metrics
spec:
  metrics:
  - overrides:
    - match:
        metric: requests_total
      disabled: false
    - match:
        metric: request_duration_milliseconds
      operation: UPSERT
      value: "response_time | 0"
  - providers:
    - name: prometheus
```

## Commands

### istioctl Commands
```bash
# Install and upgrade
istioctl install
istioctl upgrade
istioctl uninstall --purge

# Configuration analysis
istioctl analyze
istioctl analyze -n production
istioctl analyze deploy/productpage.yaml

# Proxy configuration
istioctl proxy-config cluster productpage-v1-123456
istioctl proxy-config listener productpage-v1-123456
istioctl proxy-config route productpage-v1-123456
istioctl proxy-config endpoint productpage-v1-123456

# Proxy status
istioctl proxy-status
istioctl proxy-status productpage-v1-123456

# Authentication policies
istioctl authn tls-check productpage-v1-123456.default.svc.cluster.local

# Traffic management
istioctl describe pod productpage-v1-123456
istioctl describe service productpage

# Debugging
istioctl bug-report
istioctl experimental describe pod productpage-v1-123456
```

### Traffic Splitting
```yaml
# Canary deployment
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: canary-deployment
spec:
  hosts:
  - productpage
  http:
  - match:
    - headers:
        canary:
          exact: "true"
    route:
    - destination:
        host: productpage
        subset: v2
  - route:
    - destination:
        host: productpage
        subset: v1
      weight: 90
    - destination:
        host: productpage
        subset: v2
      weight: 10
```

### Circuit Breaker
```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: circuit-breaker
spec:
  host: httpbin
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 1
      http:
        http1MaxPendingRequests: 1
        maxRequestsPerConnection: 1
    outlierDetection:
      consecutiveErrors: 1
      interval: 1s
      baseEjectionTime: 3m
      maxEjectionPercent: 100
```

### Timeout and Retry
```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: timeout-retry
spec:
  hosts:
  - ratings
  http:
  - route:
    - destination:
        host: ratings
    timeout: 10s
    retries:
      attempts: 3
      perTryTimeout: 2s
      retryOn: gateway-error,connect-failure,refused-stream
```

## Troubleshooting
```bash
# Check Istio installation
istioctl verify-install

# Check proxy configuration sync
istioctl proxy-status

# Analyze configuration issues
istioctl analyze

# Check sidecar injection
kubectl get pods -o jsonpath='{.items[*].spec.containers[*].name}'

# Debug traffic flow
istioctl proxy-config cluster <pod-name>
istioctl proxy-config listeners <pod-name>

# Check mTLS status
istioctl authn tls-check <pod-name>

# View Envoy logs
kubectl logs <pod-name> -c istio-proxy

# Enable debug logging
istioctl proxy-config log <pod-name> --level debug

# Check certificates
istioctl proxy-config secret <pod-name>
```

## Performance Tuning
```yaml
# Resource limits for sidecar
apiVersion: v1
kind: ConfigMap
metadata:
  name: istio-sidecar-injector
  namespace: istio-system
data:
  config: |
    template: |
      spec:
        containers:
        - name: istio-proxy
          resources:
            limits:
              cpu: 200m
              memory: 128Mi
            requests:
              cpu: 100m
              memory: 64Mi

# Optimize mesh configuration
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  name: control-plane
spec:
  values:
    pilot:
      env:
        EXTERNAL_ISTIOD: false
        PILOT_ENABLE_CROSS_CLUSTER_WORKLOAD_ENTRY: false
    global:
      meshConfig:
        defaultConfig:
          concurrency: 2
          proxyStatsMatcher:
            exclusionRegexps:
            - ".*circuit_breakers.*"
            - ".*upstream_rq_retry.*"
```

## Official Links
- [Istio Documentation](https://istio.io/latest/docs/)
- [Traffic Management](https://istio.io/latest/docs/concepts/traffic-management/)
- [Security](https://istio.io/latest/docs/concepts/security/)
- [Observability](https://istio.io/latest/docs/concepts/observability/)