# Kubernetes Clusters with Minikube — Beginner Notes

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

Pods created by a Deployment are usually named automatically, with a generated suffix. For example:

```text
hello-minikube-fb6fc86f8-44lwx
└──── Deployment/ReplicaSet-related name ────┘
```

You normally change the Deployment's name or configuration rather than manually naming each Pod it creates.

---

## 9. Quick reference: common commands

| Goal | Command |
|---|---|
| Create/start a named profile's cluster | `minikube start -p sample-cluster` |
| List Minikube profiles | `minikube profile list` |
| Check a profile's status | `minikube status -p sample-cluster` |
| Open its Dashboard | `minikube dashboard -p sample-cluster` |
| Show current context | `kubectl config current-context` |
| List contexts | `kubectl config get-contexts` |
| List kubeconfig cluster entries | `kubectl config get-clusters` |
| Show cluster information | `kubectl --context=sample-cluster cluster-info` |
| List Nodes | `kubectl --context=sample-cluster get nodes -o wide` |
| List Pods in all namespaces | `kubectl --context=sample-cluster get pods -A -o wide` |
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

