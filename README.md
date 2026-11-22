# Kargo secure delivery demo

One commit goes from Git to production through Kargo: it is **scanned** for vulnerabilities,
**verified** in staging (and **rolled back** automatically if it fails), **approved** by a named person
for production, and **recorded** in the audit log. Production can be **frozen**. All of it is defined in
this repo.

## What is Kargo?

[Kargo](https://kargo.io) orchestrates **promotion**: moving a release from one environment to the next.
Argo CD keeps each cluster in sync with Git; Kargo decides **what** goes **where**, **when**, and **on
which conditions**.

## What This Demonstrates

How to combine **automated delivery with security and release controls**:

- Promote the same release across environments (dev → staging → prod).
- Promote release automatically to dev.
- Scan every release with Trivy, run as a Kargo **custom promotion step**.
- Smoke-test staging and roll back automatically on failure.
- Freeze prod, require a named approver, and audit every release.
- Use SSO and RBAC: developers promote, only approvers release to prod.

**Kargo orchestrates the promotion, Git is the source of truth, and Argo CD deploys to each cluster.**

## End-to-End Flow

```text
Commit to app/ (or new image)
   │
   ▼
Warehouse creates Freight (image digest + Git commit)
   │
   ▼
dev (automatic)
   └── Trivy scan → deploy
   │
   ▼
staging
   └── Trivy scan → deploy → smoke test   (fails → auto-rollback)
   │
   ▼
prod
   └── freeze check → Trivy scan → approval → deploy → audit
```

Each deploy renders the manifests, commits them to `stage/<env>`, and Argo CD syncs that environment's
cluster.

## Environments

| Stage | Deploys to (Argo CD cluster) | Runs on (Kargo agent) | What happens on promotion |
|---|---|---|---|
| **dev** | `kargo-quickstart` | `self-hosted` | automatic · security scan · deploy |
| **staging** | `cluster-uat` | `self-hosted-uat` | security scan · deploy · smoke test · auto-rollback |
| **prod** | `cluster-prod` | `self-hosted-prod` | freeze check · security scan · approval · deploy · audit events |

The Stage names are the environments. The cluster and agent names are the existing names of the
clusters they run on: for example, the **staging** Stage deploys to the cluster registered in Argo CD as
`cluster-uat`.

## Demo scenes

Each scene is a small edit, a commit and a push.

| Scene | Change | What happens |
|---|---|---|
| **Happy path** | Bump the `release` label in `app/base/deployment.yaml` | New Freight, dev promotes automatically and the scan passes. Promote to staging (smoke test passes), then prod: it waits for approval, you approve, and the audit log records it |
| **Security gate blocks** | Set an expiry in the past in `app/.trivyignore`, e.g. `CVE-2026-31789 exp:2026-11-01` | The scan blocks the promotion and lists the CVE with a link. Restore the date to unblock |
| **Bad release rolled back** | Set `targetPort: 8080` in `app/base/service.yaml` | Dev looks healthy, but the staging smoke test fails, the release can't reach prod, and staging rolls back by itself. Restore `3000` |
| **Change freeze** | Set the `change-freeze` window to `disabled: false` in `kargo/project.yaml` | Prod promotions are blocked until you set it back to `true` |
| **Access control** | Log in as a developer (Okta, `kargo-developers`) | They can promote to staging and request prod, but can't approve it |

Notes:

- The Trivy scan downloads a fresh vulnerability database on every run, so it takes from about 20
  seconds to a few minutes depending on the network of the cluster it runs on.
- Rollbacks skip the scan: they promote a release that already passed it.

## Repository structure

```text
.
├── app/                        # The application (the Warehouse watches this folder)
│   ├── base/                   # guestbook Deployment and Service
│   ├── stages/dev|staging|prod # One Kustomize overlay per Stage
│   └── .trivyignore            # Accepted vulnerabilities, with expiry dates
├── kargo/                      # The pipeline, synced into Kargo by Argo CD
│   ├── project.yaml            # Project, auto-promotion, auto-rollback, freeze window
│   ├── warehouse.yaml          # What makes a release
│   ├── stages.yaml             # dev, staging, prod (approval and audit events on prod)
│   ├── scan-task.yaml          # Security gate (trivy-image custom step)
│   ├── promotion-task.yaml     # Render, commit to stage/<stage>, sync Argo CD
│   └── smoke-test.yaml         # Staging verification
└── argocd/
    ├── appset.yaml             # One Argo CD Application per Stage
    └── kargo-resources.yaml    # Syncs kargo/ into Kargo
```

## Setup

You need:

- A Kargo instance on the Akuity Platform (v1.12 or later) with a **self-hosted** Kargo agent per
  cluster. The Trivy custom step and the smoke-test Job don't run on Akuity-managed agents.
- The `trivy-image` custom promotion step registered on the Kargo instance.
- An Argo CD instance with the three clusters, and the Kargo instance registered as `kargo-resources`.

Steps:

1. Create the Argo CD Applications; `kargo-resources` creates the pipeline in Kargo:

   ```shell
   akuity argocd apply --name <argo-cd-instance> -f argocd/appset.yaml -f argocd/kargo-resources.yaml
   ```

   Each Stage's Application shows a ComparisonError until its first promotion creates its branch.

2. Give Kargo write access to this repo, so it can push the `stage/*` branches (the CLI prompts for a
   GitHub token):

   ```shell
   kargo create repo-credentials --project=kargo-demo github --git \
     --repo-url=https://github.com/shimagrawal/kargo-demo-new.git --username=<github-user>
   ```

3. For access control, configure OIDC on the Kargo instance and map your groups to the project's roles,
   for example `kargo-developers` to `kargo-promoter`. The prod approver is set in `kargo/stages.yaml`
   (`claim: email`); use `claim: groups` to allow a group instead.
