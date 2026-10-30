# Kubernetes Clusters with Minikube — Beginner Notes

- **Last updated: 29 October 2026**
- **Reviewer: Lakshmi Saranya Dhulipala**

> **Goal:** Understand what a Kubernetes cluster is, how Minikube creates and manages one, how to inspect clusters, open the right Dashboard, and trace a container image to its Pod.
>
> These notes use the Windows + Minikube + Docker-driver setup and the example profiles `minikube` and `sample-cluster`.

---

## 1. What is a Kubernetes cluster?

A **Kubernetes cluster** is a complete environment in which Kubernetes runs and manages containerized applications. It includes a **control plane** and one or more **Nodes**.

- **Control plane:** makes decisions and manages the cluster.
- **Node:** a machine (physical, virtual, or a container-backed environment in local Minikube) where Pods can run.
- **Pod:** the smallest deployable unit Kubernetes manages. A Pod contains one or more containers.
- **Container:** runs an application process using a container image.
- **Image:** the packaged template used to create a container.

### Visual hierarchy

```text
KUBERNETES CLUSTER
│
├── Control plane
│   ├── API server       ← entry point for Kubernetes API requests
│   ├── Scheduler       ← chooses a Node for a Pod
│   └── Controllers     ← work to keep actual state close to desired state
│
└── Node(s)
    └── Pod(s)
        └── Container(s)
            └── created from image(s)
```

**Remember:** A cluster can have multiple Nodes; each Node can run multiple Pods; each Pod can contain one or more containers. The same image can be used by many containers in different Pods or clusters.

### Can a Kubernetes cluster have multiple control-plane Nodes?

Yes. A Kubernetes cluster can have one or multiple Nodes running control-plane components.

Remember:
- Multiple control-plane Nodes can manage the **same Kubernetes cluster**.
- They do not represent multiple independent Kubernetes clusters.
- Having multiple control-plane Nodes helps avoid depending on a single Node and can provide **high availability**.

#### Understanding with a company analogy

Imagine a company with three managers.

If each manager has a completely different responsibility:

- Manager 1 → Finance
- Manager 2 → Marketing
- Manager 3 → Legal

This is a division of responsibilities.

However, imagine the company has three managers who are all capable of handling the company's management responsibilities.

Why have three managers instead of depending on only one?

To avoid depending on a single manager. If one manager becomes unavailable, another manager can take over certain responsibilities.

A similar idea is used in Kubernetes:

- Multiple control-plane Nodes run control-plane components.
- They work together to manage the same Kubernetes cluster.
- If one control-plane Node becomes unavailable, the remaining healthy components may continue managing the cluster, provided the cluster is configured for high availability.

#### What is leader election?

Even when multiple instances of a control-plane component are available, they do not always perform the same work simultaneously.

Think about our three managers:

- All three are capable of handling a particular responsibility.
- One manager takes the lead for that responsibility.
- The others remain available.
- If the leading manager becomes unavailable, another can take over.

Kubernetes uses a mechanism called **leader election** for certain control-plane components.

For example, multiple Scheduler instances can be available, but one is elected as the active leader. If that leader becomes unavailable, another instance can take over.

**Important distinction:**
- Multiple instances provide alternatives if one becomes unavailable.
- Leader election coordinates which instance actively performs a particular responsibility.
- High availability is the overall design that helps the cluster continue operating despite certain failures.

**Remember:** Multiple control-plane Nodes do not mean multiple clusters. They work together to manage one Kubernetes cluster.

#### Visual hierarchy — Multiple control-plane Nodes

```text
                 KUBERNETES CLUSTER HA (HA - High Availability)
                         |
                         |
          +--------------+--------------+
          |              |              |
       Node 1          Node 2          Node 3
    (Control plane) (Control plane) (Control plane)
          |              |              |
     +----+----+    +----+----+    +----+----+
     |    |    |    |    |    |    |    |    |
    API  Sched Ctrl API  Sched Ctrl API  Sched Ctrl
    Srv  uler  Mgr  Srv  uler  Mgr  Srv  uler  Mgr

          All work together to manage
             THE SAME CLUSTER
```

**How to read this diagram:**

1. There is only **one Kubernetes cluster**.
2. The cluster has three Nodes running control-plane components.
3. The control-plane components are replicated across these Nodes in this simplified example.
4. They work together to manage the same cluster.

**Why three Nodes?**

To avoid depending on a single Node. If one becomes unavailable, the remaining healthy components may continue managing the cluster, provided high availability is configured correctly.

**Remember:** Three control-plane Nodes do not mean three Kubernetes clusters.
One distinction to remember: having multiple control-plane Nodes is not automatically enough for HA; they must be configured to work together and tolerate failures.

### What happens when a control-plane Node becomes unavailable?

Suppose a Kubernetes cluster has multiple control-plane Nodes configured for high availability.

If one control-plane Node becomes unavailable:

- The cluster does not automatically become a new cluster.
- The remaining healthy control-plane components may continue managing the same cluster.
- This depends on the cluster's high-availability configuration and whether enough components remain available.

If all required control-plane services become unavailable, Kubernetes may no longer be able to manage the cluster properly.

However, this does not necessarily mean that existing application containers immediately stop running.

#### Running an application vs. managing an application

These are two different responsibilities.

**1. Running an application**

An application container that is already running on a healthy Worker Node may continue executing even when the control plane is unavailable.

For example, an existing Nginx Pod may continue serving HTTP requests.

**2. Managing an application**

The control plane is responsible for managing Kubernetes resources and maintaining the desired state.

If the API Server is unavailable:

- `kubectl` cannot communicate with the cluster through that API Server.
- New Kubernetes management requests, such as creating a Deployment, cannot be submitted successfully.
- Kubernetes controllers cannot reliably perform their normal reconciliation work through the unavailable API Server.

#### Does application traffic pass through the API Server?

**No.** The API Server handles Kubernetes API requests. It is not responsible for processing every application request.

There are two different communication paths:

**Kubernetes management communication:**

```text
User
  |
  v
kubectl
  |
  v
API Server
  |
  v
Kubernetes resources and management
```

Example:

```powershell
kubectl get pods
kubectl create deployment nginx --image=nginx
```

These commands require communication with the API Server.

**Application communication:**

```text
Browser / User
      |
      v
Service / Kubernetes Networking
      |
      v
Application Pod
      |
      v
Application Response
```

For example, when a user opens an Nginx website, the request reaches the Nginx Pod through the application's networking path. It does not need to pass through the API Server.

Therefore, an already-running Nginx Pod may continue serving requests even when the API Server is unavailable, provided the Worker Node, Pod, and networking remain healthy.

#### Example: What happens if the API Server crashes?

Assume:
- The API Server is unavailable.
- The Worker Node is healthy.
- An Nginx Pod is already running on that Worker Node.

| Activity | Expected behaviour |
|---|---|
| `kubectl get pods` | Cannot communicate with the unavailable API Server. |
| User opens the Nginx website | May continue working. |
| Nginx serves an HTTP request | Can continue normally. |
| Create a new Deployment using `kubectl` | Cannot submit the request successfully through the unavailable API Server. |
| Kubernetes manages workload changes | Management and reconciliation may be affected. |

**Remember:**

- API Server → Kubernetes management communication.
- Service / Networking → Application traffic.
- Existing application containers may continue running without the API Server handling every request.
- Kubernetes management operations require an available API Server.

### Why is a cluster useful?

A cluster gives Kubernetes a place to:
- schedule application Pods onto Nodes;
- monitor and manage workloads;
- restart or replace failed containers/Pods according to workload configuration;
- scale workloads and coordinate application networking;
- manage applications consistently through declarative resources such as Deployments and Services.

A cluster is not required for every application, but it is useful when you want Kubernetes to orchestrate workloads.

---

## 2. How do I create a Minikube cluster?

To create and start a cluster with the profile name `sample-cluster`:

```bash
minikube start -p sample-cluster
```

Command parts:

| Part | Meaning |
|---|---|
| `minikube` | The tool that manages a local Kubernetes cluster |
| `start` | Creates and starts the cluster if it does not exist; starts the existing cluster if it does |
| `-p` | Short for `--profile`; selects the Minikube profile name |
| `sample-cluster` | The profile name used to manage this cluster |

**Important:** Running the same command again normally starts the existing profile's cluster; it does not create another cluster with the same profile name.

### What gets set up with a cluster?

Minikube provisions a working Kubernetes environment, including:

- a Kubernetes **control plane** and its components;
- one or more **Nodes** (a default Minikube setup is usually single-node);
- Kubernetes configuration/context information so tools such as `kubectl` can connect;
- supporting local infrastructure required by the chosen Minikube driver.

With the **Docker driver**, Minikube typically uses a Docker container as the local Kubernetes Node. The cluster's Kubernetes components run within that environment, and the configured container runtime runs workload containers.

### Why are these pieces needed?

| Component | Why it is useful |
|---|---|
| API server | Receives Kubernetes API requests |
| Scheduler | Chooses a suitable Node for a newly created Pod |
| Controllers | Reconcile actual state with desired state |
| Node | Provides a place to run Pods |
| Container runtime | Starts and manages containers for Pods |
| Kubernetes context | Lets `kubectl` target the intended cluster/configuration |

**Creating a cluster does not automatically create your application Deployment or its Pods.** A fresh cluster can have system Pods, but your application workloads must be created separately.


### Minikube vs. `kubectl`

These are different command-line tools with different main responsibilities.

**Minikube** manages the lifecycle of local Minikube clusters:

```powershell
minikube start -p sample-cluster
minikube status -p sample-cluster
minikube stop -p sample-cluster
minikube dashboard -p sample-cluster
```

- `start`: creates and starts the profile's cluster if needed, or starts the existing cluster.
- `status`: checks the profile's cluster status.
- `stop`: stops the cluster without deleting the profile.
- `dashboard`: opens the Dashboard for the selected profile.

**`kubectl`** communicates with the Kubernetes API to inspect and manage resources inside a selected cluster:

```powershell
kubectl get nodes
kubectl get pods
kubectl get deployments
kubectl get services
```

For example, `kubectl --context=sample-cluster create deployment nginx-demo --image=nginx:latest` creates a Deployment in the `sample-cluster` context.

**Remember:** Minikube manages the local cluster lifecycle. Kubernetes manages resources within the cluster, and `kubectl` is the command-line tool used to interact with those resources.

---

## 3. Is a profile the same as a cluster?

**Not exactly.** They are closely connected, but they mean different things.

- **Cluster:** the actual Kubernetes environment.
- **Minikube profile:** Minikube's named way to manage a local cluster and its configuration.

Think of the profile as an **identity card and management record** for the cluster—not the cluster itself.

```text
Minikube profile: sample-cluster
          │
          └── manages/identifies ──> Kubernetes cluster
                                      ├── Control plane
                                      └── Node(s) → Pod(s)
```

When you run:

```bash
minikube start -p sample-cluster
```

Minikube creates and manages a cluster under that profile if it does not already exist. If the profile already exists, Minikube generally starts that cluster.

### Names that can look similar but mean different things

| Name/concept | Meaning | Command |
|---|---|---|
| Minikube profile | Minikube's profile name | `minikube profile list` |
| Kubernetes context | The `kubectl` connection selection (cluster, user, and optionally namespace) | `kubectl config current-context` |
| Kubernetes cluster entry | A cluster connection definition in kubeconfig | `kubectl config get-clusters` |
| Node name | Name of a Node inside the selected cluster | `kubectl get nodes` |

A profile and a context often share a name, such as `sample-cluster`, but don't assume they are identical concepts.


### Active Minikube profile vs. current Kubernetes context

These selections are related but independent.

- **Active Minikube profile:** the default profile used by Minikube commands when no profile is explicitly specified.
- **Current Kubernetes context:** the default cluster connection used by `kubectl` commands when no context is explicitly specified. A context can also include a user and namespace.

It is possible for the active Minikube profile to be `minikube` while the current Kubernetes context is `sample-cluster`. Changing one does not automatically change the other.

#### Check the active Minikube profile

```powershell
minikube profile
minikube profile list
```

#### Change the active Minikube profile

```powershell
minikube profile sample-cluster
```

To switch back:

```powershell
minikube profile minikube
```

Use `minikube profile list` to verify the selection in the **ACTIVE PROFILE** column.

#### Check or change the current Kubernetes context

```powershell
kubectl config current-context
kubectl config get-contexts
```

In `kubectl config get-contexts`, the `*` marks the current context.

Switch to `sample-cluster`:

```powershell
kubectl config use-context sample-cluster
```

Switch back to `minikube`:

```powershell
kubectl config use-context minikube
```

Verify with `kubectl config current-context`.

#### Target one cluster for a single command

Use `--context` without changing the current context:

```powershell
kubectl --context=minikube get deployments
kubectl --context=sample-cluster get deployments
```

Resources do not automatically move or appear in another cluster. The same image can be used in both clusters, but each cluster has its own Kubernetes resources.

---

## 4. How do I list and inspect existing clusters?

### List Minikube profiles

```bash
minikube profile list
```

Shows Minikube profiles and information such as their status.

### Check one cluster's status

```bash
minikube status -p sample-cluster
```

Checks the status of the named Minikube profile's cluster.

### See which context `kubectl` is using

```bash
kubectl config current-context
```

### List configured Kubernetes contexts

```bash
kubectl config get-contexts
```

### List cluster entries in kubeconfig

```bash
kubectl config get-clusters
```

### See cluster connection information

```bash
kubectl --context=sample-cluster cluster-info
```

### List Nodes in a specific cluster

```bash
kubectl --context=sample-cluster get nodes -o wide
```


Check the `STATUS` column. A Node marked `Ready` is reporting that it is ready to accept workloads.

Do not confuse `kubectl config get-contexts` with `kubectl get nodes`: the first lists configured contexts, while the second lists Nodes in the current context. Use `--context=PROFILE_OR_CONTEXT_NAME` when you want to target a specific cluster explicitly.

**Tip:** Use an explicit `--context=...` when you want to be certain which cluster a `kubectl` command targets.

---

## 5. How do I open the Dashboard for a chosen cluster?

For the `sample-cluster` profile:

```bash
minikube dashboard -p sample-cluster
```

For the original `minikube` profile:

```bash
minikube dashboard -p minikube
```

The `-p` flag selects the Minikube profile whose Dashboard should be launched. The command may open a browser tab or provide a URL, depending on the environment.

### What can I inspect in the Dashboard?

```text
Dashboard for the selected cluster
│
├── Nodes         → machines registered in that cluster
├── Workloads
│   ├── Pods      → running/created workload units
│   └── Deployments → desired replica management
├── Services      → stable network access to workloads
└── Namespaces    → logical groups for resources
```

**Important:** The standard Kubernetes Dashboard is not a universal screen listing all Minikube profiles with a click-to-switch control. Use `minikube profile list` to list Minikube profiles and `minikube dashboard -p PROFILE_NAME` to open the Dashboard for a chosen one.

---

## 6. What is the Dashboard “Cluster name” UI label for?

The Dashboard's **Cluster name** setting is a display label. It is separate from the Minikube profile name and is not a cluster selector.

- You can set a human-readable label, if you want.
- You can leave it blank if you don't need the label.
- Entering a name there does **not** create, rename, or switch a Minikube cluster.
- Minikube's `-p` flag selects the profile for the Dashboard command.

The label's exact display behavior can depend on the Dashboard version. Do not use that label as the authoritative way to identify the connected cluster; verify the profile/context using the terminal.

---

## 7. If I know an image name, how do I find the Pod name?

Suppose the image is `nginx:latest` and you want to search the `minikube` cluster.

### Option A — list Pod names and their container images

```bash
kubectl --context=minikube get pods -A -o jsonpath="{range .items[*]}{.metadata.namespace}{' '}{.metadata.name}{' '}{range .spec.containers[*]}{.image}{' '}{end}{'\n'}{end}"
```

This prints each Pod's namespace, Pod name, and image(s) specified for its containers. Then look for the row containing `nginx:latest`.

### What does each part mean?

| Command part | Meaning |
|---|---|
| `kubectl` | Kubernetes command-line tool |
| `--context=minikube` | Query the context named `minikube` |
| `get pods` | Request Pod resources |
| `-A` | Search all namespaces in that cluster |
| `-o jsonpath="..."` | Format the response using a JSONPath template |
| `{range .items[*]}` | Iterate through every Pod in the returned `.items` list |
| `{.metadata.namespace}` | Print the current Pod's namespace |
| `{.metadata.name}` | Print the current Pod's name |
| `{' '}` | Print a space |
| `{range .spec.containers[*]}` | Iterate through the current Pod's regular containers |
| `{.image}` | Print each container's configured image |
| `{end}` | End the nearest `range` loop |
| `{'\\n'}` | Print a newline |
| Final `{end}` | End the outer loop over Pods |

### Why use `.items[*]`?

The command asks Kubernetes for a JSON representation of the Pods. The returned object has an `items` list containing the Pod objects.

- `.items` means the list of returned Pods.
- `[*]` means every element in that list.
- `range` behaves like a loop: process one Pod at a time.

Conceptually:

```python
for pod in items:
    print(pod.namespace, pod.name)
    for container in pod.containers:
        print(container.image)
```

The `range` expression does not create the list; it iterates over the list returned by Kubernetes.

### Option B — filter for exactly `nginx:latest` using `jq`

If `jq` is installed, this prints only matching Pod namespaces and names:

```bash
kubectl --context=minikube get pods -A -o json | jq -r '.items[] | select(any(.spec.containers[]?; .image == "nginx:latest")) | "\(.metadata.namespace) \(.metadata.name)"'
```

This compares the image string exactly. An image such as `nginx:1.27` or `nginx` will not match `nginx:latest`.

If `jq` is not installed, use Option A and inspect the output.

### Can an image name identify the cluster by itself?

**No.** The same image can be used by many Pods, Nodes, and clusters. Search each cluster's Pods using its own context:

```bash
kubectl --context=minikube get pods -A -o wide
kubectl --context=sample-cluster get pods -A -o wide
```

A Pod's `NODE` column identifies its Node. The context used for the query identifies the cluster being inspected.

**Also remember:** An image may be downloaded onto a Node without any current Pod using it. Pod specifications show which images the Pod's containers are configured to use; they do not list every image cached on the Node.

---

## 8. Can I customise a Pod name?

Yes, when creating a standalone Pod, you can choose its name:

```bash
kubectl run my-nginx --image=nginx:latest
```

Here, `my-nginx` is the Pod name and `nginx:latest` is the image.

Pods created by a Deployment are usually named automatically, with generated name parts. For example:

```text
Deployment: gamehub                 ← Name you choose
└── ReplicaSet: gamehub-7abc123     ← Generated name
    └── Pod: gamehub-7abc123-x1y2z  ← Generated name
```

You can choose the name of a Deployment, for example:

```powershell
kubectl create deployment gamehub --image=nginx:latest
```

Here, `gamehub` is the Deployment name, not the Pod name. The ReplicaSet and Pod names are generated by Kubernetes.

| Resource | Can you choose its name with the basic command? |
|---|---|
| Standalone Pod created with `kubectl run my-nginx --image=nginx:latest` | Yes — `my-nginx` is the Pod name |
| Deployment created with `kubectl create deployment gamehub --image=nginx:latest` | Yes — `gamehub` is the Deployment name |
| ReplicaSet created by a Deployment | No — its name is generated |
| Pod created by a Deployment | No — its name is generated |

### What happens to the Pod name if a Pod is replaced?

If a Pod managed by a Deployment/ReplicaSet is replaced, the replacement Pod can have a different name even though the Deployment name remains the same.

```text
Deployment: gamehub                    ← Same name
└── ReplicaSet: gamehub-7abc123        ← Same in this example
    ├── Old Pod: gamehub-7abc123-x1y2z  ← Crashed/removed
    └── New Pod: gamehub-7abc123-p9q8r  ← Replacement
```

The Deployment-related prefix usually remains the same when the same ReplicaSet creates the replacement, while the Pod-specific suffix changes. If a Deployment update creates a new ReplicaSet, the ReplicaSet-related part can change too.

**Important distinction:** A standalone Pod does not automatically get recreated just because it crashes. A Deployment manages Pods through a ReplicaSet and works to maintain the desired number of replicas.

Because Pods can be replaced, applications should generally be reached through a Service rather than by relying on a specific Pod name or Pod IP.

---

## 9. Quick reference: common commands

| Goal | Command |
|---|---|
| Create/start a named profile's cluster | `minikube start -p sample-cluster` |
| List Minikube profiles | `minikube profile list` |
| Show active Minikube profile | `minikube profile` |
| Change active Minikube profile | `minikube profile sample-cluster` |
| Show current Kubernetes context | `kubectl config current-context` |
| List Kubernetes contexts | `kubectl config get-contexts` |
| Switch Kubernetes context | `kubectl config use-context sample-cluster` |
| Check a profile's status | `minikube status -p sample-cluster` |
| Open its Dashboard | `minikube dashboard -p sample-cluster` |
| Show current context | `kubectl config current-context` |
| List contexts | `kubectl config get-contexts` |
| List kubeconfig cluster entries | `kubectl config get-clusters` |
| Show cluster information | `kubectl --context=sample-cluster cluster-info` |
| List Nodes | `kubectl --context=sample-cluster get nodes -o wide` |

| List Deployments in a specific cluster | `kubectl --context=sample-cluster get deployments` |
| List Pods in all namespaces in a specific cluster | `kubectl --context=sample-cluster get pods -A` || List Pods in all namespaces | `kubectl --context=sample-cluster get pods -A -o wide` |
| List Pods with image names | Use the JSONPath command in Section 7 |

---

## 10. Final mental model

```text
MINIKUBE
│
├── Profile: minikube
│   └── Kubernetes Cluster
│       ├── Control plane
│       └── Node(s)
│           └── Pod(s)
│               └── Container(s) ← image(s)
│
└── Profile: sample-cluster
    └── Separate Kubernetes Cluster
        ├── Control plane
        └── Node(s)
            └── Pod(s), if workloads are created
```

- **Profile:** Minikube's named management identity for a local cluster.
- **Cluster:** the actual Kubernetes environment.
- **Context:** tells `kubectl` which configured cluster connection to use.
- **Dashboard:** a UI for exploring resources in the selected cluster.
- **Image → Pod search:** inspect Pod container specifications in each cluster; the image name alone does not identify one unique cluster.
