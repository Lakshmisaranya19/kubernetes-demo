# GameHub Kubernetes Deployment — Learning Notes

**Last updated:** 30 October 2026  
**Current checkpoint:** Frontend Deployment created; Nginx startup issue diagnosed  
**Minikube profile:** `gamehub-cluster`

---

## 1. Project goal

Deploy the existing GameHub application, previously run with Docker Compose, to a local Kubernetes cluster using Minikube. We are proceeding one component at a time, inspecting each resource and understanding errors before moving forward.

## 2. GameHub application architecture

| Component | Technology | Container port | Purpose |
|:--|:--|--:|:--|
| Frontend | React, Tailwind, Nginx | 80 | User interface and reverse proxy |
| Auth service | Python Flask, JWT, Gunicorn | 8080 | Registration and login |
| Game service | Python Flask, Gunicorn | 8081 | Game statistics and scores |
| Database | PostgreSQL 13 | 5432 | Application data |

The frontend's Nginx configuration proxies requests as follows:

| Frontend path | Upstream hostname | Upstream port |
|:--|:--|--:|
| `/api/auth/*` | `auth-service` | 8080 |
| `/api/game/*` | `game-service` | 8081 |

Docker Compose provides DNS resolution for Compose service names on its network. Kubernetes uses Kubernetes Services and cluster DNS to provide stable names for backend workloads.

The database is initialized using the project's `init.sql`. The Compose setup uses a persistent `pgdata` volume. Kubernetes database configuration and persistence are still pending.

## 3. Environment and cluster

| Component | Recorded value |
|:--|:--|
| Operating system | Windows 11 |
| Docker Engine | `29.9.2` |
| Docker Desktop | `4.97.0` |
| Minikube | `v1.39.0` |
| Kubernetes | `v1.37.0` |
| Minikube container runtime | `containerd://2.3.4` |
| Active Minikube profile | `gamehub-cluster` |
| Current kubectl context | `gamehub-cluster` |
| Kubernetes node name | `gamehub-cluster` |
| Number of nodes | 1 |

Project directory in PowerShell:

```powershell
C:\Users\Ara\OneDrive\Desktop\k8s-project\Kubernetes-crash-course-2025
```

### 3.1 Cluster name versus node name

These are different concepts, even though they currently have the same value.

| Name | What it identifies | Current value |
|:--|:--|:--|
| Minikube profile name | The Minikube cluster/profile managed by Minikube | `gamehub-cluster` |
| Kubernetes node name | A machine registered with Kubernetes | `gamehub-cluster` |

This is a single-node cluster, so it has one Kubernetes Node object. Minikube commonly uses the profile name as the node name.

**Can the node be named `gamehub-node`?** Yes. The kubelet supports a hostname override that controls the node identity it registers with. Minikube supports Kubernetes component customization through `--extra-config`. However, this is not a simple edit to the existing Node object: the kubelet must register using the new identity. For an existing Minikube cluster, that generally requires reconfiguring/recreating the node or cluster.

We are keeping the current name. Renaming it has no functional benefit for GameHub and is not worth disturbing the working cluster.

Useful checks:

```powershell
minikube profile list
kubectl config current-context
kubectl get nodes -o wide
```

### 3.2 Docker Desktop and Minikube use separate runtime environments

Docker Desktop's Docker Engine/image store and the container runtime inside Minikube are separate environments. The Minikube node reports containerd as its Kubernetes runtime. A host-side `docker images` listing does not automatically mean the image exists inside Minikube. We pushed the custom images to Docker Hub so Kubernetes can pull them.

## 4. Rebuild the application images

The project already contained Dockerfiles and a Docker Compose configuration. We changed to the project root and rebuilt the images:

```powershell
cd C:\Users\Ara\OneDrive\Desktop\k8s-project\Kubernetes-crash-course-2025
docker compose build
```

The three custom images are frontend, auth service, and game service. The database uses the upstream `postgres:13` image.

### Issue: Compose command run from the wrong directory

**Problem:** Docker Compose initially reported that no configuration file was provided.

**Reason:** The command was run from `C:\Users\Ara`, while the Compose file is in the GameHub project directory.

**Resolution:** Change to the project root and run Compose commands there.

## 5. Docker Hub repository, tagging, and pushing

We created one **public** Docker Hub repository:

```text
lakshmisaranya/gamehub
```

All three custom images were tagged in this same repository; separate repositories were not needed.

| Component | Docker Hub image |
|:--|:--|
| Frontend | `lakshmisaranya/gamehub:frontend-v1` |
| Auth service | `lakshmisaranya/gamehub:auth-v1` |
| Game service | `lakshmisaranya/gamehub:game-v1` |

Docker Hub login succeeded and all three images were pushed successfully. Because the repository is public, Kubernetes does not need registry credentials to pull these images.

General workflow:

```powershell
docker tag <local-image> lakshmisaranya/gamehub:<tag>
docker push lakshmisaranya/gamehub:<tag>
```

The frontend Deployment uses:

```text
lakshmisaranya/gamehub:frontend-v1
```

Recorded image IDs at the time of tagging/pushing:

| Tag | Recorded image ID |
|:--|:--|
| `frontend-v1` | `63ac82a71298` |
| `auth-v1` | `d8718f2ffaf0` |
| `game-v1` | `e1b8f74bff3c` |

**For future rebuilds:** Verify that the intended newly built local image is the one being tagged before pushing. A tag must point to the image built from the source version we intend to deploy.

## 6. Create the Kubernetes namespace

A namespace provides a logical scope for grouping Kubernetes resources. We created a dedicated `gamehub` namespace.

File: `namespace.yaml`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: gamehub
```

Apply:

```powershell
kubectl apply -f namespace.yaml
```

Output:

```text
namespace/gamehub created
```

Verify:

```powershell
kubectl get namespaces
```

The output included the `gamehub` namespace with status `Active`.

We also checked:

```powershell
kubectl get all -n gamehub
```

Output:

```text
No resources found in gamehub namespace.
```

This was expected: the namespace existed, but no application resources had been created inside it yet.

### 6.1 Why does Namespace use `apiVersion: v1`?

Namespace belongs to Kubernetes' core API group. The core group has no group-name prefix, so its version is written simply as `v1`.

## 7. Kubernetes API groups and how to find API versions

Kubernetes organizes resources into API groups based on functionality.

| API group | Example resources | Example `apiVersion` |
|:--|:--|:--|
| Core API group (no group prefix) | Namespace, Pod, Service, ConfigMap | `v1` |
| `apps` | Deployment, ReplicaSet, StatefulSet, DaemonSet | `apps/v1` |
| `batch` | Job, CronJob | `batch/v1` |
| `networking.k8s.io` | Ingress, NetworkPolicy | `networking.k8s.io/v1` |

For `apps/v1`:
- `apps` is the API group.
- `v1` is the version within that API group.
- `Deployment` is the resource kind.

API groups organize the Kubernetes API; they are not Kubernetes namespaces.

### 7.1 Command to find the API version for a resource

The command we discussed was:

```powershell
kubectl api-resources
```

It lists resource names, API versions, kinds, and whether resources are namespaced.

Useful columns:

| Column | Meaning |
|:--|:--|
| `NAME` | Resource name |
| `APIVERSION` | API group and version used by that resource |
| `KIND` | Kind name used in YAML |
| `NAMESPACED` | Whether the resource is namespace-scoped |

Related command:

```powershell
kubectl api-versions
```

Difference:

| Command | What it displays | When to use it |
|:--|:--|:--|
| `kubectl api-resources` | Resources and the API version each resource uses | Finding the `apiVersion` for a YAML resource |
| `kubectl api-versions` | API group/version combinations available in the cluster | Listing API versions supported by the cluster |

For finding the API version to put in a particular resource YAML, `kubectl api-resources` is generally the more useful command.

## 8. Create the frontend Deployment

A Deployment manages the desired state of an application workload. It creates and manages ReplicaSets; a ReplicaSet ensures that the desired number of Pods exists.

We started with one frontend replica.

File: `frontend-deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gamehub-frontend
  namespace: gamehub
spec:
  replicas: 1
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
          image: lakshmisaranya/gamehub:frontend-v1
          ports:
            - containerPort: 80
```

Apply from the project root:

```powershell
kubectl apply -f frontend-deployment.yaml
```

Output:

```text
deployment.apps/gamehub-frontend created
```

### 8.1 Why does Deployment use `apps/v1`?

Deployment belongs to the `apps` API group, and its version is `v1`; therefore, the YAML uses `apps/v1`.

| Resource | API version |
|:--|:--|
| Namespace | `v1` |
| Pod | `v1` |
| Service | `v1` |
| Deployment | `apps/v1` |
| ReplicaSet | `apps/v1` |
| Job | `batch/v1` |

### 8.2 Important selector and label relationship

These values must match:

```yaml
selector:
  matchLabels:
    app: gamehub-frontend
```

```yaml
template:
  metadata:
    labels:
      app: gamehub-frontend
```

The Deployment selector identifies the Pods it manages. The Pod template supplies the labels those Pods receive.

`replicas: 1` means the Deployment should maintain one Pod, not one ReplicaSet. A Deployment may manage multiple ReplicaSets during a rolling update, but the replica count describes the desired number of Pods.

`containerPort: 80` documents the port the container is expected to listen on. It does not, by itself, expose the application outside the Pod. A Service is a separate Kubernetes resource.

## 9. Check the Deployment and Pod

We checked the Deployment:

```powershell
kubectl get deployments -n gamehub
```

Initial output:

```text
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
gamehub-frontend   0/1     1            0           33s
```

| Column | Value | Meaning |
|:--|:--:|:--|
| `READY` | `0/1` | Zero Pods ready out of one desired replica |
| `UP-TO-DATE` | `1` | One Pod created from the latest Deployment configuration |
| `AVAILABLE` | `0` | No Pod available to serve traffic |
| `AGE` | `33s` | Approximate age at the time of the check |

Then we checked Pods:

```powershell
kubectl get pods -n gamehub
```

Output at that moment:

```text
NAME                               READY   STATUS    RESTARTS      AGE
gamehub-frontend-9c85bb695-ld2nk   1/1     Running   3 (29s ago)   76s
```

The container was running at that instant, but it had restarted three times. `Running` alone did not prove the container was healthy.

## 10. Troubleshoot the frontend container restarts

We inspected the Pod:

```powershell
kubectl describe pod gamehub-frontend-9c85bb695-ld2nk -n gamehub
```

Important details:

```text
Image:          lakshmisaranya/gamehub:frontend-v1
State:          Terminated
Reason:         Error
Exit Code:      1
Last State:     Terminated
Reason:         Error
Exit Code:      1
Restart Count:  3
```

Events showed that the Pod was scheduled, the image was pulled, and the container was created and started. Later, Kubernetes reported:

```text
Back-off restarting failed container frontend
```

### 10.1 What this established

- Kubernetes scheduled the Pod onto the Minikube node.
- Kubernetes successfully pulled the frontend image from Docker Hub.
- The container was created and started.
- The container process then exited with error code `1`.
- Kubernetes attempted to restart it. Repeated failures led to a restart back-off delay.

Therefore, this was not an image-pull failure.

### 10.2 Read logs from the previous container instance

Because the container had restarted, we used `--previous` to read logs from the last terminated instance:

```powershell
kubectl logs gamehub-frontend-9c85bb695-ld2nk -n gamehub --previous
```

Important log lines:

```text
Configuration complete; ready for start up
2026/10/30 08:04:28 [emerg] 1#1: host not found in upstream "auth-service" in /etc/nginx/conf.d/default.conf:12
nginx: [emerg] 1#1: host not found in upstream "auth-service" in /etc/nginx/conf.d/default.conf:12
```

### 10.3 Root cause identified

The frontend image runs Nginx. Its Nginx configuration refers to the backend upstream hostname `auth-service`.

At this point:

- The `gamehub` namespace existed.
- The frontend Deployment and Pod had been created.
- We had **not** deployed the auth-service workload or created its Kubernetes Service.
- We had **not** deployed the game-service workload or created its Kubernetes Service.

The frontend's Nginx configuration expects backend names that are not yet resolvable in the Kubernetes environment. Nginx reports `host not found in upstream "auth-service"` and exits. That causes the container restart loop.

This is different from an image-pull problem: the image was pulled successfully, but the application process could not complete startup with the current upstream configuration.

### 10.4 How this will be addressed

We need to deploy the backend components and create Kubernetes Services with DNS names that the frontend can resolve. A Kubernetes Service provides a stable network endpoint and DNS name for the Pods selected by that Service.

Intended high-level request path:

```text
Browser / user
      |
      v
Frontend Service (to be created)
      |
      v
Frontend Pod (Nginx + React)
      |
      +---- /api/auth/* ----> auth-service (Kubernetes Service)
      |                           |
      |                           v
      |                       Auth Pod
      |
      +---- /api/game/* ----> game-service (Kubernetes Service)
                                  |
                                  v
                              Game Pod
```

The dependency causing the Nginx startup failure has been identified. We have **not yet applied a fix or verified that the frontend starts successfully**. The next steps must account for the backend workloads, Services, and DNS names.

## 11. Current progress checkpoint

| Item | Status |
|:--|:--|
| GameHub source inspected | Completed |
| Docker images rebuilt | Completed |
| Docker Hub public repository created | Completed |
| Frontend image tagged and pushed | Completed |
| Auth image tagged and pushed | Completed |
| Game image tagged and pushed | Completed |
| Minikube `gamehub-cluster` selected | Completed |
| `gamehub` namespace created | Completed |
| Frontend Deployment YAML created | Completed |
| Frontend Deployment applied | Completed |
| Frontend image pulled by Kubernetes | Completed |
| Frontend container starts and stays healthy | **Not yet** |
| Auth workload and Service | Not yet deployed |
| Game workload and Service | Not yet deployed |
| PostgreSQL workload and storage | Not yet deployed |
| Frontend Service / external access | Not yet configured |
| End-to-end GameHub verification | Not yet completed |

## 12. Commands used so far

Run these from the GameHub project root unless otherwise stated.

### Docker image preparation

```powershell
docker compose build
```

### Minikube and Kubernetes inspection

```powershell
minikube profile list
kubectl config current-context
kubectl get nodes -o wide
kubectl api-resources
kubectl api-versions
```

### Namespace and frontend Deployment

```powershell
kubectl apply -f namespace.yaml
kubectl get namespaces
kubectl get all -n gamehub
kubectl apply -f frontend-deployment.yaml
kubectl get deployments -n gamehub
kubectl get pods -n gamehub
kubectl describe pod gamehub-frontend-9c85bb695-ld2nk -n gamehub
kubectl logs gamehub-frontend-9c85bb695-ld2nk -n gamehub --previous
```

## 13. Concepts learned in this stage

1. A Kubernetes Namespace groups and scopes resources.
2. `apiVersion` identifies the API group and version used for a resource.
3. Namespace and Service use the core API version `v1`; Deployment uses `apps/v1`.
4. `kubectl api-resources` lists resources and the API version each resource uses.
5. `kubectl api-versions` lists API group/version combinations available in the cluster.
6. A Minikube profile name and Kubernetes node name are different concepts, even if their values match.
7. A Deployment manages ReplicaSets, and ReplicaSets maintain the desired number of Pods.
8. `replicas: 1` means one desired Pod, not one ReplicaSet.
9. A Pod can be `Running` at a moment in time and still have restarted repeatedly.
10. `kubectl describe pod` helps inspect container state, restart count, scheduling, and events.
11. `kubectl logs --previous` retrieves logs from the previous terminated container instance.
12. A successful image pull does not guarantee that the application inside the container can start.
13. Kubernetes Services provide stable endpoints and DNS names for accessing Pods.
14. Docker Compose service-name DNS and Kubernetes Service DNS are separate networking environments.

---

**Learning checkpoint:** Stop here. The next step is to plan and implement the backend Deployment and Service sequence needed to satisfy the frontend's Nginx upstream dependencies. Do not treat the frontend as fixed until its logs and readiness confirm that it starts successfully.
