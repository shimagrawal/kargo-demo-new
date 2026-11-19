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
3. **A promotion runs a series of steps.** The `scan` task checks the image for vulnerabilities, and the
   `deploy` task renders the Stage's manifests with the image digest, commits them to the
   `stage/<stage>` branch and syncs the Stage's Argo CD Application.
4. **Staging is verified after every promotion.** A smoke test checks that guestbook answers. When it
   fails, Kargo promotes the last verified Freight to staging again on its own.
5. **Prod is guarded.** A change-freeze window can close prod, and every prod promotion waits for a named
   approver and is recorded in the Akuity Platform audit log.

## Kargo features shown

| Feature | How it's used here | Where |
|---|---|---|
| **Warehouse and Freight** | Freight combines two artifacts: the image digest and the Git commit of `app/`, so a release is traceable from commit to production | `kargo/warehouse.yaml` |
| **PromotionTasks** | Reusable `scan` and `deploy` tasks shared by all three Stages | `kargo/scan-task.yaml`, `kargo/promotion-task.yaml` |
| **Custom promotion steps** | `trivy-image`, a `CustomPromotionStep` that runs Trivy in its own container, used as a security gate on every Stage | `kargo/scan-task.yaml` |
| **Risk exceptions as code** | Accepted vulnerabilities with a reason, a ticket and an expiry date; each change is a commit that becomes part of the Freight | `app/.trivyignore` |
| **`wait-for-approval`** | Every prod promotion pauses until the named approver approves it; a rejection stops it | `kargo/stages.yaml` (prod) |
| **`record-audit-event`** | Records the approval and the deployment in the Akuity Platform audit log, as `custom-approved` and `custom-deployed` | `kargo/stages.yaml` (prod) |
| **Promotion windows** | A `Deny` window freezes prod; it blocks every kind of promotion, including automatic ones and rollbacks | `kargo/project.yaml` |
| **Verification** | An `AnalysisTemplate` smoke-tests staging after each promotion | `kargo/smoke-test.yaml`, `kargo/stages.yaml` (staging) |
| **Automatic rollback** | When staging's verification fails, Kargo promotes the last verified Freight again | `kargo/project.yaml` |
| **Conditional steps** | Rollbacks skip the scan, since they promote Freight that already passed it | `kargo/scan-task.yaml` |
| **Auto-promotion** | New Freight is promoted to dev automatically | `kargo/project.yaml` |
| **Shards and agents** | Each Stage runs its promotions on the Kargo agent of its own cluster | `kargo/stages.yaml` |
| **Argo CD integration** | `argocd-update` syncs each Stage's Application; the Applications authorize their Stage with the `kargo.akuity.io/authorized-stage` annotation | `kargo/promotion-task.yaml`, `argocd/appset.yaml` |
| **SSO and RBAC** | Users log in through an OIDC provider (Okta). The `kargo-developers` group maps to the project's `kargo-promoter` role and can promote to every Stage, but only the named approver can approve prod | Kargo instance OIDC settings and project roles, `kargo/stages.yaml` (prod) |
| **Pipeline as code** | Argo CD syncs `kargo/` into the Kargo instance, so every pipeline change, including a freeze, goes through Git | `argocd/kargo-resources.yaml` |

Approvals are limited to one named approver (the `email` claim). To allow an SSO group instead, use
`claim: groups` with the group name, for example a release managers group.

## Repository structure

```text
.
├── app/                        # The application manifests (the Warehouse watches this folder)
│   ├── base/                   # guestbook Deployment and Service
│   ├── stages/
│   │   ├── dev/                # One Kustomize overlay per Stage
│   │   ├── staging/
│   │   └── prod/
│   └── .trivyignore            # Accepted vulnerabilities
│
├── kargo/                      # The Kargo pipeline, synced into Kargo by Argo CD
│   ├── project.yaml            # Project, promotion policies, promotion window
│   ├── warehouse.yaml
│   ├── stages.yaml
│   ├── scan-task.yaml
│   ├── promotion-task.yaml
│   └── smoke-test.yaml
│
└── argocd/                     # Argo CD Applications
    ├── appset.yaml             # One Application per Stage, each on its own cluster
    └── kargo-resources.yaml    # Syncs kargo/ into the Kargo instance
```

Kargo renders each Stage's manifests to its own branch, `stage/dev`, `stage/staging` and `stage/prod`,
which the Argo CD Applications deploy.

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
4. For the RBAC part, configure OIDC on the Kargo instance and map your groups to the project's roles,
   for example `kargo-developers` to `kargo-promoter`.
