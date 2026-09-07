# Day 41 — Kubernetes Architecture & Multi-Node kind Cluster

## 1. Objective

The objective of Day 41 was to understand **Kubernetes architecture by actually bringing up, inspecting, observing, and breaking a Kubernetes cluster**.

The focus was not simply on learning Kubernetes commands. The goal was to understand the internal flow of Kubernetes:

```text
Desired State
     ↓
kube-apiserver
     ↓
etcd
     ↓
Controllers
     ↓
Scheduler
     ↓
Kubelet
     ↓
Container Runtime
     ↓
Running Workload
     ↓
Observed Status
     ↓
Reconciliation
     ↺
```

The central mental model for the day was:

> **Kubernetes continuously reconciles the actual state of the cluster toward the desired state.**

Unlike an imperative deployment script that runs once and stops, Kubernetes continuously watches and corrects the cluster.

---

# 2. Why Kubernetes?

Before using Kubernetes, it is important to understand what Docker Compose could not provide.

Docker Compose was useful for running multiple containers on a single machine:

```text
Docker Host
   ├── Container A
   ├── Container B
   └── Container C
```

However, Compose does not provide the same cluster-level orchestration capabilities as Kubernetes.

| Capability                      | Docker Compose | Kubernetes   |
| ------------------------------- | -------------- | ------------ |
| Run containers                  | Yes            | Yes          |
| Multiple machines               | No             | Yes          |
| Cluster scheduling              | No             | Yes          |
| Self-healing                    | Limited        | Yes          |
| Declarative desired state       | Limited        | Core feature |
| Continuous reconciliation       | No             | Yes          |
| Rolling deployments             | Limited        | Yes          |
| Node failure handling           | No             | Yes          |
| Automatic workload rescheduling | No             | Yes          |

The key difference is:

```text
Docker Compose:

Run commands
     ↓
Containers start
     ↓
Done
```

Kubernetes:

```text
Declare desired state
       ↓
Kubernetes observes actual state
       ↓
Compare desired vs actual
       ↓
Take corrective action
       ↓
Observe again
       ↓
Repeat forever
```

This is called **reconciliation**.

---

# 3. Kubernetes Is More Than a Container Runner

Kubernetes is not simply a more advanced `docker run`.

Kubernetes is an **API-driven orchestration system composed of an API server and multiple controllers/components**.

The container runtime is responsible for actually running containers.

The relationship is approximately:

```text
Kubernetes
     ↓
kubelet
     ↓
CRI
     ↓
containerd / CRI-O
     ↓
Container
```

Therefore:

```text
Kubernetes = orchestration/control
containerd = container execution
```

---

# 4. kind Multi-Node Cluster

For today's practical work, a local Kubernetes cluster was created using **kind**.

The cluster contained:

```text
1 Control Plane
2 Workers
```

The configuration was:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4

nodes:
  - role: control-plane
  - role: worker
  - role: worker
```

The cluster was created using:

```bash
kind create cluster --name day41 --config kind-config.yaml
```

This produced:

```text
day41-control-plane
day41-worker
day41-worker2
```

---

# 5. Important kind Architecture Observation

One of the most important observations of the day came from:

```bash
docker ps
```

The Kubernetes nodes themselves appeared as Docker/OrbStack containers.

The architecture therefore looked like:

```text
macOS
  ↓
OrbStack
  ↓
kind node containers
  ├── day41-control-plane
  ├── day41-worker
  └── day41-worker2
       ↓
    Kubernetes
       ↓
   containerd
       ↓
   Kubernetes workloads
```

This means:

> **Containers are being used as Kubernetes nodes, and Kubernetes is running containers inside those nodes.**

This is one of the reasons kind is useful for learning Kubernetes.

---

# 6. Cluster Verification

The cluster was verified with:

```bash
kubectl cluster-info
```

Nodes were inspected using:

```bash
kubectl get nodes -o wide
```

The final cluster showed:

```text
NAME                   STATUS   ROLES
day41-control-plane    Ready    control-plane
day41-worker           Ready    <none>
day41-worker2          Ready    <none>
```

The final runtime information showed:

```text
CONTAINER-RUNTIME
containerd://2.3.4
```

This confirmed that kubelet was using **containerd** as the container runtime.

The Kubernetes version was:

```text
v1.37.0
```

---

# 7. kubeconfig and Contexts

Kubernetes uses a kubeconfig file to determine:

* Which cluster to connect to
* Which user/credentials to use
* Which context is currently active
* Which namespace is associated with a context

Commands used:

```bash
kubectl config view
```

```bash
kubectl config get-contexts
```

```bash
kubectl config current-context
```

A context is conceptually:

```text
Context
 ├── Cluster
 ├── User
 └── Namespace
```

The important operational lesson was:

> **Always check the current Kubernetes context before executing destructive commands.**

For example:

```bash
kubectl config current-context
```

should become a normal safety check before:

```bash
kubectl delete ...
kubectl scale ...
kubectl apply ...
```

This becomes especially important when working with:

```text
local
stage
production
EKS
```

clusters.

---

# 8. Kubernetes Control Plane

The control plane contains several major components.

```text
Control Plane
 ├── kube-apiserver
 ├── etcd
 ├── kube-scheduler
 ├── kube-controller-manager
 └── cloud-controller-manager
```

Each component has a different responsibility.

---

# 9. kube-apiserver

The **kube-apiserver** is the front door to Kubernetes.

Almost all Kubernetes operations go through it.

The conceptual flow is:

```text
Client
  ↓
kube-apiserver
  ↓
Authentication
  ↓
Authorization
  ↓
Admission
  ↓
Validation
  ↓
Persist state
```

For example:

```bash
kubectl apply -f deployment.yaml
```

does not directly start a container.

Instead:

```text
kubectl
   ↓
kube-apiserver
   ↓
request processing
   ↓
etcd
```

The API server is therefore the central API interface.

Important distinction:

> **The API server does not directly run containers.**

If the API server goes down, existing workloads may continue running, but Kubernetes API-driven changes cannot normally be made.

---

# 10. etcd

`etcd` is Kubernetes' persistent distributed key-value database.

It stores cluster state such as:

```text
Pods
Deployments
ReplicaSets
Services
Nodes
ConfigMaps
Secrets
RBAC objects
and other Kubernetes resources
```

The conceptual relationship is:

```text
kubectl
   ↓
kube-apiserver
   ↓
etcd
```

The API server is the component that interacts with etcd.

etcd uses **Raft consensus**.

Production etcd clusters commonly use an odd number of members such as:

```text
3 members
5 members
```

because quorum is required.

For example:

```text
3 members → 2 required for quorum
5 members → 3 required for quorum
```

The key operational lesson is:

> **etcd contains the persistent Kubernetes cluster state, so etcd backups are a major disaster-recovery concern.**

---

# 11. etcd Inspection

The etcd Pod was inspected from the `kube-system` namespace.

The important objective was to see Kubernetes registry keys such as:

```text
/registry/pods/...
/registry/deployments/...
/registry/services/...
```

This made the statement:

> "etcd is the Kubernetes database"

concrete.

The inspection was performed read-only.

No data was modified or deleted from etcd.

---

# 12. kube-scheduler

The scheduler answers:

> **Which node should an unscheduled Pod run on?**

It does not start the container.

The scheduler conceptually works in two major stages.

### Filtering

It removes nodes that cannot run the Pod.

Examples:

```text
Insufficient CPU
Taints
Node selector mismatch
Affinity constraints
Other scheduling constraints
```

### Scoring

Among the remaining eligible nodes, the scheduler scores them and selects the best candidate.

Conceptually:

```text
Pod
 ↓
Filtering
 ↓
Eligible nodes
 ↓
Scoring
 ↓
Best node
 ↓
Pod bound to node
```

Important distinction:

```text
Scheduler = decides WHERE
Kubelet   = makes it RUN
```

Scheduler logs were inspected with:

```bash
kubectl logs -n kube-system kube-scheduler-day41-control-plane
```

---

# 13. kube-controller-manager

The controller manager contains multiple controllers.

Examples include:

```text
Deployment controller
ReplicaSet controller
Node controller
Job controller
```

Each controller follows the reconciliation pattern:

```text
Watch state
    ↓
Compare desired vs actual
    ↓
Take action
    ↓
Observe again
    ↓
Repeat forever
```

For example, if:

```text
Desired replicas = 5
Actual replicas  = 4
```

the appropriate controller works to restore:

```text
Desired replicas = 5
Actual replicas  = 5
```

---

# 14. cloud-controller-manager

The cloud controller manager contains cloud-specific control loops.

Depending on the cloud environment, these can include functionality related to:

```text
Load balancers
Volumes
Nodes
Cloud infrastructure
```

The important concept is:

```text
Kubernetes
    ↓
cloud-controller-manager
    ↓
Cloud provider
```

This becomes particularly important later when working with managed Kubernetes such as EKS.

---

# 15. Node Components

Each Kubernetes worker node has important node-level components:

```text
Worker Node
 ├── kubelet
 ├── container runtime
 ├── kube-proxy
 └── CNI
```

---

# 16. kubelet

The kubelet is the main Kubernetes agent running on a node.

Its responsibility is to ensure that Pods assigned to its node are actually running.

Conceptually:

```text
API server
   ↓
Pod assigned to Node
   ↓
kubelet
   ↓
container runtime
   ↓
container
```

The kubelet also:

* Reports Pod status
* Runs health probes
* Manages volumes
* Watches Pod assignments
* Works with the container runtime

An important distinction is:

> **kubelet manages Kubernetes workloads, not arbitrary containers manually started outside Kubernetes.**

---

# 17. Container Runtime

The cluster showed:

```text
containerd://2.3.4
```

The modern relationship is:

```text
kubelet
   ↓
CRI
   ↓
containerd
   ↓
OCI runtime
   ↓
container
```

Kubernetes does not require Docker Engine to run containers.

However, Docker images remain useful because they are generally distributed using OCI-compatible image formats.

Therefore:

> **Docker image knowledge from earlier days is still directly useful with Kubernetes.**

---

# 18. kube-proxy

`kube-proxy` handles the node-level networking rules required for Kubernetes Services.

A Service provides a stable virtual endpoint in front of Pods.

Conceptually:

```text
Service
   ↓
Pod A
Pod B
Pod C
```

kube-proxy programs packet-routing/load-balancing rules so Service traffic can reach the appropriate Pods.

It is important not to think of kube-proxy as an application-level proxy sitting in the middle of every request.

---

# 19. CNI

The CNI plugin provides Pod networking.

The Kubernetes networking model expects Pods to be able to communicate with other Pods without requiring NAT between Pods.

The kind cluster used:

```text
kindnet-cni
```

The conceptual separation is:

```text
CNI
 ↓
Pod networking

kube-proxy
 ↓
Service networking rules
```

CNI and kube-proxy were visible on each node because networking functionality is required wherever Pods can run.

---

# 20. Kubernetes System Pods

The following command was used:

```bash
kubectl get pods -n kube-system -o wide
```

This allowed the cluster components to be mapped to actual Pods.

Components observed included:

```text
kube-apiserver
etcd
kube-scheduler
kube-controller-manager
kube-proxy
kindnet-cni
CoreDNS
```

This connected the architecture diagram to actual processes running inside the cluster.

---

# 21. Static Pod Manifests

The control-plane node was inspected using:

```bash
docker exec -it day41-control-plane bash
```

Then:

```bash
ls /etc/kubernetes/manifests/
```

Control-plane static Pod manifests were visible, including components such as:

```text
etcd.yaml
kube-apiserver.yaml
kube-controller-manager.yaml
kube-scheduler.yaml
```

Static Pods are important during control-plane bootstrap.

The key idea is:

```text
Node starts
   ↓
kubelet starts
   ↓
kubelet reads static Pod manifests
   ↓
control-plane Pods start
   ↓
API server becomes available
   ↓
normal Kubernetes API operation
```

This answers an important bootstrap question:

> **How can the API server start if the API server itself doesn't exist yet?**

Static Pods are part of the solution in kubeadm-style control planes.

---

# 22. Kubernetes Object Structure

Kubernetes objects generally contain:

```yaml
apiVersion:
kind:
metadata:

spec:

status:
```

The important distinction is:

```text
spec
 ↓
desired state

status
 ↓
observed/current state
```

For example:

```yaml
spec:
  replicas: 5
```

means:

> "I want five replicas."

The controller updates status to reflect reality.

Therefore:

> **You declare `spec`; Kubernetes controllers maintain `status`.**

---

# 23. Namespaces

Namespaces provide logical scope within a cluster.

Examples:

```text
default
kube-system
monitoring
dev
stage
prod
```

Some resources are namespaced:

```text
Pods
Deployments
Services
ConfigMaps
Secrets
```

Other resources are cluster-scoped:

```text
Nodes
Namespaces
PersistentVolumes
ClusterRoles
```

Commands can target a specific namespace:

```bash
kubectl get pods -n kube-system
```

or all namespaces:

```bash
kubectl get pods -A
```

---

# 24. Labels and Selectors

Kubernetes components are loosely coupled using labels and selectors.

For example:

```yaml
labels:
  app: web
```

and:

```yaml
selector:
  matchLabels:
    app: web
```

Instead of referring to Pods by individual names, Kubernetes can select Pods based on labels.

Conceptually:

```text
Deployment
    ↓
label selector
    ↓
Pods matching app=web
```

This is one of the fundamental mechanisms connecting Kubernetes resources.

---

# 25. kubectl API Exploration

The Kubernetes API catalogue was explored using:

```bash
kubectl api-resources
```

This shows:

* Resource names
* Short names
* API versions
* Whether resources are namespaced
* Resource kinds

This demonstrated that Kubernetes is fundamentally an API containing many resource types.

---

# 26. kubectl explain

Instead of guessing YAML fields, Kubernetes provides built-in documentation.

Commands used:

```bash
kubectl explain pod
```

```bash
kubectl explain pod.spec
```

```bash
kubectl explain pod.spec.containers
```

```bash
kubectl explain pod.spec.containers.image
```

This should become a normal Kubernetes workflow:

> **Use `kubectl explain` instead of guessing API fields.**

---

# 27. Kubernetes API Is HTTP

The command:

```bash
kubectl get pods --v=8
```

was used to inspect the underlying API requests made by kubectl.

This demonstrates:

```text
kubectl
   ↓
HTTP request
   ↓
kube-apiserver
   ↓
Kubernetes API
```

`kubectl` is therefore a client of the Kubernetes API.

It is not Kubernetes itself.

---

# 28. Direct API Access

The API was also accessed without directly using kubectl for the request.

First:

```bash
kubectl proxy
```

Then:

```bash
curl localhost:8001/api/v1/namespaces/default/pods
```

This returned the Pods API as JSON.

Conceptually:

```text
curl
 ↓
HTTP
 ↓
Kubernetes API
 ↓
Pods endpoint
 ↓
JSON
```

This reinforced the idea:

> **Kubernetes is fundamentally an HTTP API with controllers and components operating around it.**

---

# 29. Deployment Creation

A Deployment was created with:

```bash
kubectl create deployment web --image=nginx --replicas=3
```

The resulting ownership chain was:

```text
Deployment/web
      ↓
ReplicaSet
      ↓
Pods
```

The relationship was inspected using:

```bash
kubectl get deployment,rs,pods
```

and:

```bash
kubectl describe rs <replicaset-name>
```

The Pod ownership was also inspected using:

```bash
kubectl describe pod <pod-name>
```

The resulting structure was:

```text
Deployment/web
      ↓
ReplicaSet/web-xxxxx
      ↓
Pod
Pod
Pod
```

---

# 30. Pod Lifecycle Observation

The Pod lifecycle was watched using:

```bash
kubectl get pods -w
```

Events were watched using:

```bash
kubectl get events -w
```

The lifecycle was observed conceptually as:

```text
Pod created
    ↓
Pending
    ↓
Scheduled
    ↓
ContainerCreating
    ↓
Running
```

The Pod was inspected using:

```bash
kubectl describe pod <pod-name>
```

Important information included:

```text
Node
Pod IP
Events
Container status
```

---

# 31. Matching Events to Components

Events such as:

```text
Scheduled
Pulling
Pulled
Created
Started
```

were used to connect actions to Kubernetes components.

The key mapping was:

```text
Scheduled
    ↓
Scheduler

Pulling / Created / Started
    ↓
Kubelet
```

This provides a useful debugging technique:

> **Kubernetes events show which stage of the workload lifecycle has progressed or stalled.**

---

# 32. Proving the Reconciliation Loop

This was the most important experiment of Day 41.

The Deployment initially had:

```text
replicas = 3
```

One Pod was deleted:

```bash
kubectl delete pod <pod-name>
```

The Pod disappeared.

Then Kubernetes created a replacement.

The important point was:

> **Nothing manually recreated the Pod.**

The ReplicaSet controller observed:

```text
Desired = 3
Actual = 2
```

and reconciled the difference:

```text
2 → 3
```

This experimentally demonstrated Kubernetes reconciliation.

---

# 33. Scaling the Deployment

The Deployment was then scaled:

```bash
kubectl scale deployment web --replicas=5
```

The Pods were watched using:

```bash
kubectl get pods -w
```

The system converged from:

```text
3 replicas
```

to:

```text
5 replicas
```

Again:

```text
Desired = 5
Actual = 3
```

then:

```text
Desired = 5
Actual = 5
```

This was another direct demonstration of reconciliation.

---

# 34. Killing a Container Behind Kubernetes' Back

The worker node was inspected using:

```bash
docker exec -it day41-worker bash
```

The container runtime was inspected using:

```bash
crictl ps
```

A container was then deliberately removed using its **container ID**:

```bash
crictl rm -f <container-id>
```

An important learning point occurred here.

`crictl ps` displays both:

```text
CONTAINER ID
```

and:

```text
POD ID
```

For `crictl rm`, the **container ID** must be used.

A Pod ID is not the same thing as a container ID.

After the container was removed, Kubernetes/kubelet worked to restore the container.

The Pod showed:

```text
RESTARTS = 1
```

This demonstrated another reconciliation loop:

```text
Container disappears
      ↓
kubelet notices
      ↓
kubelet works with runtime
      ↓
container restored
```

This is different from the ReplicaSet controller replacing an entire Pod.

---

# 35. Node Failure Experiment

The most important failure drill was simulating a worker-node failure.

Before failure, five nginx replicas existed across two workers.

The worker was identified using:

```bash
docker ps
```

Then the worker was stopped:

```bash
docker stop day41-worker
```

---

# 36. Node Became NotReady

The cluster was monitored using:

```bash
kubectl get nodes -w
```

The failed node eventually changed from:

```text
Ready
```

to:

```text
NotReady
```

The final state temporarily became:

```text
day41-control-plane   Ready
day41-worker          NotReady
day41-worker2         Ready
```

This demonstrated that a worker node can fail while the control plane remains operational.

---

# 37. Workload Recovery After Node Failure

Before the failure, three nginx Pods were running on:

```text
day41-worker
```

and two were running on:

```text
day41-worker2
```

After the worker failed, the Pods on the failed node became:

```text
Terminating
```

Kubernetes then created replacement Pods.

The replacement Pods were placed on:

```text
day41-worker2
```

The final workload state became:

```text
day41-worker2
 ├── web-7bv4k
 ├── web-99tg8
 ├── web-fq9w4
 ├── web-jfx56
 └── web-xdv44
```

All five were:

```text
1/1 Running
```

The Deployment showed:

```text
READY          5/5
UP-TO-DATE     5
AVAILABLE      5
```

This proved:

> **Kubernetes maintains the desired workload, not individual Pod identities.**

The original Pods on the failed node were not moved.

Instead, new Pods were created.

---

# 38. Pod Identity Is Disposable

The original Pods included:

```text
web-...-gn99r
web-...-vt5xx
web-...-zx45n
```

After the node failure, replacement Pods included:

```text
web-...-7bv4k
web-...-jfx56
web-...-xdv44
```

The names and IP addresses changed.

The Deployment did not care about the identity of individual Pods.

It cared about:

```text
Desired replicas = 5
```

This leads to an important Kubernetes principle:

> **Pods are disposable. Controllers maintain the desired workload.**

---

# 39. Restarting the Failed Worker

The failed node was restarted:

```bash
docker start day41-worker
```

The node was watched with:

```bash
kubectl get nodes -w
```

It eventually returned to:

```text
Ready
```

The final cluster showed:

```text
day41-control-plane   Ready
day41-worker          Ready
day41-worker2         Ready
```

---

# 40. Scheduler Does Not Continuously Rebalance

After the failed worker returned, the five nginx Pods remained on:

```text
day41-worker2
```

This is expected.

The scheduler does not continuously rebalance every existing Pod simply because another node becomes available.

The scheduler primarily handles **unscheduled Pods**.

Therefore:

```text
Worker fails
    ↓
Replacement Pods need placement
    ↓
Scheduler selects surviving worker
```

Later:

```text
Failed worker returns
    ↓
Existing Pods are already running
    ↓
No automatic redistribution is required
```

This is an important distinction between:

> **Scheduling**

and:

> **Continuous workload balancing.**

---

# 41. Final Cluster Verification

The final cluster state was verified using:

```bash
kubectl get nodes -o wide
```

Result:

```text
day41-control-plane   Ready
day41-worker          Ready
day41-worker2         Ready
```

Deployment:

```bash
kubectl get deployment web
```

Result:

```text
READY       5/5
UP-TO-DATE  5
AVAILABLE   5
```

Pods:

```bash
kubectl get pods -o wide
```

Result:

```text
5 Pods
1/1 Running
```

All five nginx Pods were healthy.

Therefore the cluster was completely recovered after the node failure experiment.

---

# 42. Control Plane vs Data Plane

One of the most important architectural lessons from Day 41 is the difference between the control plane and the data plane.

### Control plane

Responsible for making decisions and maintaining cluster state:

```text
kube-apiserver
etcd
scheduler
controllers
```

### Data plane

Responsible for actually running workloads:

```text
Nodes
kubelet
container runtime
Pods
CNI
kube-proxy
```

This distinction becomes extremely important during outages.

A control-plane failure does not necessarily mean that every existing application container immediately stops running.

The key operational idea is:

> **Control plane unavailable = Kubernetes control operations may fail; existing data-plane workloads may continue running.**

---

# 43. The Complete Kubernetes Flow

The entire Day 41 architecture can now be represented as:

```text
                     USER
                       │
                       │ kubectl apply
                       ↓
                ┌──────────────┐
                │ API SERVER   │
                └──────┬───────┘
                       │
          Authentication / Authorization
             Admission / Validation
                       │
                       ↓
                    etcd
                       │
                desired state
                       │
                       ↓
                Controllers
                       │
                 create objects
                       │
                       ↓
                    Pods
                       │
                       ↓
                  Scheduler
                       │
                  choose node
                       │
                       ↓
                  kubelet
                       │
                       ↓
                     CRI
                       │
                       ↓
                 containerd
                       │
                       ↓
                   Container
                       │
                       ↓
                Actual state
                       │
                       ↓
                 kubelet reports
                       │
                       ↓
                  API Server
                       │
                       ↓
                     etcd

              RECONCILIATION
                     ↺
                  FOREVER
```

Networking sits underneath:

```text
CNI
 ↓
Pod networking

kube-proxy
 ↓
Service networking
```

---

# 44. Three Reconciliation Experiments

Day 41 demonstrated reconciliation at multiple levels.

## Experiment 1 — Delete a Pod

```text
Pod deleted
     ↓
ReplicaSet controller notices
     ↓
Replacement Pod
```

## Experiment 2 — Delete a container

```text
Container deleted
     ↓
kubelet notices
     ↓
Container restored
```

## Experiment 3 — Kill a worker node

```text
Worker dies
     ↓
Node becomes NotReady
     ↓
Workloads become unavailable
     ↓
Controllers maintain desired replicas
     ↓
Replacement Pods created
     ↓
Scheduler places them
     ↓
Surviving kubelet runs them
```

The common principle is:

```text
Actual state != Desired state
           ↓
      Kubernetes acts
           ↓
Actual state → Desired state
```

---

# 45. Important Architecture Distinctions

These distinctions should not be confused.

## API server vs etcd

```text
API server = Kubernetes API/front door
etcd       = persistent cluster state
```

## Scheduler vs kubelet

```text
Scheduler = decides WHERE
kubelet   = makes it RUN
```

## Controller vs scheduler

```text
Controller = determines WHAT should exist
Scheduler  = determines WHERE it should run
```

## CNI vs kube-proxy

```text
CNI        = Pod networking
kube-proxy = Service networking rules
```

## Kubernetes vs container runtime

```text
Kubernetes = orchestration/control
containerd = container execution
```

These five distinctions are fundamental Kubernetes concepts.

---

# 46. Key Commands Learned

## Cluster

```bash
kind create cluster --name day41 --config kind-config.yaml
kubectl cluster-info
kubectl get nodes -o wide
kubectl version
```

## kubeconfig

```bash
kubectl config view
kubectl config get-contexts
kubectl config current-context
```

## System components

```bash
kubectl get pods -n kube-system -o wide
kubectl logs -n kube-system <pod>
```

## Objects

```bash
kubectl get pods
kubectl get deployment
kubectl get deployment,rs,pods
kubectl describe pod <pod>
kubectl describe rs <rs>
```

## API exploration

```bash
kubectl api-resources
kubectl explain pod
kubectl explain pod.spec
kubectl explain pod.spec.containers
kubectl get pod <pod> -o yaml
```

## Namespaces

```bash
kubectl get pods -n kube-system
kubectl get pods -A
```

## Events

```bash
kubectl get events -w
kubectl get events --sort-by=.lastTimestamp
```

## Watching workloads

```bash
kubectl get pods -w
kubectl get nodes -w
```

## Deployment

```bash
kubectl create deployment web --image=nginx --replicas=3
kubectl scale deployment web --replicas=5
kubectl delete pod <pod>
```

## Runtime inspection

```bash
docker ps
docker exec -it day41-worker bash
crictl ps
crictl rm -f <container-id>
```

## Kubernetes API

```bash
kubectl get pods --v=8
kubectl proxy
curl localhost:8001/api/v1/namespaces/default/pods
```

---

# 47. Final Day 41 Lessons

The most important lessons from Day 41 were:

1. **Kubernetes is declarative.**
2. **Kubernetes continuously reconciles desired and actual state.**
3. **The API server is the front door to Kubernetes.**
4. **etcd is the persistent store for Kubernetes state.**
5. **Controllers continuously reconcile resources.**
6. **The scheduler decides where unscheduled Pods should run.**
7. **The kubelet executes Pods assigned to its node.**
8. **containerd is the container runtime in this cluster.**
9. **CNI provides Pod networking.**
10. **kube-proxy provides Service networking rules.**
11. **Pods are disposable.**
12. **Deployments and ReplicaSets maintain workload counts rather than Pod identities.**
13. **A deleted Pod can be automatically replaced.**
14. **A deleted container can be restarted by the kubelet.**
15. **A failed worker can cause workloads to be recreated elsewhere.**
16. **A recovered node does not automatically cause Pods to rebalance.**
17. **`spec` represents desired state; `status` represents observed state.**
18. **Labels and selectors loosely connect Kubernetes objects.**
19. **kubeconfig context must be checked before destructive operations.**
20. **Kubernetes is fundamentally an HTTP API surrounded by controllers and agents.**

---

# 48. Final Mental Model

The most important sentence to remember from Day 41 is:

> **Kubernetes is an API-driven reconciliation system: you declare desired state, the API server accepts it, etcd stores it, controllers continuously work toward it, the scheduler decides placement, kubelets execute assigned Pods through the container runtime, and the resulting state is reported back so reconciliation can continue forever.**

The practical experiments proved this rather than merely describing it:

```text
Delete Pod
    ↓
Controller replaces it

Delete container
    ↓
Kubelet restores it

Kill worker
    ↓
Node becomes NotReady
    ↓
Workloads are replaced
    ↓
Scheduler places replacements
    ↓
Surviving node runs them

Restart worker
    ↓
Node returns Ready
```

**Day 41 completed successfully.**
