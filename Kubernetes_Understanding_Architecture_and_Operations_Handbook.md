**KUBERNETES**

**Understanding, Architecture & Operations**

_A practical beginner-to-production handbook_

Concepts • Architecture • Workloads • Networking • Storage • Security • Troubleshooting • Learning path

# How to use this handbook

This handbook explains the concepts behind Kubernetes and how the parts fit together. It is designed to complement the separate Kubernetes command reference. Read the concepts first, then practise the commands against a local learning cluster before applying them to shared or production clusters.

- If you are new: read Sections 1–8, then follow the learning path.
- If you already deploy apps: focus on probes, rollouts, storage, scaling, security, and troubleshooting.
- For production: follow your organization's runbooks, access controls, backup procedures, and change process.

**Safety note:** Commands that delete resources, drain nodes, alter access, or change production workloads can cause outages or data loss. Confirm the current context and namespace before running them.

# Contents

- 1\. What Kubernetes is
- 2\. Desired state and reconciliation
- 3\. Kubernetes architecture
- 4\. Core objects and workload hierarchy
- 5\. Images, containers, and registries
- 6\. Scheduling and resource management
- 7\. Kubernetes networking
- 8\. Configuration and secrets
- 9\. Health checks and probes
- 10\. Deployments and safe rollouts
- 11\. Storage and stateful workloads
- 12\. Scaling and high availability
- 13\. Security fundamentals
- 14\. Monitoring, logging, and tracing
- 15\. Jobs, CronJobs, and background work
- 16\. CI/CD and GitOps
- 17\. Troubleshooting mental models
- 18\. Common misconceptions
- 19\. Hands-on learning path
- 20\. Practice environments and official resources
- 21\. Glossary and mental model summary

# 1. What Kubernetes is

Kubernetes (often called K8s) is an open-source platform for deploying, scheduling, scaling, networking, and managing containerized applications across a cluster of machines. You describe the state you want, and Kubernetes controllers work continuously to make actual state match that desired state.

## Why use it?

- Restart or replace failed application containers and Pods.
- Run multiple replicas and distribute traffic through Services.
- Perform controlled application updates and rollbacks.
- Schedule workloads across machines based on available resources and constraints.
- Manage configuration, secrets, persistent storage, and access controls.
- Provide APIs and automation for repeatable deployments.

## Docker versus Kubernetes

| **Technology**                     | **Main job**                           | **Typical use**                                                            |
| ---------------------------------- | -------------------------------------- | -------------------------------------------------------------------------- |
| Container runtime / Docker tooling | Build images and run containers        | Package an app and run it locally or on a host                             |
| Kubernetes                         | Orchestrate workloads across a cluster | Keep replicas running, schedule Pods, expose services, and manage rollouts |

**Remember:** Docker and Kubernetes are not direct substitutes. Kubernetes normally uses a CRI-compatible container runtime on each node; Docker is commonly used to build images, and those images can run in Kubernetes.

# 2. Desired state and reconciliation

Kubernetes is declarative. Instead of telling it every step to execute, you normally declare the intended result in a manifest—for example, three replicas of a web application. Controllers repeatedly compare actual state with desired state and take actions to reduce the difference. This is called reconciliation.

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: web
          image: nginx:1.27
```

- If one Pod disappears, the Deployment/ReplicaSet control loop works toward restoring three replicas.
- If you change the image in the Deployment, Kubernetes creates a new rollout according to its update strategy.
- A manifest is a declaration of intent—not a guarantee that every requested state can be achieved. Capacity, policy, image availability, and configuration can prevent convergence.

**Key idea:** Kubernetes keeps working after the initial command. It is not simply a script that runs once.

# 3. Kubernetes architecture

## 3.1 Control plane

| **Component**            | **Responsibility**                                                                                 |
| ------------------------ | -------------------------------------------------------------------------------------------------- |
| kube-apiserver           | Main API entry point. Validates and serves Kubernetes API requests.                                |
| etcd                     | Distributed key-value store holding Kubernetes cluster state.                                      |
| kube-scheduler           | Chooses a suitable node for Pods that have not yet been assigned to a node.                        |
| kube-controller-manager  | Runs controllers that reconcile resources, such as Deployments and Nodes.                          |
| cloud-controller-manager | Optional integration with a cloud provider for services such as load balancers and node lifecycle. |

## 3.2 Worker node

| **Component**          | **Responsibility**                                                                                      |
| ---------------------- | ------------------------------------------------------------------------------------------------------- |
| kubelet                | Node agent that makes sure assigned Pods are running as specified.                                      |
| Container runtime      | Pulls images and runs containers, commonly containerd or CRI-O.                                         |
| Network implementation | Provides Pod networking and may implement Services and NetworkPolicies, depending on the cluster setup. |

## 3.3 What happens after kubectl apply?

1. kubectl sends the manifest to the Kubernetes API server.
2. The API server authenticates and authorizes the request, validates it, and persists accepted state.
3. Controllers observe the new desired state and create or update dependent objects.
4. The scheduler selects a node for each unscheduled Pod, considering resources and constraints.
5. The kubelet on the chosen node asks the container runtime to prepare the Pod and run its containers.
6. Networking and storage components attach the required connectivity and volumes.
7. Readiness checks determine when the application should receive normal Service traffic.
8. Controllers continue reconciling the system if Pods or nodes fail or the desired state changes.

**Important distinction:** The API server accepts and stores intent; the scheduler chooses placement; the kubelet and runtime run workloads on nodes.

# 4. Core objects and workload hierarchy

| **Object**              | **Meaning**                                                                  | **Use**                                            |
| ----------------------- | ---------------------------------------------------------------------------- | -------------------------------------------------- |
| Cluster                 | Control plane plus worker nodes and cluster services                         | The overall Kubernetes environment                 |
| Namespace               | Logical scope for namespaced resources                                       | Separate teams, environments, or applications      |
| Pod                     | Smallest deployable Kubernetes unit; one or more tightly coupled containers  | Run an application instance                        |
| ReplicaSet              | Maintains a target number of matching Pod replicas                           | Usually managed by a Deployment                    |
| Deployment              | Manages stateless application rollout and ReplicaSets                        | Web APIs, frontends, and other stateless services  |
| Service                 | Stable network identity for a set of Pods                                    | Reach Pods despite Pod IP changes                  |
| Ingress / Gateway       | HTTP(S) entry routing, when a compatible controller or implementation exists | Expose selected services externally                |
| ConfigMap / Secret      | Configuration data / sensitive configuration data                            | Separate runtime settings from container images    |
| PersistentVolumeClaim   | A workload's request for persistent storage                                  | Request storage without hardcoding backend details |
| Job / CronJob           | Run-to-completion work / scheduled work                                      | Migrations, reports, and periodic tasks            |
| StatefulSet / DaemonSet | Manage stable stateful replicas / workload on selected nodes                 | Databases and node-level agents, respectively      |

## Pod versus container

- A container is a running packaged process.
- A Pod is the Kubernetes scheduling unit and can contain one or more containers that share network identity and can share volumes.
- Pods are replaceable. Do not rely on a Pod IP or local writable container filesystem as permanent identity or durable storage.

## Namespace basics

```
kubectl get namespaces
kubectl get pods -n production
kubectl config set-context --current --namespace=production
```

**Caution:** Changing the current namespace affects later commands that omit -n. Always verify the active context and namespace before making changes.

# 5. Images, containers, and registries

A container image is a versioned package containing an application, its dependencies, and metadata. A registry stores images. When a Pod is scheduled, the node runtime pulls the required image if it is not already available locally.

## Example Flask Dockerfile

```
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 5000
CMD ["gunicorn", "--bind", "0.0.0.0:5000", "main:app"]
```

Example build and push workflow:

```
docker build -t registry.example.com/team/my-flask-app:1.0.0 .
docker login registry.example.com
docker push registry.example.com/team/my-flask-app:1.0.0
```

- Use explicit version tags; avoid relying on mutable tags such as latest for controlled production releases.
- A tag can be moved to another image. A digest identifies image content and is useful for reproducibility.
- Private registries may require imagePullSecrets or an integration with the platform's identity system.
- ImagePullBackOff usually means Kubernetes is retrying after an image pull failure; inspect Pod events, image name, registry access, and credentials.

# 6. Scheduling and resource management

The scheduler places Pods on eligible nodes. It considers resource requests, node availability, taints and tolerations, affinity rules, topology constraints, and other scheduling policies.

## Requests and limits

| **Setting**    | **Meaning**                                            | **Practical note**                                                     |
| -------------- | ------------------------------------------------------ | ---------------------------------------------------------------------- |
| CPU request    | CPU amount used for scheduling and resource guarantees | A Pod can remain Pending if no eligible node has enough requested CPU. |
| Memory request | Memory amount used for scheduling                      | Memory requests should reflect realistic workload needs.               |
| CPU limit      | Caps CPU usage; throttling can occur                   | Too-low limits can hurt latency and throughput.                        |
| Memory limit   | Memory ceiling enforced by the runtime/kernel          | Exceeding it can cause an OOM kill.                                    |

## Placement controls

- Taints repel Pods unless the Pod has a matching toleration.
- Node affinity expresses which nodes are preferred or required.
- Pod affinity and anti-affinity express co-location or separation preferences.
- Topology spread constraints can distribute replicas across zones or nodes.

**Troubleshooting:** A Pending Pod often points to insufficient resources, unsatisfied affinity, untolerated taints, unbound storage, or policy constraints. Describe the Pod and inspect events rather than guessing.

# 7. Kubernetes networking

Kubernetes networking generally gives Pods routable IP addresses within the cluster, but the exact implementation depends on the cluster's network plugin. Pod IPs can change when Pods are recreated, so applications normally connect through Services or DNS names.

| **Networking concept** | **What it does**                                                                   |
| ---------------------- | ---------------------------------------------------------------------------------- |
| Pod networking         | Enables communication between Pods according to cluster network rules.             |
| Service                | Stable virtual endpoint and discovery for a set of matching Pods.                  |
| ClusterIP              | Internal cluster-only Service endpoint; common default.                            |
| NodePort               | Exposes a Service on a port on nodes, subject to cluster configuration.            |
| LoadBalancer           | Requests an external load balancer where the environment supports it.              |
| Ingress                | HTTP/HTTPS routing API that requires an Ingress controller.                        |
| Gateway API            | A newer, expressive API for traffic routing; requires a compatible implementation. |
| NetworkPolicy          | Declares allowed Pod traffic; enforcement depends on the network implementation.   |

## Example: a web app reaching PostgreSQL

If a PostgreSQL Service is named postgres and is in namespace data, an application in another namespace can commonly use postgres.data.svc.cluster.local as its DNS name. Use the database Service port, credentials stored securely, and network policies appropriate to the environment.

**Exposure warning:** Do not expose databases publicly unless there is a clear, secured requirement. Prefer private networking, least-privilege access, and managed access paths.

# 8. Configuration and secrets

| **Resource** | **Typical content**                                        | **Notes**                                                                             |
| ------------ | ---------------------------------------------------------- | ------------------------------------------------------------------------------------- |
| ConfigMap    | Non-sensitive settings, feature flags, configuration files | Can be injected as environment variables or mounted files.                            |
| Secret       | Passwords, tokens, certificates, keys                      | Kubernetes Secrets are not automatically safe merely because they are called Secrets. |

- Avoid hardcoding credentials into source code, Docker images, or checked-in manifests.
- Restrict Secret access using RBAC; configure encryption at rest where available; consider an external secret manager.
- Environment variables are convenient, while mounted files may be better for configuration that must be re-read.
- Changing a ConfigMap or Secret does not guarantee every application automatically reloads it. Environment-variable values generally require Pod replacement to update.

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: web-config
data:
  APP_ENV: "production"
  LOG_LEVEL: "INFO"
```

# 9. Health checks and probes

| **Probe** | **Question answered**                                | **Failure behavior**                                                              |
| --------- | ---------------------------------------------------- | --------------------------------------------------------------------------------- |
| Startup   | Has the application finished starting?               | Delays liveness/readiness checks until startup succeeds.                          |
| Readiness | Should this Pod receive normal Service traffic now?  | The Pod is marked not ready and is normally removed from ready Service endpoints. |
| Liveness  | Is the process stuck or unhealthy enough to restart? | Repeated failures can trigger container restart.                                  |

Example HTTP probes (the application must actually implement /health):

```
readinessProbe:
  httpGet:
    path: /health
    port: 5000
  initialDelaySeconds: 5
  periodSeconds: 10
livenessProbe:
  httpGet:
    path: /health
    port: 5000
  initialDelaySeconds: 15
  periodSeconds: 20
```

**Design tip:** Keep liveness checks lightweight. If liveness depends on a temporary database outage, Kubernetes may repeatedly restart an otherwise functioning process. Readiness can include checks that determine whether the app can safely serve requests.

# 10. Deployments and safe rollouts

A Deployment manages ReplicaSets and supports rolling updates for stateless applications. A rolling update gradually replaces old Pods with new ones, subject to the rollout strategy and readiness.

- Set replicas appropriate to availability needs and capacity.
- Use readiness probes so traffic does not reach an app before it is ready.
- Choose maxSurge and maxUnavailable deliberately; rollout behavior depends on them.
- Watch rollout status and application metrics during release.
- Rollback restores a previous Deployment revision, but it does not automatically reverse database migrations or external side effects.

```
kubectl rollout status deployment/web
kubectl rollout history deployment/web
kubectl rollout undo deployment/web
```

**Release safety:** Use backward-compatible database changes and a tested recovery plan. A Kubernetes rollback alone cannot undo every change made by a release.

# 11. Storage and stateful workloads

| **Concept**                 | **Meaning**                                                                       |
| --------------------------- | --------------------------------------------------------------------------------- |
| Volume                      | Storage made available to containers in a Pod.                                    |
| PersistentVolume (PV)       | Cluster storage resource or provisioned storage representation.                   |
| PersistentVolumeClaim (PVC) | A request for storage by a workload.                                              |
| StorageClass                | Describes a class of storage and how it may be provisioned.                       |
| CSI driver                  | Interface used by storage vendors/platforms to integrate storage with Kubernetes. |
| StatefulSet                 | Workload controller for stable Pod identity and storage patterns.                 |

- A container's writable layer is ephemeral and should not be treated as durable application data.
- A PVC can outlive an individual Pod, but retention and deletion behavior depend on the PV reclaim policy, StorageClass, and workload configuration.
- Persistent storage is not the same as a backup. Test restores and confirm consistency requirements.
- Databases need deliberate planning for replication, recovery, upgrades, backups, and failure domains; Kubernetes does not automatically solve these application-level concerns.

# 12. Scaling and high availability

| **Mechanism**                   | **Purpose**                                    | **Dependencies / caveats**                                          |
| ------------------------------- | ---------------------------------------------- | ------------------------------------------------------------------- |
| Manual replica scaling          | Change the number of Pod replicas              | Requires sufficient cluster capacity.                               |
| Horizontal Pod Autoscaler (HPA) | Adjust replicas based on metrics               | Metrics and resource requests/configuration must be appropriate.    |
| Vertical Pod Autoscaler (VPA)   | Recommend or adjust resource requests          | Availability and behavior depend on installation and configuration. |
| Cluster autoscaler              | Add/remove nodes when supported and configured | Requires infrastructure integration and quota/capacity.             |

- High availability requires more than several replicas: spread replicas across failure domains, ensure capacity, and avoid single points of failure.
- A single-node local cluster is useful for learning but is not highly available.
- PodDisruptionBudgets can limit voluntary disruption, but they do not prevent every outage or involuntary failure.
- Autoscaling is not instantaneous; applications and downstream systems must tolerate scale-up delays and load spikes.

# 13. Security fundamentals

## Security layers

- Authentication: establish who or what is making a request.
- Authorization: decide what that identity is allowed to do, often using RBAC.
- Admission controls and policy: validate or constrain requested resources.
- Network controls: limit communication paths where supported.
- Workload hardening: run as non-root where possible, drop unnecessary Linux capabilities, and use read-only filesystems when practical.
- Supply-chain security: use trusted base images, scan vulnerabilities, pin versions/digests where appropriate, and protect build credentials.
- Secret management: tightly restrict access and rotate credentials.

## Example securityContext

```
securityContext:
  runAsNonRoot: true
  allowPrivilegeEscalation: false
  capabilities:
    drop: ["ALL"]
  seccompProfile:
    type: RuntimeDefault
```

**Compatibility:** A securityContext must match the image and application requirements. Test it before enforcing it broadly; some applications need specific file permissions or capabilities.

## RBAC principle

Grant the minimum verbs and resources needed, scoped to the minimum namespace or cluster scope. Avoid sharing administrator credentials with applications or routine users.

# 14. Monitoring, logging, and tracing

| **Signal** | **Helps answer**                                                    | **Common tooling examples**                                                         |
| ---------- | ------------------------------------------------------------------- | ----------------------------------------------------------------------------------- |
| Metrics    | How much CPU/memory is used? Is latency or error rate rising?       | Metrics Server for resource metrics; Prometheus and Grafana for broader monitoring. |
| Logs       | What did the application or system component report?                | kubectl logs for immediate checks; centralized log systems for multi-Pod history.   |
| Traces     | Where did a request spend time across services?                     | OpenTelemetry-compatible instrumentation and a tracing backend.                     |
| Events     | What did Kubernetes report while scheduling or starting a resource? | kubectl describe and kubectl get events.                                            |

- Collect application metrics such as request rate, error rate, latency, queue depth, and business-level outcomes.
- Centralize logs for production because Pods can be replaced and local container logs may not remain available indefinitely.
- Set actionable alerts with ownership and runbooks; avoid alerting on every transient event.

# 15. Jobs, CronJobs, and background work

A Job runs work to completion, while a CronJob creates Jobs on a schedule. These are useful for data imports, reports, cleanup, and maintenance tasks.

- Design jobs to be retry-safe and idempotent where possible.
- Use deadlines, retry limits, resource requests, and logging appropriate to the task.
- A CronJob schedule does not guarantee exactly-once business execution. Consider missed schedules, retries, overlapping runs, and duplicate effects.
- For durable asynchronous workflows, use a queue or workflow system when it fits the problem; Kubernetes alone is not a message broker.

# 16. CI/CD and GitOps

Kubernetes runs and manages workloads; CI/CD builds, tests, packages, and releases them.

9. Developer commits code and opens a pull request.
10. CI runs tests, linting, security checks, and builds a container image.
11. The image is pushed to a registry with a traceable version or digest.
12. A deployment manifest or chart is updated for the intended environment.
13. A CD system applies the change, or a GitOps controller reconciles the cluster to the desired Git state.
14. The team checks rollout health, metrics, logs, and release results.

- Helm packages Kubernetes resources into reusable charts and values.
- GitOps tools commonly watch a repository and reconcile cluster state with declared configuration.
- Keep secrets out of plain-text Git repositories; use a suitable secret-management workflow.
- A successful deployment command is not proof that the application works; verify readiness and user-facing behavior.

# 17. Troubleshooting mental models

| **Symptom**                           | **Likely areas to inspect first**                                                                               |
| ------------------------------------- | --------------------------------------------------------------------------------------------------------------- |
| Pod Pending                           | Events, resource requests, node capacity, taints, affinity, PVC binding, quotas.                                |
| ImagePullBackOff / ErrImagePull       | Image name/tag, registry reachability, credentials, pull policy, certificate or rate-limit issues.              |
| CrashLoopBackOff                      | Container logs, previous logs, command/entrypoint, configuration, permissions, probes, resource limits.         |
| Service has no working traffic        | Service selector, Pod labels, readiness, endpoints, port/targetPort, NetworkPolicy, DNS.                        |
| Pod is Running but app is unavailable | Readiness, listening address/port, application logs, Service and ingress/gateway routing.                       |
| OOMKilled                             | Memory usage, memory limit, workload behavior, leaks, and whether the request/limit matches reality.            |
| Rollout stuck                         | New Pod readiness, scheduling, image pull, capacity, admission policies, rollout status and events.             |
| Node NotReady                         | Node conditions, kubelet/runtime health, disk/memory pressure, network, and control-plane connectivity.         |
| API request is slow or failing        | API server health, authentication/authorization, network, admission webhooks, and cluster control-plane status. |

## A safe troubleshooting sequence

15. Confirm the current context and namespace.
16. Check the resource's status and events.
17. Inspect Pod details, labels, scheduling decisions, and conditions.
18. Read current and previous container logs as applicable.
19. Verify the image, command, configuration, resources, and probes.
20. Trace networking from Pod to Service to ingress/gateway, checking ports and policy at each hop.
21. Change one thing at a time and verify the result.
22. For production, preserve evidence and follow the incident/change process before deleting or restarting resources.

```
kubectl config current-context
kubectl get pods -A
kubectl describe pod <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace> --previous
kubectl get events -n <namespace> --sort-by=.metadata.creationTimestamp
```

# 18. Common misconceptions

| **Misconception**                               | **Reality**                                                                          |
| ----------------------------------------------- | ------------------------------------------------------------------------------------ |
| A Pod is permanent.                             | Pods are replaceable; design for replacement.                                        |
| A Running Pod means the app is healthy.         | Running is not the same as ready or functionally healthy.                            |
| A Service has a permanent Pod IP behind it.     | A Service provides a stable endpoint for changing Pod backends.                      |
| Kubernetes automatically backs up databases.    | Backup and restore need explicit design and testing.                                 |
| A Secret is automatically encrypted everywhere. | Storage encryption and access controls depend on configuration and platform.         |
| Rollback undoes all release changes.            | It changes workload revision; external side effects and database changes may remain. |
| More replicas always mean high availability.    | Placement, capacity, dependencies, and failure domains matter.                       |
| Kubernetes replaces CI/CD.                      | It runs workloads; build/test/release automation is a separate concern.              |

# 19. Hands-on learning path

## Stage 1 — Foundations

- Understand containers, images, registries, Pods, Deployments, Services, and namespaces.
- Use kubectl get, describe, logs, and explain.
- Practice only in a local learning cluster.

## Stage 2 — Deploy a real application

- Build a small Flask or Nginx image and push it to a registry, or use a public learning image.
- Deploy it with a Deployment and expose it through a Service.
- Scale replicas and observe how the Service routes to Pods.

## Stage 3 — Configuration and health

- Move settings into a ConfigMap and sensitive values into a Secret.
- Add startup, readiness, and liveness probes.
- Intentionally break a test deployment and diagnose the events/logs.

## Stage 4 — Release and recovery

- Update an image tag, watch the rollout, and practice a rollback.
- Learn what a rollout rollback does not undo.
- Use resource requests and limits, then observe scheduling and resource behavior.

## Stage 5 — Production operations

- Study RBAC, NetworkPolicy, storage, backups, monitoring, and incident response.
- Learn node maintenance and disruption budgets in a disposable cluster.
- Document a repeatable deployment and recovery runbook.

## Stage 6 — Advanced topics

- Explore Helm, GitOps, autoscaling, StatefulSets, Jobs/CronJobs, Gateway API, and policy controls.
- Study multi-zone design and application-level resilience.

# 20. Practice environments and official resources

| **Option** | **Good for**                                              | **Notes**                                                     |
| ---------- | --------------------------------------------------------- | ------------------------------------------------------------- |
| Minikube   | Learning Kubernetes on a workstation                      | Provides a local cluster and learning-oriented tooling.       |
| kind       | Local clusters using containers; testing manifests and CI | Useful for repeatable local test clusters.                    |
| K3s        | Lightweight Kubernetes distributions and edge/lab setups  | Good for small environments; production design still matters. |

Official resources:

- Kubernetes documentation: <https://kubernetes.io/docs/>
- Kubernetes concepts: <https://kubernetes.io/docs/concepts/>
- kubectl reference: <https://kubernetes.io/docs/reference/kubectl/>
- Interactive tutorials: <https://kubernetes.io/docs/tutorials/>
- Minikube: <https://minikube.sigs.k8s.io/docs/>
- kind: <https://kind.sigs.k8s.io/>
- Helm documentation: <https://helm.sh/docs/>

# 21. Glossary and mental model summary

| **Term**      | **Short definition**                                                             |
| ------------- | -------------------------------------------------------------------------------- |
| API           | Interface through which clients and components request or observe cluster state. |
| Control plane | Components that expose the API and manage cluster decisions/state.               |
| Node          | Machine that can run Kubernetes Pods.                                            |
| Manifest      | Declarative YAML or JSON describing desired resources.                           |
| Controller    | Control loop that reconciles actual state toward desired state.                  |
| Scheduler     | Chooses a suitable node for an unscheduled Pod.                                  |
| Runtime       | Software that runs containers on a node.                                         |
| Registry      | Service that stores container images.                                            |
| Readiness     | Whether a Pod should receive normal traffic.                                     |
| RBAC          | Role-based access control for permissions.                                       |
| PVC           | Request for persistent storage.                                                  |
| GitOps        | Practice of reconciling deployed state from version-controlled declarations.     |

## The one-minute mental model

- Image: the packaged application.
- Container: the running application process.
- Pod: Kubernetes' smallest deployable unit.
- Deployment: keeps a stateless workload's replicas and rollout aligned with the declaration.
- Scheduler: chooses where a Pod should run.
- Kubelet/runtime: run the assigned Pod on the node.
- Service: stable discovery and traffic endpoint for Pods.
- Ingress/Gateway: routes incoming traffic when a controller/implementation is installed.
- ConfigMap/Secret: supply configuration and sensitive values.
- PVC/PV/StorageClass: connect workloads to persistent storage.
- Controllers: continually reconcile actual state with desired state.
- Observability: metrics, logs, traces, and events show whether the system is working.

# Appendix: useful kubectl commands

These are starter commands; see the separate command reference for a broader operational command catalog.

```
# Cluster and context
kubectl version
kubectl cluster-info
kubectl get nodes -o wide
kubectl config get-contexts
kubectl config current-context

# Resources
kubectl get all -n <namespace>
kubectl get pods -A -o wide
kubectl get deploy,rs,svc -n <namespace>
kubectl describe deployment <name> -n <namespace>

# Logs and debugging
kubectl logs <pod> -n <namespace>
kubectl logs <pod> -c <container> -n <namespace>
kubectl exec -it <pod> -n <namespace> -- sh
kubectl get events -n <namespace> --sort-by=.metadata.creationTimestamp

# Apply and rollout
kubectl apply -f app.yaml
kubectl diff -f app.yaml
kubectl rollout status deployment/<name> -n <namespace>
kubectl rollout history deployment/<name> -n <namespace>
kubectl rollout undo deployment/<name> -n <namespace>

# Always check before a change
kubectl config current-context
kubectl get namespace
kubectl auth can-i <verb> <resource> -n <namespace>
```

**Final operational habit:** Before applying, deleting, scaling, draining, or changing permissions, verify the target cluster, namespace, object, expected impact, and recovery plan.
