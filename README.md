# argocd-repo

ArgoCD `Application` definitions for the Valkey demo stack. This ties
together:

- chart: [`valkey-repo-helm`](https://github.com/dragos1993/valkey-repo-helm)
- values: [`valkey-repo-values`](https://github.com/dragos1993/valkey-repo-values)

via ArgoCD's [multiple sources](https://argo-cd.readthedocs.io/en/stable/user-guide/multiple_sources/)
feature (`apps/valkey-app.yaml`), auto-synced with pruning and self-heal.

## Status: not yet applied

This sandbox cluster (`qxy2756-dev` on the Red Hat Developer Sandbox)
doesn't have the OpenShift GitOps operator installed, and this account
doesn't have permission to install cluster operators, so there is no
`openshift-gitops` namespace / ArgoCD instance to apply this to yet.

In the meantime, `valkey-repo-helm` is deployed directly with
`helm install`/`helm upgrade` — see that repo's README.

## Once ArgoCD is available

1. Get the OpenShift GitOps operator installed (needs cluster-admin, or a
   sandbox/cluster that already has it — e.g. via
   Administrator console → OperatorHub → "Red Hat OpenShift GitOps").
2. Apply this Application:

   ```bash
   oc apply -f apps/valkey-app.yaml -n openshift-gitops
   ```

3. If the destination namespace needs the Valkey image pull secret
   (e.g. after switching to `registry.redhat.io`), create it out-of-band
   first — ArgoCD should not manage real registry credentials as
   plaintext git-tracked Secret manifests. See `valkey-repo-helm`'s README
   for the recommended `--set-file` approach, or use a secrets operator
   (e.g. External Secrets, Sealed Secrets) if managing it via GitOps is a
   requirement.
