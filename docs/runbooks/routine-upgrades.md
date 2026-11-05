# Runbook: routine upgrades (standing procedure)

**Written:** 2026-11-04, from the October 2026 platform upgrade
([`platform-upgrade-2026-10.md`](platform-upgrade-2026-10.md)). That runbook is the one-off plan and
its record; this one is what you follow every time after it.
**Covers:** every OS, Kubernetes distribution, platform component, app image, tool and provider
in `homelab-ansible`, `homelab-gitops` and `homelab-tofu`: where each version is pinned, how often
to check it, how to roll it out, and how to prove it worked.

> **The short version.** Check for new versions on a schedule ([Cadence](#cadence)). Change one pin
> per PR. Snapshot before anything stateful. Roll dev → edge → mgmt. After every merge, run
> [`labhealth`](#lab-health-check) and don't start the next change until it's green.

---

## Cadence

| When | What | Section |
|---|---|---|
| **Weekly, automatic** | ansible CI's `version-sync` job (Mondays 06:17 UTC) fails if an ansible day-0 pin drifts from gitops. A red run means someone bumped one side only. | [Pins in two repos](#pins-in-two-repos) |
| **Monthly** | OS patches on every host, least critical first | [OS patches](#os-patches) |
| **Monthly** | Check for new versions ([how](#checking-for-new-versions)). Take security releases now, everything else at the quarterly pass | |
| **Quarterly** | Platform components: Gateway API CRDs, Traefik, cert-manager, Argo CD + KSOPS, app images, Pi-hole, Portainer | [Platform components](#platform-components-gitops) |
| **Quarterly** | Kubernetes **patch** releases (dev → edge → mgmt), kube-vip | [Kubernetes](#kubernetes) |
| **Quarterly** | Tooling: CI actions, kubeconform, CI Helm, Ansible collections, tofu provider, workstation CLIs | [Tooling](#tooling) |
| **When Rancher ships a new minor** (about twice a year) | The big chain: Rancher → Harvester → Kubernetes **minor**. Plan it like October 2026, with its own runbook | [The compatibility chain](#the-compatibility-chain) |
| **At least every ~9 months** | Restart RKE2/K3s on every server so leaf certs rotate. Any version upgrade does this, so a quarterly Kubernetes patch keeps you covered. | `homelab-ansible/docs/k3s-ops.md` (cert rotation) |
| **Out of band** | Security advisories (Rancher, Argo CD, cert-manager, Traefik, Pi-hole FTL, kernel CVEs) | Same procedures, sooner |

Write each run's results in a log table like the one at the end of the October runbook (date,
snapshot name, result, deviations). Mistakes repeat when nobody wrote them down.

---

## Standing rules

These are the October 2026 rules, kept because each one was earned.

1. **One change, one PR, one gate.** Never combine two components in one merge. When something
   breaks, you know which change did it.
2. **Snapshot before anything stateful**: Rancher, Argo CD, Kubernetes, Harvester, OS reboots on
   mgmt or edge. See [Snapshots](#snapshots). The mgmt etcd snapshot **is** the Rancher backup;
   Rancher can't be downgraded in place.
3. **Least critical first.** dev → edge → mgmt for Kubernetes, codeserver Pi → edge → mgmt →
   Proxmox → Pi-hole for OS patches.
4. **`git pull` before editing**, then check `git rev-parse --short HEAD origin/main` prints the
   same hash twice. **Verify rendered output, not the diff** (`kubectl kustomize`, `helm template`).
5. **gitops before ansible** for anything [pinned in both repos](#pins-in-two-repos).
6. **Never `tofu plan`/`apply` from a checkout behind `origin/main`.** State is shared, so an old
   checkout reads as drift; in October one planned **4 to destroy**.
7. **Never upgrade edge, dev or Harvester's guest clusters from the Rancher UI.** Edge's version is
   owned by ansible and dev's by gitops. A UI upgrade drifts from git and Argo fights it.
8. **Never take yavin1 (primary Pi-hole) and the edge cluster (backup Pi-hole + its VIP) down at the
   same time.** They are each other's DNS fallback.
9. **Wait for Argo to pick up a merge.** Polling is 180 s + up to 60 s jitter, and the repo-server
   caches the branch head, so allow **up to ~8 minutes**. Don't read "nothing happened yet" as
   "nothing will happen".
10. If Rancher shows downstream clusters `Connected=True / Ready=False` after a mgmt-side change and
    they don't recover within ~5 min:
    `kubectl --context rke2-mgmt -n cattle-system rollout restart deploy/rancher`.
11. **Don't advance past a failed gate.** Stop, investigate, write it down.

---

## Where every version lives

| Component | Pinned in | Applied by | Notes |
|---|---|---|---|
| Rancher (+ fleet/webhook/turtles, which follow it) | gitops `clusters/mgmt/rancher.yaml` `targetRevision` | Argo CD (`rancher` app) | Also ansible `rancher_chart_version` (day-0, CI-checked) |
| Argo CD chart | gitops `bootstrap/argocd-app.yaml` `targetRevision` | Argo CD (self-managed) | Also ansible `argocd_chart_version` (day-0, CI-checked) |
| KSOPS + CMP sidecar | gitops `bootstrap/argocd-values.yaml` (`viaductoss/ksops`, `quay.io/argoproj/argocd`) | Argo CD | Sidecar tag **must equal** the Argo CD appVersion |
| cert-manager | gitops `platform/cert-manager/appset.yaml` | Argo CD (all 3 clusters) | Also `.github/workflows/validate.yml` render step **and** ansible `certmanager_chart_version` |
| Traefik chart | gitops `platform/traefik/appset.yaml` | Argo CD (all 3 clusters) | Also `.github/workflows/validate.yml` render step |
| Gateway API CRDs | gitops `platform/gateway-api/manifests/standard-install.yaml` (vendored) | Argo CD (all 3 clusters) | Match what the running Traefik implements |
| whoami | gitops `apps/hello-world/deployment.yaml` | Argo CD (edge + dev) | |
| Pi-hole (backup, on edge) + nebula-sync | gitops `apps/pihole-edge/deployment.yaml` | Argo CD | Keep in step with the yavin1 tag |
| Pi-hole (primary, yavin1) | Portainer stack editor on yavin1 | Portainer | Not in git |
| Portainer | ansible `inventory/group_vars/all.yml` `portainer_version` | `playbooks/portainer.yml` | |
| code-server | upstream latest | `playbooks/codeserver.yml --tags codeserver_update` | |
| mgmt RKE2 | ansible `inventory/group_vars/rke2-mgmt/main.yml` `rke2_version` | `playbooks/rke2-mgmt-update.yml` | `rke2_ingress_controller: none` in the same file must stay (see [Kubernetes](#kubernetes)) |
| edge K3s | ansible `inventory/group_vars/k3s-edge.yml` `k3s_version` | `playbooks/k3s-edge-update.yml` | |
| dev K3s | gitops `apps/harvester-clusters/dev.yaml` `kubernetesVersion` | Argo CD → Rancher provisioning | |
| kube-vip | ansible `kubevip_version` in `k3s-edge.yml` **and** `rke2-mgmt/main.yml` | `playbooks/kubevip-update.yml` | One cluster at a time |
| Harvester | — (Harvester UI) | Harvester upgrade controller | Its own RKE2 comes with it |
| dev node OS image | gitops `apps/harvester-clusters/dev.yaml` `diskInfo.imageName` | Rancher replaces machines | |
| OS packages (all hosts) | Debian/Ubuntu/PVE repos | `playbooks/site.yml --tags os_upgrade` | |
| Ansible collections | ansible `requirements.yml` (`==` pins) | `ansible-galaxy collection install -r requirements.yml --upgrade` | |
| CI actions | ansible `.github/workflows/ci.yml`, gitops `.github/workflows/validate.yml` | GitHub | |
| CI kubeconform / Helm | gitops `.github/workflows/validate.yml` | GitHub | |
| bpg/proxmox provider | tofu `provider.tf` **and** `modules/proxmox-vm/main.tf` | `tofu init -upgrade` | Commit `.terraform.lock.hcl` |
| Workstation CLIs | Homebrew | `brew upgrade` | Rancher needs Helm ≥ 3.18 |

### Pins in two repos

cert-manager, Argo CD and Rancher are installed by ansible on day 0 and owned by Argo from day 1
(ADR-0004), so their version lives in both repos. `homelab-ansible/.github/scripts/check-chart-pins.py`
reads gitops **`main`** and fails ansible CI if they differ. So:

1. Merge the gitops PR. Wait for the rollout and its gate.
2. Re-run the ansible PR's check (it reads gitops `main`, so it's red until step 1 lands), then merge it.

The ansible pin is never applied to a running cluster. Don't re-run `rke2-mgmt-rancher.yml`,
`rke2-mgmt-argocd.yml` or `rke2-mgmt-certmanager.yml` on a live cluster: they fight Argo for the
release.

---

## The compatibility chain

Most upgrades are independent. These aren't, and they decide the order of the big upgrades:

| Rule | Check it at |
|---|---|
| Each Rancher minor supports a window of Kubernetes minors (2.15: 1.34–1.36). **Only use RKE2/K3s versions in that Rancher release's certified list**, even when a newer patch exists. The Rancher chart refuses to install on a Kubernetes minor above its `kubeVersion`. | SUSE Rancher support matrix; the Rancher release notes' "Kubernetes versions" list |
| Each Harvester minor works with one Rancher minor (1.9 ↔ 2.15). **Upgrade Rancher first, then Harvester.** Harvester upgrades only from the previous minor's latest patch. | Harvester support matrix and upgrade docs |
| cert-manager and Argo CD must support the **new** Kubernetes minor before any cluster moves to it. | cert-manager supported-releases table; Argo CD tested-versions table |
| Gateway API CRDs track what Traefik implements. | Traefik release notes |
| The KSOPS CMP sidecar image tracks the Argo CD appVersion. | `bootstrap/argocd-values.yaml` comment |

For a Kubernetes **minor** the order is always:
1. cert-manager and Argo CD versions that support it.
2. A Rancher version that certifies it.
3. Harvester to match that Rancher.
4. Kubernetes dev → edge → mgmt.
5. kube-vip.

For a **patch** release, only rule 1 matters.

---

## Checking for new versions

Run this monthly. It prints the latest stable release of each upstream next to what's running.
Pre-releases and `-rc` tags are skipped.

```zsh
latest() { curl -s "https://api.github.com/repos/$1/releases?per_page=20" \
  | python3 -c 'import json,sys; print(next((r["tag_name"] for r in json.load(sys.stdin) if not r["prerelease"] and not r["draft"]), "?"))'; }
for r in rancher/rancher harvester/harvester argoproj/argo-cd cert-manager/cert-manager traefik/traefik \
         kubernetes-sigs/gateway-api viaduct-ai/kustomize-sops kube-vip/kube-vip rancher/rke2 k3s-io/k3s \
         traefik/whoami pi-hole/docker-pi-hole lovelaze/nebula-sync portainer/portainer \
         bpg/terraform-provider-proxmox yannh/kubeconform helm/helm; do
  printf '%-32s %s\n' $r "$(latest $r)"
done
# chart versions (what the pins actually reference)
helm repo add argo https://argoproj.github.io/argo-helm >/dev/null 2>&1
helm repo add jetstack https://charts.jetstack.io >/dev/null 2>&1
helm repo add rancher-stable https://releases.rancher.com/server-charts/stable >/dev/null 2>&1
helm repo update >/dev/null
helm search repo argo/argo-cd; helm search repo jetstack/cert-manager; helm search repo rancher-stable/rancher
helm show chart oci://ghcr.io/traefik/helm/traefik 2>/dev/null | grep -E '^(version|appVersion)'
# Harvester: what this cluster is offered
kubectl --context harvester -n harvester-system get versions.harvesterhci.io
```

Before taking any version:
- **Read the release notes**, and especially the "breaking" / "upgrade notes" section of every
  release you'd skip over.
- **Wait about two weeks after a `.0`** unless it fixes a security issue. Skip a major chart bump
  that ships the same app version, since it's all risk and no gain.
- For community Rancher, only `rancher-stable` versions count. Some patch releases are Prime-only
  (October 2026: 2.14.4–2.14.6).
- **Read the "Important" notes and any "do not install" notice too**, not just "breaking". In
  November 2026 both mattered: kube-vip's v1.3.0 notes said not to install it (its binary reported
  v1.2.4; v1.3.1 is the fixed re-release), and RKE2 1.36's "Important Note" changed its bundled
  ingress default. That change would have taken down mgmt ([Kubernetes](#kubernetes)).
- **Kubernetes:** take the highest patch in Rancher's certified list, not the highest that exists.
  The list is in the KDM metadata the running Rancher reads (`kdm-branch` setting, e.g.
  `release-v2.15`). This prints the newest three per minor for the running Rancher version:
  ```zsh
  RV=$(kubectl --context rke2-mgmt get settings.management.cattle.io server-version -o jsonpath='{.value}')
  BR=$(kubectl --context rke2-mgmt get settings.management.cattle.io kdm-branch -o jsonpath='{.value}')
  curl -sL https://releases.rancher.com/kontainer-driver-metadata/$BR/data.json | python3 -c '
  import json,sys; d=json.load(sys.stdin); v=lambda s: tuple(int(x) for x in s.lstrip("v").split("-")[0].split("."))
  R=v(sys.argv[1])
  for dist in ("k3s","rke2"):
      ok=[r["version"] for r in d[dist]["releases"] if v(r.get("minChannelServerVersion","v0.0.0"))<=R<=v(r.get("maxChannelServerVersion","v99.0.0"))]
      for m in sorted({x.split(".")[1] for x in ok}): print(dist, "1."+m, [x for x in ok if x.split(".")[1]==m][-3:])' "$RV"
  ```

---

## Lab health check

Run it before you start (it's the baseline) and after every change. Paste once per shell:

```zsh
# dev kubeconfig, rebuilt fresh: the Rancher-generated one points at an in-cluster IP + internal CA
kubectl --context rke2-mgmt -n fleet-default get secret dev-kubeconfig -o jsonpath='{.data.value}' | base64 -d \
  | sed 's#server: https://10\.43\.[0-9.]*/#server: https://rancher.mgmt.wretchedhive.io/#' \
  | grep -v 'certificate-authority-data:' > /tmp/dev.kubeconfig

labhealth() {
  echo "== nodes (cluster name status version)"
  for c in rke2-mgmt k3s-edge harvester; do
    kubectl --context $c get nodes --no-headers 2>&1 | awk -v c=$c '{print "  "c, $1, $2, $5}'
  done
  KUBECONFIG=/tmp/dev.kubeconfig kubectl get nodes --no-headers 2>&1 | awk '{print "  dev", $1, $2, $5}'
  echo "== rancher clusters"
  kubectl --context rke2-mgmt get clusters.management.cattle.io \
    -o custom-columns='  NAME:.spec.displayName,CONNECTED:.status.conditions[?(@.type=="Connected")].status,READY:.status.conditions[?(@.type=="Ready")].status' --no-headers
  echo "== argo"
  kubectl --context rke2-mgmt -n argocd get applications.argoproj.io --no-headers \
    | awk '$2!="Synced"||$3!="Healthy"{bad++; print "  NOT GREEN:",$0} END{print "  "NR" apps, "bad+0" not green"}'
  echo "== endpoints"
  for h in rancher.mgmt argocd.mgmt hello.edge hello.dev; do
    printf '  %-12s %s\n' $h "$(curl -s -o /dev/null -w 'http=%{http_code} verify=%{ssl_verify_result}' --max-time 10 https://$h.wretchedhive.io)"
  done
  echo "== API VIPs (401 = answering)"
  for v in 10.0.3.40 10.0.3.20; do printf '  %-10s %s\n' $v "$(curl -sk -o /dev/null -w 'http=%{http_code}' --max-time 5 https://$v:6443/version)"; done
}
labhealth
```

Use `applications.argoproj.io`, not `applications` or `app`: on mgmt, Rancher's
`apps.catalog.cattle.io` takes the short name.

**Healthy:**
- Every node is `Ready`.
- `local` and the 3 downstream clusters (`edge-pi`, `yavin-prime`, `dev`) show `True/True`.
- Every Argo app is green (20 as of 2026-11-04).
- All four endpoints return `http=200 verify=0`.
- Both VIPs return `http=401`.

For anything that rolls pods, also keep an endpoint watch running in a second terminal and check
it never shows anything but 200:

```zsh
while true; do for h in rancher.mgmt argocd.mgmt hello.edge hello.dev; do
  printf '%s %s %s\n' "$(date +%T)" $h "$(curl -s -o /dev/null -m 3 -w '%{http_code}' https://$h.wretchedhive.io/)"; done; sleep 1; done \
  | grep -v ' 200$'
```

---

## Snapshots

```zsh
SNAP=pre-<change>-$(date +%Y%m%d)

# mgmt etcd: Rancher's entire state lives here. This IS the Rancher backup.
ssh -t ubuntu@10.0.3.41 "sudo rke2 etcd-snapshot save --name $SNAP && sudo rke2 etcd-snapshot ls | tail -3"

# edge etcd (embedded etcd on all three Pis)
ssh -t pancakes@10.0.3.3 "sudo k3s etcd-snapshot save --name $SNAP && sudo k3s etcd-snapshot ls | tail -3"

# dev: Rancher snapshots every 5 h. Confirm a recent one exists:
kubectl --context rke2-mgmt -n fleet-default get etcdsnapshots.rke.cattle.io --sort-by=.metadata.creationTimestamp | grep '^dev' | tail -2
```

Before a Proxmox host reboot, also take VM backups. Before any Pi-hole change, take a Teleporter
export. Both commands are in the October runbook's snapshot section. Restoring etcd:
[October runbook, Rollback: mgmt etcd restore](platform-upgrade-2026-10.md#rollback-mgmt-etcd-restore).

---

## OS patches

Monthly. Run from `homelab-ansible` after `git pull`, in this order, running `labhealth` between
hosts:

| # | Hosts | Command | Impact |
|---|---|---|---|
| 1 | codeserver Pi (canary) | `ansible-playbook playbooks/site.yml --tags os_upgrade --limit codeserver-pi --ask-become-pass` | none |
| 2 | edge Pis | `ansible-playbook playbooks/site.yml --tags os_upgrade --limit k3s-edge --ask-become-pass` | none (Traefik and Pi-hole backup are HA) |
| 3 | mgmt VMs | `ansible-playbook playbooks/site.yml --tags os_upgrade --limit rke2-mgmt --ask-become-pass` | none; drained, serial, etcd-gated |
| 4 | Proxmox | see [Proxmox host](#proxmox-host) | **mgmt down ~5 min** if it reboots |
| 5 | Pi-hole Pi (yavin1) | `ansible-playbook playbooks/site.yml --tags os_upgrade --limit portainer-pi --ask-become-pass` | DNS falls back to edge's Pi-hole (rule 8) |

**Raspberry Pis and Proxmox never write `/var/run/reboot-required`** (it comes from
`update-notifier-common`, which neither installs), so `os_upgrade` patches them and **skips the
reboot**. A new kernel sits there unbooted. After patching, compare the running kernel with the
newest installed one **of the same flavor**. Pis carry both `-v8` and `-2712` kernels, so a plain
"newest" is wrong:

```zsh
for h in pancakes@10.0.3.15 pancakes@10.0.3.3 pancakes@10.0.3.4 pancakes@10.0.3.5 pancakes@10.0.3.2 root@10.0.3.6; do
  printf '%-20s %s\n' $h "$(ssh $h 'r=$(uname -r); n=$(ls -1v /lib/modules | grep -- "-${r##*-}$" | tail -1); echo "running $r newest $n $([ "$r" = "$n" ] && echo ok || echo REBOOT)"')"
done
```

If they differ, reboot:
- **codeserver Pi or yavin1:** `ansible-playbook playbooks/site.yml --tags manual_reboot --limit <host>`.
- **edge:** `ansible-playbook playbooks/k3s-edge-reboot.yml --ask-become-pass`, drained, serial
  and etcd-gated.
- **Proxmox:** [Proxmox host](#proxmox-host).

Ubuntu (mgmt VMs) does write the file, so `os_upgrade` reboots those itself through the safe path.

Both clusters' serial plays wait for etcd before they uncordon a node and move on. A node being
`Ready` isn't enough, which is what caused the 2026-10-18 mgmt quorum loss. mgmt uses
`rke2/tasks/control-plane-settled.yml` (`rke2_settle_seconds`), edge uses
`k3s/tasks/control-plane-settled.yml` (`k3s_settle_seconds`). If a play stops at that gate, don't
re-run it until etcd is healthy on all three nodes.

### Proxmox host

The base role never reboots Proxmox. When the kernel check says it needs one:

```zsh
ansible-playbook playbooks/site.yml --tags os_upgrade --limit proxmox --ask-become-pass
ssh root@10.0.3.6 'for id in 100 101 102; do echo "$id $(qm config $id | grep ^onboot)"; done'   # must be onboot: 1
ssh root@10.0.3.6 'for id in 100 101 102; do qm shutdown $id --timeout 180 & done; wait; qm list'
ssh root@10.0.3.6 'reboot'
until ssh -o ConnectTimeout=5 root@10.0.3.6 'qm list' 2>/dev/null | grep -c running | grep -q 3; do sleep 15; done
```

Afterwards edge and Harvester may sit at `Connected/Ready=False`. Apply standing rule 10.

---

## Platform components (gitops)

Every change here is one gitops PR. Before merging, render it locally the same way CI does. After
merging, allow up to 8 minutes for Argo, then check the gate and run `labhealth`.

```zsh
cd ~/Developer/Personal/homelab/homelab-gitops && git pull && git switch -c bump-<component>-<version>
```

### Gateway API CRDs

```zsh
V=v1.6.3   # what the running Traefik implements
curl -sL https://github.com/kubernetes-sigs/gateway-api/releases/download/$V/standard-install.yaml \
  -o platform/gateway-api/manifests/standard-install.yaml
grep -m1 'bundle-version' platform/gateway-api/manifests/standard-install.yaml
```

**Gate:** the `gateways.gateway.networking.k8s.io` CRD's `bundle-version` annotation shows the new
version on all three clusters, and every `yavin` Gateway is still `Programmed`.

### Traefik

Change `targetRevision` in `platform/traefik/appset.yaml`, **and** `--version` in the *Render
platform charts* step of `.github/workflows/validate.yml`. CI keeps its own copy of the pin; miss
it and CI keeps validating the old chart.

```zsh
helm template traefik oci://ghcr.io/traefik/helm/traefik --version <new> >/dev/null && echo chart-ok
```

- **mgmt runs a hostPort DaemonSet.** It needs `maxSurge: 0` / `maxUnavailable: 1`, because the
  chart default `maxSurge: 1` deadlocks: the new pod can't bind 80/443 while the old one holds them.
  That's set in the appset's hostPorts element. Check it survives any chart values restructure.
- **edge and dev run 2 replicas** with a PDB and anti-affinity. Check the new chart still honours
  those keys.

**Gate:**
- `traefik-{dev,edge,mgmt}` are Synced/Healthy.
- Pods run the new Traefik version.
- The endpoint watch stayed at 200.

### cert-manager *(gitops, then ansible)*

1. gitops: change `targetRevision` in `platform/cert-manager/appset.yaml` and `--version` in
   `validate.yml`. `crds.enabled: true` means the chart upgrades its own CRDs. Read the release notes
   for removed RBAC and renamed ports.
2. After the rollout: ansible `certmanager_chart_version` in `inventory/group_vars/rke2-mgmt/main.yml`.

**Gate:**
- `cert-manager-{dev,edge,in-cluster}` are Synced/Healthy.
- `kubectl get certificates -A` shows every Certificate `Ready` on all three clusters.

A single "ACME client not initialised" requeue right after startup is normal.

### Argo CD + KSOPS *(gitops, then ansible; snapshot mgmt first)*

Argo upgrades itself, and `bootstrap/argocd-app.yaml` is git-managed (from `clusters/mgmt` via
`bootstrap/kustomization.yaml`), so the chart bump lands on merge. One PR:

| File | Change |
|---|---|
| `bootstrap/argocd-app.yaml` | chart `targetRevision` |
| `bootstrap/argocd-values.yaml` | CMP sidecar `quay.io/argoproj/argocd:<tag>`, **equal to the new chart's appVersion** |
| `bootstrap/argocd-values.yaml` | KSOPS init image, if KSOPS has a new release |

```zsh
helm repo update argo >/dev/null
helm show chart argo/argo-cd --version <new> | grep appVersion          # sidecar tag must match
helm template argocd argo/argo-cd --version <new> -n argocd -f bootstrap/argocd-values.yaml \
  | grep -E 'image: .*(argocd|ksops)' | sort -u
helm template argocd argo/argo-cd --version <new> -n argocd -f bootstrap/argocd-values.yaml \
  | grep -c '^kind: NetworkPolicy'     # 0; global.networkPolicy.create is pinned false
```

For a **chart major**, read the chart's upgrade notes: 10.0.0 silently flipped the NetworkPolicy
default. KSOPS ≥ 4.4 is distroless and nonroot, so the init container must stay
`ksops install /plugin` with the `plugins/viaduct.ai/v1/ksops` subPath.

`argocd-server` and `argocd-repo-server` run 2 replicas with a PDB (`minAvailable: 1`) since
2026-11. Before that, every mgmt node drain cost ~10–15 s of Argo UI 503s. Check a chart upgrade
keeps the `replicas` / `pdb` keys. The application-controller, applicationset-controller and redis
stay single: a drain briefly pauses reconciliation and the cache, not the UI.

Then ansible `argocd_chart_version`. Workstation CLI: `brew upgrade argocd`.

**Gate:**
- All Argo pods are `Running` on the new version.
- `labhealth` shows every app green.
- A KSOPS-rendered app still decrypts. Hard-refresh `pihole-edge` or `cert-manager-in-cluster`; the
  repo-server log should show `Plugin command successful`.

**Rollback:** revert the PR. If Argo can no longer sync itself, there is **no Helm release to
`helm rollback`**: Argo applies the manifests directly. Re-apply the previous version from the
workstation:

```zsh
helm template argocd argo/argo-cd --version <old> -n argocd -f bootstrap/argocd-values.yaml \
  | kubectl --context rke2-mgmt apply -n argocd --server-side --force-conflicts -f -
```

As a last resort, restore the mgmt etcd snapshot.

### Rancher *(gitops, then ansible; snapshot mgmt first)*

Rancher has been an Argo app since 2026-11-04 (`clusters/mgmt/rancher.yaml`). The ansible playbook
is day-0 only.

1. Check the target against [the compatibility chain](#the-compatibility-chain): the new version must
   support every cluster's current Kubernetes minor, and Harvester's.
2. Snapshot mgmt etcd. That snapshot is the only rollback.
3. gitops: change `targetRevision` in `clusters/mgmt/rancher.yaml`. Render it with the `valuesObject`
   from that file and read the release notes. Merge.
   - The chart's **pre-upgrade hook runs automatically** as an Argo PreSync Job. It's a read-only
     RKE1-resource check that blocks the sync if RKE1 resources exist.
   - `postDelete.enabled: false` must stay. Argo would run Rancher's post-delete cleanup if the
     Application were ever deleted.
4. Watch the rollout:
   ```zsh
   kubectl --context rke2-mgmt -n cattle-system rollout status deploy/rancher --timeout=15m
   kubectl --context rke2-mgmt get settings.management.cattle.io server-version -o jsonpath='{.value}{"\n"}'
   helm --kube-context rke2-mgmt list -A | grep -E 'fleet|turtles|webhook'   # Rancher upgrades these itself
   ```
5. ansible: `rancher_chart_version`, after the gitops merge.

**Gate:**
- `server-version` shows the new version, and the `rancher` app is Synced/Healthy.
- All downstream agents are on the new version and `True/True`.
- The Harvester UI loads under Virtualization Management.
- `harvester-clusters` is still Synced.

A few seconds of `rancher.mgmt` 502s during the roll is normal.

> **Watch `harvester-clusters` after a Rancher upgrade.** 2.15.3 stripped `field.cattle.io/creatorId`
> from the live `dev` Cluster, and the Rancher webhook then denied every self-heal. That's why the app
> now ignores that annotation. A future release may change another webhook-guarded field the same
> way. Read the denial message and add an `ignoreDifferences` entry rather than fighting the webhook.

### Apps and images

- **whoami:** `apps/hello-world/deployment.yaml`. One file covers edge and dev. **Gate:** `hello.edge`
  and `hello.dev` return 200 with a `Hostname:` line.
- **Pi-hole:** take a Teleporter export first, then update both copies:
  - **Primary (yavin1):** Portainer → Stacks → Pi-hole stack → Editor. Change the tag, tick
    **Re-pull image and redeploy**, then **Update the stack**.
  - **Backup (edge):** `apps/pihole-edge/deployment.yaml` (`pihole/pihole` and
    `ghcr.io/lovelaze/nebula-sync`).

  **Never both at once** (rule 8). FTL v6.8 started refusing Teleporter for app passwords unless
  `webserver.api.app_sudo=true`. nebula-sync failures are **silent** (pods stay Ready on stale
  config), so check its logs after any Pi-hole bump. Keep `RUN_GRAVITY=false`.

  **Gate:** `pihole -v` on yavin1 shows the new tag; `dig +short <local record> @10.0.3.2` and
  `@10.0.3.3` both answer; nebula-sync logs show a successful sync.
- **Portainer:** ansible `portainer_version` in `inventory/group_vars/all.yml`, then
  `ansible-playbook playbooks/portainer.yml --ask-become-pass`. Back up `portainer_data` on yavin1
  first. Prefer LTS releases.
- **code-server:** `ansible-playbook playbooks/codeserver.yml --tags codeserver_update --ask-become-pass`.

---

## Kubernetes

Snapshot first. Only take versions in the running Rancher's certified list. Order: **dev → edge →
mgmt**, one per day if you like, each gated.

| Cluster | Change | Roll out | Watch |
|---|---|---|---|
| dev | gitops `apps/harvester-clusters/dev.yaml` `kubernetesVersion`. Must be in the KDM list above (that list is what the Rancher UI's version dropdown shows) | merge; Rancher's system agent restarts K3s in place, one node at a time, with no drain or VM replacement. In 2026-11 all three took ~30 s | `KUBECONFIG=/tmp/dev.kubeconfig kubectl get nodes -w` |
| edge | ansible `k3s_version` in `inventory/group_vars/k3s-edge.yml` | `ansible-playbook playbooks/k3s-edge-update.yml --ask-become-pass` (serial, etcd-gated: `k3s_settle_seconds`) | the endpoint watch, VIP `10.0.3.20` |
| mgmt | ansible `rke2_version` in `inventory/group_vars/rke2-mgmt/main.yml`; snapshot again right before | `ansible-playbook playbooks/rke2-mgmt-update.yml --ask-become-pass` (serial, etcd-gated: `rke2_settle_seconds`) | `labhealth`, `helm --kube-context rke2-mgmt list -n kube-system` (bundled charts update too) |

**Gate (each):**
- All nodes are on the new version and `Ready`.
- The VIP answers 401.
- The endpoint watch stayed at 200.
- Argo is all green. Allow ~5 min after mgmt nodes cycle.

**RKE2 (mgmt): check the bundled components, not just Kubernetes.** RKE2 ships CNI, CoreDNS,
metrics-server and an ingress controller, and a minor can change their defaults. Read the "Important
Note" and "Charts" sections of the target release. If config needs to change, remember that
`rke2-mgmt-update.yml` **doesn't rewrite `config.yaml`**. Land the config first with
`rke2-mgmt-reconfigure.yml` while still on the old version, then bump the version.

- **The ingress default.** From 1.36 RKE2 deploys its own Traefik unless an ingress-nginx HelmChart
  exists. mgmt removed ingress-nginx in W2, so without `rke2_ingress_controller: none` (in
  `config.yaml` as `ingress-controller: none`) it would deploy `rke2-traefik` and `rke2-traefik-crd`.
  That means a second controller on hostPorts 80/443 and a second owner for the Gateway API CRDs.
- **After an upgrade, confirm it** on each node:
  `journalctl -u rke2-server | grep 'Using ingress-controller'` shows `[none]`, and
  `kubectl -n kube-system get helmcharts` lists no ingress chart.

**Edge after a roll:** the last node drained ends up with no Traefik, Pi-hole or whoami pods,
because nothing moves pods back. They're still spread across two nodes, so this is fine.

### kube-vip

Change `kubevip_version` in **one** cluster's group vars per PR, edge first.

```zsh
while true; do printf '%s %s\n' "$(date +%T)" "$(curl -sk -o /dev/null -w '%{http_code}' --max-time 3 https://10.0.3.20:6443/version)"; sleep 2; done   # 10.0.3.40 for mgmt
ansible-playbook playbooks/kubevip-update.yml --limit k3s-edge --ask-become-pass                                                                     # then --limit rke2-mgmt
```

**Gate:**
- Every kube-vip pod is on the new image.
- The VIP watch showed at most a few seconds of non-401. 2026-11 numbers for reference: edge 0
  misses, mgmt ~3 s while the VIP moved nodes.
- Each pod's log reports the new version (`kube-vip.io version=`). That catches a mislabelled build
  like v1.3.0.
- Exactly one node holds the VIP: `ip -4 -o addr | grep <vip>` on each.

Start the VIP watch **before** the play: the whole roll takes ~10 s.

Check the RBAC against the new release's generated rules (`kube-vip manifest rbac`, or
`pkg/kubevip/config_generator.go` in its source). Extra rules for features we don't run
(`svc_enable: "false"`) can be skipped.

---

## Harvester *(dev outage ~30–60 min)*

Upgrade Rancher to the matching minor first. Harvester upgrades only from the previous minor's
latest patch.

1. **Pre-check** on the node:
   ```zsh
   ssh rancher@10.0.3.9
   # sudo -i
   # curl -sLf https://raw.githubusercontent.com/harvester/upgrade-helpers/main/pre-check/v1.x/check.sh -o check.sh
   # chmod +x check.sh && ./check.sh -l /tmp/precheck.txt    -> every test must say Pass
   ```
2. **Power off the dev VMs yourself.** With `restoreVM: false`, Harvester won't upgrade around running
   non-migratable VMs. If it stops them itself, it **doesn't start them again afterwards**:
   ```zsh
   for vm in $(kubectl --context harvester -n default get vm -o name | grep dev-yavin-pool); do
     kubectl --context harvester -n default patch $vm --type merge -p '{"spec":{"runStrategy":"Halted"}}'
   done
   kubectl --context harvester -n default get vmi      # wait until none remain
   ```
3. If the UI doesn't offer the version yet:
   `curl -sL https://releases.rancher.com/harvester/<v>/version.yaml | kubectl --context harvester apply -f -`.
   Then use Harvester UI → Dashboard → Upgrade, and watch with
   `kubectl --context harvester -n harvester-system get upgrades.harvesterhci.io -w`.
4. Start dev again by patching `runStrategy` back to `RerunOnFailure` with the same loop. If
   Harvester stopped the VMs itself, `runStrategy` never changed, so the patch does nothing; use
   **Start** in the UI or `virtctl start`.

**Gate:**
- `kubectl --context harvester get settings.harvesterhci.io server-version` shows the new version.
- The dev VMs are Running.
- `labhealth` is green, and Harvester shows `True/True` in Rancher.

### dev node image *(optional, yearly)*

Upload a current `resolute-server-cloudimg-amd64.img` to Harvester as a new image, then point
`diskInfo.imageName` in `apps/harvester-clusters/dev.yaml` at it. Rancher replaces the machines one
at a time (`drainBeforeDelete: true`).

---

## Tooling

No runtime impact. CI is the gate.

| Tool | Change | Gate |
|---|---|---|
| GitHub Actions | `uses:` tags in ansible `ci.yml` and gitops `validate.yml` | CI green on both |
| kubeconform, CI Helm | URL / `setup-helm` `version:` in gitops `validate.yml` | CI green |
| Ansible collections | `requirements.yml` `==` pins, then `ansible-galaxy collection install -r requirements.yml --upgrade` | ansible CI green; `--syntax-check` every playbook |
| bpg/proxmox provider | `version` in tofu `provider.tf` **and** `modules/proxmox-vm/main.tf`, then `tofu init -upgrade && tofu plan` | the plan says **No changes** (see below); commit `.terraform.lock.hcl` |
| Workstation | `brew upgrade helm kubectl argocd sops age opentofu` | `helm version` ≥ 3.18 |

**Tofu provider bumps must plan clean.** A provider upgrade that suddenly wants to change existing
VMs is a read-back change in the provider, not real drift. 0.117 wanted to rewrite every VM's
cloud-init `password`, and that's why the pin is still `~> 0.109.0`. Never apply a plan that touches
`initialization` or the image on a running VM. Cloud-init changes can re-run first-boot modules, and
an image replacement replaces the VMs. Stop and investigate instead, for example with an
`ignore_changes` on the field, then confirm that was the only diff.

---

## After each run

- Record what you took, the snapshot names and anything surprising in the run's log.
- If a gate needed a workaround, fix the procedure **here** so the next run doesn't need it.
