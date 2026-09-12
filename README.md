# argocd-repo

ArgoCD `Application` definitions for apps deployed to the local OpenShift
Local (CRC) cluster via GitOps.

## penpot-app (`apps/penpot-app.yaml`)

Deploys Penpot. Ties together:

- chart: [`release-penpot`](https://github.com/dragos1993/release-penpot)
- values: [`envirenment-penpot`](https://github.com/dragos1993/envirenment-penpot) (`values-dev.yaml`)

via ArgoCD's [multiple sources](https://argo-cd.readthedocs.io/en/stable/user-guide/multiple_sources/)
feature, auto-synced with pruning and self-heal, into the `penpot`
namespace. Of the 6 workloads this creates, only 2 carry a PVC —
`penpot-postgres` and `penpot-minio` (see `release-penpot/README.md`)
— so `prune: true` deleting/recreating a Deployment here is harmless,
but never delete the `penpot` namespace itself expecting data to
survive: that takes the PVCs with it (details in
[`release-penpot/INSTALL.md`](https://github.com/dragos1993/release-penpot/blob/main/INSTALL.md)).

### Prerequisites (manual, out-of-band — not managed by this repo)

1. The `penpot` namespace must exist:
   ```bash
   oc create ns penpot
   ```
2. A `penpot-secrets` Secret must exist in that namespace, holding
   `postgres-password`, `minio-root-user`, `minio-root-password`, and
   `penpot-secret-key`. ArgoCD/Helm never generate or store these —
   plaintext credentials should not live in a git-tracked chart or
   values repo. See `release-penpot/README.md` for the exact
   `oc create secret` command.
3. **Specific to how ArgoCD happens to be installed on this cluster**
   (community Argo CD Operator, not something CRC itself requires): its
   `in-cluster` registration only manages the `argocd` namespace by
   default (see the `namespaces` field on the `argocd-default-cluster-config`
   Secret in namespace `argocd`) — anything targeting another namespace
   fails with `Failed to load live state: namespace "<ns>" for Route
   "<name>" is not managed`, even though the `default` AppProject itself
   allows any destination. Opt the `penpot` namespace in by labeling it;
   the operator updates that Secret's `namespaces` list automatically:
   ```bash
   oc label namespace penpot argocd.argoproj.io/managed-by=argocd
   ```
   (One-time, per namespace. Confirmed working on this cluster; not
   needed at all with the OpenShift GitOps operator instead, which
   manages cluster-wide by default.)

4. `apps/penpot-app.yaml` pins `helm.releaseName: penpot` on its first
   source. This must match whatever release name Penpot was ever
   installed under directly with `helm install` on this namespace (see
   `release-penpot/README.md`) — ArgoCD otherwise defaults the Helm
   release name to the *Application's own name* (`penpot-app`), which
   changes the rendered `app.kubernetes.io/instance` selector label on
   every Deployment. Deployment selectors are immutable, so a mismatch
   here makes every sync fail with `field is immutable` on every
   Deployment. Full story in
   [`release-penpot/INSTALL.md`](https://github.com/dragos1993/release-penpot/blob/main/INSTALL.md).

### Apply

This cluster already has ArgoCD running (installed via the community
Argo CD Operator, namespace `argocd` — see
[`release-penpot/INSTALL.md`](https://github.com/dragos1993/release-penpot/blob/main/INSTALL.md)
for how to reach its UI). Apply the Application into that namespace:

```bash
oc apply -f apps/penpot-app.yaml -n argocd
```

Then watch it sync:

```bash
oc get application penpot-app -n argocd -w
```

### Verifying the sync

```bash
oc get application penpot-app -n argocd
```

Expect `SYNC STATUS: Synced` and `HEALTH STATUS: Healthy`. If it's
stuck on anything else, check the per-resource breakdown — this is
usually faster than reading through controller logs:

```bash
oc get application penpot-app -n argocd -o jsonpath='{range .status.resources[*]}{.kind}{" "}{.name}{" "}{.status}{" "}{.health.status}{"\n"}{end}'
```

Every row should read `Synced Healthy` (Jobs read `Synced Succeeded`
briefly, then disappear — the bucket-creation Job self-deletes on
success via `helm.sh/hook-delete-policy`). If a sync is actually
failing (not just slow), the operation's error message is here:

```bash
oc get application penpot-app -n argocd -o jsonpath='{.status.operationState.phase}{"\n"}{.status.operationState.message}{"\n"}'
```

To force a fresh comparison against the git repos right now (useful
after pushing a change, rather than waiting for ArgoCD's normal poll
interval):

```bash
oc annotate application penpot-app -n argocd argocd.argoproj.io/refresh=hard --overwrite
```

## valkey-app (`apps/valkey-app.yaml`)

Deploys the Valkey demo stack. Ties together:

- chart: [`valkey-repo-helm`](https://github.com/dragos1993/valkey-repo-helm)
- values: [`valkey-repo-values`](https://github.com/dragos1993/valkey-repo-values)

### Status: not yet applied

This app was built against a different cluster (`qxy2756-dev` on the Red
Hat Developer Sandbox) that didn't have the OpenShift GitOps operator
installed and didn't allow installing cluster operators. In the
meantime, `valkey-repo-helm` is deployed directly with
`helm install`/`helm upgrade` there — see that repo's README.

If applying it to *this* cluster instead, adjust `namespace: openshift-gitops`
in `apps/valkey-app.yaml` to `argocd` (this cluster's ArgoCD instance
namespace) and its `destination.namespace` as needed, then:

```bash
oc apply -f apps/valkey-app.yaml -n argocd
```

If the destination namespace needs the Valkey image pull secret
(e.g. after switching to `registry.redhat.io`), create it out-of-band
first — ArgoCD should not manage real registry credentials as
plaintext git-tracked Secret manifests. See `valkey-repo-helm`'s README
for the recommended `--set-file` approach, or use a secrets operator
(e.g. External Secrets, Sealed Secrets) if managing it via GitOps is a
requirement.
