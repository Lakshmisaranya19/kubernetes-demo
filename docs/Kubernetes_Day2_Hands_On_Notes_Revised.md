# Kubernetes Day 2 - Revised Beginner Notes

**Environment:** Windows laptop, PowerShell, Minikube using the Docker
driver, Kubernetes.\
**Sample workload:** Nginx.

> Examples are illustrative unless explicitly labelled as output from
> your session. Generated names, IP addresses, ports, and ages can vary.

## 1. First understand how your local cluster exists

Installing Minikube installs the `minikube` command-line tool; it does
**not by itself create a running cluster**. The cluster is
created/started when you run `minikube start` (for example,
`minikube start --driver=docker`). Your cluster was already running when
we began Day 2, so the Deployment command did not create it.

Check the environment:

``` powershell
minikube status
kubectl get nodes
```

-   `minikube status` checks Minikube's environment and component
    status.
-   `kubectl get nodes` asks Kubernetes to list its Nodes.

### What is the Node in *your* setup?

Your physical Windows laptop provides the hardware and runs Windows.
Docker Desktop's Docker Engine provides container functionality. Because
you use Minikube's **Docker driver**, Minikube runs the Kubernetes Node
as a Docker container. So the Node named `minikube` is **not your entire
laptop** and is **not a separate traditional VM** in this setup; it is a
Docker container acting as the Kubernetes Node.

Simplified view:

``` text
Physical Windows laptop
└── Docker Desktop / Docker Engine
    └── Minikube Node (Docker container)
        ├── Kubernetes control-plane components
        ├── kubelet
        ├── containerd (runtime for Kubernetes workload containers)
        └── Pod(s)
            └── Nginx container
```

This is a conceptual diagram; implementation details can vary by
version/configuration.

### What is a cluster and what is a Node?

-   **Cluster:** The complete Kubernetes environment: its control plane
    and one or more Nodes.
-   **Node:** A machine/environment where Kubernetes runs Pods.
    Depending on the setup, it can be physical, virtual, or---in this
    Minikube Docker-driver setup---a Docker container acting as a Node.
-   Your current Minikube cluster has one Node named `minikube`. It can
    run both control-plane components and workloads.

### What are Docker and containerd?

A **container runtime** is software that creates and runs containers for
Kubernetes. **containerd** is a container runtime that handles container
lifecycle tasks.

Docker is a broader container platform/toolset; Docker Desktop provides
Docker Engine on your Windows system. Docker also uses containerd
internally, but Docker and containerd are not the same thing.

In your setup, distinguish two jobs:

1.  **Docker Engine on Windows:** Minikube uses Docker as the *driver*
    to run the Minikube Node container.
2.  **containerd inside the Minikube Node:** Kubernetes uses the Node's
    runtime to run Pod workload containers, such as Nginx.

-   **Driver** = how Minikube creates/runs its Node environment.
-   **Container runtime** = what Kubernetes uses on a Node to run Pod
    containers.

## 2. What is a Deployment?

You already know that a container runs an application process, a Pod
wraps one or more containers, a Node runs Pods, and a cluster contains
the Kubernetes control plane and Nodes.

A **Deployment** is a Kubernetes resource that declares and manages the
desired state of an application workload. For example, it can specify
that one replica of an application should run from a particular
container image. Kubernetes controllers work to make the actual state
match that desired state. A Deployment also supports controlled
updates/rollouts.

The common relationship is:

``` text
Deployment
└── ReplicaSet
    └── Pod
        └── Container running Nginx
```

-   **Deployment:** Declares the desired state and manages rollouts.
-   **ReplicaSet:** Maintains the desired number of matching Pods.
-   **Pod:** Runs the container(s).
-   **Container:** Runs the application process.

The command below creates a **Deployment for an existing application
image**; it does not create the Nginx application itself.

## 3. Create the Deployment

``` powershell
kubectl create deployment hello-minikube --image=nginx:latest
```

Term by term:

-   `kubectl`: Kubernetes command-line client.
-   `create`: Request creation of a resource.
-   `deployment`: Resource type to create.
-   `hello-minikube`: Name of the Deployment.
-   `--image=nginx:latest`: Image to use in the Pod template; `nginx` is
    the image name and `latest` is its tag. For repeatable production
    deployments, a specific version or immutable digest is usually safer
    than relying on `latest`.

Why: This asks Kubernetes to create a Deployment. By default, it
requests one replica; Kubernetes manages the ReplicaSet and Pod needed
to run it.

Illustrative output:

``` text
deployment.apps/hello-minikube created
```

### Did it create a local YAML file?

No. The command creates a **live Deployment resource in the cluster**,
not a `.yaml` file on your computer. Kubernetes stores cluster state in
its backing datastore (etcd). The Dashboard's **Edit** view can display
the live resource configuration as YAML; that does not mean a local file
was saved. You can separately write a manifest to a file if you want to
manage it locally.

## 4. Namespaces and `kubectl get all`

A **namespace** is a logical partition within a Kubernetes cluster. It
helps group/separate resources, for example by team or environment.
Unless you specify another namespace, your commands have been using
`default`.

Examples of **namespaced resources**: - Deployments - ReplicaSets -
Pods - Services

These resources belong to a namespace. Two namespaces can each contain a
Deployment with the same name.

**Nodes are cluster-scoped**, not namespaced. A Node belongs to the
cluster as a whole.

``` powershell
kubectl get all
```

This lists selected common resource types in the current namespace.
Despite its name, it does **not** literally list every Kubernetes
resource. Nodes are not included because they are cluster-scoped; use
`kubectl get nodes` to list them.

To explicitly select a namespace:

``` powershell
kubectl get pods -n default
```

`-n` means `--namespace`.

### Your `kubectl get all` output

``` text
NAME                                 READY   STATUS    RESTARTS   AGE
pod/hello-minikube-fb6fc86f8-44lwx   1/1     Running   0          10m

NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   5d16h

NAME                             READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/hello-minikube   1/1     1            1           10m

NAME                                       DESIRED   CURRENT   READY   AGE
replicaset.apps/hello-minikube-fb6fc86f8   1         1         1       10m
```

-   `pod/...`: Pod running Nginx.
-   `READY 1/1`: One container is ready out of one.
-   `STATUS Running`: Pod is running.
-   `RESTARTS 0`: No container restarts recorded for this Pod.
-   `deployment.apps/...`: Deployment.
-   `UP-TO-DATE`: Replicas using the latest Deployment template.
-   `AVAILABLE`: Replicas considered available.
-   `replicaset.apps/...`: ReplicaSet maintaining the Pod.
-   `service/kubernetes`: Built-in ClusterIP Service for access to the
    Kubernetes API; it is not the Service we later create for Nginx.

### List resources individually

``` powershell
kubectl get deployments
kubectl get replicasets
kubectl get pods -o wide
kubectl get nodes
```

-   `kubectl get deployments` lists Deployments in the current
    namespace.
-   `kubectl get replicasets` lists ReplicaSets in the current
    namespace. If there are multiple Deployments, their ReplicaSets can
    all appear. Use the exact ReplicaSet name to query one.
-   `kubectl get pods -o wide` lists Pods with extra columns. `-o` means
    output format; `wide` includes details such as Pod IP and Node.
-   `kubectl get nodes` lists Nodes at cluster scope.

Your actual `kubectl get pods -o wide` output:

``` text
NAME                             READY   STATUS    RESTARTS   AGE   IP           NODE       NOMINATED NODE   READINESS GATES
hello-minikube-fb6fc86f8-44lwx   1/1     Running   0          51m   10.244.0.3   minikube   <none>           <none>
```

Extra columns: - `IP: 10.244.0.3`: Pod IP inside the cluster network;
not the Service's ClusterIP. - `NODE: minikube`: The Node hosting the
Pod. In your setup, this is the Docker container acting as the Node. -
`NOMINATED NODE: <none>`: No Node is nominated for a special
scheduling/preemption decision; normal for this running Pod. -
`READINESS GATES: <none>`: No extra custom readiness gates are
configured; also normal.

## 5. Kubernetes Dashboard

To check whether the Dashboard add-on is available/enabled:

``` powershell
minikube addons list
```

Look for `dashboard` and its status; exact formatting depends on
Minikube version.

To start/open the Dashboard:

``` powershell
minikube dashboard
```

This can enable/start the Dashboard and open a local proxy URL. Keep the
terminal open if the command requires it.

In the Dashboard: 1. Select namespace `default`. 2. Open **Workloads →
Deployments** to see `hello-minikube`. 3. Open **Workloads → Pods** to
inspect the Pod. 4. Open the **Services** section to inspect Services.
5. **Edit** can show the live resource configuration as YAML; it does
not mean a local YAML file was automatically created.

Sidebar labels can vary by Dashboard version.

## 6. Create a Service for Nginx

A **Service** provides a stable network endpoint for a set of Pods. Pod
IPs can change when Pods are replaced, so other workloads should
generally reach them through a Service instead of depending on a
particular Pod IP.

### Two Service types to understand

**ClusterIP (internal access):** - The default Service type. - Provides
a stable internal IP for access from within the cluster. - Example: a
frontend Pod can contact a backend Service by name without needing to
know the backend Pod's current IP. - It is not directly exposed to
outside clients by default.

**NodePort (access through a Node port):** - Allocates a port on the
Node and routes traffic arriving at that port to the Service's selected
Pods. - A NodePort Service also normally has a ClusterIP for internal
access. - NodePort does not guarantee public internet access;
reachability depends on cluster and network configuration.

### Command used

``` powershell
kubectl expose deployment hello-minikube --type=NodePort --port=80 --target-port=80
```

Term by term:

-   `kubectl`: Kubernetes CLI.
-   `expose`: Creates a Service for the selected resource.
-   `deployment hello-minikube`: Deployment whose Pods the Service
    targets.
-   `--type=NodePort`: Chooses NodePort Service type.
-   `--port=80`: Port on the Service.
-   `--target-port=80`: Destination port on the selected Pods; Nginx
    listens on port 80 by default.

Illustrative output:

``` text
service/hello-minikube exposed
```

This command creates a **Service**, not another Pod, Deployment, Node,
or cluster.

### Service port vs target port vs NodePort

``` text
Client -> Service port 80 -> Pod target port 80
```

-   **Service port:** Port clients address on the Service.
-   **Target port:** Destination port on the selected Pod.
-   **NodePort:** Port allocated on the Node for a NodePort Service.

They happen to use port 80 for the Service and target port in this
example, but they are different concepts.

## 7. Inspect Services: TYPE vs CLUSTER-IP

``` powershell
kubectl get services
```

Your actual output:

``` text
NAME             TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
hello-minikube   NodePort    10.102.177.26   <none>        80:32339/TCP   2m6s
kubernetes       ClusterIP   10.96.0.1       <none>        443/TCP        5d16h
```

### What is the difference between `TYPE` and `CLUSTER-IP`?

-   **TYPE** describes the Service's exposure behavior. `ClusterIP`
    means internal access by default; `NodePort` means a Node port is
    allocated as well.
-   **CLUSTER-IP** is the virtual internal IP assigned to the Service.

A NodePort Service normally has a ClusterIP too. Therefore, your
`hello-minikube` row correctly says `TYPE = NodePort` while showing
`CLUSTER-IP = 10.102.177.26`.

Interpret `80:32339/TCP`: - `80`: Service port. - `32339`: NodePort
assigned in your session. - `TCP`: Transport protocol.

`10.102.177.26` is the Service ClusterIP; it is different from your Pod
IP `10.244.0.3`. `<none>` under `EXTERNAL-IP` is normal for this Service
and does not mean it failed.

In the Dashboard, choose namespace `default`, open **Services**, and
select `hello-minikube` to inspect it.

## 8. Open Nginx in your browser

### Command

``` powershell
minikube service hello-minikube --url
```

Terms: - `minikube`: Minikube CLI. - `service`: Requests access
information for a Kubernetes Service. - `hello-minikube`: Service
name. - `--url`: Prints a URL for reaching the Service.

Your actual output:

``` text
http://127.0.0.1:57352
! Because you are using a Docker driver on windows, the terminal needs to be open to run it.
```

You opened the URL in your browser and saw the **Nginx welcome page**.

### Why is port `57352` different from NodePort `32339`?

-   `32339` is the Kubernetes NodePort allocated to the Service.
-   `57352` is the local port in the URL Minikube printed for your
    Windows browser.
-   With Minikube's Docker driver on Windows, Minikube provides a local
    access/forwarding mechanism to reach the Service.

The command does **not** create another Service or change the Service
type. Your Kubernetes Service remains a **NodePort** Service. The
command gives you a convenient local URL for reaching it. Keep that
PowerShell terminal open while using the URL if Minikube prints the
warning.

## 9. End-to-end picture

``` text
Windows browser
  |
  | Local URL from `minikube service ... --url`
  | Example: http://127.0.0.1:57352
  v
Minikube local access/forwarding mechanism
  |
  v
Service: hello-minikube (TYPE NodePort)
  | Service port: 80
  | NodePort: 32339 (your session)
  | ClusterIP: 10.102.177.26 (your session)
  v
Pod: hello-minikube-fb6fc86f8-44lwx
  | Pod IP: 10.244.0.3 (your session)
  | Node: minikube
  v
Nginx container (target port 80)
  |
  v
Nginx welcome page
```

## 10. Command reference

  ---------------------------------------------------------------------------------------------------------------------------
  Command                                                                                 Purpose
  --------------------------------------------------------------------------------------- -----------------------------------
  `minikube status`                                                                       Check Minikube status

  `minikube addons list`                                                                  Check Dashboard add-on status

  `minikube dashboard`                                                                    Start/open the Dashboard

  `kubectl get nodes`                                                                     List cluster Nodes

  `kubectl create deployment hello-minikube --image=nginx:latest`                         Create the Deployment

  `kubectl get deployments`                                                               List Deployments in the current
                                                                                          namespace

  `kubectl get replicasets`                                                               List ReplicaSets in the current
                                                                                          namespace

  `kubectl get pods -o wide`                                                              List Pods with Pod IP and Node
                                                                                          details

  `kubectl get all`                                                                       List selected common resource types
                                                                                          in the current namespace

  `kubectl expose deployment hello-minikube --type=NodePort --port=80 --target-port=80`   Create the NodePort Service

  `kubectl get services`                                                                  List Services, types, ClusterIPs,
                                                                                          and ports

  `minikube service hello-minikube --url`                                                 Print a local URL for accessing the
                                                                                          Service
  ---------------------------------------------------------------------------------------------------------------------------

## Quick Recap - How URL, NodePort, Service Port, and Target Port Work Together

When I deployed Nginx using Kubernetes, I created a Deployment, then a Service, and finally accessed Nginx through a URL provided by Minikube.

### Request flow: From my browser to Nginx

**1. URL — The address I type into my browser**

Example from my practice:

`http://127.0.0.1:57352`

The URL is the address I open in my browser to access the Nginx webpage.

**2. NodePort — The entry port on the Kubernetes Node**

My NodePort was `32339`.

NodePort exposes the Service through a port on the Kubernetes Node, allowing traffic to reach the Service.

**3. Service Port — The port exposed by the Service**

My Service port was `80`.

The Service provides a stable network endpoint and directs traffic to the appropriate Pod.

**4. Target Port — The destination port inside the Pod**

My target port was `80`.

Nginx listens on port `80` inside the Pod, so the Service forwards traffic to that port.

**5. Nginx Pod — Where my application runs**

The Nginx container runs inside the Pod. It processes the HTTP request and returns the Nginx welcome page to my browser.

### Visual flow

```text
1. Browser
   |
   | Opens the Minikube URL
   | http://127.0.0.1:57352
   v
2. Minikube local access / forwarding
   |
   | Reaches the Kubernetes NodePort
   v
3. NodePort: 32339
   |
   v
4. Service Port: 80
   |
   | Service forwards traffic to the Pod
   v
5. Target Port: 80
   |
   v
6. Nginx container inside the Pod
   |
   v
   Nginx welcome page returned to the browser
```

*Note: This is a simplified request flow. In my Windows setup, Minikube provides local forwarding to reach the NodePort. The URL port `57352` is not the NodePort `32339`.*

### My actual Service output

```text
NAME             TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)
hello-minikube   NodePort   10.102.177.26   <none>        80:32339/TCP
```

How to read `80:32339/TCP`:

- `80` = Service port.
- `32339` = NodePort.
- `TCP` = Network transport protocol.

### My command

```bash
kubectl expose deployment hello-minikube --type=NodePort --port=80 --target-port=80
```

- `expose deployment hello-minikube` = Create a Service for the Deployment's Pods.
- `--type=NodePort` = Expose the Service through a port on the Node.
- `--port=80` = Set the Service port to `80`.
- `--target-port=80` = Forward traffic to port `80` on the Pod, where Nginx listens.

### Important distinction

| Item | My value | Purpose |
|---|---:|---|
| Minikube URL port | `57352` | Local browser access through Minikube |
| NodePort | `32339` | Entry port on the Kubernetes Node |
| Service port | `80` | Port exposed by the Service |
| Target port | `80` | Destination port on the Pod |

**Remember:** The URL is the address I open. NodePort provides an entry point on the Node. The Service port is the port exposed by the Service. The target port is where the application receives traffic inside the Pod.

The Service provides a stable endpoint even if a Pod is replaced and its IP address changes. The Deployment manages the desired number of Pods; the Service routes traffic to matching Pods.


## Day 2 completion summary

**Completed:** Verified the Deployment, ReplicaSet, and Pod; created and
inspected the NodePort Service; opened the Nginx welcome page in the
browser.

**Key takeaways:** - Installing Minikube is different from starting a
Minikube cluster. - In this setup, the Minikube Node is a Docker
container, not the whole laptop. - Docker is Minikube's driver;
containerd inside the Node runs Kubernetes workload containers. - A
Deployment declares and manages desired application state. - Namespaces
logically partition namespaced resources; Nodes are cluster-scoped. -
`TYPE` and `CLUSTER-IP` are different Service fields; NodePort Services
also have a ClusterIP. - Service port, target port, NodePort, Pod IP,
ClusterIP, and Minikube's local URL are distinct.
