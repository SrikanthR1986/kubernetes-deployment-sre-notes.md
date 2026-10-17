# Kubernetes Deployment, DevOps & SRE Operations Handbook

## 1. Kubernetes Overview

Kubernetes (K8s) is a container orchestration platform used to deploy, scale, manage, and recover containerized applications.

### Kubernetes architecture

- **Control plane:** API Server, Scheduler, Controller Manager, and etcd.
- **Worker nodes:** Run application workloads.
- **Pod:** Smallest deployable unit; contains one or more containers.
- **Deployment:** Manages ReplicaSets and stateless application replicas.
- **Service:** Provides stable network access to Pods.
- **Ingress / Gateway API:** Manages HTTP/HTTPS routing into the cluster.
- **ConfigMap:** Stores non-sensitive application configuration.
- **Secret:** Stores sensitive configuration; enable encryption and restrictive RBAC.
- **HPA:** Horizontal Pod Autoscaler adjusts replica counts.
- **StatefulSet:** Manages workloads requiring stable identities or storage.
- **DaemonSet:** Runs Pods on selected nodes.
- **Job / CronJob:** Runs batch or scheduled workloads.

### Typical production deployment flow

```text
Developer pushes code to GitHub
           |
           v
CI pipeline builds and tests application
           |
           v
Container image pushed to Artifact Registry
           |
           v
CD pipeline validates Kubernetes YAML / Helm
           |
           v
Deploy to Development / QA / Staging
           |
           v
Smoke tests + Security checks
           |
           v
Production approval
           |
           v
Rolling / Canary / Blue-Green deployment
           |
           v
Monitor health, logs, metrics and alerts
           |
           v
Rollback or roll forward if necessary
```

---

## 2. Kubernetes and GKE Prerequisites

Install and configure the tools appropriate to your environment:

```bash
kubectl version --client
gcloud version
helm version
docker version
```

### Connect to a GKE cluster

```bash
gcloud auth login
gcloud config set project PROJECT_ID

gcloud container clusters get-credentials CLUSTER_NAME \
  --region REGION \
  --project PROJECT_ID
```

For a zonal cluster, use `--zone ZONE` instead of `--region REGION`.

Verify the connection:

```bash
kubectl config get-contexts
kubectl config current-context
kubectl cluster-info
kubectl get nodes -o wide
kubectl get namespaces
```

**SRE practice:** Always verify the current cluster, context, namespace, and environment before making production changes.

---

## 3. Essential kubectl Commands

### Cluster and node operations

```bash
kubectl cluster-info
kubectl get nodes
kubectl get nodes -o wide
kubectl describe node NODE_NAME
kubectl top nodes
kubectl get componentstatuses
```

`kubectl top` requires Metrics Server or a compatible metrics provider. The legacy `componentstatuses` command is deprecated in modern Kubernetes; prefer cluster-specific health checks and monitoring.

### Namespace operations

```bash
kubectl get namespaces
kubectl create namespace payments
kubectl get all -n payments
kubectl describe namespace payments
```

Delete only after confirming the impact:

```bash
kubectl delete namespace payments
```

Namespace deletion can remove many resources and disrupt applications.

### Pod operations

```bash
kubectl get pods
kubectl get pods -A
kubectl get pods -n payments -o wide
kubectl describe pod POD_NAME -n payments
kubectl logs POD_NAME -n payments
kubectl logs POD_NAME -n payments --previous
kubectl logs POD_NAME -n payments --tail=100 -f
kubectl exec -it POD_NAME -n payments -- /bin/sh
```

For multi-container Pods:

```bash
kubectl logs POD_NAME -n payments -c CONTAINER_NAME
kubectl exec -it POD_NAME -n payments -c CONTAINER_NAME -- /bin/sh
```

`--previous` helps investigate a terminated container instance. Minimal container images may not include `/bin/sh` or debugging utilities.

### Inspect resources

```bash
kubectl get all -n payments
kubectl get events -n payments \
  --sort-by=.metadata.creationTimestamp

kubectl get deployment,replicaset,pod,service -n payments
kubectl get configmap,secret -n payments
kubectl api-resources
```

---

## 4. Create a Deployment Using YAML

Create `deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-app
  namespace: payments
spec:
  replicas: 3
  selector:
    matchLabels:
      app: payment-app
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: payment-app
    spec:
      containers:
        - name: payment-app
          image: nginx:1.28
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 10
            periodSeconds: 20
```

This is a deployment exercise using an NGINX image, not a production payment application. NGINX listens on port 80 by default; the manifest's `containerPort: 8080` is illustrative and should be changed to 80 for consistency. In a real application, configure the actual image, port, health endpoints, security context, and probes.

Create the namespace and deploy:

```bash
kubectl create namespace payments
kubectl apply -f deployment.yaml
kubectl get deployments -n payments
kubectl rollout status deployment/payment-app -n payments
kubectl get pods -n payments -o wide
```

Validate the manifest before deployment:

```bash
kubectl apply --dry-run=client -f deployment.yaml
kubectl diff -f deployment.yaml
```

A client-side dry run validates client-side processing; it does not guarantee the cluster will accept the resource or that the application will run correctly.

---

## 5. Service Configuration

Create `service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: payment-service
  namespace: payments
spec:
  type: ClusterIP
  selector:
    app: payment-app
  ports:
    - name: http
      port: 80
      targetPort: 80
      protocol: TCP
```

Apply and inspect:

```bash
kubectl apply -f service.yaml
kubectl get services -n payments
kubectl describe service payment-service -n payments
kubectl get endpointslices -n payments
```

### Service types

- `ClusterIP`: Internal cluster access.
- `NodePort`: Exposes a port on cluster nodes.
- `LoadBalancer`: Requests a supported external or internal load balancer.
- `ExternalName`: Maps a service name to an external DNS name.

### Test service connectivity

```bash
kubectl port-forward service/payment-service 8080:80 -n payments
```

In another terminal:

```bash
curl http://localhost:8080/
```

Port forwarding is useful for diagnostics, not a production traffic-routing solution.

---

## 6. ConfigMap and Secret

### Create a ConfigMap

```bash
kubectl create configmap payment-config \
  --from-literal=APP_ENV=dev \
  -n payments
```

Example YAML:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: payment-config
  namespace: payments
data:
  APP_ENV: "dev"
  LOG_LEVEL: "INFO"
```

Reference it from a Deployment:

```yaml
envFrom:
  - configMapRef:
      name: payment-config
```

### Create a Secret

```bash
kubectl create secret generic payment-secret \
  --from-literal=DB_USERNAME='appuser' \
  -n payments
```

Supply sensitive values through an approved secret-management workflow; do not place real credentials in Git or shell history.

Inspect only metadata when possible:

```bash
kubectl describe secret payment-secret -n payments
```

Kubernetes Secrets are not automatically encrypted at rest in every configuration. Configure encryption at rest, least-privilege RBAC, and preferably an approved external secret manager.

---

## 7. Rolling Updates, Scaling, and Rollbacks

### Update an image

```bash
kubectl set image deployment/payment-app \
  payment-app=YOUR_IMAGE:YOUR_TAG \
  -n payments
```

Use a valid, immutable image tag from your approved registry.

### Monitor the rollout

```bash
kubectl rollout status deployment/payment-app -n payments
kubectl rollout history deployment/payment-app -n payments
kubectl describe deployment payment-app -n payments
```

### Roll back a deployment

```bash
kubectl rollout undo deployment/payment-app -n payments
kubectl rollout status deployment/payment-app -n payments
```

Roll back to a particular revision:

```bash
kubectl rollout history deployment/payment-app -n payments
kubectl rollout undo deployment/payment-app \
  --to-revision=2 -n payments
```

A Deployment rollback restores a previous Pod template. It does not automatically reverse database migrations or external system changes.

### Scale replicas

```bash
kubectl scale deployment/payment-app \
  --replicas=5 -n payments
```

If GitOps or another controller manages replicas, update the source configuration too; otherwise, reconciliation may revert the manual change.

---

## 8. Autoscaling with HPA

Example HPA manifest:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: payment-hpa
  namespace: payments
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: payment-app
  minReplicas: 3
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

Apply and monitor:

```bash
kubectl apply -f hpa.yaml
kubectl get hpa -n payments
kubectl describe hpa payment-hpa -n payments
kubectl top pods -n payments
```

HPA CPU utilization generally depends on configured CPU requests and an available metrics API. HPA scales Pods, not necessarily nodes; cluster autoscaling is a separate capability.

---

## 9. Ingress and External Access

Ingress requires a compatible Ingress controller or implementation. On GKE, select the supported load-balancing approach for your cluster and requirements.

Inspect routing resources:

```bash
kubectl get ingress -A
kubectl describe ingress INGRESS_NAME -n payments
kubectl get gateway,httproute -A
```

Typical troubleshooting areas:

- DNS record points to the correct address.
- Load balancer and health checks are healthy.
- Ingress class or Gateway configuration is correct.
- Service selectors match the Pod labels.
- Endpoints exist and the target port is correct.
- Firewall, TLS certificate, and application routing settings are valid.

For modern Kubernetes networking, evaluate Gateway API when supported by the chosen implementation.

---

## 10. Helm Operations

Helm packages Kubernetes resources into reusable charts.

```bash
helm repo add stable https://charts.helm.sh/stable
helm repo update
helm list -A
```

For your own application chart:

```bash
helm lint ./chart
helm template payment-app ./chart -n payments
helm upgrade --install payment-app ./chart \
  --namespace payments --create-namespace
helm status payment-app -n payments
helm history payment-app -n payments
helm rollback payment-app 2 -n payments
```

Inspect rendered manifests and values before applying a production change. Do not assume Helm rollback reverses external side effects or database migrations.

---

## 11. CI/CD Integration with Jenkins and GitHub

Typical workflow:

1. Developer pushes code or opens a pull request.
2. Jenkins checks out source.
3. Unit tests, linting, and security checks run.
4. Docker image is built.
5. Image is pushed to Artifact Registry or another approved registry.
6. Deployment manifests or Helm values are updated.
7. A deployment pipeline deploys to the target cluster.
8. Rollout and smoke tests validate the release.
9. Production approval and monitoring follow organizational policy.

Example Jenkins pipeline stage:

```groovy
stage('Deploy to Kubernetes') {
    steps {
        sh '''
          kubectl apply -f k8s/
          kubectl rollout status deployment/payment-app \
            -n payments --timeout=180s
        '''
    }
}
```

Ensure the agent has the correct cluster credentials, context, namespace, and RBAC. For production, use protected credentials, reviewed changes, and preferably a GitOps workflow where appropriate.

### GitOps with Argo CD or Flux

- Git stores the desired cluster configuration.
- A GitOps controller reconciles the cluster with the declared state.
- Pull requests provide review and auditability.
- Drift is detected and can be reconciled according to policy.
- Rollback is normally performed by reverting the relevant Git change and allowing reconciliation.

---

## 12. Troubleshooting: Common Production Scenarios

### A. Pod is in `Pending`

```bash
kubectl describe pod POD_NAME -n payments
kubectl get events -n payments \
  --sort-by=.metadata.creationTimestamp
kubectl get nodes
```

Check CPU/memory requests, node capacity, affinity, taints/tolerations, PVC binding, and quota restrictions.

### B. Pod is in `CrashLoopBackOff`

```bash
kubectl logs POD_NAME -n payments --previous
kubectl describe pod POD_NAME -n payments
kubectl get events -n payments
```

Check application startup errors, incorrect configuration, missing secrets, command/arguments, health probes, and memory limits.

### C. `ImagePullBackOff`

Check the image name/tag, registry connectivity, image pull permissions, registry secrets, and node/workload identity configuration.

```bash
kubectl describe pod POD_NAME -n payments
kubectl get serviceaccount -n payments
```

### D. Service is not reachable

```bash
kubectl get svc -n payments
kubectl get pods -n payments --show-labels
kubectl get endpointslices -n payments
kubectl describe svc payment-service -n payments
```

Verify selectors, readiness, port/targetPort, network policies, DNS, and application listening address.

### E. Deployment rollout is stuck

```bash
kubectl rollout status deployment/payment-app -n payments
kubectl describe deployment payment-app -n payments
kubectl get replicasets -n payments
kubectl get pods -n payments -o wide
```

Investigate failed readiness probes, image pulls, resource constraints, quota, scheduling, and application startup. Roll back if necessary and safe.

### F. High CPU or memory

```bash
kubectl top nodes
kubectl top pods -n payments
kubectl describe pod POD_NAME -n payments
```

Check requests/limits, throttling, memory growth, traffic spikes, and application-level metrics. Increase resources only after evaluating capacity and root cause.

### G. DNS or cross-namespace communication failure

Use the full service DNS name:

```text
payment-service.payments.svc.cluster.local
```

Check service existence, endpoints, CoreDNS health, network policies, and port configuration.

### H. Node is `NotReady`

```bash
kubectl get nodes
kubectl describe node NODE_NAME
kubectl get events -A --sort-by=.metadata.creationTimestamp
```

Check node health, resource pressure, kubelet/runtime issues, networking, and cloud-provider events. Follow the cluster's node-repair and escalation procedure.

---

## 13. SRE Monitoring and Reliability

### Golden signals

- **Latency:** How long requests take.
- **Traffic:** Request volume.
- **Errors:** Failed requests and application failures.
- **Saturation:** Resource pressure and capacity limits.

### Useful Kubernetes commands

```bash
kubectl top pods -A
kubectl top nodes
kubectl get events -A
kubectl get pods -A -o wide
kubectl logs deployment/payment-app -n payments --since=30m
```

Use Cloud Monitoring and Cloud Logging for GKE, along with Prometheus and Grafana where appropriate.

### SRE practices

- Define SLIs and SLOs for latency, availability, and errors.
- Alert on user impact rather than every transient symptom.
- Use readiness probes to control traffic eligibility.
- Use liveness probes carefully to detect unrecoverable application states.
- Use startup probes for slow-starting applications.
- Set appropriate resource requests and limits.
- Configure PodDisruptionBudgets for suitable workloads.
- Use multiple replicas and topology spread/anti-affinity where required.
- Conduct incident reviews and track corrective actions.
- Test recovery, rollback, and capacity assumptions.

---

## 14. Production Security Checklist

- Apply least-privilege Kubernetes RBAC.
- Use non-root containers where possible.
- Configure security contexts and restrict Linux capabilities.
- Store secrets securely and enable encryption at rest.
- Use network policies where supported.
- Scan container images and dependencies.
- Pin image versions and avoid mutable `latest` tags in production.
- Use workload identity rather than long-lived cloud keys where possible.
- Restrict privileged containers and host access.
- Keep clusters and nodes on supported versions.
- Protect deployment pipelines and production credentials.
- Review audit logs and restrict access to production namespaces.

---

## 15. Kubernetes Interview Questions and Sample Answers

### Q1. Explain the Kubernetes deployment process.

**Answer:** Code is committed to Git, CI builds and tests the application, and a container image is published to a registry. CD deploys the image through YAML, Helm, or a GitOps controller. Kubernetes creates or updates the Deployment, manages ReplicaSets and Pods, and performs a rolling update. I verify rollout status, service endpoints, application health, logs, and metrics, then roll back or roll forward if required.

### Q2. What is the difference between a Pod and a Deployment?

**Answer:** A Pod runs one or more containers. A Deployment manages the desired replica count and rolling updates for stateless workloads through ReplicaSets.

### Q3. What is the difference between readiness and liveness probes?

**Answer:** Readiness determines whether a Pod should receive traffic. Liveness detects a container that needs restarting. Startup probes give slow-starting applications additional time before liveness checks begin.

### Q4. How do you troubleshoot `CrashLoopBackOff`?

**Answer:** I inspect current and previous logs, describe the Pod, review events, and check the exit code, startup command, configuration, secrets, probes, and resource limits. I fix the root cause and verify the new rollout.

### Q5. What is the difference between ClusterIP, NodePort, and LoadBalancer?

**Answer:** ClusterIP exposes a service internally, NodePort exposes a port on cluster nodes, and LoadBalancer requests a supported load balancer. Ingress or Gateway API can provide HTTP routing and TLS functionality.

### Q6. How do you perform a rolling update?

**Answer:** I update the Deployment's image or Pod template, monitor the rollout, and verify readiness and application metrics. If the release is unhealthy, I follow the rollback runbook after checking compatibility.

### Q7. What is HPA?

**Answer:** The Horizontal Pod Autoscaler adjusts replica counts using configured metrics such as CPU, memory, or custom metrics. It requires an available metrics source and appropriate resource configuration.

### Q8. How do you troubleshoot `Pending` Pods?

**Answer:** I inspect Pod events and scheduling details, then check resource requests, node capacity, affinity rules, taints, tolerations, PVCs, and namespace quotas.

### Q9. How do Pods communicate across namespaces?

**Answer:** Pod-to-Pod connectivity depends on the cluster networking implementation and policies. Services can be addressed using names such as `payment-service.payments.svc.cluster.local`, subject to DNS, routing, and NetworkPolicy rules.

### Q10. How do you manage secrets?

**Answer:** I avoid storing plaintext secrets in Git, use Kubernetes Secrets with encryption and restrictive RBAC, and prefer an approved external secret manager or workload identity integration where appropriate.

### Q11. What is the difference between Helm and Kustomize?

**Answer:** Helm uses charts, templates, and values to package and configure applications. Kustomize composes and patches Kubernetes manifests without requiring a template language. The choice depends on the team's reuse, configuration, and delivery needs.

### Q12. What is GitOps?

**Answer:** GitOps uses Git as the reviewed source of desired configuration. A controller such as Argo CD or Flux continuously reconciles the cluster with that configuration and helps identify configuration drift.

### Q13. How do you handle a production incident?

**Answer:** I assess impact, stabilize the service, follow the incident process, inspect metrics/logs/events, and use a safe rollback or mitigation. After recovery, I validate service health, document the timeline and root cause, and track preventive actions.

### Q14. How do you reduce Kubernetes deployment downtime?

**Answer:** I use appropriate readiness probes, multiple replicas, rolling-update settings, sufficient capacity, disruption controls, and backward-compatible application changes. I validate the rollout with smoke tests and service-level metrics.

### Q15. How do you improve cluster reliability?

**Answer:** I establish SLIs/SLOs, monitor golden signals, define resource requests and limits, use autoscaling appropriately, distribute replicas across failure domains, apply security policies, test recovery procedures, and review incidents for systemic improvements.

---

## 16. Daily DevOps / SRE Checklist

```bash
# 1. Verify cluster and context
kubectl config current-context
kubectl get nodes

# 2. Check workloads
kubectl get deployments,pods,services -n payments

# 3. Check rollout health
kubectl rollout status deployment/payment-app -n payments

# 4. Inspect events
kubectl get events -n payments \
  --sort-by=.metadata.creationTimestamp

# 5. Check resource usage
kubectl top nodes
kubectl top pods -n payments

# 6. Review application logs
kubectl logs deployment/payment-app -n payments --since=30m

# 7. Verify service endpoints
kubectl get endpointslices -n payments
```

Run these checks in accordance with access permissions and production operational procedures. Avoid automated deletion or restart commands without understanding their impact.

---

## 17. Quick Command Reference

| Operation | Command |
|---|---|
| List Pods | `kubectl get pods -n payments` |
| Describe Pod | `kubectl describe pod POD -n payments` |
| View logs | `kubectl logs POD -n payments` |
| Apply YAML | `kubectl apply -f deployment.yaml` |
| Validate YAML locally | `kubectl apply --dry-run=client -f deployment.yaml` |
| Compare desired changes | `kubectl diff -f deployment.yaml` |
| Scale Deployment | `kubectl scale deployment/payment-app --replicas=5 -n payments` |
| Update image | `kubectl set image deployment/payment-app payment-app=IMAGE:TAG -n payments` |
| Check rollout | `kubectl rollout status deployment/payment-app -n payments` |
| Undo rollout | `kubectl rollout undo deployment/payment-app -n payments` |
| View events | `kubectl get events -n payments --sort-by=.metadata.creationTimestamp` |
| Resource usage | `kubectl top pods -n payments` |
| Service forwarding | `kubectl port-forward service/payment-service 8080:80 -n payments` |
| Inspect Helm releases | `helm list -A` |
| Check GKE credentials | `gcloud container clusters get-credentials CLUSTER --region REGION --project PROJECT_ID` |

---

## 18. Official Documentation

- Kubernetes documentation: https://kubernetes.io/docs/
- kubectl reference: https://kubernetes.io/docs/reference/kubectl/
- Deployments: https://kubernetes.io/docs/concepts/workloads/controllers/deployment/
- Services: https://kubernetes.io/docs/concepts/services-networking/service/
- Probes: https://kubernetes.io/docs/concepts/configuration/liveness-readiness-startup-probes/
- HPA: https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/
- Helm documentation: https://helm.sh/docs/
- Google Kubernetes Engine: https://cloud.google.com/kubernetes-engine/docs
- Argo CD: https://argo-cd.readthedocs.io/
