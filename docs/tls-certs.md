# TLS certs — how they work, how to get one, how to keep them working

Runbook for anything touching TLS certs. Conceptual overview lives in [`platform-layer.md`](./platform-layer.md); this is the *how do I actually do X* reference.

## Current architecture

**Certificate authority:** [Let's Encrypt](https://letsencrypt.org/) via ACME. Two [`ClusterIssuer`](../platform/cert-manager/extras/) resources are defined cluster-wide:

- `letsencrypt-staging` — LE staging environment. Fake CA (browsers won't trust it), no meaningful rate limits. Use while iterating.
- `letsencrypt-prod` — LE production. Publicly-trusted (chains to ISRG Root X1). [Rate-limited](https://letsencrypt.org/docs/rate-limits/) — use only after staging is green.

Both live at `platform/cert-manager/extras/letsencrypt-{staging,prod}-clusterissuer.yaml` and are fanned out to every cluster by `platform/cert-manager/appset.yaml`.

**Domain validation:** DNS-01. cert-manager writes a temporary `_acme-challenge.<hostname>` TXT record via the Cloudflare API, LE validates it, then the record is deleted. No HTTP-01 (that would require inbound network exposure, which the lab does not permit — see [ADR-0001](./adr/0001-internal-only-no-public-exposure.md)).

**Domain:** `wretchedhive.io`, hosted on Cloudflare. Split-horizon: no public A records exist for `*.wretchedhive.io`, so LAN clients see Pi-hole's overrides and the outside world sees NXDOMAIN. Only the `_acme-challenge.*` TXT records ever land in public DNS, and only transiently while a challenge is in flight.

**API credential:** Cloudflare API token, SOPS-encrypted at `platform/cert-manager/extras/cloudflare-token.enc.yaml`, decrypted at manifest-generation time by the argocd-repo-server KSOPS sidecar (see [ADR-0005](./adr/0005-secrets-sops-age.md)). Token scope: `Zone → Zone → Read` + `Zone → DNS → Edit` on `wretchedhive.io` only.

**End-consumer:** the Traefik Gateway on edge terminates TLS using a wildcard `Certificate` (`platform/traefik/extras/wildcard-cert.yaml`) that references `letsencrypt-prod`. Any hostname under `*.edge.wretchedhive.io` gets a valid cert for free — no per-hostname `Certificate` needed.

## Prerequisites (should already be true)

- `argocd-repo-server` running with the KSOPS sidecar (`bootstrap/argocd-values.yaml`).
- `sops-age` Secret exists in the `argocd` namespace with the age private key at `keys.txt`. If this is ever destroyed, all encrypted secrets in git become unrecoverable — back up the age private key at `~/.config/sops/age/keys.txt` offline.
- `cert-manager` running on the target cluster (fanned out from `platform/cert-manager/appset.yaml`).
- Local tooling: `sops` (`brew install sops` on macOS), `age`, and access to the Cloudflare account for the target domain.

## Getting a cert for a new hostname (most common)

The typical case: you're adding a new service like `harbor.edge.wretchedhive.io`. **You do not need a new `Certificate`.** The existing edge wildcard `*.edge.wretchedhive.io` already covers it. Two things to do:

1. **Add a Pi-hole A record** for the new hostname pointing at the edge node IPs (`10.0.3.3`, `10.0.3.4`, `10.0.3.5`). Round-robin A records — one entry per node IP.
2. **Reference the wildcard cert Secret** from your service's Gateway/HTTPRoute. Traefik already terminates TLS on `wildcard-edge-wretchedhive-io-tls` for the whole `*.edge.wretchedhive.io` listener; your `HTTPRoute` just attaches to that Gateway. Nothing else to configure.

That's it. Wildcard matching (RFC 6125) is one label deep, so `harbor.edge.wretchedhive.io` matches `*.edge.wretchedhive.io` but `foo.harbor.edge.wretchedhive.io` would not.

## Getting a cert for a new hostname on the mgmt plane

**Fully migrated as of 2026-09-09 (ADR-0009 W1, Phases 2–4).** The mgmt wildcard
`*.mgmt.wretchedhive.io` is live on **LE prod** and the same pattern as the edge wildcard applies —
the `Certificate` lives in `apps/mgmt-tls/`, synced to the mgmt cluster only by the `mgmt-tls`
Application in `clusters/mgmt/apps.yaml` (deliberately *not* in `platform/cert-manager/extras/`,
which fans out to every registered cluster).

| Consumer | Hostname | Secret | Note |
|---|---|---|---|
| Argo CD | `argocd.mgmt.wretchedhive.io` | `argocd-server-tls` | name hardcoded by the argo-cd chart for `server.ingress.tls: true` |
| Rancher | `rancher.mgmt.wretchedhive.io` | `tls-rancher-ingress` | name hardcoded by the Rancher chart for `ingress.tls.source: secret` |

Both verify with no `-k` (`http=200 verify=0`). Downstream Rancher agents validate against the
public LE chain with `agent-tls-mode: system-store` and **no** `CATTLE_CA_CHECKSUM` pin.

**Both chart-hardcoded secret names are a trap worth remembering.** Neither chart lets you choose
the name, so a `Certificate` must target it exactly — and the `Certificate` must be `READY` *before*
the chart's ingress is pointed at it, or the ingress controller serves its self-signed default in
the gap. See [the migration runbook](./runbooks/mgmt-tls-migration.md).

**Rate-limit note:** all three mgmt-plane issuances share the SAN set
{`mgmt.wretchedhive.io`, `*.mgmt.wretchedhive.io`}, so LE counts them as *duplicates* of each other
against the 5-per-week ceiling. **3 of 5 used as of 2026-09-09** (argocd staging → argocd prod →
argocd secret-rename, then rancher). Re-issuing on this SAN set this week has little headroom; use
`letsencrypt-staging` to iterate.

## Getting a wildcard cert for a new cluster (same domain)

Case: a new cluster comes online (dev, or the eventual mgmt migration) and needs its own `*.<cluster>.wretchedhive.io` wildcard. Cheaper than the new-domain flow — cert-manager, both LE ClusterIssuers, and the Cloudflare token are already fanned out to every registered cluster by the `cert-manager` ApplicationSet, so the only new pieces are a `Certificate` resource on the new cluster and its consumer.

Assuming the cluster is already registered with Argo CD and cert-manager reports `Ready` there:

1. **Pick a component to host the Certificate.** Almost always the same one that hosts the cluster's Gateway/ingress data plane. On edge that's `platform/traefik/extras/wildcard-cert.yaml`. For a new cluster, either extend the existing component's `extras/` to include the new cluster (if the same chart is going to run there) or add a parallel component (e.g. a `platform/traefik-mgmt/`) if the topology is different.

2. **Write the `Certificate`.** Model on `platform/traefik/extras/wildcard-cert.yaml`. For `*.dev.wretchedhive.io`:

    ```yaml
    apiVersion: cert-manager.io/v1
    kind: Certificate
    metadata:
      name: wildcard-dev-wretchedhive-io
      namespace: <ns-where-consumer-lives>
    spec:
      secretName: wildcard-dev-wretchedhive-io-tls
      commonName: "*.dev.wretchedhive.io"
      dnsNames:
        - "dev.wretchedhive.io"
        - "*.dev.wretchedhive.io"
      issuerRef:
        name: letsencrypt-staging   # flip to letsencrypt-prod after validating
        kind: ClusterIssuer
        group: cert-manager.io
    ```

    Same shape as edge; only the subdomain and Secret name change.

3. **Reference the Secret** from whatever terminates TLS on that cluster (Gateway `certificateRefs`, Ingress `tls[].secretName`, etc.). Its namespace must match the consumer's namespace — cert-manager creates the Secret next to the `Certificate` resource.

4. **Add Pi-hole records** for the hostnames that will live under the new wildcard, pointing at the new cluster's node IPs (round-robin per node, same pattern as edge). Without them the wildcard cert issues fine, but nothing resolves for LAN clients.

5. **Verify against staging first,** then flip `issuerRef.name` to `letsencrypt-prod` per the staging↔prod section below.

**Why per-cluster wildcards, not one shared wildcard:** blast radius is scoped to one cluster. A leaked private key can impersonate `foo.<that-cluster>.wretchedhive.io` for any `foo`, but not hostnames on other clusters. It also means cert-manager on each cluster manages its own Certificate lifecycle independently — no cross-cluster coupling for renewals or reissuance.

**What if the new cluster is on a network segment that can't reach Cloudflare's API?** DNS-01 is initiated by cert-manager, which makes outbound HTTPS calls to `api.cloudflare.com`. If that outbound path is blocked, DNS-01 will fail. The lab today has unrestricted outbound; if that changes, the mitigation is to route cert issuance through a delegated `_acme-challenge` subdomain (CNAME to a small dedicated zone) that a proxy cluster can write to. Out of scope for now.

## Getting a cert for a new domain (a different DNS zone)

Rare — most services fit under an existing `*.<cluster>.wretchedhive.io` wildcard. But if you register a second domain (say `example.com`) and want LE certs for it:

1. **Get a Cloudflare API token scoped to the new zone.** Log into Cloudflare → My Profile → API Tokens → Create Token → Custom Token. Permissions: `Zone → Zone → Read` and `Zone → DNS → Edit`. Zone Resources: `Include → Specific zone → example.com`.
2. **Encrypt the token as a new Secret.** Create the file locally with the plaintext:

    ```yaml
    apiVersion: v1
    kind: Secret
    metadata:
      name: cloudflare-api-token-example-com
      namespace: cert-manager
    type: Opaque
    stringData:
      api-token: <paste-token>
    ```

    Save as `platform/cert-manager/extras/cloudflare-token-example-com.enc.yaml`. Then encrypt in place:

    ```bash
    sops -e -i platform/cert-manager/extras/cloudflare-token-example-com.enc.yaml
    ```

    Verify `head -2` shows plaintext YAML but `data.api-token` is `ENC[...]`.

3. **Register the encrypted file with KSOPS.** Add it to the `files:` list in `platform/cert-manager/extras/ksops-generator.yaml`:

    ```yaml
    files:
      - ./cloudflare-token.enc.yaml
      - ./cloudflare-token-example-com.enc.yaml
    ```

4. **Add a `ClusterIssuer` for the new zone.** Create `platform/cert-manager/extras/letsencrypt-prod-example-com-clusterissuer.yaml`:

    ```yaml
    apiVersion: cert-manager.io/v1
    kind: ClusterIssuer
    metadata:
      name: letsencrypt-prod-example-com
      annotations:
        argocd.argoproj.io/sync-wave: "1"
    spec:
      acme:
        server: https://acme-v02.api.letsencrypt.org/directory
        email: chris.runo@gmail.com
        privateKeySecretRef:
          name: letsencrypt-prod-example-com-account-key
        solvers:
          - dns01:
              cloudflare:
                apiTokenSecretRef:
                  name: cloudflare-api-token-example-com
                  key: api-token
    ```

    Add it to the `resources:` list in `platform/cert-manager/extras/kustomization.yaml`.

5. **Do the same for staging.** Create `letsencrypt-staging-example-com-clusterissuer.yaml` with the staging ACME URL (`https://acme-staging-v02.api.letsencrypt.org/directory`) and a distinct `letsencrypt-staging-example-com-account-key`. Reference it from a `Certificate` first, validate, then flip to the prod ClusterIssuer.

6. **Write the `Certificate` that consumes the new issuer.** Under whichever component owns the TLS listener for the new domain — probably a new wildcard cert under a `platform/traefik/extras/` (or wherever the Gateway lives), following the shape of the existing `wildcard-cert.yaml`.

7. **Commit and push.** After `platform-mgmt` picks up the appset changes, watch reconciliation:

    ```bash
    kubectl --context k3s-edge -n cert-manager get clusterissuer letsencrypt-prod-example-com
    kubectl --context k3s-edge -n <ns-of-cert> get certificate <cert-name> -w
    ```

**Why a distinct token + issuer per zone:** the token is scoped per-Cloudflare-zone, so a token for `wretchedhive.io` cannot write TXT records under `example.com`. cert-manager needs the right token for the zone it's validating.

## Should I ever issue an apex `*.wretchedhive.io` wildcard?

Almost certainly not, but the option exists.

**What it would cover:** any single-label hostname under `wretchedhive.io` — `foo.wretchedhive.io`, `home.wretchedhive.io`. RFC 6125 wildcards only match one label, so an apex wildcard does **not** cover `hello.edge.wretchedhive.io` or anything two labels deep. It doesn't replace the per-cluster wildcards; it complements them for hostnames that sit directly under the apex.

**Why per-cluster wildcards were chosen instead:** blast radius. A single leaked apex-wildcard private key can impersonate anything at `<foo>.wretchedhive.io` including the labels that name whole clusters (`edge.wretchedhive.io`, `mgmt.wretchedhive.io`). Per-cluster wildcards contain a leak to one cluster's namespace.

**If you do need one** — e.g., a lab-wide landing page at `home.wretchedhive.io` — the flow is identical to a per-cluster wildcard but with different Common Name / SANs:

```yaml
spec:
  secretName: wildcard-wretchedhive-io-tls
  commonName: "*.wretchedhive.io"
  dnsNames:
    - "wretchedhive.io"
    - "*.wretchedhive.io"
  issuerRef:
    name: letsencrypt-prod
    ...
```

Keep it on whichever cluster hosts the apex-level services, ideally scoped tightly (small namespace, tight RBAC on the Secret) to compensate for the wider matching set. And prefer nothing that ever leaves the cluster it lives on — no cross-cluster Secret replication of the apex wildcard.

## Day-2 ops

### Renewals

Automatic. cert-manager renews any `Certificate` roughly 30 days before its `notAfter`. LE certs have 90-day validity, so a renewed cert becomes the working one at ~60 days of age. No human action required.

Verify a cert isn't drifting toward expiry:

```bash
kubectl --context k3s-edge -n traefik get certificate wildcard-edge-wretchedhive-io \
  -o custom-columns='NAME:.metadata.name,READY:.status.conditions[?(@.type=="Ready")].status,NOTAFTER:.status.notAfter,RENEWAT:.status.renewalTime'
```

If `notAfter` is in the past or `READY=False` for more than a few minutes, jump to the diagnosis section below.

### Rotating the Cloudflare API token

Do this if the token is leaked, expired, or you're revoking access. The token is stored in one place — the encrypted Secret in git — so rotation is a git operation, not a `kubectl edit`.

1. Create a new token in Cloudflare with the same scope (or revoke first if compromised — LE issuance stalls for the outage window but existing certs keep serving until their `notAfter`).
2. Regenerate the encrypted Secret:

    ```bash
    cat > /tmp/cloudflare-token.plain.yaml <<'EOF'
    ---
    apiVersion: v1
    kind: Secret
    metadata:
      name: cloudflare-api-token
      namespace: cert-manager
    type: Opaque
    stringData:
      api-token: <paste-new-token>
    EOF
    sops -e /tmp/cloudflare-token.plain.yaml > platform/cert-manager/extras/cloudflare-token.enc.yaml
    rm /tmp/cloudflare-token.plain.yaml
    ```

3. Commit and push. On next sync, cert-manager sees the updated Secret and uses the new token for the next challenge.
4. Revoke the old token in Cloudflare.

### Flipping between staging and prod

Useful during debugging — e.g., you're changing the `Certificate` resource shape or the solver config and want to iterate without touching LE prod rate limits.

One line: change `issuerRef.name` on the `Certificate` (e.g., `platform/traefik/extras/wildcard-cert.yaml`) from `letsencrypt-prod` to `letsencrypt-staging` (or back). Both ClusterIssuers are permanently defined — no other change needed. cert-manager reissues the cert against the new issuer on the next sync; the Secret name stays the same, downstream consumers don't notice.

### Backing up the age key

Not cert-manager per se, but the entire encrypted-secrets story (including the Cloudflare token that DNS-01 depends on) depends on the age private key. If it's lost, encrypted Secrets in git become unrecoverable — the cluster can be rebuilt, but the CF token file must be re-encrypted from scratch with a new key.

The key lives at `~/.config/sops/age/keys.txt` on the operator workstation. Back it up somewhere offline. The `sops-age` Secret in the cluster's `argocd` namespace is a copy — not a backup.

## Diagnosing a stalled cert

Symptoms: `Certificate` shows `READY=False` for more than ~3 minutes. Work outward from the `Certificate` toward the ACME server:

1. **Describe the `Certificate`** — usually points at a specific `CertificateRequest` and gives a human-readable reason.

    ```bash
    kubectl --context k3s-edge -n <ns> describe certificate <name> | tail -30
    ```

2. **Check the `Order`** — one per issuance attempt. If in state `pending`, the challenge is still running. If `errored`, the message field explains what LE rejected.

    ```bash
    kubectl --context k3s-edge -n <ns> get order -o wide
    kubectl --context k3s-edge -n <ns> describe order <name>
    ```

3. **Check the `Challenge`** — where DNS-01-specific errors surface.

    ```bash
    kubectl --context k3s-edge -n <ns> get challenge -o wide
    kubectl --context k3s-edge -n <ns> describe challenge <name>
    ```

    Common failures:
    - `DNS record for "<host>" not yet propagated` — normal for the first ~60–120s. If it persists past 5 minutes, likely the Cloudflare token can't write TXT records for that zone (wrong permissions or wrong zone scope).
    - `secret "cloudflare-api-token" not found` — KSOPS didn't render the Secret. Check the argocd-repo-server sidecar logs and the cert-manager Application's sync status.
    - `unauthorized` from Cloudflare — token revoked or expired. Rotate per the section above.

4. **cert-manager controller logs** for anything the resource status didn't surface:

    ```bash
    kubectl --context k3s-edge -n cert-manager logs deploy/cert-manager --tail=100
    ```

5. **Verify DNS challenge propagation manually** if step 3 is inconclusive:

    ```bash
    dig +short TXT _acme-challenge.<host> @1.1.1.1
    ```

    Query Cloudflare's public resolver directly (`1.1.1.1`), not Pi-hole. LE checks propagation from multiple public resolvers; if `dig` at Cloudflare doesn't see the TXT record, LE won't either.

## Rate limits (LE prod)

Worth internalizing before you start iterating heavily:

- **50 certs per registered domain per week.** The wildcard cert counts as 1 (not one per SAN).
- **300 new orders per account per 3h.**
- **5 duplicate certs per week** — identical hostname sets. This is the one that bites during debugging: if you keep reissuing the same wildcard, you'll hit this before the 50/week.

If you're iterating, flip the `issuerRef` to `letsencrypt-staging` — those limits are effectively unlimited (30,000 certs/week).

## Related

- [ADR-0001](./adr/0001-internal-only-no-public-exposure.md) — why the lab has no public inbound (drives the DNS-01 choice)
- [ADR-0005](./adr/0005-secrets-sops-age.md) — SOPS + age (drives the encrypted-Secret-in-git choice)
- [ADR-0009](./adr/0009-letsencrypt-dns01-split-horizon.md) — the decision to use LE + DNS-01 (supersedes ADR-0006)
- [ADR-0006](./adr/0006-defer-real-tls-to-stepca-phase2.md) — historical; the step-ca route the LE decision replaced
- [`platform-layer.md`](./platform-layer.md) — what fans out what, and why
