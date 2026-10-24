# tencentcloud-k8s-gitops

GitOps manifests for Argo CD and the Jenkins application promotion pipeline.

## Environment mapping

| Environment | Kubernetes namespace | Argo CD Application | Manifest directory |
| --- | --- | --- | --- |
| dev | dev | k8s-demo-app-dev | apps/k8s-demo-app/environments/dev/manifests |
| uat | uat | k8s-demo-app-uat | apps/k8s-demo-app/environments/uat/manifests |
| preprod | preprod | k8s-demo-app-preprod | apps/k8s-demo-app/environments/preprod/manifests |
| prod | prod | k8s-demo-app-prod | apps/k8s-demo-app/environments/prod/manifests |

Each directory contains a `deployment.json` and `service.json`. JSON manifests
are consumed directly by Argo CD. The Deployment must retain the name
`k8s-demo-app` and exactly one container named `app`: Jenkins updates that
container's image while preserving environment settings. Services are internal
ClusterIP services on port 80, forwarding to application port 18080.

## Bootstrap and first promotion

Application-specific Deployment and Service configurations are maintained in
this GitOps repository. Initialize environments by reviewing and merging
these files through a PR; the promotion pipeline only updates existing
environment configuration and creates release records and Applications.

1. Merge these files into this repository's `main` branch. Jenkins reads remote
   `main`, so local files alone do not resolve the missing-manifest error.
2. Ensure Terraform's `module.k8s_app` in the infrastructure repository has
   created `dev`, `uat`, `preprod` and `prod`. Namespace lifecycle and restricted
   Pod Security labels remain owned by Terraform; `business-apps` does not grant
   Argo CD permission to create cluster-scoped resources.
3. For private Harbor images, provision a `kubernetes.io/dockerconfigjson`
   Secret named `harbor-docker-config` in each target namespace, containing
   pull credentials for `harbor.core.com`. The Secret in `cicd-build` is not
   available to application namespaces. Keep credentials outside Git. Cluster
   nodes must also resolve and support pulling from the HTTP Harbor registry.
4. Ensure the infrastructure's Argo CD bootstrap watches `argocd/projects` and
   `argocd/applications` in this repository. The `business-apps` project permits
   Deployments and Services in the four mapped namespaces.
5. Run Jenkins promotion with `TARGET_ENV=dev` and the repository, digest,
   version and Jira key from the same successful application build. Review
   and merge the generated GitOps PR.
6. After verifying the deployed environment, promote the same release in order:
   `dev → uat → preprod → prod`.

Initial Applications intentionally omit automatic sync. The `:unreleased`
image is a placeholder, not a published release; do not manually sync an
environment before its first promotion. Promotion replaces the placeholder
with the verified `repository@sha256:...`, creates `release.json`, and enables
automatic sync (`prune: false`, `selfHeal: true`) for that environment.
No `release.json` is seeded: higher environments require the exact release
already declared in the preceding environment. Jenkins checks declarations;
operators must also verify the actual deployment health before promotion.

Deployments use UID/GID 10001, matching the application's Dockerfile, and
security settings compatible with the namespaces' restricted Pod Security
policy. Configure per-environment replica counts and resources in the
Deployment files; promotion preserves them.
