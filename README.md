# k8s-gitops

GitOps repository for Kubernetes cluster `k8s-ws-terra-v2` managed by **Flux CD**. This repo provisions Istio service mesh, cert-manager for TLS, and deploys the LMS application — all through git commits.

---

## Prerequisites

- A running Kubernetes cluster (AKS — provisioned via Terraform)
- `kubectl` configured to access the cluster
- `flux` CLI installed
- A GitHub Personal Access Token (PAT) with repo permissions

---

## Repository Structure

```
clusters/k8s-ws-terra-v2/
├── flux-system/                    # Flux bootstrap (auto-generated + custom)
│   ├── gotk-components.yaml
│   ├── gotk-sync.yaml
│   ├── kustomization.yaml
│   └── helmrepositories/
│       ├── cert-manager.yaml       # HelmRepository — Jetstack charts
│       └── sealed-secrets.yaml     # HelmRepository — Bitnami Sealed Secrets
├── istio-system/                   # Istio service mesh + TLS
│   ├── namespace.yaml
│   ├── helm-istio-repository.yaml  # HelmRepository — Istio charts
│   ├── helm-release-istio-base.yaml
│   ├── helm-release-istiod.yaml
│   ├── helm-release-istio-gateway.yaml
│   ├── prod-cluster-issuer.yaml    # Let's Encrypt ClusterIssuer
│   ├── certificate-kubeflex.yaml   # TLS Certificate
│   └── gateway-kubeflex.yaml       # Istio Gateway (HTTP + HTTPS)
├── cert-manager/                   # cert-manager installation
│   ├── namespace.yaml
│   └── cert-manager.yaml           # HelmRelease — cert-manager
└── lms/                            # LMS application
    ├── namespace.yaml
    ├── deployment.yaml
    ├── service.yaml
    └── virtual-service.yaml        # Istio VirtualService routing
```

---

## Step-by-Step Setup

### Step 0 — Bootstrap Flux CD

This initializes Flux on the cluster and connects it to this GitHub repo.

```bash
flux bootstrap github \
  --owner=tharinduk001 \
  --repository=k8s-gitops \
  --branch=main \
  --path=./clusters/k8s-ws-terra-v2 \
  --personal
```

This auto-generates `flux-system/gotk-components.yaml`, `gotk-sync.yaml`, and `kustomization.yaml`. From this point on, **every file committed under `clusters/k8s-ws-terra-v2/` is automatically applied to the cluster**.

---

### Step 1 — Install Istio Service Mesh

Create the following 5 files under `clusters/k8s-ws-terra-v2/istio-system/`:

| File | Kind | Purpose |
|------|------|---------|
| `namespace.yaml` | Namespace | Creates `istio-system` namespace |
| `helm-istio-repository.yaml` | HelmRepository | Registers Istio chart repo (`https://istio-release.storage.googleapis.com/charts`) |
| `helm-release-istio-base.yaml` | HelmRelease | Installs Istio CRDs (`base` chart) |
| `helm-release-istiod.yaml` | HelmRelease | Installs Istiod control plane (depends on `istio-base`) |
| `helm-release-istio-gateway.yaml` | HelmRelease | Installs Istio Ingress Gateway as LoadBalancer (depends on `istio-base` + `istiod`) |

```bash
git add clusters/k8s-ws-terra-v2/istio-system/
git commit -m "Add Istio base, istiod, and ingress gateway"
git push
```

**Verify:**

```bash
flux reconcile kustomization flux-system --with-source
kubectl get helmreleases -n istio-system        # All 3 should be True/Ready
kubectl get svc -n istio-system                 # Wait for EXTERNAL-IP on istio-ingressgateway
```

---

### Step 2 — Deploy the LMS Application

Create the following 3 files under `clusters/k8s-ws-terra-v2/lms/`:

| File | Kind | Purpose |
|------|------|---------|
| `namespace.yaml` | Namespace | Creates `lms` namespace with label `istio-injection: enabled` |
| `deployment.yaml` | Deployment | Deploys `kalharacodes/lms-static:latest` |
| `service.yaml` | Service | ClusterIP service on port 80 |

```bash
git add clusters/k8s-ws-terra-v2/lms/
git commit -m "Add LMS app deployment and service"
git push
```

**Rollout restart to inject the Istio sidecar** (pods were created before the injection label took effect):

```bash
kubectl rollout restart deployment lms-static -n lms
```

**Verify:**

```bash
kubectl get pods -n lms     # Should show 2/2 containers (app + istio-proxy sidecar)
```

---

### Step 3 — Add Helm Repositories for cert-manager and Sealed Secrets

Create 2 files under `clusters/k8s-ws-terra-v2/flux-system/helmrepositories/`:

| File | Kind | Purpose |
|------|------|---------|
| `cert-manager.yaml` | HelmRepository | Registers `https://charts.jetstack.io` |
| `sealed-secrets.yaml` | HelmRepository | Registers `https://bitnami-labs.github.io/sealed-secrets` |

**Also edit** `flux-system/kustomization.yaml` to include the new files:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - gotk-components.yaml
  - gotk-sync.yaml
  - helmrepositories/cert-manager.yaml       # add this
  - helmrepositories/sealed-secrets.yaml      # add this
```

```bash
git add clusters/k8s-ws-terra-v2/flux-system/
git commit -m "Add cert-manager and sealed-secrets HelmRepositories"
git push
```

---

### Step 4 — Install cert-manager

Create the following 2 files under `clusters/k8s-ws-terra-v2/cert-manager/`:

| File | Kind | Purpose |
|------|------|---------|
| `namespace.yaml` | Namespace | Creates `cert-manager` namespace |
| `cert-manager.yaml` | HelmRelease | Installs cert-manager v1.14.4 with `installCRDs: true` |

> **Note:** The HelmRelease references the HelmRepository created in Step 3 by name (`cert-manager` in `flux-system` namespace).

```bash
git add clusters/k8s-ws-terra-v2/cert-manager/
git commit -m "Add cert-manager HelmRelease"
git push
```

**Verify:**

```bash
flux reconcile kustomization flux-system --with-source
kubectl get pods -n cert-manager    # cert-manager, webhook, and cainjector should be running
```

---

### Step 5 — Add TLS with Let's Encrypt (ClusterIssuer + Certificate + Gateway)

Create the following 3 files under `clusters/k8s-ws-terra-v2/istio-system/`:

| File | Kind | Purpose |
|------|------|---------|
| `prod-cluster-issuer.yaml` | ClusterIssuer | Let's Encrypt prod issuer with HTTP-01 solver via Istio ingress class |
| `certificate-kubeflex.yaml` | Certificate | Requests TLS cert for `workshop.tharindukalhara.me`, stores in secret `kubeflex-tls` |
| `gateway-kubeflex.yaml` | Gateway | Istio Gateway — HTTPS on port 443 using `kubeflex-tls`, HTTP on port 80 |

```bash
git add clusters/k8s-ws-terra-v2/istio-system/
git commit -m "Add ClusterIssuer, Certificate, and Gateway for TLS"
git push
```

**Verify:**

```bash
flux reconcile kustomization flux-system --with-source
kubectl get clusterissuer                       # letsencrypt-prod-cluster should be Ready
kubectl get certificate -n istio-system         # kubeflex should be True/Ready
kubectl get gateway -n istio-system             # gateway-kubeflex should exist
```

---

### Step 6 — Route Traffic to LMS via VirtualService

Create the final file under `clusters/k8s-ws-terra-v2/lms/`:

| File | Kind | Purpose |
|------|------|---------|
| `virtual-service.yaml` | VirtualService | Routes `workshop.tharindukalhara.me` through `istio-system/gateway-kubeflex` to `lms-static-svc` on port 80 |

```bash
git add clusters/k8s-ws-terra-v2/lms/virtual-service.yaml
git commit -m "Add VirtualService to route traffic to LMS via gateway"
git push
```

**Verify:**

```bash
kubectl get virtualservice -n lms
curl -I http://workshop.tharindukalhara.me
```

---

## Dependency Chain

```
Flux Bootstrap
 ├── Step 1: Istio (namespace → helm repo → istio-base → istiod → ingressgateway)
 ├── Step 2: LMS App (namespace + sidecar injection → deployment → service → rollout restart)
 ├── Step 3: HelmRepositories (cert-manager + sealed-secrets → added to kustomization.yaml)
 ├── Step 4: cert-manager (namespace → HelmRelease)
 ├── Step 5: TLS (ClusterIssuer → Certificate → Gateway)
 └── Step 6: VirtualService (routes domain → LMS through Gateway)
```

**Key ordering:** cert-manager (Step 4) must be running before creating ClusterIssuer/Certificate (Step 5), and Istio ingress gateway (Step 1) must exist before the Gateway resource (Step 5) can bind to it.