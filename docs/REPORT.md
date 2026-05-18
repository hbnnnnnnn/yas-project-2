# Project Report: YAS CI/CD + Service Mesh Deployment

## 1. Project Overview

**Project name:** YAS — Yet Another Shop  
**Goal:** Build a complete CI/CD pipeline and service mesh for a Java microservice e-commerce application, deployed on Kubernetes using Jenkins, ArgoCD, and Istio.

The YAS application is an e-commerce system built with Java Spring Boot, consisting of 13+ independent microservices (product, cart, order, customer, tax, etc.), backed by PostgreSQL, Kafka, Redis, Keycloak (authentication), and Elasticsearch.

Our job was **not** to write the application code — it already exists at github.com/nashtech-garage/yas. Our job was to build the entire deployment infrastructure around it.

---

## 2. What Was Built

### 2.1 CI/CD Pipeline (Jenkins)

| File | Purpose |
|---|---|
| `Jenkinsfile` | Main CI pipeline. Detects which services changed in a commit, builds Docker images tagged with the commit ID, pushes to Docker Hub (`hbnnn/<service>:<commitid>`) |
| `Jenkinsfile.developer_build` | Lets a developer pick a branch per service and deploy only that service's branch to `dev`, everything else stays on `main` |
| `Jenkinsfile.deploy_dev` | Triggered on every push to `main`. Updates the gitops repo with the new image tag for `dev` |
| `Jenkinsfile.deploy_staging` | Triggered when a git tag like `v1.2.3` is pushed. Builds release image, updates `staging` gitops values |
| `Jenkinsfile.cleanup_dev` | Resets all `dev` image tags back to `main` |

**How it connects:** When a developer pushes code → Jenkins builds a Docker image → Jenkins updates a file in the `yas-gitops` repo → ArgoCD detects the change → ArgoCD deploys to Kubernetes.

### 2.2 Kubernetes Infrastructure

All infrastructure is deployed via Helm charts under `k8s/deploy/`:

| Component | Namespace | Purpose |
|---|---|---|
| PostgreSQL (Zalando operator) | `postgres` | Primary database for all services |
| Redis | `redis` | Caching, session storage |
| Keycloak | `keycloak` | Authentication / OAuth2 / JWT tokens |
| Kafka (Strimzi KRaft) | `kafka` | Event streaming between services |
| Elasticsearch (ECK) | `elasticsearch` | Search index for `search` service |
| ArgoCD | `argocd` | GitOps controller |
| Istio | `istio-system` | Service mesh |

### 2.3 Helm Charts for Services

All 13 services have their own Helm chart under `k8s/charts/<service>/`. Each chart depends on a shared base chart `k8s/charts/backend` (or `k8s/charts/ui` for frontend services), which defines the standard Deployment, Service, and Ingress templates.

### 2.4 GitOps with ArgoCD

We use two repos:
- `yas-project-2` — contains the Helm charts (the "what to deploy")
- `yas-gitops` — contains the values files (the "how to configure each environment")

ArgoCD watches both repos. When Jenkins updates a values file in `yas-gitops`, ArgoCD automatically deploys the new image to Kubernetes.

We use **ApplicationSet** — a single ArgoCD resource that generates one Application per service automatically, instead of creating 13+ individual Applications by hand.

### 2.5 Istio Service Mesh

Istio runs as a sidecar proxy (Envoy) injected into every pod in the `dev` namespace. All traffic between services passes through these proxies, giving us:
- Encrypted mTLS between all services automatically
- Fine-grained access control (AuthorizationPolicy)
- Retry logic without changing application code
- Traffic observability via Kiali

---

## 3. Step-by-Step: What We Did and Why

### Phase 1: Fix Kafka (Strimzi 1.0.0 compatibility)

**Problem:** The original Helm chart used Strimzi v1beta2 API and ZooKeeper-based Kafka. Strimzi 1.0.0 removed ZooKeeper and renamed the API to `v1`.

**What we changed:**
- Rewrote `kafka-cluster.yaml` to use KRaft mode with `KafkaNodePool`
- Updated `debezium-connect-cluster.yaml` to new `KafkaConnect` spec
- Updated all three templates from `apiVersion: kafka.strimzi.io/v1beta2` → `kafka.strimzi.io/v1`
- Changed Kafka version from `3.9.0` to `4.1.0` (only version supported by Strimzi 1.0.0)

**Why:** Without Kafka, the `order` service (and any event-driven services) cannot function.

### Phase 2: Fix ArgoCD ApplicationSet

**Problem:** ArgoCD was showing apps as Synced/Healthy but no pods were actually running, because the gitops repo only had `.values.yaml` files — not actual Kubernetes manifests.

**What we changed:**
- Deleted the old single-app setup
- Created `k8s/argocd/applicationset-dev.yaml` using ArgoCD's **multi-source** feature: one source points to the Helm chart in `yas-project-2`, another source points to the values file in `yas-gitops`
- Did the same for staging (`applicationset-staging.yaml`)

**Why:** This is the correct GitOps pattern. The chart is version-controlled in the app repo; the environment-specific configuration (image tags, replica counts, etc.) is in the gitops repo.

### Phase 3: Fix Helm dependency packaging

**Problem:** Each service chart depends on `k8s/charts/backend` via a local `file://` path. ArgoCD runs `helm template` on a remote server and cannot resolve local file paths.

**What we changed:**
- Ran `helm dependency build` for all 20 service charts
- This produced `charts/backend-0.1.0.tgz` inside each service chart directory
- Committed all these `.tgz` files to the repo

**Why:** ArgoCD needs the dependency packaged inside the chart directory. It cannot do `helm dependency build` at deploy time.

### Phase 4: Fix ServiceMonitor CRD missing

**Problem:** All apps showed `OutOfSync` because the backend chart includes a `ServiceMonitor` resource (Prometheus) by default, but Prometheus Operator was not installed.

**What we changed:**
- Added Helm parameters to the ApplicationSet:
  ```yaml
  - name: backend.serviceMonitor.enabled
    value: "false"
  ```

**Why:** We don't need Prometheus for this project. Disabling it removes the dependency on the CRD.

### Phase 5: Fix ConfigMaps missing in dev namespace

**Problem:** All pods were stuck in `ContainerCreating` because they needed a ConfigMap (`yas-configuration-configmap`) that only existed in the `yas` namespace, not in `dev`.

**What we changed:**
- Added `yas-configuration` to the ApplicationSet so ArgoCD manages it
- This chart creates all ConfigMaps and Secrets needed by the services

**Why:** Each namespace is isolated. ConfigMaps must exist in the same namespace as the pods that use them.

### Phase 6: Fix UI service probe timeouts

**Problem:** `storefront-ui` and `backoffice-ui` were being killed by liveness probes before they could start, because the default probe settings had no `initialDelaySeconds` and Istio sidecar injection added startup overhead.

**What we changed:**
- Made probes configurable in `k8s/charts/ui/templates/deployment.yaml`
- Added generous defaults: `initialDelaySeconds: 30`, `failureThreshold: 12`

**Why:** With Istio sidecars, pods need more time to start. The Envoy proxy must initialize before the app container can receive traffic.

### Phase 7: Trim services to 13 core services

**What we changed:**
- Removed from ApplicationSet: `payment`, `payment-paypal`, `promotion`, `rating`, `recommendation`, `sampledata`, `webhook`, `location`
- Added: `swagger-ui`

**Why:** The removed services were either broken (CrashLoopBackOff) or not needed for the demo. Removing them saves ~4-5 GB of RAM and reduces noise during testing.

### Phase 8: Istio mTLS

**What we created:** `k8s/istio/peer-authentication.yaml`

```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: dev
spec:
  mtls:
    mode: STRICT
```

**Why:** This forces ALL traffic between services in the `dev` namespace to use mutual TLS. No service can communicate using plain HTTP. Each service must present a valid X.509 certificate (issued by Istio's built-in CA) to be accepted.

**Test result:** A pod outside the mesh (no Envoy sidecar) gets `Connection refused` — it cannot even establish a TCP connection because Istio's proxy requires the mTLS handshake.

### Phase 9: AuthorizationPolicy

**What we created:**
- `k8s/istio/authz-deny-all.yaml` — denies all traffic by default
- `k8s/istio/authz-allow-storefront-bff.yaml` — allows only `storefront-bff` service account to call any service
- `k8s/istio/authz-allow-backoffice-bff.yaml` — allows only `backoffice-bff` service account

**Why:** By default, any pod inside the mesh can call any other pod. AuthorizationPolicy restricts this: only known, authorized services may communicate. This implements the principle of least privilege.

**Test result:**
- `curl-test` (using `default` service account): `503` — blocked by policy
- `curl-authorized` (using `storefront-bff` service account): `401` from the tax app — the request reached the service (allowed by policy), but tax requires a JWT token

### Phase 10: Retry Policy


**What we created:** `k8s/istio/virtualservice-tax.yaml`

```yaml
retries:
  attempts: 3
  perTryTimeout: 2s
  retryOn: 5xx,reset,connect-failure
```

**Why:** If the `tax` service returns a 5xx error or is temporarily unavailable, Istio automatically retries up to 3 times without the calling service (e.g., `order`) needing to implement retry logic in its code. This improves resilience.

**Test result:** Envoy proxy stats showed `response_flags.UH: 4` for a single curl call — 1 original attempt + 3 retries = 4 total, confirming retries happened.

### Phase 11: Fix OOMKilled — JVM Heap Cap

**Problem:** After minikube resumed from sleep, seven backend services (product, customer, cart, tax, order, inventory, media) were in `CrashLoopBackOff` with exit code `137` (OOMKilled). The Kubernetes node killed each JVM because it was consuming unbounded heap with no container memory limit set.

**What we changed:**

For all seven services, added to `k8s/charts/<service>/values.yaml`:
```yaml
extraEnvs:
  - name: JAVA_TOOL_OPTIONS
    value: "-Xmx512m -Xms256m"
resources:
  limits:
    memory: 700Mi
  requests:
    memory: 256Mi
    cpu: 100m
```

**Why:** Spring Boot JVM starts small but grows the heap on demand. On a minikube node with limited RAM shared by 13 services + infrastructure, without a heap ceiling the JVM keeps growing until the OOM killer terminates it. `-Xmx512m` caps the heap; the 700Mi container limit gives a buffer above that cap for JVM overhead (metaspace, thread stacks, etc.).

### Phase 12: Fix Search Service — Rebuild Image

**Problem:** The `search` service crashed at startup every time with:
```
Caused by: co.elastic.clients.transport.TransportException:
  status: 400, [es/indices.exists] Expecting a response body, but none was sent
```

**Root cause:** A bug in the `co.elastic.clients` elasticsearch-java library. `SimpleElasticsearchRepository` calls `createIndexAndMappingIfNeeded()` in its constructor, which sends an HTTP HEAD request to check if the index exists. HEAD responses never contain a body (HTTP spec). The library attempted to parse a JSON error body from the HEAD response, found nothing, and threw an exception — crashing the application before it could start. No Spring property can bypass this because the call happens inside the repository constructor, before any application-level configuration is applied.

**What we changed:**

1. Located the entity class in the YAS source code (`DevOps-YAS/search/src/main/java/com/yas/search/model/Product.java`)
2. Added `createIndex = false` to the `@Document` annotation:
   ```java
   // Before
   @Document(indexName = "product")
   
   // After
   @Document(indexName = "product", createIndex = false)
   ```
   This tells `SimpleElasticsearchRepository` to skip the `exists()` + `createIndex()` lifecycle entirely — the repository assumes the index already exists.
3. Rebuilt the JAR with Java 21: `JAVA_HOME=<jdk21> mvn package -pl search -am -DskipTests`
4. Built a Docker image: `docker build -t hbnhbn/search:fixed ./search/`
5. Pushed to Docker Hub: `docker push hbnhbn/search:fixed`
6. Pre-created the `product` index in Elasticsearch manually so the service has something to connect to
7. Updated `k8s/charts/search/values.yaml` to use `hbnhbn/search:fixed`

**Why `createIndex = false` is safe:** We pre-create the index manually in Elasticsearch. The index mapping (field types, analyzers) is defined in `search/src/main/resources/esconfig/elastic-analyzer.json`. By disabling auto-creation in the code, we take ownership of index lifecycle — a common production pattern where index management is handled separately from the application.

---

## 4. Problems Encountered

| Problem | Root Cause | Resolution |
|---|---|---|
| Strimzi CRDs not found | Helm API discovery cache stale | `rm -rf ~/.kube/cache` + `helm template \| kubectl apply` |
| Strimzi API version mismatch | Strimzi 1.0.0 uses `v1` not `v1beta2` | Updated all templates |
| ZooKeeper removed | Strimzi 1.0.0 only supports KRaft | Rewrote Kafka cluster to KRaft mode |
| Kafka version unsupported | Strimzi 1.0.0 only supports Kafka 4.x | Changed version to `4.1.0` |
| ApplicationSet CRD missing | Race condition during ArgoCD install | Re-applied with `--server-side` |
| No pods deployed | GitOps repo had only values files | Switched to multi-source ApplicationSet |
| Helm dependency not packaged | `file://` deps not resolvable by ArgoCD | Pre-packaged all `backend-0.1.0.tgz` |
| ServiceMonitor CRD missing | Prometheus Operator not installed | Disabled `serviceMonitor.enabled` in ApplicationSet |
| ConfigMaps missing in dev | ConfigMaps only existed in `yas` namespace | Added `yas-configuration` to ApplicationSet |
| UI probes failing | No `initialDelaySeconds` + Istio overhead | Made probes configurable, added generous defaults |
| DestinationRule conflicting | ISTIO_MUTUAL conflicts with auto-mTLS in Istio 1.5+ | Deleted DestinationRule, relying on auto-mTLS |
| minikube NodePort unreachable | Docker driver doesn't expose NodePorts | Used `kubectl port-forward` instead |
| Backend services OOMKilled | JVM heap unbounded on memory-constrained minikube node | Added `JAVA_TOOL_OPTIONS=-Xmx512m -Xms256m` and 700Mi limit to all backend services |
| search CrashLoopBackOff | Bug in `co.elastic.clients` elasticsearch-java client — tries to read a body from a HEAD response (which HTTP forbids). Crash happens in `SimpleElasticsearchRepository.createIndexAndMappingIfNeeded()` before application starts | Rebuilt search image from source with `@Document(indexName = "product", createIndex = false)` to skip the buggy `indices.exists()` call. Pre-created `product` index in Elasticsearch. Published as `hbnhbn/search:fixed` |

---

## 5. Current State

| Component | Status |
|---|---|
| Jenkins CI pipelines | Done |
| Kubernetes cluster (Minikube) | Running |
| PostgreSQL, Redis, Keycloak, Kafka, Elasticsearch | Running |
| ArgoCD ApplicationSet (dev) — 13 services | Synced |
| ArgoCD ApplicationSet (staging) — 13 services | Synced |
| Istio (istiod + Kiali + Prometheus) | Running |
| Istio sidecar injection in dev | Active (all pods 2/2) |
| mTLS STRICT | Applied |
| AuthorizationPolicy (deny-all + allow BFFs) | Applied |
| VirtualService retry on tax | Applied |
| All 13 services | Running (2/2 with Istio sidecar) |

---

## 6. Architecture Diagram

```
Developer pushes code
        │
        ▼
   GitHub (yas-project-2)
        │
        ▼ webhook
   Jenkins CI
   - Detects changed services
   - Builds Docker images
   - Pushes to Docker Hub (hbnnn/<service>:<commitid>)
   - Updates yas-gitops repo (yq sets image tag)
        │
        ▼
   GitHub (yas-gitops)
   dev/<service>.values.yaml  ← ArgoCD watches this
        │
        ▼
   ArgoCD ApplicationSet
   - Generates 1 Application per service
   - Multi-source: chart from yas-project-2 + values from yas-gitops
   - Auto-syncs, auto-prunes
        │
        ▼
   Kubernetes (Minikube)
   ├── namespace: dev
   │   ├── 13 services (all pods 2/2 with Istio sidecar)
   │   └── Istio policies (mTLS, AuthorizationPolicy, VirtualService)
   ├── namespace: staging
   │   └── 13 services
   ├── namespace: postgres / kafka / keycloak / elasticsearch / redis
   └── namespace: istio-system (istiod, Kiali, Prometheus)
```
