# homelab-gitops

Single source of truth for every cluster in the lab. Argo CD on the `mgmt` cluster syncs this repo continuously; after the one-time bootstrap, no cluster state changes outside of Git.

## Clusters

| Name | Distro | Where | Role |
|---|---|---|---|
| `mgmt` | RKE2 | Proxmox on NUC #1 (3-node) | Rancher, Argo CD, Harbor, central observability |
| `edge` | K3s | 3x Raspberry Pi 5 | Always-on; internal DNS (Pi-hole), uptime probes |
| `dev` | RKE2 | Harvester guest cluster on NUC #2 | Long-lived golden-path deploy target |
| `sandbox-*` | RKE2 | Harvester guest clusters | Ephemeral; recreated weekly |

## Layout

```
homelab-gitops/
├── bootstrap/                # one-time apply, then self-managed
│   ├── argocd-values.yaml    # Helm values for Argo CD itself
│   ├── argocd-app.yaml       # Application that manages Argo CD (multi-source: upstream chart + $values)
│   └── root-appset.yaml      # ApplicationSet generating one bootstrap Application per clusters/* directory
├── clusters/                 # one directory per registered cluster = the cluster registry
│   ├── mgmt/
│   ├── edge/
│   ├── dev/
│   └── sandbox-template/     # excluded from the bootstrap ApplicationSet; stamped onto ephemeral clusters
├── platform/                 # shared platform components (cert-manager, gateway, observability, etc.) — populated in Phase 2
└── apps/                     # workload Application manifests — populated in Phase 3 via golden-path CI PRs
```

## Bootstrap order

Run these once, by hand. Everything after step 4 is Git. The full chronological build log is in [`docs/setup-log.md`](./docs/setup-log.md); decision rationale lives in [`docs/adr/`](./docs/adr/).

1. **Generate age key on your workstation:**
   ```
   age-keygen -o ~/.config/sops/age/keys.txt
   chmod 600 ~/.config/sops/age/keys.txt
   ```
   Copy the public key (`age1...`) into `.sops.yaml`. Distribute the private key to the `mgmt` cluster as a Secret named `sops-age` in the `argocd` namespace (one-shot via Ansible or `kubectl create secret generic`). See [ADR-0005](./docs/adr/0005-secrets-sops-age.md).

2. **Install Argo CD via Helm** (one-shot — owned by `homelab-ansible/playbooks/rke2-mgmt-argocd.yml`):
   ```
   helm repo add argo https://argoproj.github.io/argo-helm
   helm upgrade --install argo-cd argo/argo-cd \
     --namespace argocd --create-namespace \
     --version <pin same as bootstrap/argocd-app.yaml> \
     -f bootstrap/argocd-values.yaml
   ```
   Day-0 only — see [ADR-0004](./docs/adr/0004-two-stage-bootstrap-ansible-then-argocd.md) for the ownership boundary.

3. **Add Pi-hole A records for `argocd.yavin.internal`:**
   Three A records → `10.0.3.41`, `10.0.3.42`, `10.0.3.43` (round-robin straight to RKE2 nodes, same pattern as Rancher). See [ADR-0002](./docs/adr/0002-dns-pi-hole-yavin-internal.md) and [ADR-0003](./docs/adr/0003-bypass-npm-for-rancher-and-argocd.md).

4. **Apply the self-managing Application and the root ApplicationSet:**
   ```
   kubectl apply -f bootstrap/argocd-app.yaml
   kubectl apply -f bootstrap/root-appset.yaml
   ```
   From this point, Argo manages Argo. Future Helm value changes are committed to `bootstrap/argocd-values.yaml`; no more `helm upgrade` by hand.

5. **Register downstream clusters:**
   ```
   argocd cluster add <kubeconfig-context-for-edge> --name edge
   argocd cluster add <kubeconfig-context-for-dev>  --name dev
   ```
   `mgmt` is the in-cluster default; nothing to register.

## TLS today

Internal-only. `*.yavin.internal` hostnames for Rancher and Argo CD resolve directly to the RKE2 mgmt nodes; `rke2-ingress-nginx` terminates TLS with the in-cluster self-signed cert (dynamiclistener for Rancher, chart-default for Argo CD). NPM is not in the path for these two control planes (it still fronts other lab services). Browser warning on click-through is expected until step-ca lands in Phase 2 — see [ADR-0001](./docs/adr/0001-internal-only-no-public-exposure.md), [ADR-0003](./docs/adr/0003-bypass-npm-for-rancher-and-argocd.md), [ADR-0006](./docs/adr/0006-defer-real-tls-to-stepca-phase2.md).

## Secrets

SOPS + age. Encrypted files match `*.enc.yaml` (see `.sops.yaml`). KSOPS sidecar wiring will be added to `bootstrap/argocd-values.yaml` in Phase 2, when the first encrypted secret actually needs to sync. See [ADR-0005](./docs/adr/0005-secrets-sops-age.md).
