# homelab-gitops

Single source of truth for every cluster in the lab. Argo CD on the `mgmt` cluster syncs this repo continuously; after the one-time bootstrap, no cluster state changes outside of Git.

## Clusters

| Name | Distro | Where | Role |
|---|---|---|---|
| `mgmt` | RKE2 `v1.35.5+rke2r2` | Proxmox on NUC #1 (3 nodes, `10.0.3.41/42/43`) | Rancher `2.14.2`, Argo CD; Harbor planned (W5) |
| `edge` | K3s | 3x Raspberry Pi 5 (API via kube-vip VIP `10.0.3.20`) | Always-on; uptime probes |
| `dev` | K3s `v1.35.8+k3s1` | Harvester guest cluster on NUC #2 (3 nodes, `10.0.3.158/.182/.241`) | Long-lived golden-path deploy target |
| `sandbox-*` | — | Harvester guest clusters | Planned: ephemeral, recreated weekly |

Argo CD targets `mgmt` by server URL (`https://kubernetes.default.svc`), not by name ([ADR-0007](./docs/adr/0007-applicationset-mgmt-destination.md)); `edge` and `dev` are registered cluster Secrets. Harvester itself (`yavin-prime`, single node) is imported into Rancher but is not an Argo target. `dev` is defined in git as Rancher provisioning-v2 CRs in `apps/harvester-clusters/` and provisioned by Rancher ([ADR-0012](./docs/adr/0012-harvester-guest-clusters-via-provisioning-v2.md)).

## Layout

```
homelab-gitops/
├── bootstrap/                # one-time apply, then self-managed
│   ├── argocd-values.yaml    # Helm values for Argo CD itself (incl. the KSOPS sidecar)
│   ├── argocd-app.yaml       # Application that manages Argo CD (multi-source: upstream chart + $values)
│   └── root-appset.yaml      # ApplicationSet generating one <cluster>-bootstrap Application per clusters/* directory
├── clusters/                 # one directory per cluster: that cluster's Argo Applications (applied to mgmt)
│   ├── mgmt/                 # platform-mgmt, mgmt-control-plane, harvester-clusters
│   ├── edge/                 # hello-world-edge
│   ├── dev/                  # hello-world-dev
│   └── sandbox-template/     # excluded from the bootstrap ApplicationSet; stamped onto ephemeral clusters
├── platform/                 # shared platform components, one ApplicationSet per component (see ADR-0008)
│   ├── cert-manager/         # cluster generator: every registered cluster; LE staging + prod ClusterIssuers
│   ├── gateway-api/          # cluster generator: vendored standard-channel CRDs
│   └── traefik/              # list generator [dev, edge, mgmt]; per-cluster wildcard cert in extras/<cluster>/
├── apps/                     # workload manifests referenced by clusters/*/apps.yaml
│   ├── harvester-clusters/   # Harvester guest-cluster definitions (Rancher provisioning-v2 CRs, ADR-0012)
│   ├── hello-world/          # traefik/whoami smoke-test app (edge)
│   └── mgmt-control-plane/   # HTTPRoutes for Rancher + Argo CD
└── docs/                     # setup log, ADRs, runbooks, incidents
```

## Bootstrap order

Run these once, by hand. Everything after step 4 is Git. The full chronological build log is in [`docs/setup-log.md`](./docs/setup-log.md); decision rationale lives in [`docs/adr/`](./docs/adr/); a concept-level walkthrough of how the platform layer (`platform/`) fans components out across clusters is in [`docs/platform-layer.md`](./docs/platform-layer.md).

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

3. **Add Pi-hole A records for `argocd.mgmt.wretchedhive.io`:**
   Three A records → `10.0.3.41`, `10.0.3.42`, `10.0.3.43` (round-robin straight to RKE2 nodes, same pattern as `rancher.mgmt.wretchedhive.io`). Pi-hole Local DNS is exact-match, so every hostname needs its own records. See [ADR-0002](./docs/adr/0002-dns-pi-hole-yavin-internal.md) and [ADR-0009](./docs/adr/0009-letsencrypt-dns01-split-horizon.md). (Originally `argocd.yavin.internal`; those records were retired 2026-09-11.)

4. **Apply the self-managing Application and the root ApplicationSet:**
   ```
   kubectl apply -f bootstrap/argocd-app.yaml
   kubectl apply -f bootstrap/root-appset.yaml
   ```
   From this point, Argo manages Argo. Future Helm value changes are committed to `bootstrap/argocd-values.yaml`; no more `helm upgrade` by hand.

5. **Register downstream clusters:**
   ```
   argocd cluster add <kubeconfig-context-for-edge> --name edge
   ```
   Harvester guest clusters (`dev`) are registered by hand with an explicit ServiceAccount + cluster Secret — see [`docs/runbooks/register-harvester-guest-cluster.md`](./docs/runbooks/register-harvester-guest-cluster.md). Registration is deliberately not in git, and neither is the Harvester cloud credential (`cattle-global-data:cc-qf8cr`, created in the Rancher UI) that `apps/harvester-clusters/` depends on: both are manual day-0 prerequisites on a rebuild. `mgmt` is the in-cluster default; nothing to register.

## TLS today

Let's Encrypt prod everywhere, LAN-only. Each cluster runs Traefik behind a Gateway API `Gateway` (`yavin`) that terminates TLS with a per-cluster wildcard — `*.mgmt.`, `*.edge.` and `*.dev.wretchedhive.io` — issued by cert-manager over DNS-01 via Cloudflare. Split-horizon DNS: Pi-hole (`10.0.3.2`) holds the LAN A records and there are no public A records, so nothing is reachable from the internet. Rancher (`rancher.mgmt.wretchedhive.io`) and Argo CD (`argocd.mgmt.wretchedhive.io`) are served by HTTPRoutes on mgmt's Traefik, which binds hostPort 80/443 on each mgmt node; edge and dev use klipper-lb LoadBalancer Services. See [ADR-0009](./docs/adr/0009-letsencrypt-dns01-split-horizon.md), [ADR-0010](./docs/adr/0010-mgmt-traefik-hostport-daemonset.md) and the runbook in [`docs/tls-certs.md`](./docs/tls-certs.md).

## Secrets

SOPS + age. Encrypted files match `*.enc.yaml` (see `.sops.yaml`) and are decrypted at manifest-generation time by a KSOPS sidecar in `argocd-repo-server`, wired in `bootstrap/argocd-values.yaml`. The first consumer is the Cloudflare API token at `platform/cert-manager/extras/cloudflare-token.enc.yaml`. See [ADR-0005](./docs/adr/0005-secrets-sops-age.md).
