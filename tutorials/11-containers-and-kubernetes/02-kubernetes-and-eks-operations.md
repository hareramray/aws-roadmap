# Reconcile, break, and recover a Kubernetes application

[Container tutorial index](README.md) · [All tutorials](../README.md) · [Roadmap chapter](../../11-containers-and-kubernetes.md)

**Outcome:** Observe how Deployments, Services, probes, identities, and disruptions affect a two-replica application. Reuse the image from [the ECS tutorial](01-ecs-container-service.md). An already installed local Kubernetes cluster avoids AWS charges. Using EKS is optional and bills the cluster, compute, storage, networking, and observability resources even when your application is idle.

## 1. Establish the cluster boundary

Run `kubectl config current-context` and `kubectl cluster-info`. Stop unless the context is the intended sandbox. Record context, server version, node architectures, and image reference. Do not install a second network plugin into an existing cluster casually. A local cluster's image-loading and network-policy behavior depend on its distribution.

For a local cluster, load the previously built `roadmap-http:v1` image using that distribution's supported loader, then set the manifest's image to that local name and pull policy to `Never`. For EKS, use the ECR digest and `IfNotPresent`. Worker nodes need registry access through their node role/network path; an application service account does not normally provide the initial image-pull identity.

Create `workload.yaml` below. Replace `REPLACE_WITH_IMAGE_DIGEST` before applying. The Service is internal and creates no AWS load balancer. Port forwarding gives the learner a temporary local access path.

```yaml
apiVersion: v1
kind: Namespace
metadata: {name: roadmap-containers}
---
apiVersion: v1
kind: ServiceAccount
metadata: {name: web, namespace: roadmap-containers}
automountServiceAccountToken: false
---
apiVersion: apps/v1
kind: Deployment
metadata: {name: web, namespace: roadmap-containers}
spec:
  replicas: 2
  selector:
    matchLabels: {app: roadmap-web}
  template:
    metadata:
      labels: {app: roadmap-web}
    spec:
      serviceAccountName: web
      terminationGracePeriodSeconds: 20
      securityContext:
        runAsNonRoot: true
        runAsUser: 10001
        runAsGroup: 10001
        seccompProfile: {type: RuntimeDefault}
      containers:
        - name: web
          image: REPLACE_WITH_IMAGE_DIGEST
          imagePullPolicy: IfNotPresent
          ports: [{containerPort: 8080, name: http}]
          env: [{name: RELEASE, value: kubernetes-v1}]
          resources:
            requests: {cpu: 50m, memory: 64Mi}
            limits: {cpu: 250m, memory: 128Mi}
          securityContext:
            readOnlyRootFilesystem: true
            allowPrivilegeEscalation: false
            capabilities: {drop: [ALL]}
          startupProbe:
            httpGet: {path: /health, port: http}
            periodSeconds: 2
            failureThreshold: 15
          readinessProbe:
            httpGet: {path: /health, port: http}
            periodSeconds: 5
          livenessProbe:
            httpGet: {path: /health, port: http}
            periodSeconds: 10
            failureThreshold: 3
---
apiVersion: v1
kind: Service
metadata: {name: web, namespace: roadmap-containers}
spec:
  selector: {app: roadmap-web}
  ports: [{port: 80, targetPort: http}]
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata: {name: web, namespace: roadmap-containers}
spec:
  minAvailable: 1
  selector:
    matchLabels: {app: roadmap-web}
```

## 2. Apply and prove readiness

Create the dedicated namespace before the server-side dry run: dry-run requests do not persist a namespace for subsequent namespaced objects. Refuse an existing namespace until its ownership is established; do not adopt an unrelated namespace for the exercise.

```powershell
$existingNamespace = kubectl get namespace roadmap-containers --ignore-not-found -o name
if ($LASTEXITCODE -ne 0) { throw 'Cannot inspect namespace ownership' }
if ($existingNamespace) { throw 'Namespace already exists; verify ownership before continuing' }
kubectl create namespace roadmap-containers
if ($LASTEXITCODE -ne 0) { throw 'Namespace creation failed' }
kubectl apply --dry-run=server -f .\workload.yaml
if ($LASTEXITCODE -ne 0) { throw 'Server validation failed' }
kubectl apply -f .\workload.yaml
if ($LASTEXITCODE -ne 0) { throw 'Apply failed' }
kubectl rollout status deployment/web -n roadmap-containers --timeout=120s
if ($LASTEXITCODE -ne 0) { throw 'Rollout did not stabilize; inspect pods/events' }
kubectl get pods,services,endpointslices -n roadmap-containers
kubectl port-forward -n roadmap-containers service/web 8080:80
```

Port forwarding occupies that terminal. In another PowerShell window, run `Invoke-RestMethod http://127.0.0.1:8080/health`. Expect `kubernetes-v1`. Stop forwarding with Ctrl+C afterward. Inspect logs, `kubectl describe pod`, and rollout history. A Deployment creates ReplicaSets, which manage Pods; a Service selects endpoints by labels. A Running Pod need not be Ready, and a Service with no ready endpoints cannot route normally.

Readiness should describe whether a replica can receive traffic. Liveness should detect a stuck process, not restart every replica whenever a shared database is briefly unavailable. A startup probe prevents liveness from prematurely killing a legitimately slow starter. Resource requests guide placement; limits constrain runtime behavior and can introduce CPU throttling or memory termination.

## 3. Inject one reversible failure

Copy the manifest to `broken.yaml` and change only the readiness path to `/missing`. Apply it and observe rollout status with a 60-second timeout. The new Pods can be Running but not Ready. Inspect events and EndpointSlices, compare the new ReplicaSet with the old one, and check whether old ready replicas remain serving. The HTTP app deliberately returns 404 for that path.

Reapply `workload.yaml` and wait for recovery. Alternatively, inspect rollout history and use `kubectl rollout undo deployment/web -n roadmap-containers` when the previous revision is known. Rollback restores workload configuration, not data migrations. Delete one disposable Pod and observe its controller replacement. A PDB limits voluntary evictions; it does not prevent node failure or guarantee that arbitrary deletion preserves availability.

## 4. Add configuration, identity, and network controls deliberately

Move `RELEASE` into a ConfigMap and reference it with `env.valueFrom.configMapKeyRef`; change the ConfigMap and observe that environment values in existing containers do not update until they restart. A mounted ConfigMap has different refresh behavior, and the application still needs to reread it. Kubernetes Secrets require encryption/access controls; base64 encoding is not encryption. Use an approved external secret integration when appropriate and avoid committing secret manifests.

Verify Kubernetes authorization with `kubectl auth can-i get pods -n roadmap-containers` for your learner identity and separately for a constrained test service account. EKS access entries map AWS identities to cluster access; Pod Identity or IRSA provides AWS permissions to workloads. These are different directions of trust.

For an optional EKS Pod Identity experiment, provision the documented agent when required by the selected compute mode, create a role trusted for EKS Pod Identity with only access to a dedicated test resource, and associate it with the service account:

```powershell
aws eks create-pod-identity-association --cluster-name $env:LAB_EKS_CLUSTER --namespace roadmap-containers --service-account web --role-arn $env:LAB_POD_ROLE
if ($LASTEXITCODE -ne 0) { throw 'Association failed' }
```

Record the association ID for deletion. Use a compatible SDK/default credential chain in a separate diagnostic workload to call STS and one allowed resource operation; verify a different resource is denied. Do not print tokens. Understand projected credentials and agent connectivity rather than assuming a service-account label grants permission. Restrict IMDS access; containers are not a strong isolation boundary solely because they use separate roles.

Apply a default-deny NetworkPolicy only when the installed CNI actually enforces it. Test allowed and denied traffic from a temporary diagnostic Pod, allow DNS explicitly if needed, and confirm enforcement with requests. Some local cluster defaults accept policy objects without enforcing them. EKS VPC CNI also consumes subnet addresses; inspect available addresses, ENIs, prefix delegation settings, and pending-Pod events before increasing replicas.

## 5. Exercise storage, scaling, and maintenance

Create a separate tiny PVC using an existing sandbox StorageClass after inspecting its provisioner, volume binding mode, and reclaim policy. Mount it in a disposable writer Pod, write a timestamp, recreate the Pod, and verify persistence. For EBS CSI, understand single-AZ placement and access modes; test what happens when scheduling conflicts with the volume's zone. EFS serves different shared-filesystem requirements. A snapshot needs a tested restore; a Deployment manifest alone cannot recreate lost data. Delete the PVC only after recording whether the backing volume will be deleted or retained.

For scaling, first change replicas from two to three and observe scheduling. If pending, distinguish insufficient requests-capacity from IP exhaustion or affinity. HPA needs supported metrics and reasonable requests; node autoscaling/Karpenter addresses node capacity, not application readiness. Managed node groups, Auto Mode, and EKS Fargate shift infrastructure ownership differently and have different feature constraints. A local HPA experiment must not be mistaken for production capacity evidence.

Plan an EKS upgrade as a sequence: inspect deprecated APIs and upgrade insights; verify controller/add-on compatibility; back up manifests and application data; test in a disposable cluster; update the control plane through supported versions; update compatible add-ons and nodes according to AWS guidance; drain gradually; verify DNS, storage, identity, and workloads after each change. The control plane cannot simply be downgraded to recover. Normal managed-cluster upgrades do not automatically solve all VPC CNI/add-on lifecycle work. Record an application/cluster rebuild path rather than promising a universal rollback.

## 6. Cost and cleanup

Delete the namespace after saving evidence. Verify any optional LoadBalancer Services/Ingress resources are removed before deleting a cluster so their controllers can clean up cloud resources. Remove test Pod Identity associations/roles, dedicated PVCs/backing disks/snapshots, and diagnostic objects. If you created an EKS cluster, delete its dedicated compute and cluster using its provisioning tool; inspect ENIs, load balancers, NAT gateways, endpoints, EBS volumes, logs, and ECR afterward. Never delete shared infrastructure based on a name prefix alone.

Explain why readiness and liveness differ, which identity pulls an image, why Pod Identity does not grant Kubernetes administrator access, how a PDB can block a drain, and why a successful manifest apply does not prove network-policy enforcement.

Official references: [Kubernetes probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/), [EKS Pod Identity associations](https://docs.aws.amazon.com/eks/latest/userguide/pod-id-association.html), [EKS VPC CNI practices](https://docs.aws.amazon.com/eks/latest/best-practices/vpc-cni.html), [EKS cluster updates](https://docs.aws.amazon.com/eks/latest/userguide/update-cluster.html).
