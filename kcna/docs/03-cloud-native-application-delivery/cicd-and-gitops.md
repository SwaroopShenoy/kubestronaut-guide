# CI/CD and GitOps

Up: [KCNA hub](../../README.md) · Domain 3 — Cloud Native Application Delivery (16%) · Prev: [Security basics](../02-container-orchestration/security-basics.md) · Next: [Packaging with Helm and Kustomize](packaging-helm-kustomize.md)

Software reaches a cluster through a pipeline, and in GitOps the pipeline's destination is Git itself. This chapter covers continuous integration, delivery and deployment, and how GitOps tools reconcile a cluster from a repository.

## Continuous integration, delivery and deployment

- **Continuous integration (CI)**: developers merge small changes often; each merge builds and runs automated tests.
- **Continuous delivery**: every passing build is ready to release; production deployment may need approval.
- **Continuous deployment**: every passing build is deployed to production automatically.

Good CI/CD practice: everything in version control, automated tests, fast builds, image scanning, and a rollback path.

## Tools by role

| Tool | Type | Notes |
|---|---|---|
| Jenkins | CI/CD server | Long-established; large plugin ecosystem; pipelines as code |
| GitLab CI/CD | CI/CD in GitLab | Pipelines defined in `.gitlab-ci.yml` |
| GitHub Actions | CI/CD in GitHub | Workflows in `.github/workflows/` |
| Tekton | Kubernetes-native CI/CD | Pipelines built from Task and Pipeline resources |

## GitOps

GitOps uses Git as the source of truth for desired cluster state. An agent inside the cluster pulls the desired state and reconciles the cluster toward it.

Four principles:

1. **Declarative**: the desired state is described in a declarative format (YAML, Helm, Kustomize).
2. **Versioned and immutable**: desired state lives in Git with history.
3. **Pulled automatically**: an agent in the cluster pulls changes; the CI system does not push to the cluster.
4. **Continuously reconciled**: the agent corrects drift between cluster and Git.

| Tool | Notes |
|---|---|
| Argo CD | Declarative GitOps with a web UI and sync controls |
| Flux | GitOps toolkit, focused on automation and the CLI |

Benefits: audit trail through Git history, easy rollback (revert a commit), and a repeatable recovery path.

## Distinctions that get tested

- GitOps is a deployment model; CI is about building and testing. They work together.
- Git is storage for desired state; the reconciling agent does the work. Git alone does not deploy anything.
- GitOps changes are pull-based. A push-based pipeline that runs `kubectl apply` is CD, not GitOps.
