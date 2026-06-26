# How the platform layer works

Conceptual explainer for `platform/`. Use this if you're adding a new platform component and want to understand the moving parts, or if you're trying to remember what fans out what.

- Chronological build steps live in [`setup-log.md`](./setup-log.md).
- Decision rationale (why this layout vs alternatives) lives in [`adr/0008-platform-component-pattern.md`](./adr/0008-platform-component-pattern.md).
- This doc covers *what each piece does* and *how a request flows through them*.

## What cert-manager does (and why we need it)

cert-manager is a Kubernetes operator that manages TLS certificates as Kubernetes objects.

Without it, getting a TLS cert for a service like `argocd.yavin.internal` means: generate a private key, ask some Certificate Authority (Let's Encrypt, an internal CA, whatever) to sign a certificate, manually load both into the cluster as a `Secret`, remember to renew before expiry. For every service. Forever.

With it, you write a Kubernetes resource:

```yaml
kind: Certificate
metadata:
  name: argocd-tls
spec:
  dnsNames: [argocd.yavin.internal]
  issuerRef: { name: my-internal-ca }
```

…and cert-manager generates the key, asks the named Issuer to sign a cert, stores the result as a Secret, and quietly renews it before expiry. Ingress controllers and pods consume the Secret like any other.

**cert-manager is not itself a CA** — it's the automation layer that talks to CAs. The signing authority is a separate object called an `Issuer` (or `ClusterIssuer` for the cluster-scoped version). cert-manager supports many: Let's Encrypt, internal step-ca, HashiCorp Vault, AWS Private CA, or just "self-signed" for testing.

**What's installed today is the foundation, not the payoff.** The cluster has cert-manager itself plus a `SelfSigned` ClusterIssuer. cert-manager can technically issue certs now, but only self-signed ones — which doesn't help browser warnings. The real value lands when step-ca arrives (see [ADR-0006](./adr/0006-defer-real-tls-to-stepca-phase2.md)) and a new ClusterIssuer is wired to it. At that point "browser-trusted internal TLS" becomes a one-resource change per service.

cert-manager is also a dependency for several other Phase 2 things (Gateway API webhook certs, observability TLS, etc.), which is why it's the foundation that goes in first.

## The fan-out problem (and why ApplicationSets exist)

There are 3 clusters (mgmt, edge, and dev-once-it-exists). cert-manager needs to run on all of them. Each cluster will eventually run other shared things too — observability, Gateway API, Harbor.

The naive way: write `Application/cert-manager-mgmt`, `Application/cert-manager-edge`, etc. by hand. Add a new cluster? Edit Git, add another Application. Add a new platform component? Add three new Applications. With N clusters × M components, this scales miserably.

**ApplicationSet is Argo CD's templating layer for Applications.** You write *one* ApplicationSet that has:

- a **generator** — declares what to fan out over (a list of clusters, a list of git directories, a matrix, etc.)
- a **template** — declares what each fanned-out Application should look like

Argo CD then maintains the resulting Applications automatically. Add a cluster, the matching Application appears. Remove one, it disappears.

The generator used for platform components is the **cluster generator**. It looks at every cluster registered with this Argo CD (the `cluster-edge` Secret, plus the implicit in-cluster connection for mgmt) and emits one item per cluster. One ApplicationSet → as many Applications as you have clusters. Today: 2. Tomorrow when you `argocd cluster add` the Harvester guest cluster: 3, automatically, with no Git change.

That matches the Week 9 success criterion exactly: *"a new cluster registration would automatically receive both."*

## The three pieces

### `ApplicationSet/cert-manager` — the fan-out template

Lives at `platform/cert-manager/appset.yaml`. For every registered cluster, this template creates an Application that installs cert-manager onto that cluster.

The Application it stamps out is **multi-source**:

1. Source #1: the cert-manager Helm chart from Jetstack (`charts.jetstack.io`, pinned to v1.20.2).
2. Source #2: this repo's `platform/cert-manager/extras/` directory, containing the SelfSigned ClusterIssuer.

The destination is the cluster's `cert-manager` namespace.

When Argo reconciles this ApplicationSet, two Applications appear in `argocd app list`: `cert-manager-in-cluster` (mgmt) and `cert-manager-edge`. Each one independently installs cert-manager onto its target cluster.

### `Application/platform-mgmt` — the umbrella that *applies* the ApplicationSet

Lives in `clusters/mgmt/apps.yaml`. The part that's easy to miss: the ApplicationSet above is itself a Kubernetes resource that has to be applied *to* mgmt's cluster, because mgmt is where Argo CD runs and processes ApplicationSets. So something has to apply `platform/cert-manager/appset.yaml` (and any future appsets) onto mgmt.

That something is `Application/platform-mgmt`. It's a plain Argo Application targeting mgmt's `argocd` namespace, with its source pointing at the `platform/` directory in Git. When it syncs, it reads `platform/kustomization.yaml`, kustomize-builds the umbrella, and applies the resulting ApplicationSet resources to mgmt.

This is sometimes called the **app-of-appsets pattern**. The umbrella exists so adding a new platform component is just dropping a file into `platform/` — no edit to `clusters/mgmt/apps.yaml` needed.

### `platform/` — the umbrella's source

The directory itself. `platform/kustomization.yaml` is the umbrella's index — it lists every `<component>/appset.yaml` as a resource. Adding a new platform component (Harbor, observability, step-ca, …) is just dropping a file into `platform/<new-component>/appset.yaml` and adding it to the umbrella — `platform-mgmt` picks it up on next reconcile, creates the new ApplicationSet on mgmt, which fans the per-cluster Applications out. No per-cluster Git edits.

Generator choice per component is just about scope. `cert-manager/appset.yaml` uses a cluster generator because cert-manager belongs on every cluster. `traefik/appset.yaml` uses a list generator (`elements: [{name: edge}]`) because Traefik is the ingress data plane on edge today and would conflict with mgmt's `rke2-ingress-nginx` for hostPort 80/443. Add or remove a cluster from the list, and Argo stamps or prunes the matching Application — the pattern handles both fan-out shapes.

## The chain, end-to-end

```
clusters/mgmt/apps.yaml
        │
        └── Application/platform-mgmt
                source: platform/
                destination: mgmt cluster, argocd namespace
                │
                │  applies the umbrella to mgmt
                ▼
        ┌───────────────────────────────────────────┐
        │ mgmt cluster                              │
        │                                           │
        │   ApplicationSet/cert-manager             │
        │     generator: clusters (auto-discovers)  │
        │     template: install cert-manager        │
        │     │                                     │
        │     │  Argo loops over registered         │
        │     │  clusters and stamps Applications:  │
        │     ▼                                     │
        │   Application/cert-manager-in-cluster ────┼──→ installs cert-manager into mgmt
        │   Application/cert-manager-edge       ────┼──→ installs cert-manager into edge
        │                                           │
        └───────────────────────────────────────────┘
```

## Why this structure (the part ADR-0008 captures)

A few obvious-looking shortcuts that were rejected:

- **"Why not put the ApplicationSet directly in `clusters/mgmt/apps.yaml` and skip the umbrella?"** Could be done today, with one component. But every new platform component would mean editing `clusters/mgmt/apps.yaml`. With the umbrella, every new component is just a new directory + one line in `platform/kustomization.yaml`.

- **"Why one ApplicationSet per component instead of one big matrix that does cluster × component?"** Because components diverge in scope. cert-manager goes everywhere. Harbor goes mgmt-only. Observability might land on mgmt + edge but skip ephemeral sandbox clusters. Per-component scoping is a one-line label selector on each cluster generator; a matrix forces every component into the same shape.

- **"Why is the SelfSigned ClusterIssuer in `extras/` instead of in the Helm chart's values?"** Helm charts don't own ClusterIssuers — they're a separate concern. The multi-source pattern (chart + kustomize-built extras in one Application) lets each component bring its own non-chart resources without fighting the chart's authors.

Full rationale in [ADR-0008](./adr/0008-platform-component-pattern.md).

## What happens when you push

Nothing runs until the changes land in `main`. Once they do:

1. Argo CD on mgmt polls `main` (default: every 3 minutes).
2. `mgmt-bootstrap` (which already points at `clusters/mgmt/`) sees the new `platform-mgmt` Application and creates it.
3. `platform-mgmt` syncs, applying the `cert-manager` ApplicationSet onto mgmt.
4. The ApplicationSet controller runs the cluster generator and creates `cert-manager-in-cluster` and `cert-manager-edge` Applications.
5. Each of those Applications syncs, installing the chart + ClusterIssuer onto its target cluster.

Steps 2–5 typically complete within a couple of minutes once you push. After that, `kubectl -n cert-manager get pods` on either cluster shows actual cert-manager pods running, and `kubectl get clusterissuer selfsigned` reports `Ready: True`.

## Adding a new platform component (the recipe)

The pattern is reusable. For any new component (Gateway API, observability, Harbor, …):

1. `mkdir platform/<component>` and write a cluster-generator `appset.yaml` whose template installs the component (multi-source: chart + your `extras/`).
2. If the component needs cluster-scoped resources beyond what the chart provides (issuers, gateway classes, etc.), put them in `platform/<component>/extras/`.
3. Add `- <component>/appset.yaml` to `platform/kustomization.yaml`.
4. Commit + push. `platform-mgmt` picks it up; the new ApplicationSet creates one Application per registered cluster; rollout follows.

If a component is mgmt-only (Harbor), the cluster generator gets a `selector.matchLabels` that only matches mgmt's cluster Secret labels. Same shape, narrower fan-out.
