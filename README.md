# argocd-repo

ArgoCD `Application` definitions for apps deployed to the local OpenShift
Local (CRC) cluster via GitOps.

## penpot-app (`apps/penpot-app.yaml`)

Deploys Penpot. Ties together:

- chart: [`release-penpot`](https://github.com/dragos1993/release-penpot)
- values: [`envirenment-penpot`](https://github.com/dragos1993/envirenment-penpot) (`values-dev.yaml`)

via ArgoCD's [multiple sources](https://argo-cd.readthedocs.io/en/stable/user-guide/multiple_sources/)
feature, auto-synced with pruning and self-heal, into the `penpot`
namespace.

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

### Apply

This cluster already has ArgoCD running (installed via the community
Argo CD Operator, namespace `argocd` — see the root-level install docs
for how to reach its UI). Apply the Application into that namespace:

```bash
oc apply -f apps/penpot-app.yaml -n argocd
```

Then watch it sync:

```bash
oc get application penpot-app -n argocd -w
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
