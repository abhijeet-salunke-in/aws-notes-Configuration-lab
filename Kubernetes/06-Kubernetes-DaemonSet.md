# Kubernetes DaemonSet

## Overview

So far in Kubernetes we have learned:

```text
Pod
 ↓
ReplicationController
 ↓
ReplicaSet
 ↓
Deployment
```

Each Kubernetes object was introduced to solve a specific problem.

For example:

```text
Pod
 ↓
Runs Containers

ReplicaSet
 ↓
Maintains Desired Number Of Pods

Deployment
 ↓
Provides Updates, Rollbacks, Revisions
```

However, there are certain workloads where we do not want:

```text
Fixed Number Of Pods
```

Instead, we want:

```text
One Pod Running On Every Node
```

This requirement is solved using:

```text
DaemonSet
```

---

# Problem Statement

Suppose we have a cluster:

```text
Cluster
│
├── Control Plane
├── Worker Node-1
└── Worker Node-2
```

Now imagine we need:

```text
Monitoring Agent

OR

Log Collection Agent

OR

Security Agent
```

to run on every worker node.

Examples:

```text
Prometheus Node Exporter
Fluentd
Filebeat
Datadog Agent
Falco
```

These applications must run on every node because they collect:

```text
Logs
Metrics
Security Events
System Information
```

from the node itself.

---

# Why Deployment Is Not Suitable

Suppose we use Deployment:

```yaml
replicas: 2
```

Deployment only guarantees:

```text
Two Pods Running Somewhere In Cluster
```

Possible result:

```text
Worker-1 → Pod
Worker-1 → Pod

Worker-2 → No Pod
```

This is acceptable for applications.

But not acceptable for:

```text
Monitoring Agents
Log Collectors
Security Agents
```

because every node must be monitored.

---

# Kubernetes Solution

Kubernetes introduced:

```text
DaemonSet
```

DaemonSet guarantees:

```text
One Pod Per Node
```

---

# What Is DaemonSet?

DaemonSet is a Kubernetes workload object that ensures one Pod runs on every node in a cluster.

Whenever:

```text
New Node Added
```

DaemonSet automatically creates a Pod on that node.

Whenever:

```text
Node Removed
```

DaemonSet automatically removes the corresponding Pod.

---

# DaemonSet Architecture

Example Cluster:

```text
1 Control Plane
2 Worker Nodes
```

Architecture:

```text
DaemonSet
     │
     ├─────────────┐
     │             │
     ▼             ▼

Worker-1      Worker-2
    │              │
    ▼              ▼

Daemon Pod    Daemon Pod
```

Result:

```text
One Pod On Every Worker Node
```

---

# Problems Solved By DaemonSet

## 1. Automatic Node Coverage

Without DaemonSet:

```text
Manually Create Pod On Every Node
```

With DaemonSet:

```text
Automatically Create Pod On Every Node
```

---

## 2. Self-Healing

Suppose:

```text
Daemon Pod Deleted
```

DaemonSet detects:

```text
Desired = 1 Pod On Node

Current = 0 Pods On Node
```

DaemonSet automatically creates:

```text
New Pod
```

---

## 3. Automatic Node Scaling

Suppose cluster grows:

```text
Worker-1
Worker-2
```

to

```text
Worker-1
Worker-2
Worker-3
```

DaemonSet automatically creates:

```text
Daemon Pod On Worker-3
```

No manual action required.

---

## 4. Monitoring

Examples:

```text
Prometheus Node Exporter
Datadog Agent
```

Every node sends metrics.

---

## 5. Log Collection

Examples:

```text
Fluentd
Filebeat
```

Every node sends logs.

---

## 6. Security Monitoring

Examples:

```text
Falco
```

Every node reports security events.

---

# Practical Lab Performed

Cluster Initially:

```text
1 Control Plane
1 Worker Node
```

A new worker node was added.

Final Cluster:

```text
1 Control Plane
2 Worker Nodes
```

Then a DaemonSet was deployed.

Observation:

```text
DaemonSet Automatically Created
One Pod On Each Worker Node
```

---

# Step 1: Configure AWS

```bash
aws configure
```

Provide:

```text
AWS Access Key
AWS Secret Key
Region
Output Format
```

Verify:

```bash
aws sts get-caller-identity
```

---

# Step 2: Configure KOPS State Store

```bash
export KOPS_STATE_STORE=s3://abhis.kops.v1
```

Verify:

```bash
echo $KOPS_STATE_STORE
```

Expected:

```text
s3://abhis.kops.v1
```

---

# Step 3: Check Instance Groups

```bash
kops get ig --name=abhi.k8s.local
```

Example Output:

```text
NAME
master-ap-south-1a
nodes-ap-south-1a
```

---

# Step 4: Add Worker Node

Edit worker node instance group:

```bash
kops edit ig nodes-ap-south-1a
```

---

Current:

```yaml
minSize: 1
maxSize: 1
```

Change to:

```yaml
minSize: 2
maxSize: 2
```

Save and Exit.

---

# Step 5: Update Cluster

```bash
kops update cluster --yes
```

Expected:

```text
Cluster Updated
```

---

# Step 6: Apply Rolling Update

```bash
kops rolling-update cluster --yes
```

This creates the new worker node.

---

# Step 7: Verify Nodes

```bash
kubectl get nodes
```

Example:

```text
NAME               STATUS

control-plane      Ready

worker-node-1      Ready

worker-node-2      Ready
```

---

# Step 8: Create Namespace

File:

```yaml
apiVersion: v1
kind: Namespace

metadata:
  name: devops
```

Create:

```bash
kubectl create -f namespace.yml
```

Verify:

```bash
kubectl get ns
```

---

# Step 9: Create DaemonSet YAML

File:

```yaml
apiVersion: apps/v1
kind: DaemonSet

metadata:
  name: daemon-port-deploy

spec:

  selector:
    matchLabels:
      app: portfolio

  template:
    metadata:
      labels:
        app: portfolio

    spec:
      containers:
      - name: port-cont
        image: aarushtechnologies/portfolio:v1

        ports:
        - containerPort: 80
```

---

# Understanding The YAML

## apiVersion

```yaml
apiVersion: apps/v1
```

DaemonSet belongs to Apps API Group.

---

## kind

```yaml
kind: DaemonSet
```

Creates DaemonSet Object.

---

## selector

```yaml
selector:
  matchLabels:
    app: portfolio
```

Used to identify Pods managed by DaemonSet.

---

## template

```yaml
template:
```

Defines Pod Blueprint.

Contains:

```text
Labels
Container Name
Image
Port
```

---

## image

```yaml
image: aarushtechnologies/portfolio:v1
```

Container image used by all DaemonSet Pods.

---

# Step 10: Create DaemonSet

```bash
kubectl create -f Daemon-demo.yml
```

Expected:

```text
daemonset.apps/daemon-port-deploy created
```

---

# Step 11: Verify DaemonSet

```bash
kubectl get ds
```

Example:

```text
NAME                 DESIRED CURRENT READY

daemon-port-deploy      2       2      2
```

Meaning:

```text
DESIRED = Required Pods

CURRENT = Existing Pods

READY = Healthy Pods
```

---

# Step 12: Verify Pods

```bash
kubectl get pods
```

Example:

```text
daemon-port-deploy-rnlnz

daemon-port-deploy-t78z9
```

---

# Step 13: Verify Pod Placement

```bash
kubectl get pods -o wide
```

Example:

```text
NAME                      NODE

daemon-port-deploy-rnlnz  worker-node-1

daemon-port-deploy-t78z9  worker-node-2
```

Observation:

```text
One Pod Running On Each Worker Node
```

---

# Step 14: Verify Nodes

```bash
kubectl get nodes
```

Example:

```text
control-plane

worker-node-1

worker-node-2
```

---

# Self-Healing Demonstration

View Pods:

```bash
kubectl get pods
```

Delete One Pod:

```bash
kubectl delete pod daemon-port-deploy-rnlnz
```

Verify Again:

```bash
kubectl get pods
```

Result:

```text
New Pod Automatically Created
```

Reason:

```text
DaemonSet Maintains
One Pod Per Node
```

---

# Automatic Node Addition Demonstration

Current Cluster:

```text
Worker-1
Worker-2
```

Add New Worker Node.

Verify:

```bash
kubectl get nodes
```

Now:

```text
Worker-1
Worker-2
Worker-3
```

Check Pods:

```bash
kubectl get pods -o wide
```

Result:

```text
Worker-1 → Pod

Worker-2 → Pod

Worker-3 → Pod
```

Observation:

```text
DaemonSet Automatically Created
Pod On New Node
```

---

# Real-World Use Cases

## Fluentd

```text
Collect Logs From Every Node
```

---

## Filebeat

```text
Ship Logs To Central Server
```

---

## Prometheus Node Exporter

```text
Collect CPU
Memory
Disk Metrics
```

---

## Datadog Agent

```text
Infrastructure Monitoring
```

---

## Falco

```text
Runtime Security Monitoring
```

---

# Deployment vs ReplicaSet vs DaemonSet

| Feature             | ReplicaSet | Deployment | DaemonSet  |
| ------------------- | ---------- | ---------- | ---------- |
| Self-Healing        | Yes        | Yes        | Yes        |
| Fixed Replicas      | Yes        | Yes        | No         |
| Rolling Updates     | No         | Yes        | Limited    |
| Rollback            | No         | Yes        | No         |
| One Pod Per Node    | No         | No         | Yes        |
| Application Hosting | Yes        | Yes        | Usually No |
| Monitoring Agents   | No         | No         | Yes        |

---

# Useful Commands

View DaemonSets:

```bash
kubectl get ds
```

Describe DaemonSet:

```bash
kubectl describe ds daemon-port-deploy
```

View Pods:

```bash
kubectl get pods
```

View Pod Placement:

```bash
kubectl get pods -o wide
```

View Nodes:

```bash
kubectl get nodes
```

Delete DaemonSet:

```bash
kubectl delete ds daemon-port-deploy
```

OR

```bash
kubectl delete -f Daemon-demo.yml
```

---

# Interview Questions

### What is a DaemonSet?

A Kubernetes object that ensures one Pod runs on every node.

---

### Why is DaemonSet used?

To automatically run Pods on all nodes.

---

### Does DaemonSet require replicas?

```text
No
```

DaemonSet automatically calculates Pod count based on node count.

---

### What happens when a new node joins the cluster?

DaemonSet automatically creates a Pod on that node.

---

### What happens if a DaemonSet Pod is deleted?

DaemonSet automatically recreates it.

---

### Main Use Cases?

```text
Monitoring Agents

Log Collection Agents

Security Agents
```

---

# Summary

```text
DaemonSet
     │
     ├── One Pod Per Node
     ├── Self-Healing
     ├── Automatic Node Coverage
     ├── Monitoring Agents
     ├── Log Collection
     └── Security Monitoring
```

## Workflow

```text
Create DaemonSet
        │
        ▼
Pod Created On Every Node
        │
        ▼
New Node Added
        │
        ▼
New Pod Created Automatically
        │
        ▼
Pod Deleted
        │
        ▼
DaemonSet Recreates Pod
        │
        ▼
Cluster Remains Consistent
```

This version is GitHub-ready, matches your actual class practice, and is detailed enough for revision, interviews, and future reference.
