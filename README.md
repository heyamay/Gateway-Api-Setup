# Gateway API POC with NGINX Gateway Fabric on AWS EKS

Proof-of-Concept demonstrating migration from classic NGINX Ingress Controller to **Kubernetes Gateway API** using **NGINX Gateway Fabric** on AWS EKS.

Goal: Expose a simple application (initially nginx:alpine, later your real app) via HTTP/HTTPS using modern Gateway API resources instead of Ingress.

## Why Gateway API?

- NGINX Ingress Controller support ends in **March 2026**.
- Gateway API is the future-standard Kubernetes way to handle ingress traffic.
- Better extensibility, role-based configuration, cross-namespace support, and more expressive routing.

## Features Demonstrated

- AWS EKS cluster creation with eksctl
- NGINX Gateway Fabric installation via OCI Helm chart
- Gateway + HTTPRoute for routing
- HTTPS termination with self-signed certificate
- Cross-namespace routing (Gateway in `nginx-gateway`, app in `prod-app`)
- Catch-all routing to prove connectivity
- Basic proxy settings via ClientSettingsPolicy

## Prerequisites

- AWS CLI configured (`aws configure`)
- eksctl installed (`brew install eksctl` or curl method)
- kubectl
- helm v3
- openssl (for self-signed cert)

## Architecture Overview
Internet → AWS ELB (LoadBalancer svc) → NGINX Gateway Fabric (DaemonSet)
→ Gateway (nginx-gateway ns)
→ HTTPRoute (prod-app ns)
→ Service (ClusterIP) → Pod (nginx:alpine or your app)


## Step-by-Step Setup

### 1. Create EKS Cluster

```
eksctl create cluster \
  --name gateway-poc \
  --version 1.29 \
  --region us-east-1 \
  --nodegroup-name standard-workers \
  --node-type t3.small \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 3 \
  --managed \
  --profile personal
```

Wait ~15 minutes.
Verify:
```
kubectl get nodes
```

2. Install Gateway API CRDs (standard channel)
```
kubectl kustomize "https://github.com/nginx/nginx-gateway-fabric/config/crd/gateway-api/standard?ref=v2.3.0" | kubectl apply -f -
```
Verify:
```
kubectl get crds | grep gateway.networking.k8s.io
```
3. Install NGINX Gateway Fabric (from local chart)
Pull chart:
```
helm pull oci://ghcr.io/nginx/charts/nginx-gateway-fabric --version 2.3.0 --untar
cd nginx-gateway-fabric
```
Install (use --set nginx.service.type=LoadBalancer):
```
helm install ngf . \
  --namespace nginx-gateway \
  --create-namespace \
  --set nginx.service.type=LoadBalancer
```
Wait for ready:
```
kubectl wait --timeout=5m -n nginx-gateway deployment/ngf-nginx-gateway-fabric --for=condition=Available
```
4. Create Gateway Resource (exposes LB)
gateway.yaml
```
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: nginx-gateway
  namespace: nginx-gateway
spec:
  gatewayClassName: nginx
  listeners:
  - name: http
    protocol: HTTP
    port: 80
    allowedRoutes:
      namespaces:
        from: All           # or use selector + label on namespaces
  - name: https
    protocol: HTTPS
    port: 443
    tls:
      mode: Terminate
      certificateRefs:
      - kind: Secret
        name: poc-tls-secret
    allowedRoutes:
      namespaces:
        from: All
```
Apply:
```
kubectl apply -f gateway.yaml
```

Get external hostname:
```
kubectl get svc nginx-gateway-nginx -n nginx-gateway -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
```
Example: a25e4448192f64ec6864ce36af8011dc-434611334.us-east-1.elb.amazonaws.com

5. Create Self-Signed TLS Secret
```
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/CN=example.com"
kubectl create secret tls poc-tls-secret --key tls.key --cert tls.crt -n nginx-gateway
rm tls.key tls.crt
```
6. Allow Cross-Namespace Routing (ReferenceGrant)
referencegrant.yaml
```
apiVersion: gateway.networking.k8s.io/v1
kind: ReferenceGrant
metadata:
  name: allow-prod-app-routes
  namespace: nginx-gateway
spec:
  from:
  - group: gateway.networking.k8s.io
    kind: HTTPRoute
    namespace: prod-app
  to:
  - group: gateway.networking.k8s.io
    kind: Gateway
```
Apply:
```
kubectl apply -f referencegrant.yaml
```
Alternative (simpler): Label namespace
```
kubectl label namespace prod-app nginx-gateway-access=true
```
And update Gateway listeners with selector if needed.

7. Deploy Sample App & Service
deployment.yaml

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: poc-app
  namespace: prod-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: poc-app
  template:
    metadata:
      labels:
        app: poc-app
    spec:
      containers:
      - name: poc-app
        image: nginx:alpine
        ports:
        - containerPort: 80
```
service.yaml
```
apiVersion: v1
kind: Service
metadata:
  name: poc-app-svc
  namespace: prod-app
spec:
  type: ClusterIP
  ports:
  - port: 80
    targetPort: 80
    name: http
  selector:
    app: poc-app
```
Apply:
```
kubectl create namespace prod-app
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
```
8. Create HTTPRoute (Catch-all for testing)
httproute.yaml
```
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: poc-route
  namespace: prod-app
spec:
  parentRefs:
  - name: nginx-gateway
    namespace: nginx-gateway
  rules:
  - backendRefs:
    - name: poc-app-svc
      port: 80
    # matches: []   # catch-all (remove path match for root testing)
```
Apply:
```
kubectl apply -f httproute.yaml
```
9. Test
```
# Get LB hostname
LB=$(kubectl get svc nginx-gateway-nginx -n nginx-gateway -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

curl -v http://$LB/
curl -vk https://$LB/
```

Expected: NGINX welcome page (200 OK)
For specific path (after switching to your app):
```
matches:
- path:
    type: PathPrefix
    value: /adapteracc
```
10. Add Proxy Settings (Timeouts, Body Size)
clientsettingspolicy.yaml
```
apiVersion: gateway.nginx.org/v1alpha1
kind: ClientSettingsPolicy
metadata:
  name: poc-proxy-settings
  namespace: nginx-gateway
spec:
  body:
    maxSize: 100m
  request:
    connectTimeout: 600s
    readTimeout: 600s
    sendTimeout: 600s
  targetRef:
    group: gateway.networking.k8s.io
    kind: Gateway
    name: nginx-gateway
```
Apply:
```
kubectl apply -f clientsettingspolicy.yaml
```
Cleanup
```
eksctl delete cluster --name gateway-poc
```

Troubleshooting

- 404 from nginx → path not matching backend content or catch-all not applied
- No response → check Gateway status (kubectl describe gateway nginx-gateway -n nginx-gateway)
- Route not Accepted → check ReferenceGrant and allowedRoutes.from
- Pod logs: kubectl logs -l app=poc-app -n prod-app

