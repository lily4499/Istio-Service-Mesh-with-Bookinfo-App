
# 🚀 Istio Service Mesh with Bookinfo App on Kubernetes

## 🌍 Real-World Scenario

You're a DevOps engineer at a company adopting microservices. Each service is built independently but deployed on a shared Kubernetes cluster. To manage traffic, observe service health, enforce policies, and secure communication, you decide to deploy **Istio Service Mesh**. You use the **Bookinfo microservice app** to demonstrate core capabilities like **sidecar injection**, **traffic routing**, **observability**, and **service monitoring**.

---

## 🗂️ Project Structure

```
istio-mesh-demo/
├── manifests/
│   ├── gateway-virtualservice.yaml        # Istio Gateway + VirtualService for Bookinfo
│   └── istio-destinationrules.yaml        # (Optional) Destination rules
├── scripts/
│   └── cleanup.sh                         # Uninstall Bookinfo app
├── README.md
```

---

## ✅ Step-by-Step Guide

### 1. Deploy a Kubernetes Cluster

Use any managed Kubernetes provider (EKS, AKS, GKE) or a local one like Minikube.

```bash
minikube start --driver=docker
```

---

### 2. Install Istio CLI

```bash
curl -L https://istio.io/downloadIstio | sh -
cd istio-1.17.2        # Replace with your version
export PATH=$PWD/bin:$PATH
```

Or move `istioctl` to system path:

```bash
sudo mv bin/istioctl /usr/local/bin
```

---

### 3. Install Istio Core Components

```bash
istioctl install --set profile=demo -y
```

✅ This installs:
- `istio-system` namespace
- Istio control plane (istiod)
- Ingress gateway

Verify installation:

```bash
kubectl get pods -n istio-system
```

---

### 4. Enable Automatic Sidecar Injection

```bash
kubectl label namespace default istio-injection=enabled --overwrite
kubectl get ns --show-labels
```

This ensures that all pods in the namespace have the Envoy sidecar injected.

---

### 5. Deploy Sample Application (Bookinfo)

```bash
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
```

Verify:

```bash
kubectl get pods
kubectl get svc
```

Check response from within the cluster:

```bash
kubectl exec "$(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}')" \
-c ratings -- curl -sS productpage:9080/productpage | grep -o "<title>.*</title>"
```

Expected output:

```html
<title>Simple Bookstore App</title>
```

Describe a pod to confirm 2 containers (App + Envoy):

```bash
kubectl describe pod <bookinfo-pod-name>
```

---

### 6. Istio Traffic Management – Gateway & VirtualService

Create a file: `manifests/gateway-virtualservice.yaml`

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: my-microservice-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
    - port:
        number: 80
        name: http
        protocol: HTTP
      hosts:
        - "*"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: bookinfo
spec:
  hosts:
    - "*"
  gateways:
    - my-microservice-gateway
  http:
    - match:
        - uri:
            prefix: /productpage
      route:
        - destination:
            host: productpage
            port:
              number: 9080
```

Apply the configuration:

```bash
kubectl apply -f manifests/gateway-virtualservice.yaml
istioctl analyze
kubectl get gateway
kubectl get virtualservice
```

---

### 7. Access Bookinfo Application via Ingress

```bash
kubectl get svc istio-ingressgateway -n istio-system
```

Set environment variables:

```bash
export INGRESS_HOST=$(kubectl -n istio-system get svc istio-ingressgateway \
-o jsonpath='{.status.loadBalancer.ingress[0].ip}')
export INGRESS_PORT=$(kubectl -n istio-system get svc istio-ingressgateway \
-o jsonpath='{.spec.ports[?(@.name=="http2")].port}')
export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT
echo $GATEWAY_URL
```

Visit the app:

```bash
http://$GATEWAY_URL/productpage
```

---

### 8. Deploy Istio Addons: Observability Tools

```bash
kubectl apply -f samples/addons
kubectl rollout status deployment/kiali -n istio-system
```

Access dashboards:

```bash
istioctl dashboard kiali
istioctl dashboard jaeger
```

Or port-forward manually:

```bash
kubectl port-forward svc/kiali -n istio-system 20001:20001
```

Visit:

```
http://localhost:20001
```

---

### 9. (Optional) Destination Rules & Service Entries

Create a file: `manifests/istio-destinationrules.yaml`

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: productpage
spec:
  host: productpage
  subsets:
    - name: v1
      labels:
        version: v1
```

Apply it:

```bash
kubectl apply -f manifests/istio-destinationrules.yaml
```

---

### 10. Cleanup (Optional)

```bash
samples/bookinfo/platform/kube/cleanup.sh
```

---

## 📊 Monitoring

Istio provides built-in tools to visualize, trace, and debug traffic in your microservices.

### 🔹 Kiali – Service Mesh Dashboard

```bash
istioctl dashboard kiali
```

Or:

```bash
kubectl port-forward svc/kiali -n istio-system 20001:20001
```

Visit: [http://localhost:20001](http://localhost:20001)

Key features:
- Live service graph
- Metrics: latency, error rates, traffic volume
- Namespace health
- Configuration validation

---

### 🔹 Jaeger – Distributed Tracing

```bash
istioctl dashboard jaeger
```

Or:

```bash
kubectl port-forward svc/jaeger -n istio-system 16686:16686
```

Visit: [http://localhost:16686](http://localhost:16686)

Key features:
- View trace data
- Identify slow services
- Understand call chains across services

---

## 🧾 Summary

| Step                    | Purpose                                |
|-------------------------|----------------------------------------|
| Deploy Cluster          | Provide Kubernetes environment         |
| Install Istio           | Set up service mesh components         |
| Enable Sidecar Injection| Inject Envoy proxy in pods             |
| Deploy App              | Demonstrate service mesh capabilities  |
| Gateway & VirtualService| Route and expose the app externally    |
| Istio Addons            | Observe, trace, and debug traffic      |
| Ingress IP              | Expose app via public ingress gateway  |
| Cleanup                 | Remove app and Istio configurations    |

---

