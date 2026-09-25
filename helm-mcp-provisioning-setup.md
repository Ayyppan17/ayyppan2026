# Helm Provisioning Agent — Build & Test Runbook

Covers: IKS cluster → Helm/Kubernetes MCP server → GitHub chart repo → `catalog.yaml` → agent system prompt → ServiceNow RITM flow → end-to-end test.

---

## 0. Architecture at a glance

```
ServiceNow (RITM: "New Service Deployment")
        │
        ▼
Helm Provisioning Agent  (your system prompt)
        │  MCP tool calls
        ▼
Kubernetes / Helm MCP Server  (namespaces_list, resources_create_or_update,
                                helm_install, helm_upgrade*, helm_list, ...)
        │
        ▼
IBM Cloud Kubernetes Service (IKS) cluster
        │
        ▼
GitHub repo (charts/*.tgz + catalog.yaml)  ← chart source of truth
```

The agent never talks to `kubectl`/`helm` CLI directly — every action goes through the MCP server's tool calls, and the server is the only thing with cluster credentials.

---

## 1. Prerequisites

- IBM Cloud account with access to an IKS cluster (or permission to create one)
- `ibmcloud` CLI + `kubectl` + `helm` (v3) installed locally
- A GitHub repo to host packaged charts (`Ayyppan17/boarepo` in your screenshot)
- An MCP-capable agent runtime (Claude, or whatever hosts the ServiceNow ↔ MCP bridge)
- Cluster-admin (or namespace-admin) rights to create the ServiceAccount/RBAC the MCP server will run as

---

## 2. Provision / connect the IKS cluster

```bash
# Log in and target the account
ibmcloud login --sso
ibmcloud ks cluster ls

# Point kubectl at the target cluster
ibmcloud ks cluster config --cluster <CLUSTER_ID_OR_NAME>
kubectl get nodes
```

If you're creating a new cluster rather than reusing one:

```bash
ibmcloud ks cluster create classic \
  --name helm-agent-iks \
  --zone <zone> \
  --flavor <flavor> \
  --workers 3
```

Confirm `helm version` and `kubectl version` both succeed against the cluster before continuing.

---

## 3. Deploy the Helm/Kubernetes MCP server into IKS

This is the piece that exposes `namespaces_list`, `resources_create_or_update`, `helm_install`, `helm_list`, etc. as MCP tools.

### 3.1 Create a dedicated namespace and ServiceAccount

```bash
kubectl create namespace mcp-system
```

```yaml
# rbac.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: helm-mcp-server
  namespace: mcp-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: helm-mcp-server
rules:
  - apiGroups: [""]
    resources: ["namespaces", "services", "configmaps", "secrets", "pods", "events"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
  - apiGroups: ["apps"]
    resources: ["deployments", "statefulsets", "daemonsets"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
  - apiGroups: ["networking.k8s.io"]
    resources: ["ingresses"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
  - apiGroups: [""]
    resources: ["namespaces"]
    verbs: ["create"]   # needed for the agent's namespace auto-provisioning step
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: helm-mcp-server
subjects:
  - kind: ServiceAccount
    name: helm-mcp-server
    namespace: mcp-system
roleRef:
  kind: ClusterRole
  name: helm-mcp-server
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl apply -f rbac.yaml
```

Scope this down to specific namespaces with `Role`/`RoleBinding` instead of `ClusterRole` once you know the fixed set of target namespaces (`database`, `velero`, `ingress-nginx`, `kubeai-system`, plus dynamically created app namespaces) — the system prompt's "never touch anything except via Helm, except Namespace" rule maps well to RBAC scoped to `Namespace`, `Deployment`, `Service`, etc.

### 3.2 Run the MCP server in-cluster

Deploy it as a `Deployment` behind a `Service`, using the ServiceAccount above, and enable the `core` + `helm` toolsets (this is the toolset combination that matches your system prompt's tool names):

```yaml
# helm-mcp-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: helm-mcp-server
  namespace: mcp-system
spec:
  replicas: 1
  selector:
    matchLabels: { app: helm-mcp-server }
  template:
    metadata:
      labels: { app: helm-mcp-server }
    spec:
      serviceAccountName: helm-mcp-server
      containers:
        - name: helm-mcp-server
          image: <your-mcp-server-image>:<tag>
          args:
            - "--port=8080"
            - "--toolsets=core,helm"
            - "--read-only=false"
          ports:
            - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: helm-mcp-server
  namespace: mcp-system
spec:
  selector: { app: helm-mcp-server }
  ports:
    - port: 80
      targetPort: 8080
```

```bash
kubectl apply -f helm-mcp-deployment.yaml
kubectl -n mcp-system rollout status deploy/helm-mcp-server
```

> Confirm exactly which tools your chosen MCP server image ships (`helm_install`/`helm_list`/`helm_uninstall` at minimum; `helm_upgrade`, `helm_upgrade_install`, `helm_rollback` if your build supports them — some Helm-focused MCP servers add these, the pure Kubernetes-core one does not). Run its `--help` / tool-list output once deployed and reconcile against the "DEPLOYMENT EXECUTION" section of your system prompt.

### 3.3 Expose it to the agent runtime

- If the agent (Claude / your orchestration layer) runs outside the cluster: expose the MCP `Service` via an `Ingress`/`Route` with TLS and an auth token, then register that URL as an MCP server in the agent's config.
- If it runs alongside the agent (e.g. same pod/sidecar, or invoked over stdio): wire it per your agent framework's MCP-server registration mechanism instead of a network Service.

Test connectivity with the MCP inspector before wiring up the agent:

```bash
npx @modelcontextprotocol/inspector@latest http://<mcp-service-url>
```

Confirm `namespaces_list`, `resources_create_or_update`, `helm_install`, and `helm_list` all show up and respond.

---

## 4. Build the GitHub chart repo

Matches the structure in your screenshot: `docs/charts/*.tgz` (packaged charts) + `docs/catalog.yaml` (the pointer file), plus per-chart source directories (`docs/mysql-service/`).

### 4.1 Repo layout

```
boarepo/
└── docs/
    ├── catalog.yaml
    ├── charts/
    │   ├── mysql-service-0.1.0.tgz
    │   ├── velero-8.0.0.tgz
    │   ├── ingress-nginx-4.15.1.tgz
    │   └── kubeai-stack-0.1.0.tgz
    └── mysql-service/              # chart *source* for mysql (others similar)
        ├── Chart.yaml
        ├── Chart.lock
        ├── charts/                 # subchart dependencies, if any
        └── values-ibm-iks.yaml
```

### 4.2 Author or vendor each chart

- `mysql`: your own chart wrapping the `bitnamilegacy/mysql` image (source lives in `docs/mysql-service/`).
- `velero`, `ingress-nginx`, `kubeai`: pull the upstream chart at the pinned version (e.g. `helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx && helm pull ingress-nginx/ingress-nginx --version 4.15.1`), or vendor/fork them if you need IKS-specific tweaks (as `values-ibm-iks.yaml` suggests you're doing for mysql).

### 4.3 Package and publish charts

```bash
cd docs/mysql-service
helm dependency update            # if it has subcharts
helm package . -d ../charts --version 0.1.0
```

Repeat per chart, dropping the resulting `.tgz` into `docs/charts/`. Commit and push:

```bash
git add docs/charts/*.tgz docs/mysql-service
git commit -m "Package mysql-service 0.1.0"
git push origin main
```

Because the agent fetches charts as a **raw URL string only** (per the "CHART URL RULE" in your system prompt), always reference them as:

```
https://raw.githubusercontent.com/<org>/<repo>/<branch>/docs/charts/<chart-file>.tgz
```

No HTML links, no markdown, no query params — just that string, exactly matching what's in `catalog.yaml`.

### 4.4 `catalog.yaml`

This is the file your screenshot shows — it's the single source of truth the agent validates every deployment request against:

```yaml
services:

  mysql:
    chart: https://raw.githubusercontent.com/Ayyppan17/boarepo/main/docs/charts/mysql-service-0.1.0.tgz
    namespace: database
    release: orderhub-mysql

    overrides:
      mysql:
        image:
          repository: bitnamilegacy/mysql

  velero:
    chart: https://raw.githubusercontent.com/Ayyppan17/boarepo/main/docs/charts/velero-8.0.0.tgz
    namespace: velero
    release: velero

  ingress-nginx:
    chart: https://raw.githubusercontent.com/Ayyppan17/boarepo/main/docs/charts/ingress-nginx-4.15.1.tgz
    namespace: ingress-nginx
    release: ingress-nginx

  kubeai:
    chart: https://raw.githubusercontent.com/Ayyppan17/boarepo/main/docs/charts/kubeai-stack-0.1.0.tgz
    namespace: kubeai-system
    release: kubeai
```

To add a new approved service later: package its chart, push the `.tgz`, add a block here with `chart` / `namespace` / `release` (+ `overrides` if required), and list it in the system prompt's "approved services" set. The agent is explicitly gated to refuse anything not in both places.

### 4.5 Where the agent reads `catalog.yaml` from

Decide (and document) one of:
- The agent fetches `catalog.yaml` live from the raw GitHub URL on each run, or
- It's mounted/synced into the agent's runtime environment.

Either way, pin it to a branch/tag rather than always reading `main` in prod, so a mid-flight chart change can't silently alter an in-progress deployment.

---

## 5. Wire up the agent (system prompt)

Your existing system prompt already encodes the full contract — ingest fields, validation, namespace auto-provisioning, the "Helm only, never raw resources except Namespace" rule, the deployment gate, and success/failure work-note formats. To stand this up from scratch:

1. Load the system prompt into whatever hosts the agent (Claude via API/Bedrock/Vertex, or an orchestration platform).
2. Grant the agent exactly two tool surfaces:
   - The **ServiceNow** connector/API (read RITM fields, post work notes, set state) — scoped to the `New Service Deployment` catalog item.
   - The **Helm/Kubernetes MCP server** from Section 3.
3. Give the agent read access to `catalog.yaml` (Section 4.5).
4. Double-check the hard constraints are enforced by tool permissions too, not just prompt text — e.g. if your MCP server allows `resources_create_or_update` on `Deployment`, the prompt's "never use it for Deployment" rule is only a soft guardrail. Where possible, restrict the ServiceAccount's RBAC (Section 3.1) so non-Namespace `resources_create_or_update` calls fail at the API level as a backstop.

---

## 6. ServiceNow catalog item

Create (or confirm) a **"New Service Deployment"** catalog item whose RITM variables map 1:1 to the agent's required ingest fields:

| Catalog variable | RITM field | Notes |
|---|---|---|
| Service Name | `service_name` | must match a key in `catalog.yaml` |
| Namespace | `namespace` | target namespace (may differ from catalog default) |
| Environment | `environment` | dropdown: `dev` / `test` / `prod` |
| Replica Count | `replica_count` | integer |
| Image | `image` | only relevant where the chart supports an image override |
| Resource Limits | `resource_limits` | CPU/memory |
| Ingress Host | `ingress_host` | |
| Requestor | `requestor` | auto-filled from caller |
| Cost Center | `cost_center` | |

Route this catalog item's fulfillment flow to trigger the agent (webhook, scheduled job, or Flow Designer action that calls your agent's endpoint with the RITM sys_id).

---

## 7. End-to-end test plan

Run this in a **test** environment/namespace first.

1. **Submit a RITM** against "New Service Deployment" requesting `ingress-nginx`, `environment: test`, valid values for all required fields.
2. **Confirm ingestion** — agent posts the "request picked up" work note.
3. **Break validation on purpose** once (e.g. omit `replica_count` or set `environment: staging`) → confirm the agent posts an issues list, sets state to *Pending Info*, and stops (no MCP calls made).
4. **Resubmit valid values** → confirm:
   - `namespaces_list` is called before any create.
   - If the namespace doesn't exist, `resources_create_or_update` is called **only** with a `Namespace` manifest, and a work note announces the auto-provision.
5. **Pre-deploy checks** — confirm the agent verifies service exists in `catalog.yaml`, chart URL matches exactly, release name is set, and (for `mysql`) the `mysql.image.repository=bitnamilegacy/mysql` override is present before calling `helm_install`.
6. **Deployment** — since `ingress-nginx` has no prior release in the namespace, expect `helm_install`, not `helm_upgrade`.
7. **Verify in-cluster**, matching the "Configuration Overview" doc pattern you already produced:
   ```bash
   helm list -n ingress-nginx
   kubectl -n ingress-nginx get deploy,svc
   kubectl -n ingress-nginx get svc ingress-nginx-controller -o wide   # confirm external IP/hostname assigned
   ```
8. **Success work note** — confirm it contains service name, release name, namespace, chart URL, action (`install`), and status `deployed`.
9. **Re-run the same RITM** (or a new one for the same service/namespace) → confirm the second pass calls `helm_upgrade` (or `helm_upgrade_install` only if that behavior was explicitly requested), not another `helm_install`.
10. **Negative test** — request a service *not* in `catalog.yaml` → confirm the agent stops with a work note and performs zero deployment calls.
11. **Chart URL tamper test** — manually edit `catalog.yaml` so the URL for a service doesn't match what's requested → confirm the pre-deploy check catches the mismatch and stops.

Once test passes cleanly, repeat the same RITM flow with `environment: prod` pointed at a production-scoped ServiceAccount/namespace set before treating this as production-ready.

---

## 8. Ongoing maintenance checklist

- New chart version → package, push `.tgz`, bump `chart:` URL and any version-pinned fields in `catalog.yaml` in the same PR.
- Rotate/limit the MCP server's ServiceAccount token; avoid cluster-admin.
- Review RBAC periodically against the actual `apiVersion`/`kind` set your charts render (Helm-managed resources shouldn't need the agent's own credentials to touch anything Helm doesn't already manage).
- Keep `catalog.yaml` and the system prompt's "approved services" list in lockstep — treat a mismatch as a bug.
