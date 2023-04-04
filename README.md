# Anthos Service Mesh
# Deploying sample micro-services application
```sh
mkdir asm
cd asm

# Get online boutique code from git
kpt pkg get \
  https://github.com/japneet-sahni/anthos-service-mesh.git/online-boutique \
  online-boutique

# Deploy application using kubernetes manifests
kubectl apply -f online-boutique/kubernetes-manifests/namespaces
kubectl apply -f online-boutique/kubernetes-manifests/deployments
kubectl apply -f online-boutique/kubernetes-manifests/services

# Check all application namespaces and pods
kubectl get ns --show-labels | grep -w 'ad\|cart\|checkout\|currency\|email\|frontend\|loadgenerator\|payment\|product-catalog\|recommendation\|shipping'

kubectl get pods -A | grep -w 'ad\|cart\|checkout\|currency\|email\|frontend\|loadgenerator\|payment\|product-catalog\|recommendation\|shipping'

# Get Loadbalancer IP and hit through browser
kubectl get svc -n frontend
```

# Challenges
- How different micro-services are actually connected (if we want to understand the communication schema in our cluster.)
- No metrics (how many requests? what's the error rate)
- No distributed tracing (which service takes how much time for a request)
- Timeline (what if we want to dig into past)
- Security (what all traffic is encrypted and which service allows what)
- Traffic Management (what is the traffic split between different versions of micro-services)

# Installation of ASM
```sh
# Download asmcli
curl https://storage.googleapis.com/csm-artifacts/asm/asmcli_1.14 > asmcli
chmod +x asmcli

# ASM validate
./asmcli validate \
  --project_id japneet-project \
  --cluster_name cluster-1 \
  --cluster_location us-central1-a \
  --output_dir asm-install \
  --ca mesh_ca

# ASM install on GKE on GCP
./asmcli install \
  --project_id japneet-project \
  --cluster_name cluster-1 \
  --cluster_location us-central1-a \
  --output_dir asm-install \
  --enable_all \
  --ca mesh_ca

# ASM install for Anthos on VMWare
./asmcli install \
  --kubeconfig KUBECONFIG_FILE \
  --output_dir asm-install \
  --platform multicloud \
  --enable_all \
  --ca mesh_ca

# Check asm and istio namespaces and pods
kubectl get ns | grep -w 'asm-system\|istio-system\|gke-connect'
kubectl get pods -A | grep -w 'asm-system\|istio-system\|gke-connect'
```

# Automatic sidecar injection
```sh
# check namespace labels before enabling automatic sidecar injection
kubectl get ns --show-labels | grep -w 'ad\|cart\|checkout\|currency\|email\|frontend\|loadgenerator\|payment\|product-catalog\|recommendation\|shipping'

# Adding automatic sidecar injection label for all application namespaces
for ns in ad cart checkout currency email frontend loadgenerator \
  payment product-catalog recommendation shipping; do
    kubectl label namespace $ns istio-injection=enabled --overwrite
done;

# check namespace labels after enabling automatic sidecar injection
kubectl get ns --show-labels | grep -w 'ad\|cart\|checkout\|currency\|email\|frontend\|loadgenerator\|payment\|product-catalog\|recommendation\|shipping'

# Sidecars are not injected unless the deployments are restarted
kubectl get pods -A | grep -w 'ad\|cart\|checkout\|currency\|email\|frontend\|loadgenerator\|payment\|product-catalog\|recommendation\|shipping'

# Restart all application deployments in order to get sidecar injected
for ns in ad cart checkout currency email frontend loadgenerator \
  payment product-catalog recommendation shipping; do
    kubectl rollout restart deployment -n ${ns}
done;

# check pods after enabling automatic sidecar injection
kubectl get pods -A | grep -w 'ad\|cart\|checkout\|currency\|email\|frontend\|loadgenerator\|payment\|product-catalog\|recommendation\|shipping'
```

# Install Istio Ingress gateway
```sh
kubectl create namespace gatewayns
kubectl label namespace gatewayns istio-injection=enabled --overwrite
kubectl apply -n gatewayns -f asm-install/samples/gateways/istio-ingressgateway

# Gateway and VS for frontend
kubectl apply -f online-boutique/istio-manifests/frontend-gateway.yaml

# Access application now using istio ingress gateway
kubectl get service istio-ingressgateway  -n gatewayns
```

# Observability
- Topology
- Service Dashboard
- Timeline
- Traffic
![Observability](images/observability.png?raw=true "Observability")

# Security

## Mutual TLS
```sh
In Anthos Service Mesh 1.5 and later, auto mutual TLS (auto mTLS) is enabled by default. With auto mTLS, a client sidecar proxy automatically detects if the server has a sidecar. The client sidecar sends mTLS to workloads with sidecars and sends plaintext to workloads without sidecars. Note, however, services accept both plaintext and mTLS traffic. As you inject sidecar proxies to your Pods, we recommend that you also configure your services to only accept mTLS traffic.

With Anthos Service Mesh, you can enforce mTLS, outside of your application code, by applying a single YAML file.

Go to Anthos Security -> Policy Audit
```

```sh
# Go to both loadbalancer before below change

# Apply Strict authentication mode
for ns in ad cart checkout currency email frontend loadgenerator \
     payment product-catalog recommendation shipping; do
kubectl apply -n $ns -f online-boutique/istio-manifests/peer-authentication.yaml
done

Hit Ingress Gateway LB directly (Pass)
Hit Frontend LB directly (Fail)
Go to Anthos Security -> Policy Audit
```
## Result of strict mode
![Mutual TLS](images/mutual-tls.png?raw=true "Mutual TLS")

```sh
# Delete Strict authentication mode
for ns in ad cart checkout currency email frontend loadgenerator payment \
  product-catalog recommendation shipping; do
    kubectl delete peerauthentication -n $ns namespace-policy
done;
```

# Traffic Management
```sh
# Deploy your VirtualService and DestinationRule for v1 of productcatalog:
kubectl apply -f online-boutique/istio-manifests/canary/destination-vs-v1.yaml

# Deploy v2 of productcatalog
kubectl apply -f online-boutique/istio-manifests/canary/productcatalogservice-v2.yaml

# Get all pods
kubectl get pods -A | grep -w 'ad\|cart\|checkout\|currency\|email\|frontend\|loadgenerator\|payment\|product-catalog\|recommendation\|shipping'

## Apply destination rule for v2
kubectl apply -f online-boutique/istio-manifests/canary/destination-v1-v2.yaml

## Apply destination rule for v2
kubectl apply -f online-boutique/istio-manifests/canary/vs-split-traffic.yaml
```

![Traffic Management](images/traffic-management.png?raw=true "Traffic Management")

```sh
## Rollout
kubectl apply -f online-boutique/istio-manifests/canary/rollout-vs-v2.yaml

## Rollback
kubectl apply -f online-boutique/istio-manifests/canary/rollout-vs-v2.yaml
```

# Deletion

```sh
kubectl delete -f online-boutique/istio-manifests/canary/destination-vs-v1.yaml
kubectl delete -f online-boutique/istio-manifests/canary/productcatalogservice-v2.yaml
kubectl delete -f online-boutique/kubernetes-manifests/services
kubectl delete -f online-boutique/kubernetes-manifests/deployments
kubectl delete -n gatewayns -f asm-install/samples/gateways/istio-ingressgateway

for ns in ad cart checkout currency email frontend loadgenerator \
  payment product-catalog recommendation shipping gatewayns; do
    kubectl label namespace $ns istio-injection- --overwrite
done;

kubectl delete -f online-boutique/kubernetes-manifests/namespaces
kubectl delete ns gatewayns

kubectl delete validatingwebhookconfiguration,mutatingwebhookconfiguration -l operator.istio.io/component=Pilot
asm-install/istioctl x uninstall --purge
kubectl delete namespace istio-system asm-system --ignore-not-found=true
```

# Book-Info Application
```
mkdir asm
cd asm
kpt pkg get \
  https://github.com/japneet-sahni/anthos-service-mesh.git/book-info \
  book-info

curl https://storage.googleapis.com/csm-artifacts/asm/asmcli > asmcli
chmod +x asmcli

./asmcli install \
  --project_id japneet-project \
  --cluster_name cluster-1 \
  --cluster_location us-central1-c \
  --output_dir asm-install \
  --enable_all \
  --ca mesh_ca

kubectl apply -f book-info/bookinfo.yml
kubectl apply -f book-info/bookinfo-gateway.yml

kubectl create namespace gatewayns
kubectl label namespace gatewayns istio-injection=enabled --overwrite
kubectl apply -n gatewayns -f asm-install/samples/gateways/istio-ingressgateway
kubectl get service istio-ingressgateway  -n gatewayns

kubectl label namespace default istio-injection=enabled
kubectl rollout restart deployment

kubectl apply -f book-info/reviews-vs-v1.yml
kubectl apply -f book-info/reviews-vs-v2-header.yml
kubectl apply -f book-info/reviews-vs-v1-v3-split.yml
```