# Ababiel preview environments — runbook

Per-pull-request environments of [pal-odoo](https://github.com/MohanadAbugharbia/pal-odoo)
on the homelab. Put the `preview` label on a pal-odoo PR and a few minutes
later `https://ababiel-pr-<n>.tailc5dc8e.ts.net` (tailnet only) serves that
PR's Ababiel with demo data in English and Arabic. Every push upgrades the
environment in place; removing the label or closing the PR tears everything
down, database included.

## How the pieces fit

| Repo | Piece | Role |
|---|---|---|
| pal-odoo | `.github/workflows/build-latest.yaml` job `build-preview` | Builds `ghcr.io/mohanadabugharbia/pal-odoo:sha-<sha>` (+ `pr-<n>`) for labelled PRs and posts the sticky `odoo-preview` comment |
| pal-odoo | `deploy/preview/` | Kustomize base: one `OdooDeployment` (`ababiel`) and one Tailscale `Ingress`; rewritten per PR by the ApplicationSet |
| pal-odoo | `.github/workflows/preview-cleanup.yaml` | Deletes the PR's GHCR tags on close/unlabel |
| homelab | `argo-services/ababiel-preview/` | Namespace, `ResourceQuota`, `LimitRange`, `NetworkPolicy`, the two sealed secrets and the `ApplicationSet` |
| homelab | `argo-services/odoo-operator/` | The operator, pinned to 0.3.0 |
| homelab | `argo-services/shared/pg_cluster.yaml` | `shared-pg`, where the `pal_odoo` role creates one database per PR |
| ArgoCD Helm values (outside git) | `configs.cm`, `configs.repositories`, `extraObjects` | Health check for `OdooDeployment`, the pal-odoo deploy key, the generator token and the `ababiel-preview` AppProject |
| odoo-operator ≥ 0.3.0 | `OdooDeployment` | Creates/drops the database (only its own), runs init and upgrade Jobs, owns the pods |

Names for PR 12: Application `ababiel-pr-12`; OdooDeployment, Deployment and
Ingress `ababiel-pr-12`; Services `ababiel-pr-12-http` / `-poll`; database
`pal_odoo_pr_12`; URL `https://ababiel-pr-12.tailc5dc8e.ts.net`.

Teardown chain: label removed or PR closed → the generator drops the
Application → its resources finalizer prunes the `OdooDeployment` → the
operator's `odoo.abugharbia.com/database` finalizer drops `pal_odoo_pr_12`
(it only drops databases whose comment is `odoo-operator:ababiel-preview/ababiel-pr-12`)
→ the owned PVC and the Ingress are garbage collected.

## One-time prerequisites (owner)

Do these **before merging** the homelab PR that adds `argo-services/ababiel-preview/`.

### 1. Release odoo-operator 0.3.0

Merge the operator PR, dispatch its `Release` workflow with `minor`, and check
that the `install.yaml` release asset contains `spec.upgrade`:

```bash
curl -sL https://github.com/MohanadAbugharbia/odoo-operator/releases/download/0.3.0/install.yaml | grep -c 'upgrade:'
```

`argo-services/odoo-operator/kustomization.yaml` pins that URL; ArgoCD cannot
build the odoo-operator Application until the asset exists.

### 2. GitHub

- Create the label `preview` on pal-odoo.
- Add a **read-only deploy key** on pal-odoo for ArgoCD (it clones
  `git@github.com:MohanadAbugharbia/pal-odoo.git` at the PR head sha).
- Create a fine-grained PAT scoped to pal-odoo with **Pull requests: Read** and
  **Metadata: Read** for the pull-request generator.
- Create a PAT with `read:packages` for pulling the private
  `ghcr.io/mohanadabugharbia/pal-odoo` package (use a classic PAT if GHCR
  rejects the fine-grained one).
- On the `pal-odoo` package page, "Manage Actions access": grant the
  `pal-odoo` repository **Admin**, so `preview-cleanup` can delete versions.
  If `GITHUB_TOKEN` still gets 403, put a fine-grained PAT with
  `delete:packages` in the pal-odoo secret `GHCR_ADMIN_TOKEN`.

### 3. ArgoCD Helm values

ArgoCD is installed with
`helm upgrade --install --namespace argo --values helm-services/argocd/values.yaml --repo https://argoproj.github.io/argo-helm --version 9.4.17 argocd argo-cd`
(chart 9.4.17, ArgoCD v3.3.6). Anything patched into `argocd-cm` by hand is
reverted on the next upgrade, so all ArgoCD-side configuration goes into that
values file. Merge the snippet below into it (the existing keys —
`accounts.mabugharbia`, `kustomize.buildOptions`, the RBAC, notifications and
the server Ingress — stay untouched) and re-run the same `helm upgrade`.

```yaml
configs:
  cm:
    # existing keys stay (accounts.mabugharbia, kustomize.buildOptions)
    resource.customizations.health.odoo.abugharbia.com_OdooDeployment: |
      hs = {}
      if obj.status == nil then hs.status = "Progressing"; hs.message = "Waiting for the operator"; return hs end
      if obj.status.observedGeneration ~= nil and obj.metadata.generation ~= nil
         and obj.status.observedGeneration < obj.metadata.generation then
        hs.status = "Progressing"; hs.message = "Waiting for generation " .. obj.metadata.generation; return hs
      end
      if obj.status.conditions ~= nil then
        for _, c in ipairs(obj.status.conditions) do
          if c.type == "Degraded" and c.status == "True" then
            hs.status = "Degraded"; hs.message = c.reason .. ": " .. (c.message or ""); return hs
          end
        end
        for _, c in ipairs(obj.status.conditions) do
          if c.type == "Ready" then
            if c.status == "True" and obj.status.phase == "Running" then hs.status = "Healthy"; hs.message = c.reason; return hs end
            hs.status = "Progressing"; hs.message = (obj.status.phase or "") .. ": " .. c.reason; return hs
          end
        end
      end
      if obj.status.phase == "Failed" then hs.status = "Degraded"; hs.message = "Phase Failed"; return hs end
      hs.status = "Progressing"; hs.message = obj.status.phase or "Pending"; return hs
  repositories:
    pal-odoo:
      url: git@github.com:MohanadAbugharbia/pal-odoo.git
      type: git
      sshPrivateKey: |
        -----BEGIN OPENSSH PRIVATE KEY-----   # read-only deploy key on pal-odoo
        ...
extraObjects:
  - apiVersion: v1
    kind: Secret
    metadata: { name: ababiel-preview-github-token, namespace: argo }
    stringData: { token: github_pat_... }      # fine-grained PAT: pal-odoo Pull requests Read + Metadata Read
  - apiVersion: argoproj.io/v1alpha1
    kind: AppProject
    metadata: { name: ababiel-preview, namespace: argo }
    spec:
      description: Ababiel PR preview environments (ApplicationSet ababiel-preview)
      sourceRepos:
        - git@github.com:MohanadAbugharbia/pal-odoo.git
        - git@github.com:MohanadAbugharbia/homelab.git
      destinations:
        - { server: https://kubernetes.default.svc, namespace: ababiel-preview }
      clusterResourceWhitelist: []
      namespaceResourceWhitelist:
        - { group: odoo.abugharbia.com, kind: OdooDeployment }
        - { group: networking.k8s.io, kind: Ingress }
```

The values file already keeps credentials in plaintext (the notifier SMTP
password), so the deploy key and PAT follow the same convention. If you would
rather not, create the two Secrets once with `kubectl -n argo create secret`
(the repository Secret needs the label `argocd.argoproj.io/secret-type: repository`)
and put only `configs.cm` and the AppProject into the values.

Check: `argocd proj get ababiel-preview` lists the two source repos and the
`ababiel-preview` destination.

### 4. Seal the secrets

`argo-services/ababiel-preview/encrypted-secrets.yaml` ships with
`PASTE_KUBESEAL_OUTPUT` placeholders. Replace the two documents with the
output of:

```bash
# the password the shared copy unseals to
PW="$(kubectl -n shared get secret pal-odoo-db-cred -o jsonpath='{.data.password}' | base64 -d)"
kubectl create secret generic pal-odoo-db-cred -n ababiel-preview \
  --from-literal=username=pal_odoo --from-literal=password="$PW" --dry-run=client -o yaml \
  | kubeseal --controller-namespace sealed-secrets --controller-name sealed-secrets -o yaml

kubectl create secret docker-registry ghcr-pull -n ababiel-preview \
  --docker-server=ghcr.io --docker-username=MohanadAbugharbia --docker-password="$GHCR_PAT" \
  --dry-run=client -o yaml \
  | kubeseal --controller-namespace sealed-secrets --controller-name sealed-secrets -o yaml
```

Keep `metadata.namespace: ababiel-preview` and the `template` block from the
kubeseal output (the docker-registry one carries `type: kubernetes.io/dockerconfigjson`).
The sealed-secrets controller will not unseal a document whose namespace or
name differs from what it was sealed for.

### 5. Tailscale and network policy

- Confirm the tailnet ACLs allow new `ababiel-pr-*` devices (one per environment).
- `argo-services/ababiel-preview/networkpolicy.yaml` admits ingress on 8069
  from the proxy pods the Tailscale operator creates (selected by the
  `tailscale.com/managed: "true"` label in any namespace, and the `tailscale`
  namespace as a whole). If the operator runs elsewhere and labels differ,
  adjust the selector; if a preview pod never becomes Ready although its
  `kubectl logs` look healthy, check the policy first (`kubectl -n ababiel-preview describe networkpolicy`).
  k3s enforces NetworkPolicy by default; a server started with
  `--disable-network-policy` silently ignores it.

## Rollout and verification

1. Merge the homelab PR (operator on 0.3.0, `shared-pg` at 10Gi, the new
   directory with the sealed secrets pasted in).
2. Register the Application once, as for every other service directory (the
   `application.yaml` files are not part of the kustomize builds):

   ```bash
   kubectl apply -f argo-services/ababiel-preview/application.yaml
   ```

3. Check the pieces:

   ```bash
   kubectl -n odoo-operator-system rollout status deploy/odoo-operator-controller-manager
   kubectl get crd odoodeployments.odoo.abugharbia.com -o jsonpath='{.spec.versions[0].schema.openAPIV3Schema.properties.spec.properties.upgrade.type}'   # object
   kubectl -n shared get cluster shared-pg -o jsonpath='{.spec.storage.size}'     # 10Gi
   kubectl -n ababiel-preview get secret pal-odoo-db-cred ghcr-pull                # both unsealed
   kubectl -n ababiel-preview get resourcequota,limitrange,networkpolicy
   kubectl -n argo get applicationset ababiel-preview -o jsonpath='{.status.conditions}'   # no errors
   ```

4. End to end: label a pal-odoo PR `preview` → `build-preview` green and the
   sticky comment posted → Application `ababiel-pr-<n>` appears within ~60 s →
   `OdooDeployment` goes `Pending` → `Initializing` (`status.database.provisionedBy: operator`)
   → `kubectl -n ababiel-preview logs job/ababiel-pr-<n>-init` ends in
   `Modules loaded` → `Running` / `Ready` → the URL serves the login page
   (`admin` / `admin`; demo users `device_nab`, `device_ram`, `hub_manager`).
   Push a commit → the Deployment scales to 0 → `ababiel-pr-<n>-upgrade-<hash>`
   Job → back up with `status.appliedImage` on the new tag.
   Remove the label → Application, CR, Job, PVC and Ingress are gone and
   `\l` on `shared-pg` no longer lists `pal_odoo_pr_<n>`.
5. Negative test: create a database `keepme` on `shared-pg`, apply a CR with
   `spec.database.name: keepme` and `deletionPolicy: Delete` in
   `ababiel-preview` → `provisionedBy: external`; delete the CR → `keepme`
   still exists.
6. Capacity: a fourth labelled PR sits `Pending` with `Degraded: QuotaExceeded`;
   unlabel another one and it proceeds.

## Operations

```bash
# What is running
kubectl -n ababiel-preview get odoodeployment            # PHASE / READY / IMAGE / DATABASE columns
kubectl -n argo get applications -l app.kubernetes.io/part-of=ababiel-preview
kubectl -n ababiel-preview describe odoodeployment ababiel-pr-12   # conditions and events

# Why is it not Ready
kubectl -n ababiel-preview get jobs
kubectl -n ababiel-preview logs job/ababiel-pr-12-init     # or job/ababiel-pr-12-upgrade-<hash>
kubectl -n ababiel-preview get events --sort-by=.lastTimestamp | tail -20

# Retry a failed init/upgrade Job (the operator recreates a deleted Job)
kubectl -n ababiel-preview delete job ababiel-pr-12-init

# Orphaned databases: every operator-created database carries a comment
kubectl -n shared exec -it shared-pg-1 -- psql -U postgres -c '\l+' | grep odoo-operator
# drop one by hand only when no OdooDeployment references it any more
kubectl -n shared exec -it shared-pg-1 -- psql -U postgres -c 'DROP DATABASE pal_odoo_pr_12 WITH (FORCE)'

# Manual teardown when the label route is not available
kubectl -n argo delete application ababiel-pr-12       # the finalizer prunes the CR; the operator drops the DB

# Capacity
kubectl -n ababiel-preview describe resourcequota ababiel-preview
```

`shared-pg` keeps the default `max_connections` (100). Each preview uses at
most `db_maxconn: 16` connections; if `too many connections` shows up in the
Odoo logs, raise `postgresql.parameters.max_connections` in
`argo-services/shared/pg_cluster.yaml` (CNPG restarts the instance for it).

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Generated Application in error: `project ababiel-preview does not exist` | Helm values not applied | Step 3 above, then `helm upgrade` |
| ApplicationSet condition `ErrorOccurred` mentioning the token | `ababiel-preview-github-token` missing or PAT without Pull requests: Read | Step 3 |
| Application `ComparisonError` on `deploy/preview` | PR branched before `deploy/preview` existed, or the deploy key cannot read pal-odoo | Rebase the PR; check `argocd repo list` |
| Pod `ImagePullBackOff` | `ghcr-pull` not unsealed, PAT expired, or the image for that sha was not built yet (`build-preview` still running or failed) | Step 4; check the PR's checks |
| `OdooDeployment` `Pending`, `Degraded: DatabaseConnectionFailed` | `pal-odoo-db-cred` not unsealed in `ababiel-preview` | Step 4 |
| `Pending`, `Degraded: QuotaExceeded` | Three environments already running | Unlabel one, or raise the quota |
| `Failed`, `Degraded: InitJobFailed` | A module failed to install on the PR's code | `kubectl logs job/...`; fix the PR and push, or delete the Job to retry |
| Pods never Ready, logs fine | NetworkPolicy blocks the probes or the ingress proxy | Step 5 |
| Database left behind after teardown | Drop refused (comment mismatch) or connection failure at deletion time; the operator emits a `DatabaseDropRefused` / `DatabaseNotDropped` event | Inspect `\l+`, drop by hand |
