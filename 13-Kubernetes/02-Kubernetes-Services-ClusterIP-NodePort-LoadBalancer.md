# Kubernetes Services: ClusterIP, NodePort and LoadBalancer

## Overview

In Kubernetes, Pods are temporary resources.

A Pod can be deleted, restarted, or recreated at any time. When this happens, the Pod receives a new IP address.

Because Pod IP addresses are not permanent, applications should not communicate directly using Pod IPs.

Kubernetes solves this problem using **Services**.

A Service provides:

- Stable IP Address
- Stable DNS Name
- Load Balancing
- Service Discovery
- External Access (if required)

---

# Why Do We Need Services?

Without Service:

```text
User
  |
  v
Pod (10.10.1.15)

Pod Deleted
  |
  v
New Pod (10.10.2.30)

Connection Broken
```

With Service:

```text
User
  |
  v
Service
  |
  v
Pod

Pod Recreated
  |
  v
New Pod

Service Still Works
```

The Service remains unchanged even if Pods change.

---

# Kubernetes Service Architecture

```text
Client
   |
   v
Service
   |
   v
Pod
```

A Service identifies Pods using Labels and Selectors.

Example:

Pod Label:

```yaml
labels:
  app: web
```

Service Selector:

```yaml
selector:
  app: web
```

The Service forwards traffic only to Pods matching the selector.

---

# Types of Kubernetes Services

```text
Service
│
├── ClusterIP
├── NodePort
└── LoadBalancer
```

---

# Service Comparison

| Service Type | Internal Access | External Access | Use Case |
|-------------|----------------|----------------|----------|
| ClusterIP | Yes | No | Internal communication |
| NodePort | Yes | Yes | Testing and Labs |
| LoadBalancer | Yes | Yes | Production |

---

# 1. ClusterIP Service

## Theory

ClusterIP is the default Kubernetes Service type.

It exposes the application only inside the Kubernetes cluster.

External users cannot access the application directly.

### Architecture

```text
Frontend Pod
      |
      v
ClusterIP Service
      |
      v
Backend Pod
```

### Real-World Example

```text
Frontend Application
        |
        v
Backend API Service
        |
        v
Database Service
```

All communication remains inside the cluster.

---

# Practical: ClusterIP Service

## Step 1: Create Pod

### File: pod1-ClusterIP.yml

```yaml
apiVersion: v1
kind: Pod

metadata:
  name: nginx-clusterip-pod
  labels:
    app: web

spec:
  containers:
  - name: nginx-container
    image: nginx:latest

    ports:
    - containerPort: 80
```

Create Pod:

```bash
kubectl apply -f pod1-ClusterIP.yml
```

Verify:

```bash
kubectl get pods
```

Expected:

```text
nginx-clusterip-pod   Running
```

---

## Step 2: Create ClusterIP Service

### File: Service-ClusterIP.yml

```yaml
apiVersion: v1
kind: Service

metadata:
  name: nginx-clusterip-service

spec:
  type: ClusterIP

  selector:
    app: web

  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```

Create Service:

```bash
kubectl apply -f Service-ClusterIP.yml
```

Verify:

```bash
kubectl get svc
```

Example Output:

```text
NAME                      TYPE        CLUSTER-IP
nginx-clusterip-service   ClusterIP   100.64.12.45
```

---

## Step 3: Test ClusterIP Service

Get Service IP:

```bash
kubectl get svc
```

Copy the Cluster-IP.

Example:

```text
100.64.12.45
```

---

## Step 4: Connect to Worker Node

AWS Console:

```text
EC2
→ Instances
→ Worker Node
→ Connect
→ EC2 Instance Connect
```

Open terminal.

---

## Step 5: Test Using curl

Run:

```bash
curl http://100.64.12.45
```

Example:

```bash
curl http://100.64.12.45
```

Expected Output:

```html
Welcome to nginx!
```

You will receive the Nginx default webpage HTML.

### Why Are We Testing From Worker Node?

Because:

```text
ClusterIP
=
Internal Cluster Access Only
```

It cannot be opened directly from your local browser.

---

## Key Point

```text
ClusterIP = Accessible Only Inside Kubernetes Cluster
```

---

# 2. NodePort Service

## Theory

NodePort exposes an application through a port on every Kubernetes Node.

Users can access the application from outside the cluster using:

```text
<Node-IP>:<NodePort>
```

---

## Architecture

```text
Browser
   |
Node Public IP
   |
NodePort
   |
Service
   |
Pod
```

---

## NodePort Range

```text
30000 - 32767
```

---

# Practical: NodePort Service

## Step 1: Create Pod

### File: pod1-NodePort.yml

```yaml
apiVersion: v1
kind: Pod

metadata:
  name: nginx-nodeport-pod

  labels:
    app: nginx-nodeport-app

spec:
  containers:
  - name: nodeport-container
    image: nginx

    ports:
    - containerPort: 80
```

Create Pod:

```bash
kubectl apply -f pod1-NodePort.yml
```

Verify:

```bash
kubectl get pods
```

---

## Step 2: Create NodePort Service

### File: Service-NodePort.yml

```yaml
apiVersion: v1
kind: Service

metadata:
  name: nginx-nodeport-service

spec:
  type: NodePort

  selector:
    app: nginx-nodeport-app

  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
    nodePort: 30080
```

Create Service:

```bash
kubectl apply -f Service-NodePort.yml
```

Verify:

```bash
kubectl get svc
```

Expected:

```text
nginx-nodeport-service
80:30080/TCP
```

---

## Step 3: Get Node Public IP

```bash
kubectl get nodes -o wide
```

Or

AWS Console:

```text
EC2
→ Worker Node
→ Public IPv4 Address
```

Example:

```text
13.233.xx.xx
```

---

## Step 4: Open Security Group Port

AWS Console:

```text
EC2
→ Security Groups
→ Worker Node Security Group
→ Inbound Rules
→ Edit Inbound Rules
```

Add:

```text
Type: Custom TCP
Port: 30080
Source: 0.0.0.0/0
```

Save.

---

## Step 5: Test NodePort

Open Browser:

```text
http://13.233.xx.xx:30080
```

Expected:

```html
Welcome to nginx!
```

---

## Alternative Testing Using curl

```bash
curl http://13.233.xx.xx:30080
```

Expected:

```html
Welcome to nginx!
```

---

## Key Point

```text
NodePort = Accessible Inside and Outside Cluster
```

---

# 3. LoadBalancer Service

## Theory

LoadBalancer is primarily used in cloud environments like AWS.

When a LoadBalancer Service is created:

Kubernetes automatically creates an AWS Load Balancer.

Traffic Flow:

```text
User
  |
AWS Load Balancer
  |
Kubernetes Service
  |
Pod
```

---

## Benefits

- Public Access
- High Availability
- Load Distribution
- Production Ready

---

# Practical: LoadBalancer Service

## Step 1: Create Pod

### File: pod1-loadbalancer.yml

```yaml
apiVersion: v1
kind: Pod

metadata:
  name: nginx-lb-pod

  labels:
    app: nginx-lb-app

spec:
  containers:
  - name: nginx-lb-container
    image: nginx:latest

    ports:
    - containerPort: 80
```

Create Pod:

```bash
kubectl apply -f pod1-loadbalancer.yml
```

Verify:

```bash
kubectl get pods
```

---

## Step 2: Create LoadBalancer Service

### File: Service-loadbalancer.yml

```yaml
apiVersion: v1
kind: Service

metadata:
  name: nginx-loadbalancer-service

spec:
  type: LoadBalancer

  selector:
    app: nginx-lb-app

  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```

Create Service:

```bash
kubectl apply -f Service-loadbalancer.yml
```

Verify:

```bash
kubectl get svc
```

Initially:

```text
EXTERNAL-IP
<pending>
```

Wait 2-5 minutes.

---

## Step 3: Get Load Balancer DNS

Run:

```bash
kubectl get svc
```

Example:

```text
a1b2c3d4e5.ap-south-1.elb.amazonaws.com
```

---

## Step 4: Verify in AWS

AWS Console:

```text
EC2
→ Load Balancers
```

You should see a new AWS Load Balancer automatically created.

---

## Step 5: Test LoadBalancer

Open Browser:

```text
http://a1b2c3d4e5.ap-south-1.elb.amazonaws.com
```

Expected:

```html
Welcome to nginx!
```

---

## Alternative Testing Using curl

```bash
curl http://a1b2c3d4e5.ap-south-1.elb.amazonaws.com
```

Expected:

```html
Welcome to nginx!
```

---

## Key Point

```text
LoadBalancer = Automatically Creates AWS Load Balancer
```

---

# create vs apply

## kubectl create

Used when creating a resource for the first time.

```bash
kubectl create -f file.yml
```

---

## kubectl apply

Used for:

- Creating resources
- Updating resources
- Day-to-day deployments

```bash
kubectl apply -f file.yml
```

Recommended approach:

```bash
kubectl apply -f file.yml
```

---

# Useful Commands

View Pods:

```bash
kubectl get pods
```

View Services:

```bash
kubectl get svc
```

Detailed Service Information:

```bash
kubectl describe svc nginx-clusterip-service

kubectl describe svc nginx-nodeport-service

kubectl describe svc nginx-loadbalancer-service
```

View Service Endpoints:

```bash
kubectl get endpoints
```

---

# Cleanup

Delete ClusterIP Resources:

```bash
kubectl delete -f pod1-ClusterIP.yml
kubectl delete -f Service-ClusterIP.yml
```

Delete NodePort Resources:

```bash
kubectl delete -f pod1-NodePort.yml
kubectl delete -f Service-NodePort.yml
```

Delete LoadBalancer Resources:

```bash
kubectl delete -f pod1-loadbalancer.yml
kubectl delete -f Service-loadbalancer.yml
```

Verify:

```bash
kubectl get pods

kubectl get svc
```

---

# Interview Questions

### Why do we need Services in Kubernetes?

Because Pod IP addresses are temporary and can change whenever Pods are recreated.

---

### Which Service type is default?

```text
ClusterIP
```

---

### Which Service type allows external access using a custom port?

```text
NodePort
```

---

### Which Service type automatically creates an AWS Load Balancer?

```text
LoadBalancer
```

---

### What is the default NodePort range?

```text
30000 - 32767
```

---

### Which Service type is mostly used in production?

```text
LoadBalancer
```

---

# Summary

```text
ClusterIP
    |
    └── Internal Communication Only

NodePort
    |
    └── Access Using NodeIP:NodePort

LoadBalancer
    |
    └── Creates AWS Load Balancer Automatically
```

| Service Type | Internal Access | External Access | Typical Use |
|-------------|----------------|----------------|-------------|
| ClusterIP | Yes | No | Internal Apps |
| NodePort | Yes | Yes | Labs & Testing |
| LoadBalancer | Yes | Yes | Production Applications |
