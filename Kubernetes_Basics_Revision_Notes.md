# Kubernetes Basics - Revision Notes

*A beginner-friendly guide using the GameHub application*

These notes summarize the Kubernetes concepts we discussed, with simple
explanations, examples, and diagrams. Read them when you want to refresh
your memory without watching the lesson again.

------------------------------------------------------------------------

## 1. The big picture: GameHub

Imagine GameHub has these parts:

-   **Frontend** --- the website you see in your browser.
-   **Auth service** --- handles login and authentication.
-   **Game service** --- handles game-related features.
-   **PostgreSQL** --- stores application data.

Kubernetes helps run and manage these parts.

A simplified application flow is:

``` text
Your browser
    |
    v
  Frontend
    |
    +------> Auth service
    |
    +------> Game service
                 |
                 v
             PostgreSQL
```

The diagram shows the application's logical flow. In Kubernetes, these
parts are usually run in separate Pods so that they can be managed and
scaled independently.

## 2. Pod, Node, and Cluster

### Pod

A **Pod** is the smallest deployable unit in Kubernetes. A Pod contains
one or more containers.

For our beginner GameHub setup, we could use:

-   One Pod for the frontend container.
-   One Pod for the Auth service container.
-   One Pod for the Game service container.
-   One Pod for PostgreSQL, with appropriate persistent storage
    configured.

Related application components do not have to share one Pod. Separate
Pods make it easier to update and scale services independently.

### Node

A **Node** is a computer or virtual machine that runs Pods. Kubernetes
schedules Pods onto Nodes.

One Node can run multiple Pods.

### Cluster

A **Cluster** is a Kubernetes environment that manages its Nodes
together. A cluster normally has a control plane and worker Nodes. A
local learning cluster can also have just one Node.

### Diagram: how they fit together

``` text
KUBERNETES CLUSTER
|
+-- Node 1 (a computer or VM)
|   |
|   +-- Frontend Pod
|   +-- Auth Pod
|
+-- Node 2 (another computer or VM)
    |
    +-- Game service Pod
    +-- PostgreSQL Pod
```

This is an example only. Kubernetes could place the Pods on different
Nodes.

**Important:** Your laptop is not automatically a Kubernetes Node. It is
usually the client you use to access the application. It would be a Node
only if it were configured to participate in the Kubernetes cluster.

For local practice, tools such as Minikube can create a Kubernetes
cluster on your own computer. A multi-node local cluster may use
containers or virtual machines as Nodes, but those Nodes still depend on
the same physical computer.

## 3. Deployment and ReplicaSet

### Deployment

A **Deployment** describes the desired state of an application and
manages its releases. For example, you might ask Kubernetes to run three
replicas of the frontend using a particular container image.

A Deployment helps manage updates to the application, including rolling
updates and rollbacks.

### ReplicaSet

A **ReplicaSet** makes sure the requested number of matching Pods are
running.

Example: if the desired number is three and one Pod disappears, the
ReplicaSet works to create a replacement.

### Diagram

``` text
Deployment
    |
    v
ReplicaSet (maintains the desired number of Pods)
    |
    +------> Frontend Pod 1
    +------> Frontend Pod 2
    +------> Frontend Pod 3
```

In normal use, you create and update a Deployment; Kubernetes manages
the ReplicaSet for you.

**Remember:** 
- Deployment = manages the app's desired state and releases. 
- ReplicaSet = maintains the desired number of Pods.

## 4. Service: a stable way to reach Pods

Pods can be replaced. When a Pod is replaced, its IP address can change.

A **Service** provides a stable way to reach a set of Pods. Instead of
having the frontend depend on the current IP address of a backend Pod,
the frontend can contact a Service.

For example:

``` text
Frontend Pod
    |
    | sends request to the backend Service
    v
Backend Service
    |
    v
A matching Backend Pod
```

The Service selects matching Pods and routes traffic to them. The
frontend does not need to track each backend Pod's changing IP address.

------------------------------------------------------------------------

## 5. The key difference: ClusterIP vs NodePort

This was the most confusing part, so keep the explanation simple.

### ClusterIP --- internal access

**ClusterIP is the default Kubernetes Service type.** It gives a Service
an internal IP address that is normally reachable from within the
cluster.

GameHub example: the frontend needs to call the backend to sign a user
in. The frontend can call the backend's ClusterIP Service.

``` text
Inside the Kubernetes cluster

Frontend Pod
     |
     | calls backend Service
     v
ClusterIP Service
     |
     v
Backend Pod
```

The frontend does not need to know which Node is running the backend
Pod.

ClusterIP is for internal cluster communication; it does not normally
provide direct access from your laptop's browser.

### NodePort --- access through a Node

**NodePort exposes a Service on a port on the cluster's Nodes.** A
client that can reach a Node's IP address and the assigned port can use
that entry point.

GameHub example: you want to open the frontend from your laptop's
browser. NodePort is one possible way to expose the frontend Service.

``` text
Outside the cluster

Your laptop's browser
          |
          | connects to Node IP + NodePort
          v
     Kubernetes Node
          |
          v
      NodePort Service
          |
          v
      Frontend Pod
```

Your laptop is the **client** in this example, not automatically the
Node. The Node is a machine that participates in the Kubernetes cluster.

### Side-by-side comparison

  -----------------------------------------------------------------------
  Question                ClusterIP               NodePort
  ----------------------- ----------------------- -----------------------
  Main purpose            Internal access to a    Access to a Service
                          Service                 through a Node

  GameHub example         Frontend calls backend  Laptop's browser
                                                  accesses frontend

  Uses a Node's IP and    No, not for normal      Yes
  NodePort?               ClusterIP access        

  Is it a Kubernetes      Yes                     Yes
  Service type?                                   
  -----------------------------------------------------------------------

### The simplest memory trick

-   **Frontend → Backend:** usually use a ClusterIP Service.
-   **Laptop/browser → Frontend:** NodePort can be used to expose the
    frontend.
-   **NodePort is one way** to expose an app; it is not the only way.

These are typical examples, not strict rules about which kind of client
is allowed to make a request. The key distinction is whether the Service
is internally exposed or exposed through a Node's port.

## 6. Putting the concepts together

Here is a simplified picture of GameHub running in Kubernetes:

``` text
YOUR LAPTOP
Browser
   |
   | access through NodePort (one possible method)
   v
KUBERNETES CLUSTER
   |
   v
Frontend Service
   |
   v
Frontend Pod
   |
   | calls internal backend Service
   v
ClusterIP Service
   |
   v
Backend Pod(s)
   |
   +------> Auth service Pod(s)
   |
   +------> Game service Pod(s)
                 |
                 v
            PostgreSQL Pod
            + persistent storage
```

This diagram is conceptual: the precise arrangement of Services and Pods
depends on the application's configuration. In a real deployment, Auth,
Game, and PostgreSQL would normally have their own appropriate Services
and workload/storage configuration.

```
YOUR LAPTOP
+--------------------------------+
|  Browser                       |
|  http://<minikube-ip>:<port>    |
+--------------------------------+
              |
              | Request enters
              v
+----------------------------------------+
|          MINIKUBE CLUSTER              |
|                                        |
|   +--------------------------------+   |
|   | Kubernetes Node (Minikube)     |   |
|   |                                |   |
|   |   NodePort Service             |   |
|   |          |                     |   |
|   |          v                     |   |
|   |      Frontend Pod              |   |
|   +--------------------------------+   |
|                                        |
+----------------------------------------+
```

## 7. Quick revision

  Kubernetes concept   Remember it as
  -------------------- -----------------------------------------------
  Pod                  The unit that contains one or more containers
  Node                 A computer or VM that runs Pods
  Cluster              Kubernetes environment managing Nodes
  Deployment           Manages desired app state and releases
  ReplicaSet           Maintains the desired number of Pods
  Service              Stable way to reach matching Pods
  ClusterIP            Internal Service access
  NodePort             Service access through a Node's port

## 8. Quick self-test

Try answering before looking at the answers.

1.  Can one Node run multiple Pods?
2.  What works to create a replacement when one of three desired Pods
    disappears?
3.  Why is a Service useful when Pod IP addresses can change?
4.  In GameHub, which Service type is normally enough for the frontend
    to call the backend?
5.  Is your laptop automatically a Kubernetes Node when you open GameHub
    in a browser?

ANSWERS

1.  Yes.
2.  The ReplicaSet, managed by a Deployment in the usual setup.
3.  It provides a stable way to reach the matching Pods.
4.  ClusterIP.
5.  No. The laptop is normally the client. A Kubernetes Node is a
    machine participating in the cluster.

## 9. What to learn next

You do **not** need to master every Kubernetes concept before starting
hands-on practice. A sensible next sequence is:

1.  Connect to your local cluster and inspect it with `kubectl`.
2.  Deploy a simple sample application.
3.  Learn the basic YAML fields in a Deployment and Service.
4.  Access the app and check which Service type is being used.
5.  Practise scaling, logs, troubleshooting, rolling updates, and
    rollbacks.
6.  Later, learn ConfigMaps, Secrets, and persistent storage for your
    GameHub services and database.

**Learning tip:** Practise one command or concept at a time. First
understand what the command changes, then run it, inspect the output,
and explain what happened in your own words.
