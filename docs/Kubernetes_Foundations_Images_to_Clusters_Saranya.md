# Kubernetes Foundations — From Images to Clusters (Claude)

**Learning notes for Saranya**  
**Project example: GameHub**  
**Level:** Beginner-friendly reference notes

---

## 1. The big picture

A typical path from application code to a running application looks like this:

```text
Application code
      |
      | build
      v
 Docker image
      |
      | create and run
      v
   Container
      |
      | Kubernetes manages containers inside
      v
     Pod
      |
      | scheduled onto
      v
     Node
      |
      | belongs to
      v
   Cluster
```

These are related, but they are **not the same thing**:

- **Image** = a template used to create a container.
- **Container** = a running (or stopped) instance created from an image.
- **Pod** = Kubernetes' smallest deployable unit; it contains one or more containers.
- **Node** = a physical or virtual machine that runs Pods.
- **Cluster** = a Kubernetes environment made up of its control plane and one or more Nodes.

## 2. Image

A **container image** is a read-only template containing the filesystem layers and configuration needed to create a container. It is commonly built from a `Dockerfile`.

For GameHub, an image might include the application code, required libraries, and startup configuration.

```text
Dockerfile + build context
          |
          | docker build
          v
    GameHub image
```

### Single or multiple?

- One image can be used to create **many containers**.
- An image is a template; it is not itself a running application process.
- A base image may not contain your application code yet; the Dockerfile can add it.

Example: build one version of the GameHub frontend image, then use that same image to start several instances.

## 3. Container

A **container** is an instance created from an image. It runs the configured application process in an isolated environment.

```text
             GameHub image
             /     |     \
            v      v      v
       Container  Container  Container
           1          2         3
```

### Single or multiple?

- **One image → many containers** is normal.
- Each container is created from one image; a container is not assembled from multiple images.
- `docker run` normally creates and starts a new container.
- `docker start` starts an existing stopped container.

### GameHub example

If you start three containers from the same GameHub frontend image, you have three container instances. In Kubernetes, the usual pattern for independently scaling those instances is to put one application container in each Pod and run three Pod replicas.

## 4. Pod

A **Pod** is Kubernetes' smallest deployable unit. A Pod contains one or more containers. Most commonly, an application Pod contains **one application container**.

```text
Pod A                  Pod B
+----------------+     +----------------+
| GameHub        |     | GameHub        |
| frontend       |     | frontend       |
| container      |     | container      |
+----------------+     +----------------+
```

### Single or multiple containers in a Pod?

**One container per Pod** is the usual and recommended starting pattern for a single application component.

A Pod can contain multiple containers when those containers are tightly coupled and need to work together—for example, an application container plus a helper/sidecar container.

Containers in the same Pod:
- share the Pod's network namespace and IP address;
- can communicate over `localhost`;
- can share volumes when those volumes are configured for them.

Containers in one Pod are placed together on the **same Node**. Kubernetes does not split one Pod's containers across different Nodes.

### GameHub example: three instances

If GameHub needs three independently scalable frontend instances, the normal pattern is:

```text
Deployment asks for 3 replicas
             |
       +-----+-----+
       |     |     |
       v     v     v
     Pod 1  Pod 2  Pod 3
       |     |     |
       v     v     v
     App     App   App
   container container container
```

That means **3 Pods and 3 application containers**, assuming one application container per Pod—not one Pod containing three application containers.

### If a Pod has multiple containers

For example, if each Pod has two containers and the workload has three Pod replicas, the total is:

- 3 Pods
- 2 containers per Pod
- 6 containers total

The replica count refers to **Pods**, not containers.

## 5. Node

A **Node** is a physical or virtual machine that runs Pods. Your local Minikube environment currently has one Node named `minikube`.

One Node can run multiple Pods, depending on available CPU/memory and scheduling constraints.

```text
Node: minikube
+--------------------------------------+
| Pod: GameHub frontend                |
| Pod: GameHub backend                 |
| Pod: another workload                |
+--------------------------------------+
```

### Single or multiple Nodes?

- A Kubernetes cluster has **one or more Nodes**.
- A single Node can run multiple Pods.
- With multiple Nodes, Kubernetes can distribute Pods across them.
- A Pod is assigned to one Node at a time; the containers inside that Pod stay together on that Node.
- The Scheduler chooses a suitable Node for a new, unassigned Pod based on factors such as available resources and scheduling constraints.

Example with two Nodes and four Pods:

```text
Kubernetes Cluster
+----------------------+   +----------------------+
| Node 1               |   | Node 2               |
|  Pod A               |   |  Pod C               |
|  Pod B               |   |  Pod D               |
+----------------------+   +----------------------+
```

This is an example distribution, not a guarantee that Kubernetes will always split Pods evenly.

## 6. Cluster

A **Kubernetes cluster** is the complete Kubernetes environment. It includes:

1. **Control Plane** — coordinates and manages the cluster.
2. **Worker Node(s)** — run application Pods. In a small local setup, the same Node may also host the Control Plane.

Your Minikube practice environment has one Node named `minikube`. The Windows laptop is the physical host, Docker is used as Minikube's driver, and the Minikube Node contains the Kubernetes environment.

```text
Your Windows laptop
└── Docker (Minikube driver)
    └── Minikube Kubernetes cluster
        ├── Control Plane components
        └── Node: minikube
            ├── Kubernetes system Pods
            └── Your application Pods (when deployed)
```

### Single or multiple clusters?

You can have one or more separate Kubernetes clusters—for example, a local practice cluster and a separate production cluster. Each cluster is its own Kubernetes environment with its own resources and configuration.

Do not confuse:
- **Multiple Pods in one cluster** with multiple clusters.
- **Multiple Nodes in one cluster** with multiple clusters.
- **Multiple containers in one Pod** with multiple Pods.

## 7. How to choose the pattern

| Requirement | Usual pattern |
|---|---|
| Run one application instance in Kubernetes | One Pod with one application container |
| Run three independently scalable copies of GameHub frontend | One Deployment with 3 replicas; normally 3 Pods, 1 frontend container in each |
| Run an app plus a tightly coupled helper | One Pod with multiple containers, if they need shared lifecycle/networking |
| Run several Pods on one machine | One Node can host multiple Pods, subject to capacity and constraints |
| Spread workloads across machines | Use multiple Nodes in one cluster; Kubernetes schedules Pods onto suitable Nodes |
| Separate development and production environments | Often use separate clusters, depending on the team's requirements |

## 8. Deployment and ReplicaSet: where replicas are defined

In everyday Kubernetes work, you usually do not manually create three separate Pods for three identical copies. You define a **Deployment** with a desired replica count.

Example fragment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gamehub-frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: gamehub-frontend
  template:
    metadata:
      labels:
        app: gamehub-frontend
    spec:
      containers:
        - name: frontend
          image: your-gamehub-frontend-image:tag
```

This illustrates the structure only; `your-gamehub-frontend-image:tag` is a placeholder and must be replaced with an image that your cluster can access.

Key points:
- `replicas: 3` means the Deployment aims to keep **3 Pods**.
- The Pod template above defines **one application container per Pod**.
- The Deployment manages rollout and desired state through ReplicaSets.
- If a managed Pod disappears, Kubernetes works to create a replacement.
- Saving the YAML file alone does not create resources. You submit it to the cluster with a command such as `kubectl apply -f deployment.yaml`.

## 9. Quick reference: what is one, what can be many?

| Item | Can there be multiple? | What does “multiple” mean? |
|---|---|---|
| Image | Yes | One image can be used to create many container instances |
| Container | Yes | Multiple container instances can run from the same image |
| Containers in a Pod | Yes | Usually one; sometimes multiple tightly coupled containers |
| Pods | Yes | A workload can have multiple Pod replicas |
| Nodes | Yes | A cluster can have one or more Nodes |
| Clusters | Yes | You can operate separate Kubernetes environments |

## 10. Knowledge check

Try answering these without looking back:

1. Is an image a running instance or a template?
2. Can one image create multiple containers?
3. If a Deployment has `replicas: 3` and each Pod template has one container, how many Pods and application containers are intended?
4. Can containers from the same Pod be placed on different Nodes?
5. Can one Node run multiple Pods?
6. What is the difference between two Nodes in one cluster and two separate clusters?
7. Which Control Plane component chooses a Node for a new, unassigned Pod?

### Answers

1. A template.
2. Yes.
3. Three Pods and three application containers.
4. No. Containers in a Pod stay together on one Node.
5. Yes, subject to resources and constraints.
6. Two Nodes are machines within one Kubernetes environment; two clusters are separate Kubernetes environments.
7. The Scheduler.

---

## 11. Your current local environment

According to the continuation notes:

- Windows 11 Home Single Language.
- Minikube v1.39.0, using the Docker driver.
- Kubernetes v1.37.0.
- One Node named `minikube`, previously confirmed `Ready`.
- Kubernetes system Pods were `Running`.
- The default namespace had no application Pods at the time of the notes.

Useful commands you have already used:

```powershell
minikube status
kubectl get nodes
kubectl get pods -A
kubectl get pods
```

What they show:
- `minikube status` — status of Minikube components.
- `kubectl get nodes` — Nodes visible to the current Kubernetes context.
- `kubectl get pods -A` — Pods in all namespaces.
- `kubectl get pods` — Pods in the current namespace (usually `default`).

---

**Learning principle:** Keep these five ideas separate: **image → container → Pod → Node → cluster**. They are related, but each represents a different layer or unit.
