# Employee Task Tracker — GitOps Repository

The single source of truth for **what's actually deployed**, for one application, in two environments. ArgoCD continuously reconciles each EKS cluster to match whatever is committed here — nothing more, nothing less.

Part of a 3-repo project. The other two: [employee-task-app](https://github.com/rashmiranjandevops/employee-task-app) (application source, CI/CD, and the Helm **chart**) and [employee-task-infra](https://github.com/rashmiranjandevops/employee-task-infra) (Terraform + Ansible). Full docs, setup, and architecture reasoning live in `employee-task-app` since it's the natural starting point — this README covers only what's specific to this repo.

## Project overview

This repo intentionally contains almost nothing: 2 ArgoCD Application manifests and 2 values files. That's on purpose — see "Why this repo is this small" below.

## Architecture diagram

```
employee-task-app                              employee-task-gitops (this repo)
  helm/employee-task/                            apps/dev-application.yaml  ──┐
    Chart.yaml                                    apps/prod-application.yaml ─┤
    values.yaml (defaults)                                                    │
    templates/                                    environments/dev/values.yaml ◄┤ (values ref)
                                                    environments/prod/values.yaml◄┘
        ▲                                                       │
        │ (chart source)                                        │ (values source)
        └───────────────────┬───────────────────────────────────┘
                             │
                    ArgoCD multi-source Application
                             │
                             v
                   EKS cluster (dev or prod)
```

Each ArgoCD `Application` in `apps/` has **two sources**: the Helm chart from `employee-task-app`, and a values file from this repo, combined via ArgoCD's multi-source feature (`$values` reference). See `employee-task-app`'s [ARCHITECTURE.md](https://github.com/rashmiranjandevops/employee-task-app/blob/main/ARCHITECTURE.md#why-the-helm-chart-lives-in-employee-task-app-not-employee-task-gitops) for the full reasoning.

## Folder structure

```
apps/
  dev-application.yaml     ArgoCD Application for dev  (auto-sync)
  prod-application.yaml    ArgoCD Application for prod (manual sync)

environments/
  dev/values.yaml           Everything dev-specific: replica counts, resources,
                              hostnames, the ECR registry, the ACM cert ARN
  prod/values.yaml          Same, different values
```

There's no chart here, and no shared/default values file — both live in `employee-task-app`. If you're looking for `Chart.yaml` or Kubernetes manifest templates, they're not in this repo.

## Technology stack

ArgoCD · Helm (values only — the chart lives elsewhere) · Kubernetes (as the deploy target)

## Setup instructions

This repo doesn't need "setup" on its own — it needs `employee-task-infra`'s Terraform applied and ArgoCD installed first. Full order across all 3 repos: `employee-task-app`'s [INSTALL.md](https://github.com/rashmiranjandevops/employee-task-app/blob/main/INSTALL.md).

Once ArgoCD is installed:
```bash
kubectl apply -f apps/dev-application.yaml
kubectl apply -f apps/prod-application.yaml
```

## Deployment instructions

You don't deploy from this repo directly — CI in `employee-task-app` does, by updating a values file here:

1. CI builds and pushes an image, then bumps `backend.image.tag`/`frontend.image.tag` in the target environment's values file (`environments/dev/values.yaml` for a `develop` push, `environments/prod/values.yaml` for a `main` push).
2. **dev**: ArgoCD auto-syncs — live within ~3 minutes.
3. **prod**: the commit lands in Git immediately, but `prod-application.yaml` has no `automated` sync policy — someone runs `argocd app sync employee-task-prod` on purpose. That's the entire prod promotion gate.

## Rollback procedure

```bash
# from employee-task-app
./scripts/rollback.sh prod ../employee-task-gitops
```
Reverts the relevant `environments/<env>/values.yaml` to its previous Git commit and syncs ArgoCD to that state — a Git revert, not `argocd app rollback` (which would change the cluster without changing Git, and get silently undone by the next auto-sync).

## Troubleshooting guide

| Symptom | Likely cause |
|---|---|
| `Application` stuck `OutOfSync` in dev | `syncPolicy.automated` missing from `dev-application.yaml`, or it was never applied |
| `Application` shows `Unknown` health | A Helm rendering error — render locally from `employee-task-app`: `helm template helm/employee-task -f ../employee-task-gitops/environments/dev/values.yaml` |
| Image pull fails after a deploy | The tag CI wrote to the values file doesn't exist in ECR yet — check the CI run actually finished pushing |
| Both `values` files changed unexpectedly | Both CI pipelines (GitHub Actions + Jenkins) were auto-triggering at once — only one should be "live" at a time |

Full troubleshooting guide (covers the other 2 repos too): `employee-task-app`'s [TROUBLESHOOTING.md](https://github.com/rashmiranjandevops/employee-task-app/blob/main/TROUBLESHOOTING.md).

## Applying manually (without CI or ArgoCD)

```bash
# from employee-task-app, with employee-task-gitops as a sibling directory
helm template helm/employee-task -f ../employee-task-gitops/environments/dev/values.yaml | kubectl apply -f -
```

## Why this repo is this small

That's deliberate, not incomplete. Everything that changes when the *application* changes (a new env var, a new probe path, a new port) lives in `employee-task-app`'s chart, reviewed in the same PR as the code change that needs it. Everything that changes on every *deploy* (an image tag) — and only that — lives here. A repo this small is easy to audit: `git log` on this repo's `environments/` folder is a complete, uncluttered deploy history for both environments.

## Interview questions & lessons learned

Covered in `employee-task-app`'s [ARCHITECTURE.md](https://github.com/rashmiranjandevops/employee-task-app/blob/main/ARCHITECTURE.md#interview-questions-this-project-should-let-you-answer-confidently) — the multi-source ArgoCD pattern this repo depends on is one of the specific things covered there.
