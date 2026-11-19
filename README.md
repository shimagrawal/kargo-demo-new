# Kargo secure delivery example

A Kargo pipeline that takes a change from a Git commit to production with a security gate, automatic
verification and rollback, a change freeze, a release approval and an audit trail. Everything that
defines the pipeline lives in this repo.

## How it works

```
Warehouse "guestbook"
  image ghcr.io/akuity/guestbook + commits under app/ in this repo
        │
        ▼
  dev        cluster kargo-quickstart   auto-promotion · security scan · deploy
        │
        ▼
  staging    cluster cluster-uat        security scan · deploy · smoke test · auto-rollback
        │
        ▼
  prod       cluster cluster-prod       change-freeze window · security scan · approval · deploy · audit events
```

1. **A release is a piece of Freight.** The Warehouse watches the guestbook image and the `app/` folder
   of this repo. When either changes, it creates Freight that pins both: the image digest and the Git
   commit. Every Stage deploys exactly that combination.
2. **Each Stage runs on its own cluster.** A Stage's promotion runs on the Kargo agent of the cluster it
   deploys to (its `shard`), and Argo CD deploys to that same cluster.
3. **A promotion runs a series of steps:**
   - **Security scan** (`scan` task): checks out the Freight's commit and scans the image with Trivy.
     Fixable CRITICAL vulnerabilities block the promotion, unless they're accepted in `app/.trivyignore`.
   - **Deploy** (`deploy` task): renders `app/stages/<stage>` with the image digest, commits it to the
     `stage/<stage>` branch and syncs the `kargo-demo-<stage>` Argo CD Application. The commit message
     names the source commit, the image digest and the Freight.
4. **Staging is verified after every promotion.** A smoke test checks that guestbook answers through its
   Service. When it fails, Kargo promotes the last verified Freight to staging again on its own.
5. **Prod is guarded.** A change-freeze window can close prod. Every prod promotion waits for a named
   approver, and records the approval and the deployment in the Akuity Platform audit log.

## Controls

| Control | What it does | Where it's defined |
|---|---|---|
| Security gate | Blocks images with fixable CRITICAL vulnerabilities on every Stage | `kargo/scan-task.yaml`, using the `trivy-image` custom promotion step |
| Risk exceptions | Accepts specific vulnerabilities with a reason, a ticket and an expiry date; each exception is a reviewed commit that becomes part of the Freight | `app/.trivyignore` |
| Verification | Smoke test against the deployed app after each staging promotion | `kargo/smoke-test.yaml`, `verification` in `kargo/stages.yaml` |
| Automatic rollback | Promotes the last verified Freight again when staging's verification fails | `autoRollback` in `kargo/project.yaml` |
| Change freeze | A promotion window that blocks all promotions to prod, including automatic ones and rollbacks, while it's active | `promotionWindows` in `kargo/project.yaml` |
| Release approval | Pauses every prod promotion until a named approver approves it; a rejection stops it | `wait-for-approval` in `kargo/stages.yaml` |
| Audit trail | Records who approved and what was deployed in the Akuity Platform audit log, as `custom-approved` and `custom-deployed` | `record-audit-event` in `kargo/stages.yaml` |
| Traceability | Freight pins the image digest and commit; every stage branch commit names its source | Warehouse and `deploy` task |
| Pipeline as code | Argo CD syncs `kargo/` into Kargo, so every pipeline change, including a freeze, goes through Git | `argocd/kargo-resources.yaml` |

Approvals are limited to one named approver (the `email` claim). To allow an SSO group instead, use
`claim: groups` with the group name, for example a release managers group.

## Repository layout

| Path | What it is |
|---|---|
| `app/base/` | guestbook Deployment and Service |
| `app/stages/<stage>/` | One Kustomize overlay per Stage; prod runs two replicas |
| `app/.trivyignore` | Accepted vulnerabilities |
| `kargo/project.yaml` | Project and ProjectConfig: auto-promotion for dev, auto-rollback for staging, the change-freeze window |
| `kargo/warehouse.yaml` | Warehouse: the image and `app/` commits that make up a release |
| `kargo/scan-task.yaml` | `scan` PromotionTask: the security gate |
| `kargo/promotion-task.yaml` | `deploy` PromotionTask: render, commit to the stage branch, sync Argo CD |
| `kargo/stages.yaml` | The dev, staging and prod Stages |
| `kargo/smoke-test.yaml` | AnalysisTemplate used to verify staging |
| `argocd/appset.yaml` | ApplicationSet with one Application per Stage, each on its own cluster |
| `argocd/kargo-resources.yaml` | Application that syncs `kargo/` into the Kargo instance |

## Setup

Requirements:

- A Kargo instance on the Akuity Platform, v1.12 or later, with a self-hosted Kargo agent per cluster.
- An Argo CD instance with the three clusters, and the Kargo instance registered as the
  `kargo-resources` cluster.
- The `trivy-image` custom promotion step registered on the Kargo instance.

Steps:

1. Create the pipeline, either with `kargo apply -f kargo/` or through the `kargo-resources` Application.
2. Add Git credentials so Kargo can push the stage branches. Use a GitHub token with write access to
   this repo; the CLI prompts for it when `--password` is left out:

   ```shell
   kargo create repo-credentials --project=kargo-demo github --git \
     --repo-url=https://github.com/shimagrawal/kargo-demo-new.git --username=<github-user>
   ```

3. Create the Argo CD Applications:

   ```shell
   akuity argocd apply --name <argo-cd-instance> -f argocd/appset.yaml -f argocd/kargo-resources.yaml
   ```

   Each Application shows a ComparisonError until the first promotion creates its `stage/<stage>`
   branch.
