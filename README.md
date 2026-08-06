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

Change into the newly created version directory (e.g., cd istio-x)

Add the bin directory containing istioctl to your system path: 

```bash
export PATH=$PWD/bin:$PATH
```

```bash
istioctl install --set profile=demo -y
```
Apply the official Kiali addon and Prometheus dashboards from the Istio installation directory:

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
kubectl apply -f samples/addons/grafana.yaml
```
## 3. Deploy the Bookinfo Demo & VirtualService

```bash
kubectl create namespace bookinfo
kubectl label namespace bookinfo istio-injection=enabled
```

```bash
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
service/details created
serviceaccount/bookinfo-details created
deployment.apps/details-v1 created
service/ratings created
serviceaccount/bookinfo-ratings created
deployment.apps/ratings-v1 created
service/reviews created
serviceaccount/bookinfo-reviews created
deployment.apps/reviews-v1 created
deployment.apps/reviews-v2 created
deployment.apps/reviews-v3 created
service/productpage created
serviceaccount/bookinfo-productpage created
```

Confirm all services and pods are correctly defined and running:

```bash
$ kubectl get services
NAME          TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE
details       ClusterIP   10.0.0.31    <none>        9080/TCP   6m
kubernetes    ClusterIP   10.0.0.1     <none>        443/TCP    7d
productpage   ClusterIP   10.0.0.120   <none>        9080/TCP   6m
ratings       ClusterIP   10.0.0.15    <none>        9080/TCP   6m
reviews       ClusterIP   10.0.0.170   <none>        9080/TCP   6m

and

$ kubectl get pods
NAME                             READY     STATUS    RESTARTS   AGE
details-v1-1520924117-48z17      2/2       Running   0          6m
productpage-v1-560495357-jk1lz   2/2       Running   0          6m
ratings-v1-734492171-rnr5l       2/2       Running   0          6m
reviews-v1-874083890-f0qf0       2/2       Running   0          6m
reviews-v2-1343845940-b34q5      2/2       Running   0          6m
reviews-v3-1813607990-8ch52      2/2       Running   0          6m
```

To confirm that the Bookinfo application is running, send a request to it by a curl command from some pod, for example from ratings:

```bash
kubectl exec "$(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}')" -c ratings -- curl -sS productpage:9080/productpage | grep -o "<title>.*</title>"
```

Now that the Bookinfo services are up and running, you need to make the application accessible from outside of your Kubernetes cluster, e.g., from a browser. A gateway is used for this purpose.

Create a gateway for the Bookinfo application:

Deploy the Bookinfo sample application and tie it to an Istio Gateway and VirtualService configuration:

```bash
kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml
gateway.networking.istio.io/bookinfo-gateway created
virtualservice.networking.istio.io/bookinfo created
```
Confirm the gateway has been created:

```bash
$ kubectl get gateway
NAME               AGE
bookinfo-gateway   32s
```

## 4. Access Kiali and Generate Traffic
```bash
istioctl dashboard kiali
```

Generate incoming traffic to populate the Kiali service graph by running a continuous curl command against your local gateway endpoint or opening http://localhost:9080/productpage if your gateway maps to port 80

```bash
curl -s http://localhost:9080/productpage > /dev/null
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
for i in {1..50}; do curl -s http://localhost:9080/productpage | grep -o "color="; done
```

Steps to Port-Forward Bookinfo

```bash
kubectl port-forward svc/productpage 9080:9080
```
View / Open the page

```bash
http://localhost:9080/productpage
```
Open your Kiali Graph dashboard (istioctl dashboard kiali). Make sure you switch the graph edge display dropdown from "No edge labels" to Traffic Distribution. You will visually see the traffic line splitting into roughly 80% toward reviews-v1 and 20% toward reviews-v2
