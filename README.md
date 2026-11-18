# Kargo security demo

A small Kargo pipeline that takes a commit to production with security gates, approvals and an audit
trail.

```
Warehouse (guestbook image + this repo's app/ commits)
  └─► dev       auto-promotion · trivy scan
       └─► staging   trivy scan · smoke test · auto-rollback
            └─► prod      trivy scan · change-freeze window · wait-for-approval (named approver) · audit events
```

Each Stage renders `app/stages/<stage>` with the Freight's image digest, pushes it to the `stage/<stage>`
branch and syncs the `kargo-demo-<stage>` Argo CD Application.

| Path | What it is |
|---|---|
| `app/` | Kustomize base and one overlay per Stage |
| `kargo/` | Project, ProjectConfig (auto-promotion, promotion window), Warehouse, PromotionTask, Stages |
| `argocd/appset.yaml` | ApplicationSet with one Application per Stage, each on its own cluster |
| `argocd/kargo-resources.yaml` | Application that syncs `kargo/` into the Kargo instance, so the pipeline itself is managed through Git |

## Setup

### 1. Push this repo to GitHub

The repo must be public, or Argo CD needs read credentials as well.

```shell
cd kargo-demo-new
git init -b main
git add .
git commit -m "Kargo security demo"
gh repo create shimagrawal/kargo-demo-new --public --source . --push
```

If you use another name, replace `https://github.com/shimagrawal/kargo-demo-new.git` in
`kargo/` and `argocd/`.

### 2. Kargo resources

```shell
kargo login https://gaq379139gqc393z.kargo.akuity.cloud --sso
kargo apply -f kargo/project.yaml
kargo apply -f kargo/

# Git credentials for pushing the stage branches: a GitHub token with write access to this repo.
# Leave out --password and the CLI prompts for it.
kargo create repo-credentials --project=kargo-demo github \
  --git --repo-url=https://github.com/shimagrawal/kargo-demo-new.git \
  --username=shimagrawal
```

Only the user in the prod Stage's `approvers` rule (an `email` claim) can approve prod promotions.
To approve by SSO group instead, use `claim: groups` with the group name.

The Stages use the `trivy-image` custom step from the kargo-custom-steps repo. It must be registered on
the instance (`kargo/custom-steps/supply-chain/trivy-image/trivy-image.yaml` there).

### 3. Argo CD ApplicationSet

`argocd/appset.yaml` creates one Application per Stage, each on its own cluster: dev on
`kargo-quickstart`, staging on `cluster-uat` and prod on `cluster-prod`. Apply it to the Akuity Argo CD
instance the Kargo agent is connected to, with the Akuity CLI:

```shell
akuity login
akuity argocd apply --name <argo-cd-instance-name> -f argocd/appset.yaml -f argocd/kargo-resources.yaml
```

`kargo-demo-kargo-resources` syncs `kargo/` into the Kargo instance (registered in Argo CD as the
`kargo-resources` cluster). After that, change the pipeline through Git rather than `kargo apply`.

The Applications show a ComparisonError until the first promotion creates their `stage/<stage>` branch.

### 4. First run

The Warehouse discovers `v0.0.2` and the latest commit, and dev promotes itself. Promote to staging
and prod once before the demo to check the whole path, then promote `v0.0.1` back to every Stage so
the demo can show `v0.0.2` going through.

## Demo script (about 25 minutes)

1. **Pipeline (2 min).** Show the graph, the Warehouse and the Freight timeline. Open a Freight: it
   holds the image digest and the source commit.
2. **Security gate (5 min).** Before the demo, remove the entries from `app/.trivyignore` and push. dev
   blocks the image: the `scan` task lists the CRITICAL CVEs. Then add the entries back with a reason,
   ticket and expiry date, and push. The risk exception is a reviewed commit: it creates new Freight, and
   dev now passes. The Freight shows both the image digest and that commit.
3. **Staging (2 min).** Promote to staging. The scan runs again, on the staging cluster's Kargo agent,
   and the `smoke-test` verification checks that guestbook answers. The Freight is now verified.
4. **Auto-rollback (4 min).** Push a bad change: set `targetPort: 8080` in `app/base/service.yaml` (guestbook
   listens on 3000). The new Freight passes dev, which has no verification. Promote it to staging: the
   smoke test fails, and Kargo promotes the last verified Freight to staging again on its own. Revert the
   commit afterwards.
5. **Change freeze (3 min).** In `kargo/project.yaml`, set `change-freeze` to `disabled: false` and push:
   the freeze is a reviewed commit, synced into Kargo by Argo CD. Promoting to prod is refused with the
   window's reason. Revert the commit to lift it.
6. **Approval (5 min).** Promote to prod. The promotion waits at `wait-for-approval`, which only the
   named approver can answer; with SSO groups this would be a release-managers group. Approve with a
   message, e.g. `CHG-1234 approved`. The deployment runs.
7. **Traceability and audit (4 min).**
   - The promotion's steps show who approved, the commit Kargo pushed to `stage/prod` and the Argo CD
     sync.
   - The `stage/prod` commit message names the source commit, image digest and Freight.
   - In the Akuity portal, the organization Audit Log shows the promotion and the `custom-approved` and
     `custom-deployed` entries.
   - The Kargo instance's Logs tab shows the control plane logs.
