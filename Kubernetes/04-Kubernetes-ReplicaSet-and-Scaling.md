# Kubernetes ReplicaSet (RS) and Scaling

## Overview

ReplicaSet (RS) is a Kubernetes controller responsible for maintaining a specified number of Pod replicas.

Its primary purpose is to ensure that the desired number of Pods are always running.

If a Pod fails, crashes, or is deleted accidentally, ReplicaSet automatically creates a replacement Pod.

ReplicaSet is the modern replacement for ReplicationController.

---

# Why Do We Need ReplicaSet?

Imagine an application running with 2 Pods.

```text
Pod-1 Running
Pod-2 Running
```

If one Pod fails:

```text
Pod-1 Running
Pod-2 Deleted
```

Application capacity decreases.

ReplicaSet detects this difference and immediately creates a new Pod.

```text
Pod-1 Running
Pod-3 Running
```

Desired state is restored automatically.

---

# ReplicaSet Architecture

```text
ReplicaSet
      |
      |
      +----------------+
      |                |
      v                v
    Pod-1           Pod-2
```

ReplicaSet continuously watches Pods and maintains the desired number of replicas.

---

# ReplicationController vs ReplicaSet

| Feature | ReplicationController | ReplicaSet |
|----------|----------|----------|
| Maintains Replica Count | Yes | Yes |
| Self Healing | Yes | Yes |
| Scaling | Yes | Yes |
| Label Selectors | Equality Based | Set Based + Equality Based |
| Used By Deployment | No | Yes |
| Recommended Today | No | Yes |

---

# ReplicaSet Components

## replicas

Specifies the number of Pods that should run.

Example:

```yaml
replicas: 2
```

---

## selector

Used to identify Pods managed by the ReplicaSet.

Example:

```yaml
selector:
  matchLabels:
    app: mobile
```

---

## template

Defines the Pod configuration.

Example:

```yaml
template:
  metadata:
    labels:
      app: mobile
```

---

# Practical Lab: Create ReplicaSet

## Architecture

```text
ReplicaSet
     |
     +------+
     |      |
     v      v
   Pod-1  Pod-2
```

---

# Step 1: Create ReplicaSet YAML

## File: my-rs.yml

```yaml
apiVersion: apps/v1
kind: ReplicaSet

metadata:
  name: myrs

spec:
  replicas: 2

  selector:
    matchLabels:
      app: mobile

  template:
    metadata:
      labels:
        app: mobile

    spec:
      containers:
      - name: mob-cont
        image: aarushtechnologies/app

        ports:
        - containerPort: 80
```

---

# Understanding the YAML

## API Version

```yaml
apiVersion: apps/v1
```

ReplicaSet belongs to the apps API group.

---

## Kind

```yaml
kind: ReplicaSet
```

Creates a ReplicaSet object.

---

## Replicas

```yaml
replicas: 2
```

Maintain two Pods.

---

## Selector

```yaml
selector:
  matchLabels:
    app: mobile
```

ReplicaSet manages Pods having:

```yaml
app: mobile
```

---

## Template

Defines the Pod specification.

```yaml
template:
```

Every Pod created by ReplicaSet follows this template.

---

# Step 2: Create ReplicaSet

Execute:

```bash
kubectl create -f my-rs.yml
```

Expected Output:

```text
replicaset.apps/myrs created
```

---

# Step 3: Verify ReplicaSet

Check:

```bash
kubectl get rs
```

Expected:

```text
NAME   DESIRED   CURRENT   READY
myrs   2         2         2
```

Explanation:

```text
DESIRED = Required Pods
CURRENT = Existing Pods
READY   = Healthy Pods
```

---

# Step 4: Verify Pods

Run:

```bash
kubectl get pods
```

Example:

```text
myrs-jx5kt
myrs-lf7hp
```

Both should be:

```text
Running
```

---

# Self-Healing Demonstration

ReplicaSet automatically recreates deleted Pods.

---

## Step 1: View Pods

```bash
kubectl get pods
```

Example:

```text
myrs-jx5kt
myrs-lf7hp
```

---

## Step 2: Delete One Pod

```bash
kubectl delete pod myrs-jx5kt
```

Expected:

```text
pod "myrs-jx5kt" deleted
```

---

## Step 3: Verify Again

```bash
kubectl get pods
```

Example:

```text
myrs-lf7hp
myrs-r8w4t
```

Notice:

```text
Deleted Pod Recreated Automatically
```

This proves ReplicaSet provides:

```text
Self-Healing
```

---

# Exposing ReplicaSet Using LoadBalancer Service

Pods created by ReplicaSet need a Service for user access.

---

# Step 1: Create Service YAML

## File: myrs-svc.yml

```yaml
apiVersion: v1
kind: Service

metadata:
  name: mobile-svc

spec:
  type: LoadBalancer

  selector:
    app: mobile

  ports:
  - port: 80
    targetPort: 80
```

---

# Understanding Service YAML

Selector:

```yaml
selector:
  app: mobile
```

Matches ReplicaSet Pods.

Because ReplicaSet Pods contain:

```yaml
labels:
  app: mobile
```

Traffic automatically reaches all Pods.

---

# Step 2: Create Service

```bash
kubectl create -f myrs-svc.yml
```

Expected:

```text
service/mobile-svc created
```

---

# Step 3: Verify Service

```bash
kubectl get svc
```

Example:

```text
NAME         TYPE           CLUSTER-IP
mobile-svc   LoadBalancer   100.69.20.84
```

Initially:

```text
EXTERNAL-IP
<pending>
```

Wait 2–5 minutes.

---

# Step 4: Verify LoadBalancer

Run:

```bash
kubectl get svc
```

Example:

```text
mobile-svc
LoadBalancer
100.69.20.84

a177146dd8fd8432cb4a87af79.ap-south-1.elb.amazonaws.com
```

---

# Step 5: Verify on AWS

Navigate:

```text
AWS Console
→ EC2
→ Load Balancers
```

You should see a newly created AWS Load Balancer.

---

# Step 6: Test Application

Copy:

```text
EXTERNAL-IP
```

Example:

```text
a177146dd8fd8432cb4a87af79.ap-south-1.elb.amazonaws.com
```

Open Browser:

```text
http://a177146dd8fd8432cb4a87af79.ap-south-1.elb.amazonaws.com
```

Expected:

```text
Application Homepage
```

---

# Scaling ReplicaSet

ReplicaSet supports horizontal scaling.

Scaling means increasing or decreasing Pod count.

---

# Scale Up

Current:

```text
2 Pods
```

Increase to:

```text
5 Pods
```

Command:

```bash
kubectl scale rs myrs --replicas=5
```

Verify:

```bash
kubectl get rs
```

Expected:

```text
NAME   DESIRED   CURRENT   READY
myrs   5         5         5
```

Check Pods:

```bash
kubectl get pods
```

You should now see:

```text
5 Running Pods
```

---

# Scale Down

Current:

```text
5 Pods
```

Reduce to:

```text
2 Pods
```

Command:

```bash
kubectl scale rs myrs --replicas=2
```

Verify:

```bash
kubectl get pods
```

Expected:

```text
2 Running Pods
```

---

# Generic Scaling Command

```bash
kubectl scale rs <replicaset-name> --replicas=<number>
```

Example:

```bash
kubectl scale rs myrs --replicas=10
```

---

# Cascade Deletion Demonstration

One important concept practiced in the lab:

---

## Normal Deletion

```bash
kubectl delete rs myrs
```

Result:

```text
ReplicaSet Deleted
Pods Deleted
```

---

## Orphan Deletion

```bash
kubectl delete replicaset myrs --cascade=orphan
```

Result:

```text
ReplicaSet Deleted
Pods Remain Running
```

This is useful when you want to remove ReplicaSet ownership but keep Pods alive.

---

# Useful Commands

View ReplicaSets:

```bash
kubectl get rs
```

---

View Pods:

```bash
kubectl get pods
```

---

View Services:

```bash
kubectl get svc
```

---

Detailed ReplicaSet Information:

```bash
kubectl describe rs myrs
```

---

View Endpoints:

```bash
kubectl get endpoints
```

---

List Kubernetes Resources:

```bash
kubectl api-resources
```

---

# Cleanup

Delete Service:

```bash
kubectl delete -f myrs-svc.yml
```

---

Delete ReplicaSet:

```bash
kubectl delete -f my-rs.yml
```

OR

```bash
kubectl delete rs myrs
```

---

Verify:

```bash
kubectl get rs

kubectl get pods

kubectl get svc
```

Everything should be removed.

---

# Important Notes

Modern Kubernetes architecture:

```text
Deployment
      |
      v
ReplicaSet
      |
      v
Pods
```

In real-world environments, engineers rarely create ReplicaSets directly.

Instead:

```text
Deployment creates and manages ReplicaSets automatically.
```

However, understanding ReplicaSet is essential because Deployment works internally through ReplicaSets.

---

# Interview Questions

### What is ReplicaSet?

A Kubernetes controller that maintains a desired number of Pod replicas.

---

### What is the main purpose of ReplicaSet?

Self-healing and maintaining Pod availability.

---

### How is ReplicaSet different from ReplicationController?

ReplicaSet supports advanced label selectors and is used by Deployments.

---

### How do you scale a ReplicaSet?

```bash
kubectl scale rs myrs --replicas=5
```

---

### Which field identifies Pods managed by ReplicaSet?

```yaml
selector:
```

---

### Which field defines Pod configuration?

```yaml
template:
```

---

### What happens if a Pod managed by ReplicaSet is deleted?

ReplicaSet automatically creates a replacement Pod.

---

### What is the relationship between Deployment and ReplicaSet?

Deployment manages ReplicaSets, and ReplicaSets manage Pods.

---

# Summary

```text
ReplicaSet
      |
      +--> Maintains Desired Number of Pods
      |
      +--> Provides Self-Healing
      |
      +--> Supports Scaling
      |
      +--> Improves High Availability
      |
      +--> Used Internally By Deployments
```

## Workflow

```text
Create ReplicaSet
        |
        v
ReplicaSet Creates Pods
        |
        v
Create Service
        |
        v
AWS LoadBalancer Created
        |
        v
Application Accessible
        |
        v
Scale Pods Up/Down
        |
        v
ReplicaSet Maintains Desired State
```
