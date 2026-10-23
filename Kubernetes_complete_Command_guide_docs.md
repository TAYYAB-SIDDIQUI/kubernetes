Kubernetes Complete Command & Operations Guide

Practical reference for cluster health, Pods, images, deployments, networking, configuration, storage, security, and troubleshooting.

Commands may require different permissions or optional components depending on the Kubernetes distribution, runtime, and cluster version.

# How to use this guide

- Replace placeholders such as &lt;pod-name&gt;, &lt;namespace&gt;, and &lt;deployment-name&gt; with real values. Do not type the angle brackets literally.
- Use -n &lt;namespace&gt; for a specific namespace and -A for all namespaces.
- Check the current context before making changes: kubectl config current-context.
- Prefer declarative YAML and kubectl apply for repeatable deployments.
- Treat delete, force-delete, node drain, secret inspection, and storage deletion as potentially disruptive or sensitive operations.
- kubectl is the Kubernetes CLI; Docker, crictl, ctr, and Helm are separate tools.

# 1. Kubernetes flow and core objects

Typical flow: YAML manifests → kubectl apply → API server → controllers and scheduler → worker node → Pod/container → Service → Ingress/Gateway.

| Object                  | Purpose                                                                    |
| ----------------------- | -------------------------------------------------------------------------- |
| Cluster                 | The complete Kubernetes environment.                                       |
| Node                    | A machine that runs Kubernetes workloads.                                  |
| Namespace               | Logical separation of resources.                                           |
| Pod                     | Smallest deployable Kubernetes unit; contains one or more containers.      |
| Deployment / ReplicaSet | Manages stateless replicas and rolling updates.                            |
| StatefulSet             | Workloads needing stable identities and storage relationships.             |
| DaemonSet               | Runs a Pod on selected nodes, commonly node agents.                        |
| Job / CronJob           | One-off or scheduled tasks.                                                |
| Service                 | Stable network endpoint for matching Pods.                                 |
| Ingress / Gateway       | Manages incoming traffic; needs a compatible controller or implementation. |
| ConfigMap / Secret      | Application configuration / sensitive configuration.                       |
| PV / PVC / StorageClass | Persistent storage resources and provisioning.                             |
| ServiceAccount / RBAC   | Workload identity and authorization.                                       |
| HPA / NetworkPolicy     | Replica autoscaling / traffic restrictions when supported.                 |

Kubernetes continuously reconciles actual state toward the desired state declared in resources. For example, a Deployment with three replicas attempts to keep three Pods running.

# 2. Essential kubectl commands

```
kubectl version
kubectl version --client
kubectl cluster-info
kubectl get nodes
kubectl get namespaces
kubectl get pods
kubectl get deployments
kubectl get services
kubectl get all
kubectl describe pod <pod-name>
kubectl logs <pod-name>
kubectl apply -f deployment.yaml
kubectl delete -f deployment.yaml
kubectl help
kubectl get --help
kubectl explain deployment
kubectl explain pod.spec
```

| Command      | Purpose                                        |
| ------------ | ---------------------------------------------- |
| get          | List resources and status.                     |
| describe     | Detailed resource information and events.      |
| logs         | Container output.                              |
| exec         | Run a command in a container.                  |
| apply        | Create/update resources from manifests.        |
| delete       | Remove resources.                              |
| edit / patch | Modify a live resource.                        |
| rollout      | Inspect and manage workload updates.           |
| scale        | Change replica count.                          |
| top          | CPU/memory usage; requires a metrics pipeline. |
| explain      | Inspect resource schema fields.                |

# 3. Cluster health and nodes

```
kubectl version
kubectl cluster-info
kubectl api-resources
kubectl api-versions
kubectl get --raw='/readyz'
kubectl get --raw='/livez'
kubectl get --raw='/readyz?verbose'

kubectl get nodes
kubectl get nodes -o wide
kubectl describe node <node-name>
kubectl get nodes --show-labels
kubectl top nodes
kubectl get events -A --sort-by=.metadata.creationTimestamp
```

readyz/livez report API server health, not the health of every application or node. Node conditions worth investigating include Ready=False/Unknown, MemoryPressure, DiskPressure, and PIDPressure. kubectl top requires Metrics Server or another compatible metrics pipeline.

# 4. Namespaces and contexts

```
kubectl get namespaces
kubectl describe namespace <namespace>
kubectl get all -n <namespace>
kubectl get pods -A
kubectl get deployments -A
kubectl get services -A
kubectl config get-contexts
kubectl config current-context
kubectl config use-context <context-name>
kubectl config get-clusters
kubectl config get-users
kubectl config view
kubectl config view --minify
kubectl config set-context --current --namespace=<namespace>
kubectl config set-context <context-name> --namespace=<namespace>
kubectl get pods --context=<context-name> -n <namespace>
```

kubectl get all does not include every resource type; query Secrets, ConfigMaps, PVCs, policies, and other kinds separately. Before destructive work, confirm the current context and namespace.

# 5. Pods: list, inspect, logs, exec, debug

```
kubectl get pods
kubectl get pods -o wide
kubectl get pods -A
kubectl get pods -w
kubectl get pods --show-labels
kubectl get pods -l app=flask-api
kubectl get pods --field-selector=status.phase=Running
kubectl get pods -A --field-selector=spec.nodeName=<node-name>
kubectl get pods -o custom-columns=NAME:.metadata.name,NODE:.spec.nodeName,STATUS:.status.phase
kubectl get pods -o custom-columns=NAME:.metadata.name,IMAGES:.spec.containers[*].image,STATUS:.status.phase

kubectl describe pod <pod-name>
kubectl get pod <pod-name> -o yaml
kubectl get pod <pod-name> -o json
kubectl get pod <pod-name> --show-labels
kubectl get pod <pod-name> -o jsonpath='{.spec.containers[*].image}'
kubectl get pod <pod-name> -o jsonpath='{.status.containerStatuses[*].restartCount}'
kubectl get pod <pod-name> -o jsonpath='{.status.podIP}'
kubectl get pod <pod-name> -o jsonpath='{.spec.nodeName}'

kubectl logs <pod-name>
kubectl logs -f <pod-name>
kubectl logs <pod-name> --tail=100
kubectl logs <pod-name> --since=30m
kubectl logs <pod-name> --timestamps
kubectl logs <pod-name> -c <container-name>
kubectl logs <pod-name> -c <container-name> --previous
kubectl logs <pod-name> --all-containers=true
kubectl logs -l app=flask-api --all-containers=true --tail=100
kubectl logs deployment/<deployment-name> --all-pods=true

kubectl exec -it <pod-name> -- /bin/sh
kubectl exec -it <pod-name> -- /bin/bash
kubectl exec <pod-name> -- printenv
kubectl exec <pod-name> -- ls -la /app
kubectl exec -it <pod-name> -c <container-name> -- /bin/sh

kubectl run test-pod --image=busybox:1.36 --restart=Never -- sleep 3600
kubectl delete pod <pod-name>
kubectl cp <namespace>/<pod-name>:/app/output.txt ./output.txt
kubectl cp ./config.txt <namespace>/<pod-name>:/tmp/config.txt
kubectl run net-debug --image=busybox:1.36 --restart=Never -it --rm -- sh
```

A Pod may be in Running phase while a container restarts or fails readiness. Minimal images may not include bash, curl, or nslookup. Prefer a temporary diagnostic Pod or approved ephemeral debug container when tools are absent.

# 6. Images and image pull failures

```
kubectl get pods -o custom-columns=NAME:.metadata.name,IMAGES:.spec.containers[*].image
kubectl get pods -A -o custom-columns=NS:.metadata.namespace,NAME:.metadata.name,IMAGES:.spec.containers[*].image
kubectl get deployment <deployment-name> -o jsonpath='{range .spec.template.spec.containers[*]}{.name}{"  "}{.image}{"  "}{.imagePullPolicy}{"\n"}{end}'
kubectl get deployment <deployment-name> -o jsonpath='{.spec.template.spec.containers[*].image}'
kubectl get pod <pod-name> -o jsonpath='{range .status.containerStatuses[*]}{.name}{"  "}{.image}{"  "}{.imageID}{"\n"}{end}'
kubectl describe pod <pod-name>

kubectl set image deployment/<deployment-name> <container-name>=<registry>/<repository>:<tag>
kubectl rollout status deployment/<deployment-name>
kubectl rollout history deployment/<deployment-name>
kubectl rollout undo deployment/<deployment-name>
kubectl rollout restart deployment/<deployment-name>
```

| Problem                         | Checks                                                    |
| ------------------------------- | --------------------------------------------------------- |
| ErrImagePull / ImagePullBackOff | Pod events, registry URL, tag, credentials, TLS, network. |
| manifest unknown                | Repository or tag does not exist.                         |
| unauthorized                    | Registry credentials or imagePullSecret.                  |
| no matching manifest            | CPU architecture/platform compatibility.                  |
| x509 error                      | Registry certificate trust.                               |
| Old code still running          | Rollout status, tag reuse, pull policy, runtime imageID.  |

Kubernetes has no general kubectl pull command for arbitrary node images. On a node, Docker may provide docker images/pull/inspect; containerd environments commonly use crictl images/pull/inspecti or ctr -n k8s.io images list. These tools require node access and depend on the runtime.

```
docker images
docker pull nginx:latest
docker inspect nginx:latest
docker image history nginx:latest
docker image rm nginx:latest
docker system df

sudo crictl images
sudo crictl pull nginx:latest
sudo crictl inspecti <image-id>
sudo crictl rmi <image-id>
sudo ctr namespaces list
sudo ctr -n k8s.io images list
```

imagePullPolicy: Always asks the runtime to resolve/pull the image at container start; IfNotPresent uses a cached image if available; Never requires the image to already be present. Prefer versioned tags or immutable digests. Reusing a mutable tag does not guarantee that existing Pods change.

# 7. Deployments and rollouts

```
kubectl get deployments
kubectl get deployments -A
kubectl describe deployment <deployment-name>
kubectl get deployment <deployment-name> -o yaml
kubectl get replicasets
kubectl get replicasets --show-labels
kubectl get deployment <deployment-name> -w
kubectl rollout history deployment/<deployment-name>
kubectl rollout history deployment/<deployment-name> --revision=2

kubectl create deployment flask-api --image=nginx:1.28
kubectl scale deployment/flask-api --replicas=3
kubectl set image deployment/flask-api flask-api=nginx:1.29
kubectl rollout status deployment/flask-api
kubectl rollout pause deployment/flask-api
kubectl rollout resume deployment/flask-api
kubectl rollout undo deployment/flask-api
kubectl rollout undo deployment/flask-api --to-revision=2
kubectl rollout restart deployment/flask-api
kubectl delete deployment flask-api
kubectl scale deployment/flask-api --replicas=0
```

A Deployment is for managing stateless replicas and rolling updates. A Deployment-managed Pod usually gets replaced if deleted. Scaling to zero preserves the Deployment but stops its replicas. If an HPA controls a Deployment, it may adjust replicas after a manual scale.

# 8. Services, ports, DNS, and networking

```
kubectl get services
kubectl get services -o wide
kubectl describe service <service-name>
kubectl get service <service-name> -o yaml
kubectl get endpoints <service-name>
kubectl get endpointslices
kubectl get endpointslices -l kubernetes.io/service-name=<service-name>

kubectl expose deployment flask-api --port=80 --target-port=5000
kubectl expose deployment flask-api --type=NodePort --port=80 --target-port=5000
kubectl expose deployment flask-api --type=LoadBalancer --port=80 --target-port=5000

kubectl port-forward service/flask-api 8080:80
kubectl port-forward pod/<pod-name> 8080:5000
kubectl port-forward deployment/flask-api 8080:5000
kubectl port-forward --address 127.0.0.1 service/flask-api 8080:80

kubectl run dns-test --image=busybox:1.36 --restart=Never -it --rm -- nslookup kubernetes.default
kubectl get pods -A -o wide
kubectl get services -A -o wide
kubectl get networkpolicies -A
kubectl describe networkpolicy <policy-name>
```

| Service type | Use                                                               |
| ------------ | ----------------------------------------------------------------- |
| ClusterIP    | Internal cluster endpoint.                                        |
| NodePort     | Port exposed on eligible nodes.                                   |
| LoadBalancer | Requests an external load balancer from supported infrastructure. |
| ExternalName | Maps a Service to an external DNS name.                           |

If traffic fails, check Service selectors, Pod labels, ready endpoints, targetPort, DNS, NetworkPolicies, firewall rules, and the cluster's CNI. Port-forward is a debugging/local-access method, not a production ingress configuration.

# 9. Events and common troubleshooting

```
kubectl get events
kubectl get events -A
kubectl get events -A --sort-by=.metadata.creationTimestamp
kubectl describe pod <pod-name>
kubectl describe deployment <deployment-name>
kubectl get pods -o custom-columns=NAME:.metadata.name,STATUS:.status.phase,RESTARTS:.status.containerStatuses[*].restartCount
kubectl get pod <pod-name> -o yaml
kubectl get resourcequotas -A
kubectl get limitranges -A
kubectl top pods
kubectl top pods -A
kubectl top nodes
```

| Status / symptom           | Investigate                                          |
| -------------------------- | ---------------------------------------------------- |
| Pending                    | Scheduling, node capacity, taints, affinity, PVCs.   |
| ContainerCreating          | Image pull, mounts, runtime, network setup.          |
| Running but not Ready      | Readiness probe, app health, dependencies.           |
| CrashLoopBackOff           | Current/previous logs, exit reason, configuration.   |
| ImagePullBackOff           | Image reference, registry credentials, connectivity. |
| CreateContainerConfigError | Missing Secret/ConfigMap or invalid configuration.   |
| OOMKilled                  | Memory limit, memory use, leak, workload sizing.     |
| Evicted                    | Node pressure and eviction conditions.               |
| Terminating for too long   | Finalizers, graceful shutdown, mounts, node status.  |
| Unknown                    | Node/kubelet communication.                          |

## Standard troubleshooting sequence

- Run kubectl get pods -o wide.
- Run kubectl describe pod &lt;pod-name&gt; and read the Events section.
- Read kubectl logs &lt;pod-name&gt;; if it crashed, use --previous.
- Check node status, resource usage, PVCs, Secrets, ConfigMaps, and Service endpoints.
- Apply a targeted fix, then verify with kubectl rollout status and a fresh status/log check.

# 10. ConfigMaps, Secrets, and environment variables

```
kubectl get configmaps
kubectl describe configmap <configmap-name>
kubectl get configmap <configmap-name> -o yaml
kubectl create configmap app-config --from-file=app.conf
kubectl create configmap app-settings --from-literal=APP_MODE=production --from-literal=LOG_LEVEL=INFO
kubectl delete configmap <configmap-name>

kubectl get secrets
kubectl describe secret <secret-name>
kubectl create secret generic app-secret --from-literal=API_KEY='<your-api-key>'
kubectl create secret tls app-tls --cert=fullchain.pem --key=privkey.pem
kubectl delete secret <secret-name>

kubectl exec <pod-name> -- printenv
kubectl set env deployment/flask-api LOG_LEVEL=INFO
kubectl set env deployment/flask-api LOG_LEVEL-
kubectl get deployment flask-api -o yaml
kubectl rollout restart deployment/flask-api
```

Secret data shown in YAML is generally base64-encoded, not encrypted by virtue of that encoding. Avoid printing secret values in logs or shared terminals. Environment variables in existing containers do not change when a Secret or ConfigMap changes; applications using mounted files may see eventual updates but must reload/use them appropriately.

# 11. Persistent storage

```
kubectl get pvc
kubectl get pvc -A
kubectl describe pvc <pvc-name>
kubectl get pv
kubectl describe pv <pv-name>
kubectl get storageclass
kubectl describe storageclass <storageclass-name>
kubectl get pod <pod-name> -o yaml
kubectl get events --sort-by=.metadata.creationTimestamp
```

- PVC Pending: check storage provisioning, access modes, capacity, topology, and events.
- PVC Bound: a compatible PersistentVolume has been bound; this does not by itself guarantee the application can mount or use it.
- Deleting PVCs may lead to data loss depending on reclaim policy and storage provider. Never delete a production PVC as a casual troubleshooting step.

# 12. Resources and autoscaling

```
kubectl top nodes
kubectl top pods
kubectl top pods -A
kubectl top pods --containers
kubectl describe pod <pod-name>
kubectl get deployment <deployment-name> -o yaml
kubectl describe resourcequota <quota-name>
kubectl get hpa
kubectl describe hpa <hpa-name>
kubectl autoscale deployment flask-api --min=2 --max=10 --cpu=70%
kubectl get hpa -w
kubectl delete hpa <hpa-name>
```

Example resource configuration:

```
resources:
  requests:
    cpu: "250m"
    memory: "256Mi"
  limits:
    cpu: "1"
    memory: "512Mi"
```

Requests influence scheduling and resource accounting. CPU limits constrain CPU use; memory limit breaches can lead to termination. CPU-based HPA commonly needs resource requests and a functioning metrics pipeline. HPA scales replicas, not node capacity.

# 13. Jobs, CronJobs, StatefulSets, and DaemonSets

```
kubectl get jobs
kubectl describe job <job-name>
kubectl logs job/<job-name>
kubectl create job manual-job --image=busybox:1.36 -- echo "Hello Kubernetes"
kubectl delete job <job-name>

kubectl get cronjobs
kubectl describe cronjob <cronjob-name>
kubectl get cronjobs -o yaml
kubectl delete cronjob <cronjob-name>

kubectl get statefulsets
kubectl describe statefulset <statefulset-name>
kubectl get statefulset <statefulset-name> -o yaml
kubectl rollout status statefulset/<statefulset-name>
kubectl rollout restart statefulset/<statefulset-name>
kubectl scale statefulset/<statefulset-name> --replicas=3

kubectl get daemonsets -A
kubectl describe daemonset <daemonset-name>
kubectl rollout status daemonset/<daemonset-name>
kubectl rollout restart daemonset/<daemonset-name>
kubectl rollout undo daemonset/<daemonset-name>
```

CronJobs can retry and may produce duplicate executions; design scheduled tasks to tolerate retries. StatefulSets provide stable identities and are often paired with persistent storage. DaemonSets run Pods on selected nodes, often for logging, monitoring, or networking.

# 14. Labels, selectors, annotations, and output

```
kubectl get pods --show-labels
kubectl label pod <pod-name> environment=testing
kubectl label pod <pod-name> environment=production --overwrite
kubectl label pod <pod-name> environment-
kubectl get pods -l app=flask-api
kubectl get pods -l 'app=flask-api,environment=production'
kubectl get pods -l environment
kubectl get pods -l '!environment'

kubectl get pods -o wide
kubectl get pods -o yaml
kubectl get pods -o json
kubectl get pods -o name
kubectl get pods -o custom-columns=NAME:.metadata.name
kubectl get pods -o jsonpath='{.items[*].metadata.name}'
```

- \-A: all namespaces; -n: one namespace; -o wide: extra columns; -o yaml/json: structured output.
- \-w: watch; -l: label selector; --field-selector: supported field filter; --sort-by: sort output.
- \--dry-run=client: local simulation; --dry-run=server: API-server validation without persisting, where supported.
- Labels are for grouping/selecting; annotations are metadata. Do not put secrets in labels or annotations.

# 15. Manifests and declarative workflow

```
kubectl apply -f deployment.yaml
kubectl apply -f ./k8s/
kubectl diff -f deployment.yaml
kubectl apply --dry-run=server -f deployment.yaml
kubectl create deployment flask-api --image=nginx:1.28 --dry-run=client -o yaml
kubectl delete -f deployment.yaml
kubectl replace -f deployment.yaml
```

Use apply for the normal declarative workflow. Keep manifests in version control and review kubectl diff before production changes. Server-side dry-run can validate a request without saving it, but it does not prove the workload will become healthy.

# 16. RBAC and authorization

```
kubectl get roles -A
kubectl get clusterroles
kubectl get rolebindings -A
kubectl get clusterrolebindings
kubectl describe role <role-name> -n <namespace>
kubectl describe clusterrole <clusterrole-name>
kubectl describe rolebinding <binding-name> -n <namespace>
kubectl auth can-i get pods
kubectl auth can-i create deployments -n <namespace>
kubectl auth can-i delete pods -n <namespace>
kubectl auth can-i --list -n <namespace>

kubectl create role pod-reader --verb=get,list,watch --resource=pods -n testing
kubectl create rolebinding pod-reader-binding --role=pod-reader --user=<user-name> -n testing
```

Grant the least privilege needed. Do not use cluster-admin as a shortcut for ordinary authorization errors.

# 17. Node maintenance and cluster administration

```
kubectl cordon <node-name>
kubectl drain <node-name> --ignore-daemonsets
kubectl uncordon <node-name>
kubectl describe node <node-name>
kubectl get nodes -o wide

kubectl get pods -n kube-system
kubectl get services -n kube-system
kubectl get configmaps -n kube-system
kubectl get pods -A -o wide
kubectl get resourcequotas -A
kubectl get limitranges -A
kubectl get pdb -A
kubectl describe node <node-name>
```

Drain evicts eligible workloads and can be blocked by PodDisruptionBudgets. DaemonSet Pods are not removed by the normal drain operation. Understand availability and storage implications before maintenance. Managed Kubernetes services may not expose control-plane Pods.

# 18. Security, certificates, and policies

```
kubectl auth can-i --list
kubectl get serviceaccounts -A
kubectl get networkpolicies -A
kubectl get namespaces --show-labels
kubectl get secrets -A -o custom-columns=NS:.metadata.namespace,NAME:.metadata.name,TYPE:.type
kubectl api-resources | grep -i certificate

# If cert-manager and its CRDs are installed:
kubectl get certificates -A
kubectl get certificaterequests -A
kubectl get issuers -A
kubectl get clusterissuers
```

HTTPS commonly requires a valid hostname, certificate, TLS termination at an ingress/gateway or load balancer, and correct routing. cert-manager commands only work when the corresponding CRDs and permissions exist.

# 19. Helm

```
helm list -A
helm status <release-name> -n <namespace>
helm history <release-name> -n <namespace>
helm get values <release-name> -n <namespace>
helm get values <release-name> -n <namespace> --all
helm get manifest <release-name> -n <namespace>
helm upgrade <release-name> <chart> -n <namespace> --dry-run
helm upgrade --install <release-name> <chart> -n <namespace>
helm rollback <release-name> <revision> -n <namespace>
helm uninstall <release-name> -n <namespace>
```

Helm is a separate package manager, not a kubectl subcommand. Review release resources and persistence before uninstalling.

# 20. Production checklist and common workflows

## Before deployment

- Confirm context and namespace.
- Review YAML and run kubectl diff.
- Verify image repository, tag/digest, pull secret, and target architecture.
- Set appropriate requests, limits, readiness/liveness/startup probes, and security settings.
- Verify ConfigMaps, Secrets, PVCs, Service selectors, and ingress/gateway routing.
- Use a rollout strategy and an application-level health check.

## After deployment

```
kubectl rollout status deployment/<deployment-name>
kubectl get deployment <deployment-name>
kubectl get pods -o wide
kubectl get services
kubectl get endpointslices
kubectl logs deployment/<deployment-name> --all-pods=true
kubectl get events --sort-by=.metadata.creationTimestamp
```

## If the rollout fails

```
kubectl describe deployment <deployment-name>
kubectl describe pod <pod-name>
kubectl logs <pod-name>
kubectl logs <pod-name> --previous
kubectl rollout history deployment/<deployment-name>
# After confirming the correct revision and impact:
kubectl rollout undo deployment/<deployment-name>
kubectl rollout status deployment/<deployment-name>
```

## Fast symptom-to-command map

| Symptom             | Start with                                                   |
| ------------------- | ------------------------------------------------------------ |
| Application down    | get pods; describe pod; logs; logs --previous                |
| Image pull fails    | describe pod; check image, tag, registry credentials, events |
| Pod Pending         | describe pod; nodes; PVCs; quotas; taints and affinity       |
| Service unreachable | describe service; endpointslices; DNS; ports; NetworkPolicy  |
| Rollout unhealthy   | rollout status; describe deployment; logs; rollout history   |
| Memory/CPU issue    | top pods; top nodes; describe pod; requests and limits       |
| Node maintenance    | get nodes; pdb; cordon; drain; uncordon after maintenance    |

# 21. Official references

- **kubectl command reference:** <https://kubernetes.io/docs/reference/kubectl/>
- **Kubernetes architecture:** <https://kubernetes.io/docs/concepts/architecture/>
- **Cluster administration:** <https://kubernetes.io/docs/concepts/cluster-administration/>
- **Debugging applications:** <https://kubernetes.io/docs/tasks/debug/>
- **Debug Pods:** <https://kubernetes.io/docs/tasks/debug/debug-application/debug-pods/>
- **Debug Services:** <https://kubernetes.io/docs/tasks/debug/debug-application/debug-service/>
- **Deployments:** <https://kubernetes.io/docs/concepts/workloads/controllers/deployment/>
- **Services:** <https://kubernetes.io/docs/concepts/services-networking/service/>
- **RBAC:** <https://kubernetes.io/docs/reference/access-authn-authz/rbac/>
- **Persistent volumes:** <https://kubernetes.io/docs/concepts/storage/persistent-volumes/>

# Final notes

- This is a broad practical reference, not literally every kubectl flag or every Kubernetes API operation; the CLI has many resource- and version-specific options.
- Use kubectl &lt;command&gt; --help and kubectl explain &lt;resource&gt; to discover details for your cluster version.
- Avoid force-deleting Pods, deleting PVCs, draining nodes, deleting namespaces, or changing RBAC without understanding the consequences.
- Never paste credentials, tokens, kubeconfig contents, or decoded Secrets into public logs or chats.
