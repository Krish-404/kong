# Setting Up Kong Gateway with Nginx on Kubernetes

## Prerequisites
- Kubernetes cluster
- `kubectl` installed and configured
- Helm installed

---

## **1. Install Gateway API CRDs**
```sh
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.1.0/standard-install.yaml
```

---

## **2. Create and Apply Gateway Configuration**
Create a file `kong/gateway.yaml` and add the following content:
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: kong
  annotations:
    konghq.com/gatewayclass-unmanaged: 'true'
spec:
  controllerName: konghq.com/kic-gateway-controller
---
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: kong
  namespace: kong
spec:
  gatewayClassName: kong
  listeners:
  - name: proxy
    port: 80
    protocol: HTTP
    allowedRoutes:
      namespaces:
         from: All
```
Apply the configuration:
```sh
kubectl apply -f kong/gateway.yaml
```

Verify:
```sh
kubectl get gateway -n kong
```

---

## **3. Install Kong Gateway with Helm**
```sh
helm repo add kong https://charts.konghq.com
helm repo update
helm install kong kong/ingress -n kong --create-namespace
```
Wait for a few minutes to allow Kong to initialize.

---

## **4. Get the External IP of Kong Gateway**
```sh
export PROXY_IP=$(kubectl get svc --namespace kong kong-gateway-proxy -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo $PROXY_IP
```
Test Kong Gateway:
```sh
curl -i http://$PROXY_IP/
```
Expected output:
```
HTTP/1.1 404 Not Found
{ "message":"no Route matched with those values" }
```

---

## **5. Deploy Nginx Application**
Create a file `kong/nginx-deployment.yaml` with the following content:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx
  namespace: default
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: ClusterIP
```
Apply:
```sh
kubectl apply -f kong/nginx-deployment.yaml
```
Verify:
```sh
kubectl get pods -n default
kubectl get svc -n default nginx
```

---

## **6. Configure Kong to Route to Nginx**
Create a file `kong/nginx-httproute.yaml` with the following content:
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: nginx-route
  namespace: default
spec:
  parentRefs:
    - name: kong
      namespace: kong
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: nginx
          kind: Service
          port: 80
```
Apply:
```sh
kubectl apply -f kong/nginx-httproute.yaml
```
Verify:
```sh
kubectl get httproute -n default
```

Check Kong Gateway Service:
```sh
kubectl get svc -n kong kong-gateway-proxy
```

---

## **7. Verify HTTPRoute**

```sh
kubectl describe httproute -n default nginx-route
```
Expected output:
```
Status:
  Parents:
    Conditions:
      Reason: Accepted
      Type: Programmed
```

---

## **8. Test Connectivity**
```sh
curl -i http://$PROXY_IP/
```
Expected output:
```
HTTP/1.1 200 OK
Server: nginx
```

---

## **9. Additional Troubleshooting (If It Doesn’t Work)**
If the route still doesn’t work, reapply the Gateway:
```sh
kubectl apply -f kong/gateway.yaml
```
Then restart Kong:
```sh
kubectl rollout restart deployment -n kong kong-ingress-controller
```
Finally, verify:
```sh
kubectl describe httproute -n default nginx-route
```

---


