# Kargo secure delivery demo

One commit goes from Git to production through Kargo: it is **scanned** for vulnerabilities,
**verified** in staging (and **rolled back** automatically if it fails), **approved** by a named person
for production, and **recorded** in the audit log. Production can be **frozen**. All of it is defined in
this repo.

## What this demo shows

| | Feature | What you see |
|---|---|---|
| 📦 | **Traceable releases** | A release (Freight) pins the image digest and the Git commit together, and that exact pair moves through every environment |
| 🔒 | **Security gate** | Every promotion scans the image with Trivy and blocks on fixable CRITICAL vulnerabilities |
| 📝 | **Risk exceptions as code** | Accepted vulnerabilities live in `app/.trivyignore` with a reason, a ticket and an expiry date. When one expires, the gate blocks again |
| ✅ | **Verification** | After each promotion to staging, a smoke test checks the app answers. A failed release can't reach prod |
| ↩️ | **Automatic rollback** | When the staging smoke test fails, Kargo puts the last good release back by itself |
| 🙋 | **Release approval** | Developers can promote, but every prod promotion waits for a named approver |
| 🧾 | **Audit trail** | Approvals and deployments are recorded in the Akuity Platform audit log |
| ❄️ | **Change freeze** | A promotion window can close prod. It blocks every promotion, including automatic ones |
| 👥 | **SSO and RBAC** | Users log in with Okta. The `kargo-developers` group can promote; only the approver can approve prod |
| 🧬 | **Pipeline as code** | Argo CD syncs `kargo/` into Kargo, so every pipeline change, even a freeze, is a reviewed commit |

## Environments

| Stage | Deploys to (Argo CD cluster) | Runs on (Kargo agent) | What happens on promotion |
|---|---|---|---|
| **dev** | `kargo-quickstart` | `self-hosted` | automatic · security scan · deploy |
| **staging** | `cluster-uat` | `self-hosted-uat` | security scan · deploy · smoke test · auto-rollback |
| **prod** | `cluster-prod` | `self-hosted-prod` | freeze check · security scan · approval · deploy · audit events |

The Stage names are the environments. The cluster and agent names are the existing names of the
clusters they run on: for example, the **staging** Stage deploys to the cluster registered in Argo CD as
`cluster-uat`.

## How it works

```
commit to app/  or  new guestbook image
            │
            ▼
Warehouse ──► Freight (image digest + Git commit)
            │
            ▼
  dev ──► staging ──► prod
  each promotion:  scan ──► render manifests ──► commit to stage/<stage> ──► Argo CD sync
```

1. The **Warehouse** watches the guestbook image and the `app/` folder, and creates Freight when either
   changes.
2. Each **promotion** runs two tasks: `scan` (Trivy) and `deploy` (render the Stage's Kustomize overlay
   with the image digest, commit it to the `stage/<stage>` branch, sync the Stage's Argo CD Application).
3. Each Stage's promotion runs on the Kargo agent of the cluster it deploys to.
4. **Staging** is verified after every promotion; a failure triggers the automatic rollback.
5. **Prod** checks the freeze window, waits for approval and records audit events.

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
