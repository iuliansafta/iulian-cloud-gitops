# iulian.cloud GitOps

GitOps repository for managing Kubernetes infrastructure using ArgoCD and the App of Apps pattern.

## Overview

This repository manages multiple Kubernetes clusters using a declarative GitOps approach. Applications are automatically deployed and synchronized by ArgoCD when changes are pushed to the main branch.

## Clusters

- **homelab-1**: Primary homelab cluster running production services
- **nomad**: Nomad integration cluster for control plane services

## Architecture

### App of Apps Pattern

The repository uses ArgoCD's App of Apps pattern for hierarchical application management:

```
Root Application (clutser-apps.yaml, nomad-apps.yaml)
    |
    |--- Cluster Applications (clusters/*/*)
    |
    |--- Application Resources (apps/*/*)
```

### Directory Structure

```
.
|--- clutser-apps.yaml          # Root app for homelab-1 cluster
|--- nomad-apps.yaml            # Root app for nomad cluster
|--- clusters/                  # ArgoCD Application manifests per cluster
|   |--- homelab-1/
|   |--- nomad/
|--- apps/                      # Actual Kubernetes resources
    |--- homelab-1/
    |--- immich/
    |--- nomad/
```

## Deployed Applications

### homelab-1 Cluster

- **ClickHouse**: Analytics database with HyperDX stack
- **External Secrets**: Integration with HashiCorp Vault
- **Immich**: Photo and video management
- **Longhorn**: Distributed storage system
- **Uptime Kuma**: Monitoring and uptime tracking

### nomad Cluster

- **nomad-ctrl-plane**: Nomad control plane integration service

## Infrastructure Components

### Networking

- **Gateway API**: Kubernetes Gateway API with Cilium gateway class
- **TLS**: Self-signed certificates managed by cert-manager
- **Domain**: Internal services use `.iulian.cld` domain

### Storage

- **Longhorn**: Persistent storage provider for all clusters

### Secrets Management

- **External Secrets Operator**: Syncs secrets from HashiCorp Vault
- **Vault**: `http://192.168.1.152:8200` (KV v2 backend)

## Adding a New Application

### Kustomize-based Application

1. Create application directory:
   ```bash
   mkdir -p apps/<cluster>/<app-name>
   ```

2. Add Kubernetes manifests and create `kustomization.yaml`:
   ```yaml
   apiVersion: kustomize.config.k8s.io/v1beta1
   kind: Kustomization
   resources:
     - deployment.yaml
     - service.yaml
     - gw.yaml
     - httproute.yaml
   ```

3. Create ArgoCD Application in `clusters/<cluster>/<app-name>.yaml`:
   ```yaml
   apiVersion: argoproj.io/v1alpha1
   kind: Application
   metadata:
     name: <app-name>
     namespace: argocd
   spec:
     project: default
     source:
       repoURL: git@github.com:iuliansafta/iulian-cloud-gitops.git
       path: apps/<cluster>/<app-name>
       targetRevision: main
     destination:
       server: https://kubernetes.default.svc
       namespace: <namespace>
     syncPolicy:
       automated:
         prune: true
         selfHeal: true
       syncOptions:
         - CreateNamespace=true
   ```

4. Commit and push - ArgoCD will automatically deploy

### Helm Chart Application

Create ArgoCD Application manifest in `clusters/<cluster>/<app-name>.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: <app-name>
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://helm-repo-url.example.com
    chart: <chart-name>
    targetRevision: <version>
    helm:
      values: |
        # Helm values here
  destination:
    server: https://kubernetes.default.svc
    namespace: <namespace>
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

## Gateway API Configuration

For external access to services:

1. **Gateway Resource** (`gw.yaml`):
   ```yaml
   apiVersion: gateway.networking.k8s.io/v1beta1
   kind: Gateway
   metadata:
     name: <app>-gateway
     namespace: <namespace>
     annotations:
       cert-manager.io/issuer: ca-issuer
   spec:
     gatewayClassName: cilium
     listeners:
       - name: https
         protocol: HTTPS
         port: 443
         hostname: <app>.iulian.cld
         tls:
           certificateRefs:
             - kind: Secret
               name: <app>-tls-cert
   ```

2. **HTTPRoute Resource** (`httproute.yaml`):
   ```yaml
   apiVersion: gateway.networking.k8s.io/v1
   kind: HTTPRoute
   metadata:
     name: <app>-http-route
     namespace: <namespace>
   spec:
     parentRefs:
       - name: <app>-gateway
     rules:
       - matches:
           - path:
               type: PathPrefix
               value: /
         backendRefs:
           - name: <service-name>
             port: <port>
   ```

3. **Certificate Issuer** (`issuer.yaml`):
   ```yaml
   apiVersion: cert-manager.io/v1
   kind: Issuer
   metadata:
     name: self-signed
     namespace: <namespace>
   spec:
     selfSigned: {}
   ---
   apiVersion: cert-manager.io/v1
   kind: Certificate
   metadata:
     name: ca
     namespace: <namespace>
   spec:
     isCA: true
     privateKey:
       algorithm: ECDSA
       size: 256
     secretName: ca
     commonName: ca
     issuerRef:
       name: self-signed
       kind: Issuer
   ---
   apiVersion: cert-manager.io/v1
   kind: Issuer
   metadata:
     name: ca-issuer
     namespace: <namespace>
   spec:
     ca:
       secretName: ca
   ```

## Sync Policies

Applications use automated sync with the following policies:

- **prune: true** - Resources not in git are automatically deleted
- **selfHeal: true** - Manual changes are automatically reverted
- **CreateNamespace: true** - Target namespaces are created automatically
- **PruneLast: true** - Resources are deleted after new ones are created (used in root apps)

## Repository Information

- **Git Repository**: `git@github.com:iuliansafta/iulian-cloud-gitops.git`
- **Main Branch**: `main`
- **ArgoCD Namespace**: `argocd`
- **ArgoCD Project**: `default`

## Requirements

- ArgoCD installed and configured
- Cilium CNI with Gateway API support
- cert-manager for TLS certificate management
- Longhorn for persistent storage
- External Secrets Operator with Vault integration
