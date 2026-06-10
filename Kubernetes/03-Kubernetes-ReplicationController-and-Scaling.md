# Kubernetes ReplicationController (RC) and Scaling

## Introduction

When learning Kubernetes, most beginners start by creating Pods directly.

A Pod can run an application successfully, but running standalone Pods creates several operational problems.

To solve these problems, Kubernetes introduced **ReplicationController (RC)**.

ReplicationController was one of the first Kubernetes controllers designed to provide:

- Self-Healing
- High Availability
- Scaling
- Desired State Management

Although modern Kubernetes uses Deployments and ReplicaSets, understanding ReplicationController helps build a strong foundation for understanding how Kubernetes evolved.

---

# Problem with Standalone Pods

Suppose Kubernetes only had Pods.

```text
Pod
 └── nginx
```

The application runs successfully.

However, if the Pod crashes:

```text
Pod ❌
Application ❌
```

Nobody automatically recreates it.

Example:

```bash
kubectl delete pod nginx-pod
```

Result:

```text
Pod deleted permanently
Application unavailable
```

---

# Problems with Standalone Pods

## Problem 1: No Self-Healing

If a Pod crashes:

```text
Pod Crash
    ↓
Application Down
```

The administrator must manually create another Pod.

---

## Problem 2: No High Availability

Suppose the application runs inside a single Pod:

```text
1 Pod
```

If that Pod fails:

```text
Application Down
```

This creates a single point of failure.

---

## Problem 3: Manual Scaling

Suppose traffic increases and you need:

```text
10 Pods
```

You must manually create:

```text
Pod1
Pod2
Pod3
...
Pod10
```

Managing this manually becomes difficult.

---

# Why Kubernetes Introduced ReplicationController

Kubernetes needed a mechanism that could:

- Automatically recreate failed Pods
- Maintain a fixed number of Pods
- Support scaling
- Improve application availability

To solve these problems, Kubernetes introduced:

```text
ReplicationController (RC)
```

Think of RC as a security guard for Pods.

You tell RC:

```yaml
replicas: 3
```

RC continuously checks:

```text
Desired Pods = 3
Actual Pods = ?
```

If actual Pods become less than desired Pods, RC immediately creates new Pods.

---

# What is a ReplicationController?

A ReplicationController is a Kubernetes controller that ensures a specified number of Pod replicas are always running.

Example:

Desired:

```text
3 Pods
```

Current:

```text
Pod-1 Running
Pod-2 Running
Pod-3 Running
```

Everything is healthy.

---

Suppose Pod-2 crashes:

```text
Pod-1 Running
Pod-2 Deleted
Pod-3 Running
```

Now:

```text
Desired = 3
Actual = 2
```

RC automatically creates:

```text
Pod-4 Running
```

Result:

```text
3 Pods Running Again
```

---

# How ReplicationController Works Internally

RC continuously compares:

```text
Desired State
```

with

```text
Actual State
```

Example:

```text
Desired Pods = 3
Actual Pods = 2
```

RC detects the mismatch.

Then:

```text
Creates 1 New Pod
```

Result:

```text
Desired Pods = 3
Actual Pods = 3
```

This concept is called:

```text
Desired State Management
```

---

# Benefits of ReplicationController

## Self-Healing

```text
Pod Dies
    ↓
RC Creates New Pod
```

---

## High Availability

Instead of:

```text
1 Pod
```

You can run:

```text
3 Pods
```

Application remains available even if one Pod fails.

---

## Scaling

Need more Pods?

Change:

```yaml
replicas: 3
```

to

```yaml
replicas: 10
```

RC creates additional Pods.

---

## Fault Tolerance

Multiple Pod replicas improve application reliability.

---

# ReplicationController Architecture

```text
ReplicationController
        |
        |
        +----------------+
        |       |        |
        v       v        v
      Pod1    Pod2    Pod3
```

RC continuously ensures all desired Pods exist.

---

# Components of ReplicationController

## replicas

Defines how many Pods should run.

Example:

```yaml
replicas: 3
```

---

## selector

Used to identify Pods managed by RC.

Example:

```yaml
selector:
  app: mobile
```

RC manages Pods matching:

```yaml
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

Every Pod created by RC follows this template.

---

# Practical Lab

## Objective

Create:

```text
ReplicationController
        |
        +--> 3 Pods
```

and expose the application using a LoadBalancer Service.

---

# Step 1: Create ReplicationController YAML

## File: my-rc.yml

```yaml
apiVersion: v1
kind: ReplicationController

metadata:
  name: myrc

spec:
  replicas: 3

  selector:
    app: mobile

  template:
    metadata:
      labels:
        app: mobile

    spec:
      containers:
      - name: mobile-container
        image: aarushtechnologies/app

        ports:
        - containerPort: 80
```

---

# YAML Explanation

## API Version

```yaml
apiVersion: v1
```

Uses Kubernetes core API.

---

## Kind

```yaml
kind: ReplicationController
```

Creates a ReplicationController object.

---

## Metadata

```yaml
metadata:
  name: myrc
```

RC name:

```text
myrc
```

---

## Replicas

```yaml
replicas: 3
```

Maintain exactly 3 Pods.

---

## Selector

```yaml
selector:
  app: mobile
```

Identifies Pods managed by RC.

---

## Template

Defines Pod configuration.

Every Pod created by RC uses this template.

---

# Step 2: Create ReplicationController

```bash
kubectl create -f my-rc.yml
```

Expected:

```text
replicationcontroller/myrc created
```

---

# Step 3: Verify ReplicationController

```bash
kubectl get rc
```

Expected:

```text
NAME   DESIRED   CURRENT   READY
myrc   3         3         3
```

Meaning:

```text
DESIRED = Required Pods
CURRENT = Existing Pods
READY = Healthy Pods
```

---

# Step 4: Verify Pods

```bash
kubectl get pods
```

Example:

```text
myrc-4zmqf
myrc-7js84
myrc-ttscq
```

All Pods should be:

```text
Running
```

---

# Self-Healing Demonstration

## Step 1: View Existing Pods

```bash
kubectl get pods
```

---

## Step 2: Delete One Pod

Example:

```bash
kubectl delete pod myrc-4zmqf
```

Expected:

```text
pod deleted
```

---

## Step 3: Verify Again

```bash
kubectl get pods
```

Example:

```text
myrc-7js84
myrc-ttscq
myrc-kd9w2
```

Notice:

```text
Deleted Pod Recreated Automatically
```

This proves:

```text
Self-Healing
```

---

# Exposing RC Using LoadBalancer Service

Users cannot directly access Pods reliably.

A Service provides a stable access point.

---

# Step 1: Create Service YAML

## File: mobile-svc.yml

```yaml
apiVersion: v1
kind: Service

metadata:
  name: mobile-service

spec:
  type: LoadBalancer

  selector:
    app: mobile

  ports:
  - port: 80
    targetPort: 80
```

---

# Service YAML Explanation

## Type

```yaml
type: LoadBalancer
```

Creates an AWS Elastic Load Balancer.

---

## Selector

```yaml
selector:
  app: mobile
```

Matches Pods created by RC.

---

## Port Mapping

```yaml
port: 80
targetPort: 80
```

Traffic received on port 80 is forwarded to container port 80.

---

# Step 2: Create Service

```bash
kubectl create -f mobile-svc.yml
```

Expected:

```text
service/mobile-service created
```

---

# Step 3: Verify Service

```bash
kubectl get svc
```

Example:

```text
NAME             TYPE           CLUSTER-IP
mobile-service   LoadBalancer   100.69.20.84
```

Initially:

```text
EXTERNAL-IP
<pending>
```

Wait a few minutes.

---

# Step 4: Verify AWS Load Balancer

```bash
kubectl get svc
```

Example:

```text
mobile-service

LoadBalancer

100.69.20.84

a177146dd8fd8432cb4a87af79.ap-south-1.elb.amazonaws.com
```

AWS automatically creates:

```text
Elastic Load Balancer (ELB)
```

Verify:

```text
AWS Console
→ EC2
→ Load Balancers
```

---

# Step 5: Test Application

Copy:

```text
EXTERNAL-IP / ELB DNS
```

Example:

```text
a177146dd8fd8432cb4a87af79.ap-south-1.elb.amazonaws.com
```

Open:

```text
http://a177146dd8fd8432cb4a87af79.ap-south-1.elb.amazonaws.com
```

Expected:

```text
Application Homepage Loads
```

---

# Scaling ReplicationController

Scaling means increasing or decreasing the number of Pods.

---

# Scale Up

Current:

```text
3 Pods
```

Increase to:

```text
5 Pods
```

Command:

```bash
kubectl scale rc myrc --replicas=5
```

Verify:

```bash
kubectl get rc
```

Expected:

```text
NAME   DESIRED   CURRENT   READY
myrc   5         5         5
```

Check Pods:

```bash
kubectl get pods
```

You should see:

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
kubectl scale rc myrc --replicas=2
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
kubectl scale rc <rc-name> --replicas=<number>
```

Example:

```bash
kubectl scale rc myrc --replicas=10
```

---

# Useful Commands

View RC:

```bash
kubectl get rc
```

---

Describe RC:

```bash
kubectl describe rc myrc
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

View Endpoints:

```bash
kubectl get endpoints
```

---

# Limitations of ReplicationController

RC supports only simple selectors.

Example:

```yaml
selector:
  app: mobile
```

This means:

```text
app = mobile
```

Only exact matching is supported.

RC cannot perform:

```text
app = nginx OR apache
```

or

```text
app != mysql
```

Large applications require more advanced selection logic.

This limitation led to the introduction of:

```text
ReplicaSet
```

---

# Evolution of Kubernetes Controllers

```text
Pod
│
├─ Problem:
│   No self-healing
│   No scaling
│
▼

ReplicationController
│
├─ Solves:
│   Self-healing
│   Scaling
│
├─ Problem:
│   Simple selectors
│
▼

ReplicaSet
│
├─ Solves:
│   Better selectors
│   Self-healing
│   Scaling
│
├─ Problem:
│   No rolling updates
│   No rollback
│
▼

Deployment
│
├─ Solves:
│   Rolling updates
│   Rollbacks
│   Version history
│
▼

Modern Kubernetes
```

---

# Cleanup

Delete Service:

```bash
kubectl delete -f mobile-svc.yml
```

Delete RC:

```bash
kubectl delete -f my-rc.yml
```

OR

```bash
kubectl delete rc myrc
```

Verify:

```bash
kubectl get rc
kubectl get pods
kubectl get svc
```

---

# Interview Questions

### What is a ReplicationController?

A Kubernetes controller that ensures a specified number of Pod replicas are always running.

---

### What problem does RC solve?

- Pod failures
- Manual scaling
- High availability

---

### What is Self-Healing?

Automatic recreation of failed Pods.

---

### What is the purpose of replicas?

Defines how many Pods should run.

---

### What happens if a Pod managed by RC is deleted?

RC automatically creates a replacement Pod.

---

### How do you scale RC?

```bash
kubectl scale rc myrc --replicas=5
```

---

### Why was ReplicaSet introduced?

To provide advanced label selector capabilities.

---

# Summary

```text
ReplicationController
        |
        +--> Maintains Desired Number of Pods
        |
        +--> Provides Self-Healing
        |
        +--> Supports Scaling
        |
        +--> Improves High Availability
        |
        +--> Maintains Desired State
```

## Workflow

```text
Create RC
     |
     v
RC Creates Pods
     |
     v
Service Created
     |
     v
AWS Load Balancer Created
     |
     v
Application Accessible
     |
     v
Scale Pods Up/Down
     |
     v
RC Maintains Desired State
```
