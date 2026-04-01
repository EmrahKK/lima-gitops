# ArgoCD + Vault Secret Management via argocd-vault-plugin

## Overview

This document describes the complete setup for integrating HashiCorp Vault with ArgoCD using [argocd-vault-plugin (AVP)](https://github.com/argoproj-labs/argocd-vault-plugin). Instead of storing secrets in git, manifests contain `<path:...>` placeholders that AVP replaces with real values from Vault at sync time.

**Environment:**
- ArgoCD: v3.3.6 (Helm chart v9.4.17, release name `argocd`, namespace `argocd`)
- ArgoCD deployed via: `helm install` from `argo-helm-main/charts/argo-cd/`
- Vault: standalone, HTTP on port 8200, namespace `vault`
- Cluster node architecture: **ARM64**

---

## Architecture

```
ArgoCD repo-server
  ├── sidecar: avp            (plain YAML + placeholders)
  └── sidecar: avp-kustomize  (kustomize + placeholders)
          |
          | Kubernetes auth
          v
       Vault (KV v2: secret/*)
```

AVP runs as sidecar containers in the `argocd-repo-server` pod. It authenticates to Vault using the repo-server's Kubernetes service account token (no static credentials in git), and replaces `<path:secret/data/app#key>` placeholders at sync time.

---

## Phase 1: Vault Setup (one-time, manual)

### 1.1 Initialize and Unseal Vault

```bash
kubectl exec -n vault vault-0 -- vault operator init -key-shares=1 -key-threshold=1
# Save the Unseal Key and Root Token from the output

kubectl exec -n vault vault-0 -- vault operator unseal <UNSEAL_KEY>
```

### 1.2 Configure Vault

```bash
kubectl exec -it -n vault vault-0 -- sh
```

Inside the pod:

```bash
vault login <ROOT_TOKEN>

# Enable KV v2 secrets engine
vault secrets enable -path=secret kv-v2

# Enable Kubernetes auth
vault auth enable kubernetes

# Configure Kubernetes auth (uses the pod's own SA for cluster access)
vault write auth/kubernetes/config \
  kubernetes_host="https://kubernetes.default.svc" \
  kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt

# Create policy granting ArgoCD read access to all secrets
vault policy write argocd - <<'EOF'
path "secret/data/*" {
  capabilities = ["read"]
}
EOF

# Bind the argocd-repo-server service account to the policy
vault write auth/kubernetes/role/argocd \
  bound_service_accounts=argocd-repo-server \
  bound_service_accounts_namespaces=argocd \
  policies=argocd \
  ttl=1h
```

### 1.3 Store Secrets in Vault

```bash
vault kv put secret/myapp \
  admin-password="your-password" \
  admin-user="admin"
```

---

## Phase 2: ArgoCD Configuration

File: `argo-helm-main/charts/argo-cd/values.yaml`

### 2.1 CMP Plugin Definitions (`configs.cmp.plugins`)

Two plugins are defined — one for plain YAML and one for kustomize directories:

```yaml
configs:
  cmp:
    create: true
    plugins:
      # For plain YAML directories with AVP placeholders
      avp:
        allowConcurrency: true
        generate:
          command: [/home/argocd/custom-tools/argocd-vault-plugin, generate, "."]
        lockRepo: false

      # For kustomize directories with AVP placeholders
      avp-kustomize:
        allowConcurrency: true
        generate:
          command: [sh, -c]
          args: ["kustomize build . | /home/argocd/custom-tools/argocd-vault-plugin generate -"]
        lockRepo: false
```

> **No `discover` block.** Without auto-discovery, plugins are only used when an Application explicitly declares `plugin: name: avp` or `plugin: name: avp-kustomize`. This prevents AVP from claiming applications that don't use Vault secrets.

### 2.2 Init Container (download AVP binary)

Under `repoServer.initContainers`:

```yaml
initContainers:
  - name: download-avp
    image: alpine:3.19
    command: [sh, -c]
    args:
      - |
        wget -qO /custom-tools/argocd-vault-plugin \
          https://github.com/argoproj-labs/argocd-vault-plugin/releases/download/v1.18.1/argocd-vault-plugin_1.18.1_linux_arm64
        chmod +x /custom-tools/argocd-vault-plugin
    volumeMounts:
      - mountPath: /custom-tools
        name: custom-tools
```

> **Architecture note:** This cluster runs ARM64 nodes, so the `linux_arm64` binary is required. For AMD64 clusters use `linux_amd64`.

### 2.3 Sidecar Containers

Each plugin requires its own `argocd-cmp-server` sidecar. Under `repoServer.extraContainers`:

```yaml
extraContainers:
  - name: avp
    command: [/var/run/argocd/argocd-cmp-server]
    image: quay.io/argoproj/argocd:v3.3.6
    securityContext:
      runAsNonRoot: true
      runAsUser: 999
    env:
      - name: VAULT_ADDR
        value: http://vault.vault.svc.cluster.local:8200
      - name: AVP_TYPE
        value: vault
      - name: AVP_AUTH_TYPE
        value: k8s
      - name: AVP_K8S_ROLE
        value: argocd
    volumeMounts:
      - mountPath: /var/run/argocd
        name: var-files
      - mountPath: /home/argocd/cmp-server/plugins
        name: plugins
      - mountPath: /tmp
        name: cmp-tmp
      - mountPath: /home/argocd/custom-tools
        name: custom-tools
      - mountPath: /home/argocd/cmp-server/config/plugin.yaml
        subPath: avp.yaml
        name: argocd-cmp-cm

  - name: avp-kustomize
    command: [/var/run/argocd/argocd-cmp-server]
    image: quay.io/argoproj/argocd:v3.3.6
    securityContext:
      runAsNonRoot: true
      runAsUser: 999
    env:
      - name: VAULT_ADDR
        value: http://vault.vault.svc.cluster.local:8200
      - name: AVP_TYPE
        value: vault
      - name: AVP_AUTH_TYPE
        value: k8s
      - name: AVP_K8S_ROLE
        value: argocd
    volumeMounts:
      - mountPath: /var/run/argocd
        name: var-files
      - mountPath: /home/argocd/cmp-server/plugins
        name: plugins
      - mountPath: /tmp
        name: cmp-tmp
      - mountPath: /home/argocd/custom-tools
        name: custom-tools
      - mountPath: /home/argocd/cmp-server/config/plugin.yaml
        subPath: avp-kustomize.yaml
        name: argocd-cmp-cm
```

### 2.4 Volumes

Under `repoServer.volumes`:

```yaml
volumes:
  - name: custom-tools
    emptyDir: {}
  - name: cmp-tmp
    emptyDir: {}
  - name: argocd-cmp-cm
    configMap:
      name: argocd-cmp-cm
```

> **Mount strategy:** The AVP binary is mounted as a **directory** (`/home/argocd/custom-tools`) rather than a single file with `subPath`. Mounting a single file with `subPath` into `/usr/local/bin/` causes `ENOEXEC` when `argocd-cmp-server` tries to exec it (even though the binary itself is valid), due to how Go's `os.StartProcess` interacts with file bind mounts. Directory mounts do not have this issue.

### 2.5 Apply Changes

```bash
helm upgrade argocd argo-helm-main/charts/argo-cd -n argocd
kubectl rollout status deployment/argocd-repo-server -n argocd
```

Verify both sidecars are running:

```bash
kubectl get pods -n argocd -l app.kubernetes.io/name=argocd-repo-server
kubectl logs -n argocd deploy/argocd-repo-server -c avp
kubectl logs -n argocd deploy/argocd-repo-server -c avp-kustomize
```

---

## Phase 3: Writing Manifests with AVP Placeholders

### Placeholder syntax

```
<path:secret/data/<secret-name>#<key>>
```

- `secret/data/` — required prefix for KV v2
- `<secret-name>` — the path you used in `vault kv put`
- `<key>` — the field name within that secret

### Use `stringData`, not `data`

Secrets must use `stringData` (plain strings). Using `data` causes Kubernetes to validate the placeholder as base64 before AVP can substitute it, resulting in `illegal base64 data` errors.

```yaml
# Correct
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
stringData:
  admin-password: <path:secret/data/myapp#admin-password>
  admin-user: <path:secret/data/myapp#admin-user>
type: Opaque
```

---

## Phase 4: Configuring ArgoCD Applications

### Choosing the right plugin

| Your source directory contains | Use plugin |
|-------------------------------|-----------|
| Plain YAML files only | `avp` |
| `kustomization.yaml` + YAML files | `avp-kustomize` |

### Application manifest example

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/EmrahKK/lima-gitops.git
    path: registry/lima-local/components/vault
    targetRevision: HEAD
    plugin:
      name: avp-kustomize   # or "avp" for plain YAML
  destination:
    server: https://kubernetes.default.svc
    namespace: vault
```

### App-of-apps directories

If an ArgoCD Application manages a directory that contains other Application manifests (app-of-apps pattern), that parent directory needs a `kustomization.yaml` listing only the Application YAML files. This prevents ArgoCD from picking up subdirectory `kustomization.yaml` files and trying to apply them as Kubernetes resources.

```
registry/lima-local/
├── kustomization.yaml     ← lists vault.yaml only
└── vault.yaml             ← ArgoCD Application manifest
```

```yaml
# registry/lima-local/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - vault.yaml
```

---

## Repository Structure

```
registry/
└── lima-local/
    ├── kustomization.yaml                    # app-of-apps root
    ├── vault.yaml                            # ArgoCD Application for vault-components
    └── components/
        └── vault/
            ├── kustomization.yaml            # lists vault.yaml + tmp.yaml
            ├── vault.yaml                    # Vault StatefulSet, Service, etc.
            └── tmp.yaml                      # Secret with AVP placeholders
```

---

## Troubleshooting

### `exec format error`

The AVP binary architecture does not match the node architecture. Check your nodes:
```bash
kubectl get nodes -o jsonpath='{.items[*].status.nodeInfo.architecture}'
```
Use `linux_arm64` for ARM64 nodes, `linux_amd64` for x86 nodes.

### `illegal base64 data at input byte N`

Secret manifest uses `data:` instead of `stringData:`. Change to `stringData:`.

### `kustomize.config.k8s.io/Kustomization CRD not found`

Two possible causes:

1. **AVP plugin processes a directory with `kustomization.yaml`** — use `avp-kustomize` instead of `avp`. The `avp` plugin outputs all YAML files including `kustomization.yaml` as raw resources; `avp-kustomize` runs kustomize first so `kustomization.yaml` is treated as config.

2. **App-of-apps directory has no root `kustomization.yaml`** — ArgoCD falls back to plain-YAML mode and recursively picks up all `.yaml` files, including those in subdirectories. Add a `kustomization.yaml` at the root that lists only the Application manifests.

### `Manifest generation error (cached)`

ArgoCD caches manifest generation errors. After fixing the root cause, force a re-evaluation:
```bash
kubectl annotate -n argocd application <app-name> argocd.argoproj.io/refresh=hard --overwrite
```

### AVP plugin not claimed / `plugin not found`

If no `discover` block is set, the plugin is only used when explicitly declared in the Application. Verify the Application spec has:
```yaml
source:
  plugin:
    name: avp   # or avp-kustomize
```
