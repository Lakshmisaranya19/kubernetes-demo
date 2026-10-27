# Kubernetes Complete Foundations: Layers to Clusters (Combined)

**Learning notes for Saranya | Running example: GameHub | Local setup: Minikube (Docker driver, Windows)**

This file merges *Kubernetes Foundations: Images to Clusters* and *Kubernetes Basics Revision Notes*, then adds the foundations and the confusing parts that both files skipped or got slightly wrong. Every section ends with the "why it confuses people" fix where one is needed.

---

## 0. The one idea that removes most confusion

Your hierarchy has **two different kinds of arrows**. Mixing them is the root of every "single or multiple?" mistake.

```text
CHAIN 1: BUILD AND RUN  (arrow means "made from / created from")

Dockerfile ──docker build──▶ Image (many layers) ──docker run──▶ Container

CHAIN 2: GROUP AND SCHEDULE  (arrow means "placed into / belongs to")

Container ──▶ Pod ──▶ Node ──▶ Cluster
```

- Chain 1 is **Docker's job**. It answers "how do I package and start my app?"
- Chain 2 is **Kubernetes' job**. It answers "where does it run, how many copies, and what keeps it alive?"
- The bridge between them: a Pod's YAML names an **image**. Kubernetes asks the node's container runtime to create the **container** from that image.

---

## 1. Docker side: layer, Dockerfile, image, container

### 1.1 Layer

A **layer** is the set of file changes produced by one Dockerfile instruction (`RUN`, `COPY`, `ADD`). Layers are read-only and stacked.

- One image has **many layers**.
- One layer can be **shared by many images** (this is why a second image sharing the same base downloads fast, and why Docker Hub said "Mounted from another repo" in your Level 7 push).
- Layers are cached. If you change line 8 of a Dockerfile, lines 1 to 7 are reused and lines 8 onward rebuild. That is why `COPY package*.json` + `npm ci` come *before* `COPY . .` in good Dockerfiles.

### 1.2 Dockerfile and build context

- The **Dockerfile** is the recipe. It defines the image.
- `docker build` is the command that *executes* the recipe.
- Your **repo is only the build context**: the files Docker is allowed to `COPY` from. After the build, the image is self-contained. You do not need the repo to run it.
- `.dockerignore` controls what is excluded from the build context (for example `node_modules`, `.env`).

```text
Dockerfile + build context (your repo)
          │
          │ docker build -t gamehub-frontend:v1 .
          ▼
     GameHub image (layers)
```

### 1.3 Image

An **image** is a read-only template: an ordered stack of layers plus metadata (default command, exposed ports, environment defaults).

- Not running. Not a process. A template.
- One image can create **many** containers.
- Name format: `registry/namespace/repository:tag`, for example `lakshmisaranya/playwright-automation:v2`.
- **Tag** = a label for a version. `latest` is just a tag named "latest"; it does **not** automatically mean newest. It means whatever was last pushed with that tag. This matters in Kubernetes (see section 13).
- **Digest** (`sha256:...`) = the exact immutable identity of an image. Tags can move; digests cannot.

### 1.4 Registry

A **registry** is where images are stored and pulled from (Docker Hub, GitHub Container Registry, AWS ECR). `docker push` uploads to a registry, `docker pull` downloads. Kubernetes nodes pull images from a registry, which is why a locally built image is invisible to a cluster unless you load it (section 13).

### 1.5 Container

A **container** is a running (or stopped) instance created from **exactly one image**, plus one thin **writable layer** on top.

- **1 image → many containers**: yes.
- **1 container → many images**: **no, never.** If you see several images, you are looking at several containers.
- Class and object analogy: image = class, container = object.
- `docker run` = create + start a **new** container. `docker start` = restart an **existing** stopped container (env vars are not re-read; Level 2 lesson).
- Changes in the writable layer survive stop/start but vanish when the container is removed (Level 4C lesson).

> **Why "a container can have multiple images" feels true:** a Compose file or a Pod lists several containers, each from a different image. The "many" belongs to the Compose file or the Pod, not to the container.

### 1.5.1 Docker vs the container runtime

On your laptop, **Docker** builds images and runs containers. Inside a Kubernetes node, the thing that actually runs containers is a **container runtime**, usually **containerd** (or CRI-O). Kubernetes does not need the Docker command at all. Minikube with the Docker driver runs the *whole node as a container inside Docker*, and containerd runs inside that node.

```text
Windows laptop
└── Docker Desktop
    └── "minikube" container   ← this IS the Kubernetes node
        └── containerd         ← runs your app containers inside the node
            └── your app containers
```

This "container inside a container" layering is why `docker images` on your laptop does **not** show images the cluster pulled. They live in a different place (inside the minikube node).

---

## 2. Pod

A **Pod** is the smallest unit Kubernetes deploys. It is a wrapper around **one or more containers**.

**Why Pods exist at all:** containers in the same Pod are tightly coupled. They:

- share **one IP address and one network namespace**, so they talk over `localhost`;
- can share **volumes**;
- always land on the **same Node**, start together, and die together.

```text
Pod (one IP: 10.244.0.5)
┌──────────────────────────────┐
│ app container   :3000        │
│ sidecar container :9100      │  ← reaches app at localhost:3000
│ shared volume (optional)     │
└──────────────────────────────┘
```

### Single or multiple containers per Pod?

- **Usually one.** One app component = one container = one Pod.
- **Multiple only when tightly coupled**: an app plus a log shipper, a proxy sidecar, or an **init container** (a container that runs once to completion before the main container starts; same idea as your `volume-init` service in Compose with `service_completed_successfully`).
- Do **not** put frontend and database in one Pod. They scale and update independently.

### How a Pod is defined

In YAML. But you almost never write a bare Pod. You write a **Deployment** or **Job** whose `template:` section *is* the Pod definition, and that controller creates the Pods.

### Three facts that trip people up

1. **Pods are disposable.** A Pod that dies is replaced by a *new* Pod with a *new* IP. A bare Pod with no controller is **not** replaced at all.
2. **Replicas count Pods, not containers.** 3 replicas with 2 containers each = 3 Pods and **6 containers**.
3. **A Pod never spans Nodes.** All its containers share one machine.

### What "a Pod never spans Nodes" means

"Spans" means "stretches across". A single Pod cannot be split so that some of its containers run on one machine and others on a different machine.

```text
Not possible (Pod split across Nodes)

Node 1                     Node 2
┌──────────────┐           ┌──────────────┐
│ app          │           │ log-shipper  │
└──────────────┘           └──────────────┘
        └──── same Pod ────┘     ✗


What actually happens (Pod stays whole on one Node)

Node 1                     Node 2
┌──────────────────────┐   ┌──────────────┐
│ Pod                  │   │ (other Pods) │
│  ├ app               │   └──────────────┘
│  └ log-shipper       │
└──────────────────────┘
```

**Why the rule exists:** containers in a Pod share one IP address, talk over `localhost`, and can share a volume. `localhost` and shared folders only work inside one machine, so the whole Pod must be placed on one Node.

**Analogy:** a Pod is a family living in one flat. All members live in the same flat (Node). You cannot have one member in Hyderabad and another in Pune and still call it one household with one address.

**What this does not mean:** your application can still spread across many Nodes. If a Deployment has 3 Pods, each Pod can sit on a different Node, but each Pod is whole on its own Node.

```text
Node 1: Pod A (whole)
Node 2: Pod B (whole)
Node 3: Pod C (whole)
```

**Practical consequence:** if two parts of your app must run on different machines or scale separately (for example frontend and database), put them in separate Pods. Put containers in the same Pod only when they need to share `localhost` and volumes.

---

## 3. Node

A **Node** is a machine (physical or virtual) that runs Pods.

### What runs on every Node

| Component | Job |
|---|---|
| **kubelet** | The agent. Talks to the control plane, makes sure the Pods assigned to this node are actually running |
| **Container runtime** (containerd) | Actually pulls images and runs containers |
| **kube-proxy** | Programs network rules on the node so Services route traffic to the right Pods |

### How a Node is "defined"

**You do not write YAML to create a Node.** A Node is a machine that has been set up and registered with a cluster. Once registered, Kubernetes keeps an entry (a `Node` object) for it, and you see that entry with `kubectl get nodes`.

How the machine gets set up depends on where you run Kubernetes:

| Where | Who creates the Node | What you do |
|---|---|---|
| **Minikube (your laptop)** | Minikube does it for you | Run `minikube start`. Nothing else |
| **Your own servers** | You | Install the container runtime and kubelet on the machine, then join it to the cluster with `kubeadm join` |
| **Cloud (EKS/GKE/AKS)** | The cloud service | Choose a node group (machine type and count) in the console or Terraform. The cloud creates the machines |

**What happened on your machine with Minikube:**

```text
You ran:  minikube start

Minikube then:
1. Created a Docker container named "minikube"   ← this container IS your Node
2. Installed the kubelet and containerd inside it
3. Started the control plane components inside it
4. Registered the machine with the cluster
```

You can verify each of these yourself:

```powershell
kubectl get nodes                 # shows one Node named "minikube" with STATUS Ready
docker ps                         # shows a container named "minikube" (the Node)
kubectl describe node minikube    # shows the Node's CPU, memory, and the Pods running on it
```

**Why there is no YAML:** a Pod or Deployment describes *what you want to run*. A Node is *where it runs*. You describe the work in YAML, and the Node is the machine that already exists to run it.

**Optional experiment:** `minikube node add` adds a second Node to your local cluster. After that, `kubectl get nodes` shows two, and you can scale a Deployment and use `kubectl get pods -o wide` to see Pods spread across both.
### Single or multiple?

- One Node runs **many Pods** (limited by CPU, memory, and scheduling rules).
- One Node belongs to **exactly one cluster**.
- A cluster has **one or more Nodes**. Minikube default is one.
- You do not normally choose which Node a Pod goes to. The **Scheduler** decides.

> **Correction to a common claim:** "a node has one or more pods" describes what usually happens, but a Node does not *own* Pods in its definition. The Scheduler assigns Pods to it. An empty Node is still a valid Node.

---

## 4. Cluster

A **cluster** is the whole Kubernetes environment: a **control plane** plus **one or more worker nodes**.

### Control plane components

| Component | Job |
|---|---|
| **kube-apiserver** | The front door. `kubectl` and every other component talk to it. Nothing bypasses it |
| **etcd** | The database holding all cluster state (what should exist) |
| **kube-scheduler** | Picks a Node for each new, unassigned Pod |
| **kube-controller-manager** | Runs the control loops (Deployment controller, ReplicaSet controller, Node controller...) that keep reality matching the desired state |

### Where does the cluster "live"?

It does not live *inside* a node, and it is not *outside* them. It is the logical grouping created when you run `kubeadm init`, `minikube start`, `kind create cluster`, or create an EKS/GKE/AKS cluster.

- **Production:** control plane on separate machines (managed services hide them from you), workers elsewhere.
- **Minikube:** everything collapses into one node. That is why it looks like "the cluster is inside my laptop."

### Single or multiple clusters?

You can have many separate clusters (local practice, staging, production). Do not confuse:

- many **Pods** in one cluster with many clusters;
- many **Nodes** in one cluster with many clusters;
- many **containers** in one Pod with many Pods.

`kubectl` talks to whichever cluster the current **context** points at. Check with `kubectl config current-context` before running destructive commands.

---

## 5. The complete hierarchy with cardinality

```text
Layer ──(many make up)──▶ Image ──(1 image → many)──▶ Container
Container ──(1 or more)──▶ Pod ──(many)──▶ Node ──(1 or more)──▶ Cluster
```

| Unit | Made of / contains | Single or multiple? | How it is defined |
|---|---|---|---|
| **Layer** | File changes from one Dockerfile instruction | Shared by many images | Created automatically by Dockerfile instructions |
| **Image** | Many layers | 1 image → many containers | **Dockerfile** + `docker build` |
| **Container** | Exactly **1** image + a writable layer | 1 container = 1 image, always | `docker run`, a Compose `services:` entry, or a Pod's `containers:` list |
| **Pod** | **1 or more** containers (usually 1) | Lives on exactly 1 node | YAML, normally inside a Deployment/Job `template:` |
| **Node** | Runs many Pods | Belongs to exactly 1 cluster | Provisioned machine joined to a cluster; not written as YAML |
| **Cluster** | Control plane + 1 or more Nodes | Many clusters can exist separately | `minikube start`, `kubeadm init`, or a managed cloud service |

**Layers are smaller than images**, so "the first smallest unit is the image" is not quite right. The smallest *thing you build and ship* is the image, but it is made of layers.

---

## 6. Controllers: the layer between Pod and Cluster

Pods are rarely created directly. **Controllers** create and watch them. This is the self-healing that Docker Compose lacked (Level 4 and 10.5 findings).

| Controller | Use it for | GameHub example |
|---|---|---|
| **Deployment** | Stateless apps that run forever and can have identical replicas | Frontend, Auth service, Game service |
| **ReplicaSet** | Keeps N matching Pods alive. Created and managed *by* a Deployment; you rarely touch it | (automatic) |
| **Job** | Run-to-completion work | Your Playwright test run |
| **CronJob** | A Job on a schedule | Nightly test run |
| **StatefulSet** | Apps needing stable identity and their own storage per replica | PostgreSQL (in real use) |
| **DaemonSet** | One Pod on every Node | Log or monitoring agents |

### How replicas are defined

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gamehub-frontend
spec:
  replicas: 3                  # desired number of Pods
  selector:
    matchLabels:
      app: gamehub-frontend    # which Pods this Deployment owns
  template:                    # this block IS the Pod definition
    metadata:
      labels:
        app: gamehub-frontend  # must match the selector above
    spec:
      containers:
        - name: frontend
          image: your-gamehub-frontend-image:tag   # placeholder
          ports:
            - containerPort: 80
```

Key points:

- `replicas: 3` = 3 **Pods**.
- The `selector` and the Pod `labels` **must match**, or the Deployment is rejected.
- Saving the file does nothing. You submit it: `kubectl apply -f deployment.yaml`.
- **Deployment manages releases (rolling updates, rollbacks). ReplicaSet keeps the Pod count.**

### Desired state and the reconciliation loop

You declare *what you want* ("3 replicas of this image"). Controllers continuously compare reality with that declaration and fix differences. Delete a Pod and the ReplicaSet creates another. This is the core idea of Kubernetes.

### Rolling update vs rollback

- **Rolling update:** change the image in the Deployment, and Kubernetes replaces Pods gradually so the app stays up.
- **Rollback:** `kubectl rollout undo deployment/gamehub-frontend` returns to the previous ReplicaSet.
- A **canary** release (small percentage of traffic to the new version) is a *different* strategy and needs extra tooling. A rolling update does not give you traffic-percentage control.

---

## 7. Labels, selectors, and namespaces

### Labels and selectors

A **label** is a key/value tag on an object (`app: gamehub-frontend`). A **selector** finds objects by label. This is how Deployments find their Pods and how **Services find the Pods to send traffic to**. It is the glue of Kubernetes. If a Service has no endpoints, a label/selector mismatch is the first thing to check.

### Namespaces

A **namespace** is a named partition *inside one cluster* (`default`, `kube-system`, or your own such as `gamehub`). It groups and separates resources. It is **not** a separate cluster.

- `kubectl get pods` shows only the **current namespace** (normally `default`).
- `kubectl get pods -A` shows **all namespaces** (including the system Pods).
- This is why a fresh Minikube shows system Pods with `-A` but "No resources found" without it.

---

## 8. Services: a stable way to reach Pods

Pod IPs change every time a Pod is replaced, so nothing should depend on a Pod IP. A **Service** gives a **stable IP and DNS name** and load-balances across all Pods matching its selector.

```text
Frontend Pod ──▶ http://auth-service:80 ──▶ Service ──▶ one of the matching Auth Pods
```

- Inside a cluster, Pods reach a Service by name: `auth-service` (same namespace) or `auth-service.gamehub.svc.cluster.local` (full form).
- This is the Kubernetes version of Compose's name-based networking, except it is stable and load-balanced.
- **kube-proxy** on each node implements the routing rules.

### Service types

| Type | Reachable from | Typical use | Notes |
|---|---|---|---|
| **ClusterIP** (default) | Inside the cluster only | Frontend → backend, backend → database | No direct access from your browser |
| **NodePort** | Outside, via `<node-ip>:<port>` (ports 30000 to 32767) | Simple exposure, local practice | Also creates a ClusterIP underneath |
| **LoadBalancer** | Outside, via a cloud load balancer IP | Real cloud deployments | Needs a cloud provider (or `minikube tunnel` locally) |
| **Ingress** (not a Service type) | Outside, via HTTP host/path rules | Many web apps behind one entry point | Needs an Ingress controller installed |

**Memory rule:**

- Pod → Pod, inside the cluster: **ClusterIP**.
- Browser → app, for practice: **NodePort** (or port-forward).
- Browser → app, in production: **LoadBalancer or Ingress**.

### Important Minikube gotcha: NodePort may not work from your Windows browser

The older notes show `http://<minikube-ip>:<port>` reaching the app. With the **Docker driver on Windows**, the Minikube node is a container, and its IP is generally **not reachable from your host**. Use one of these instead:

```powershell
minikube service gamehub-frontend --url     # opens a tunnel and prints a localhost URL (keep the terminal open)
kubectl port-forward svc/gamehub-frontend 8080:80   # then open http://localhost:8080
```

If a NodePort "doesn't work," the Service is probably fine. The network path from your laptop to the node is the problem.

---

## 9. Configuration and storage

### ConfigMap and Secret

- **ConfigMap:** non-sensitive config (`DB_HOST`, `HEADLESS`, base URLs). Same idea as your non-sensitive env vars.
- **Secret:** sensitive values (`DB_PASSWORD`).
- **Warning:** a Kubernetes Secret is **base64-encoded, not encrypted**, by default. Anyone who can read the Secret object can decode it. Real protection needs RBAC (restricting who can read Secrets), etcd encryption at rest, and often an external manager (Infisical, Vault) feeding it. This is the same lesson as your Docker secrets work: secrets files limit casual exposure, access control decides who can read them.
- Both can be injected as **environment variables** or **mounted as files** (files are the analogue of `/run/secrets`).

### Volumes, PersistentVolume, PersistentVolumeClaim

Pod storage is **ephemeral** by default (like a container's writable layer). For a database:

- **PersistentVolume (PV):** the actual storage.
- **PersistentVolumeClaim (PVC):** a Pod's *request* for storage. The Pod mounts the PVC.
- Data survives Pod deletion and rescheduling. This is the Kubernetes counterpart of your named volume `test-results-volume`.
- In Minikube, a default storage class provisions PVs for you automatically.

---

## 10. What happens when you run `kubectl apply -f deployment.yaml`

```text
1. kubectl sends the YAML to the API server
2. API server validates it and stores the desired state in etcd
3. Deployment controller creates a ReplicaSet
4. ReplicaSet controller creates 3 Pod objects (not yet assigned to a node)
5. Scheduler picks a Node for each Pod
6. kubelet on that Node sees "a Pod is assigned to me"
7. kubelet tells containerd to pull the image and start the container
8. Pod becomes Running; the Service starts routing traffic to it
```

Failures at each step show up as different statuses, which helps debugging:

| Status | Usually means |
|---|---|
| `Pending` | Not scheduled yet (no node has enough resources) or waiting on a volume |
| `ContainerCreating` | Scheduled, image is being pulled |
| `ErrImagePull` / `ImagePullBackOff` | The node cannot pull the image (wrong name/tag, private registry, or a local image the cluster cannot see) |
| `CrashLoopBackOff` | Container starts, then exits repeatedly (app error; check `kubectl logs`) |
| `OOMKilled` (in `describe`) | Memory limit exceeded (your Level 8 lesson applies here too) |
| `Running` but not working | Check readiness, Service selector labels, and ports |

---

## 11. Probes and resource limits (Level 4 and 8 ideas, Kubernetes version)

- **Readiness probe:** "Should this Pod receive traffic right now?" Fails = removed from the Service, **not** restarted.
- **Liveness probe:** "Is this Pod still alive?" Fails = the container is **restarted**.
- **Compose `healthcheck` vs probes:** Compose can only mark a container unhealthy. Kubernetes acts on it.
- **Requests and limits:** `requests` = what the scheduler reserves when choosing a node; `limits` = the hard cap. Memory over limit = OOM kill. CPU over limit = throttling. Same behavior as your Level 8 findings.

```yaml
resources:
  requests: { cpu: "100m", memory: "128Mi" }
  limits:   { cpu: "500m", memory: "256Mi" }
```

---

## 12. Your repo mapped to Kubernetes

| Your Docker/Compose piece | Kubernetes equivalent |
|---|---|
| `playwright-repo-db-container` (runs and exits) | **Job** (not a Deployment) |
| `postgres-db-container` | **StatefulSet**, or Deployment + PVC for learning |
| `report-viewer` (Nginx) | **Deployment + Service** (best first hands-on target) |
| `test-results-volume` | **PVC** |
| `./db/init` mounted to `docker-entrypoint-initdb.d` | ConfigMap mounted as files, or an init container |
| `volume-init` service with `service_completed_successfully` | **Init container** |
| `.env` non-sensitive values | **ConfigMap** |
| Docker secrets in `/run/secrets` | **Secret** mounted as files |
| `restart: unless-stopped` | Controller behavior (Deployment recreates Pods automatically) |
| `mem_limit`, `cpus` | `resources.limits` |
| `healthcheck` + `service_healthy` | Readiness/liveness probes |
| User-defined network, name-based DNS | **Service** names + cluster DNS |
| `ports: "8080:80"` | NodePort / port-forward / Ingress |

GameHub mapping:

| GameHub part | Workload | Service type |
|---|---|---|
| Frontend | Deployment (replicas 2 to 3) | NodePort (practice) or Ingress |
| Auth service | Deployment | ClusterIP |
| Game service | Deployment | ClusterIP |
| PostgreSQL | StatefulSet (or Deployment + PVC) | ClusterIP |

---

## 13. Minikube environment and the local image trap

```text
Your Windows laptop
└── Docker Desktop (Minikube driver)
    └── Minikube cluster
        ├── Control plane components
        └── Node: minikube
            ├── Kubernetes system Pods (kube-system)
            └── Your application Pods
```

**The most common beginner failure:** you build an image on your laptop with `docker build`, reference it in a Deployment, and get `ErrImagePull`. The cluster's node has its own image store and cannot see your laptop's Docker images.

Fixes (pick one):

```powershell
minikube image load gamehub-frontend:v1      # copy the local image into the node
# or push the image to Docker Hub and reference lakshmisaranya/gamehub-frontend:v1
```

Also set `imagePullPolicy: IfNotPresent` for locally loaded images. Kubernetes defaults to `Always` when the tag is `latest` (or missing), which forces a registry pull and fails. **Use explicit version tags instead of `latest`** (same traceability point as Level 7).

---

## 14. Confusing parts, answered directly

| Confusion | Answer |
|---|---|
| "Can a container have multiple images?" | No. 1 container = 1 image. The many is at the Pod or Compose level |
| "Is the image the smallest unit?" | Layers are smaller. The image is the smallest thing you build and ship |
| "Is the repo part of the image?" | Only the files you `COPY` from the build context end up inside it |
| "Pod vs container?" | The Pod is Kubernetes' wrapper. Docker has no Pod concept |
| "Does `replicas: 3` mean 3 containers?" | 3 **Pods**. Containers = 3 × containers per Pod |
| "Is a Pod the thing I manage?" | No. You manage the Deployment/Job. A bare Pod that dies stays dead |
| "Do I write YAML for a Node?" | No. You provision a machine and join it |
| "Is my laptop a Node?" | Not automatically. It is the client. In Minikube, the node is a container on your laptop |
| "Is the cluster inside the node?" | Neither inside nor outside. It is the control plane + nodes, as a logical group |
| "Are Nodes in one cluster the same as two clusters?" | No. Nodes share one control plane and one API. Separate clusters do not |
| "Is a namespace a mini-cluster?" | No. It is a partition inside one cluster, with the same nodes |
| "Why can't I open ClusterIP in my browser?" | It is an internal-only virtual IP |
| "NodePort doesn't open on Windows Minikube" | Use `minikube service ... --url` or `kubectl port-forward` |
| "Is a Secret encrypted?" | Not by default. Base64 only |
| "Is Kubernetes the same as Docker?" | No. Docker builds and runs containers. Kubernetes orchestrates containers across machines through a runtime like containerd |
| "Does `down` in Compose equal `delete` in Kubernetes?" | Roughly. Deleting a Deployment removes its Pods; PVCs may remain depending on settings |

---

## 15. Command cheat sheet

```powershell
# Cluster and context
minikube status
minikube start
kubectl config current-context
kubectl get nodes
kubectl get pods -A                       # all namespaces

# Create and inspect
kubectl apply -f deployment.yaml
kubectl get deployments
kubectl get pods
kubectl get svc
kubectl describe pod <pod-name>           # events show why a Pod is failing
kubectl logs <pod-name>
kubectl logs <pod-name> --previous        # logs of the crashed container

# Scale, update, roll back
kubectl scale deployment gamehub-frontend --replicas=3
kubectl set image deployment/gamehub-frontend frontend=<image>:<new-tag>
kubectl rollout status deployment/gamehub-frontend
kubectl rollout undo deployment/gamehub-frontend

# Get inside and reach the app
kubectl exec -it <pod-name> -- sh         # same idea as docker exec
kubectl port-forward svc/gamehub-frontend 8080:80
minikube service gamehub-frontend --url

# Clean up
kubectl delete -f deployment.yaml
```

Debug order when something is wrong: `get pods` → `describe pod` → `logs` → check Service selector labels → check ports.

---

## 16. Knowledge check

1. Is an image a running instance or a template?
2. Can one container be built from two images?
3. What is the difference between `docker build` and the Dockerfile?
4. A Deployment has `replicas: 3` and each Pod has 2 containers. How many Pods and containers?
5. Can containers from one Pod run on different Nodes?
6. Can one Node run multiple Pods? Can one Pod run on two Nodes?
7. Which control plane component picks the Node for a new Pod?
8. Two Nodes in one cluster vs two separate clusters: what is the difference?
9. What recreates a Pod that disappears from a Deployment?
10. Which Service type should the frontend use to call the backend inside the cluster?
11. Your Pod shows `ErrImagePull` for an image you just built locally. What is the likely cause and fix?
12. Is a Kubernetes Secret encrypted?
13. Which workload type fits your Playwright test run, and why not a Deployment?
14. What is a namespace?

### Answers

1. A template.
2. No. A container comes from exactly one image.
3. The Dockerfile is the recipe. `docker build` executes it to produce the image.
4. 3 Pods and 6 containers.
5. No. A Pod's containers always share one Node.
6. Yes to the first. No to the second.
7. The Scheduler.
8. Two Nodes share one control plane and API. Two clusters are fully separate environments.
9. The ReplicaSet, which is managed by the Deployment.
10. ClusterIP.
11. The node cannot see your laptop's Docker images. Use `minikube image load` or push to a registry, and set `imagePullPolicy: IfNotPresent`.
12. Not by default. It is base64-encoded. Encryption at rest and access control are separate setups.
13. A Job. It runs to completion. A Deployment would keep restarting it after it exits.
14. A named partition inside one cluster, not a separate cluster.

---

## 17. Suggested hands-on order

1. `minikube status`, `kubectl get nodes`, `kubectl get pods -A` (read the output; explain each line in your own words).
2. Deploy Nginx as a Deployment with 1 replica. Reach it with `kubectl port-forward`.
3. Delete its Pod and watch the ReplicaSet replace it. This proves self-healing.
4. Scale to 3 replicas. Run `kubectl get pods -o wide` to see which Node each one is on.
5. Add a ClusterIP Service, then a NodePort. Compare how you reach each.
6. Do a rolling update to a new image tag, then `rollout undo`.
7. Break something on purpose (wrong image tag → `ImagePullBackOff`; wrong selector → Service with no endpoints) and diagnose it with `describe` and `logs`.
8. Move `report-viewer` (Nginx) and its volume to Kubernetes as Deployment + Service + PVC.
9. Run the Playwright suite as a Job.
10. Add ConfigMap and Secret for the DB settings, then PostgreSQL as a StatefulSet.

**Learning principle:** keep these ideas separate: **layer → image → container → Pod → Node → cluster**, with **controllers** managing Pods and **Services** giving them a stable address. Each is a different layer of responsibility.
