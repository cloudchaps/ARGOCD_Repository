# Kubernetes NGINX Ingress Migration Guide

This document summarizes the migration from a custom NGINX Deployment + ConfigMap reverse proxy to a Kubernetes-native NGINX Ingress setup.

## 1. Previous Architecture

The application originally used an NGINX Deployment with a custom `nginx.conf` mounted from a ConfigMap.

Traffic flow:

```text
User → NGINX Deployment → Services → Pods
```

Example previous NGINX ConfigMap logic:

```nginx
server {
    listen 80;

    location / {
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_pass http://client/;
    }

    location /api {
        proxy_pass http://nodeapi-service:5000;
    }

    location /webapi {
        proxy_pass http://javaapi-service:9000;
    }
}
```

## 2. New Kubernetes-Native Architecture

The new approach uses the NGINX Ingress Controller and Kubernetes Ingress resources.

Traffic flow:

```text
User → DNS / hosts file → NGINX Ingress Controller → Ingress Rules → Services → Pods
```

This is closer to a real-world Kubernetes architecture.

## 3. Enable NGINX Ingress Controller in Minikube

```bash
minikube addons enable ingress
```

Verify that the ingress controller is running:

```bash
kubectl get pods -n ingress-nginx
```

Expected result:

```text
ingress-nginx-controller-xxxxx   1/1   Running
```

## 4. Final Working Ingress Configuration

Configured location:

```text
/Kubernetes_Manifest_Files/01-Frontend/05-IngressController/Ingress.yaml
```

Apply the Ingress:

```bash
kubectl apply -f k8s/ingress/emart-ingress.yaml
```

## 5. Important Lesson Learned

The first Ingress attempt used regex-style paths and rewrite behavior:

```yaml
path: /api(/|$)(.*)
pathType: ImplementationSpecific
```

That approach can break backend routing if the application expects the original path.

For this application, the working solution was simpler:

```yaml
path: /api
pathType: Prefix
```

and:

```yaml
path: /webapi
pathType: Prefix
```

No rewrite annotation was needed.

## 6. Local DNS Setup

Get the Minikube IP:

```bash
minikube ip
```

Edit the local hosts file:

```bash
sudo nano /etc/hosts
```

Add:

```text
<minikube-ip> emart.local
```

Example:

```text
192.168.49.2 emart.local
```

## 7. Test External Routing

Test the frontend:

```bash
curl -v http://emart.local/
```

Test NodeJS backend:

```bash
curl -v http://emart.local/api
```

Test Java backend:

```bash
curl -v http://emart.local/webapi
```

## 8. Troubleshooting Checklist

Use this flow when the application is not reachable through Ingress.

### Step 1: Check Ingress Controller

```bash
kubectl get pods -n ingress-nginx
```

Expected:

```text
STATUS = Running
READY = 1/1
```

Check logs:

```bash
kubectl logs -n ingress-nginx <ingress-controller-pod-name>
```

Find controller pod:

```bash
kubectl get pods -n ingress-nginx
```

### Step 2: Check the Ingress Resource

```bash
kubectl get ingress -n emartapp-application
```

Describe it:

```bash
kubectl describe ingress emart-ingress -n emartapp-application
```

Check:
- Host is correct: `emart.local`
- Paths are correct: `/`, `/api`, `/webapi`
- Backend services exist
- Backend service ports are correct

### Step 3: Check Services

```bash
kubectl get svc -n emartapp-application
```

Verify these services exist:

```text
client
nodeapi-service
javaapi-service
```

Expected ports:

```text
client           4200
nodeapi-service  5000
javaapi-service  9000
```

### Step 4: Check Service Endpoints

```bash
kubectl get endpoints -n emartapp-application
```

Each service should show pod IPs.

If a service has no endpoints, check:
- Service selector
- Pod labels
- Pod readiness

Useful commands:

```bash
kubectl get svc <service-name> -n emartapp-application -o yaml
kubectl get pods -n emartapp-application --show-labels
kubectl describe pod <pod-name> -n emartapp-application
```

Common issue:

```text
Service selector does not match Pod labels.
```

### Step 5: Test Service Connectivity from Inside the Cluster

Create a temporary curl pod:

```bash
kubectl run test-curl \
  -n emartapp-application \
  --rm -it \
  --image=curlimages/curl \
  -- sh
```

Inside the pod, test:

```bash
curl -v http://client:4200
curl -v http://nodeapi-service:5000/api
curl -v http://javaapi-service:9000/webapi
```

If these fail, the problem is not Ingress. Check the backend service or application.

### Step 6: Test External Access

From your local machine:

```bash
curl -v http://emart.local/
curl -v http://emart.local/api
curl -v http://emart.local/webapi
```

If internal service tests work but external access fails, check:
- `/etc/hosts`
- Ingress resource
- Ingress controller
- Minikube IP
- Firewall/local network

## 9. Common Errors and Fixes

### Error: 502 Bad Gateway

Meaning:

```text
Ingress reached the route but failed to connect to the backend service.
```

Check:

```bash
kubectl get endpoints -n emartapp-application
kubectl describe ingress emart-ingress -n emartapp-application
kubectl logs -n ingress-nginx <ingress-controller-pod-name>
```

Common causes:
- Service has no endpoints
- Wrong service port
- Backend pod not ready
- App not listening on expected port

### Error: 503 Service Unavailable

Meaning:

```text
No healthy backend is available.
```

Check:

```bash
kubectl get pods -n emartapp-application
kubectl describe pod <pod-name> -n emartapp-application
kubectl get endpoints -n emartapp-application
```

Common causes:
- Readiness probe failing
- Service selector mismatch
- Pod not ready

### Error: 404 Not Found

Meaning:

```text
Ingress rule does not match the requested host/path, or backend app does not serve that path.
```

Check:

```bash
kubectl describe ingress emart-ingress -n emartapp-application
curl -v http://emart.local/api
curl -v http://emart.local/webapi
```

Common causes:
- Wrong path
- Wrong host
- Backend expects different route
- Rewrite annotation changing the request path

### Error: Backend works internally but not externally

Check:

```bash
kubectl get ingress -n emartapp-application
kubectl describe ingress emart-ingress -n emartapp-application
cat /etc/hosts
minikube ip
```

Common causes:
- Wrong `/etc/hosts` entry
- Wrong Ingress host
- Ingress controller not running

## 10. Key Concepts

### Service

A Kubernetes Service provides stable internal access to pods.

```text
Service → Pods
```

### Ingress

Ingress manages external HTTP/HTTPS routing into the cluster.

```text
External User → Ingress → Service → Pod
```

### Ingress Controller

The Ingress Controller is the actual reverse proxy implementation, such as NGINX.

```text
Ingress Resource = routing rules
Ingress Controller = component that applies the routing rules
```

### Prefix Path

This matches all requests starting with a path.

Example:

```yaml
path: /api
pathType: Prefix
```

Matches:

```text
/api
/api/products
/api/users/1
```

## 11. Final Working Flow

```text
http://emart.local/       → client:4200
http://emart.local/api    → nodeapi-service:5000
http://emart.local/webapi → javaapi-service:9000
```

## 12. Interview Notes

This setup mirrors a real-world Kubernetes ingress architecture.

A strong explanation:

> I replaced a custom NGINX reverse proxy deployment with a Kubernetes-native Ingress model. The NGINX Ingress Controller handles external traffic, while Ingress rules route requests based on path to internal ClusterIP services. This is more scalable and closer to production Kubernetes patterns.