# istio-virtualservice-observability-demo

To run Istio and Kiali on a local Kind (Kubernetes in Docker) cluster on macOS with an active VirtualService demo, create a Kind cluster with mapped ports 80 and 443, install Istio with its demo profile and Kiali addon, deploy a sample app with a Gateway and VirtualService, and open the dashboard

## Prerequisites

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) installed and running
- [Homebrew](https://brew.sh/) installed

## 1. Create Kind Cluster on macOS

Create a cluster configuration file named kind-config.yaml to map HTTP/HTTPS ports from your Mac to the Docker container node

```bash
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
```

Run the cluster creation command in your terminal

```bash
kind create cluster --name istio-mesh --config kind-config.yaml
```
## 2. Install Istio and Kiali

Download and extract istioctl, then install Istio using the demo profile which includes components like Prometheus and tracing support:

Run the automated installation script in your terminal: 

```bash
curl -L https://istio.io/downloadIstio | sh
```

Change into the newly created version directory (e.g., cd istio-1.30.0):cd istio-*

Add the bin directory containing istioctl to your system path: 

```bash
export PATH=$PWD/bin:$PATH
```

```bash
istioctl install --set profile=demo -y
```
Apply the official Kiali addon and Prometheus dashboards from the Istio installation directory:

```bash
git clone https://github.com/istio/istio.git
```

YAML: 
```bash
istio/samples/addons/prometheus.yaml
istio/samples/addons/kiali.yaml
istio/samples/bookinfo/platform/kube/bookinfo.yaml
istio/samples/bookinfo/networking/bookinfo-gateway.yaml
```

```bash
kubectl apply -f samples/addons/prometheus.yaml
kubectl apply -f samples/addons/kiali.yaml
```
## 3. Deploy the Bookinfo Demo & VirtualService

```bash
kubectl create namespace bookinfo
kubectl label namespace bookinfo istio-injection=enabled
```

Deploy the Bookinfo sample application and tie it to an Istio Gateway and VirtualService configuration:

```bash
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml -n bookinfo
kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml -n bookinfo
```

## 4. Access Kiali and Generate Traffic
```bash
istioctl dashboard kiali
```

Generate incoming traffic to populate the Kiali service graph by running a continuous curl command against your local gateway endpoint or opening http://localhost/productpage if your gateway maps to port 80

```bash
curl -s http://localhost/productpage > /dev/null
```

## 5. Configure traffic splitting in Bookinfo Demo 

To configure traffic splitting in your Bookinfo demo, you must create a DestinationRule to define the version subsets (v1 and v2) and update the VirtualService to distribute the traffic 80/20

Define the Subsets (DestinationRule)

```bash
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: reviews
  namespace: bookinfo
spec:
  host: reviews
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
```
Apply it to your cluster:

```bash
kubectl apply -f reviews-destinationrule.yaml
```

## 6. Configure 80/20 Splitting (VirtualService)

Apply this configuration to route 80% of incoming traffic to v1 (no stars) and 20% to v2

```bash
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
  namespace: bookinfo
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
      weight: 80
    - destination:
        host: reviews
        subset: v2
      weight: 20
```

Apply it to your cluster:

```bash
kubectl apply -f reviews-traffic-split.yaml
```

## 7. Generate Traffic and Verify in Kiali 

Send 50 rapid requests in your terminal to see the weighted distribution take effect:

```bash
for i in {1..50}; do curl -s http://localhost/productpage | grep -o "color="; done
```

Open your Kiali Graph dashboard (istioctl dashboard kiali). Make sure you switch the graph edge display dropdown from "No edge labels" to Traffic Distribution. You will visually see the traffic line splitting into roughly 80% toward reviews-v1 and 20% toward reviews-v2
