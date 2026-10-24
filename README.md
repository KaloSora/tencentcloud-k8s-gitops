# tencentcloud-k8s-gitops

GitOps manifests for Argo CD and the Jenkins application promotion pipeline.

## ArtifactId-based application paths

Promotion derives the application identity from the image's
`org.opencontainers.image.title` label, written from Maven `project.artifactId`
during the build. It must match the final image repository component and be a
valid lowercase Service name (1–63 characters, starting with a letter).
For any artifactId, Jenkins generates:

```text
apps/<artifactId>/environments/<environment>/manifests/deployment.json
apps/<artifactId>/environments/<environment>/manifests/service.json
apps/<artifactId>/environments/<environment>/release.json
argocd/applications/<artifactId>-<environment>.yaml
```

The Argo CD Application is named `<artifactId>-<environment>`. App-owned JSON
sources can use `${ARTIFACT_ID}` in names, labels, selectors and references.
ArtifactIds must be unique across projects sharing this GitOps repository;
groupId is not part of the destination path. Renaming an artifact creates a new
application and leaves the old application in place.

## Example environment mapping (`artifactId=k8s-demo-app`)

| Environment | Kubernetes namespace | Argo CD Application | Manifest directory |
| --- | --- | --- | --- |
| dev | dev | k8s-demo-app-dev | apps/k8s-demo-app/environments/dev/manifests |
| uat | uat | k8s-demo-app-uat | apps/k8s-demo-app/environments/uat/manifests |
| preprod | preprod | k8s-demo-app-preprod | apps/k8s-demo-app/environments/preprod/manifests |
| prod | prod | k8s-demo-app-prod | apps/k8s-demo-app/environments/prod/manifests |

Each directory contains a `deployment.json` and `service.json`. JSON manifests
are consumed directly by Argo CD. The Deployment must retain the name
`<artifactId>` and exactly one container named `app`: Jenkins updates that
container's image with the verified digest. Application settings come from the
image source commit in the app repository. Services are internal
ClusterIP services on port 80, forwarding to application port 18080.

## Bootstrap and first promotion

Application-specific Deployment and Service configurations are authored in
[`tencentcloud-k8s-app`](https://github.com/KaloSora/tencentcloud-k8s-app) under
`deploy/environments/<environment>/manifests/`. Jenkins reads them from the exact
source commit recorded in the selected image and generates this repository's
target environment files through a PR. Treat the Deployment and Service files
here as generated deployment declarations; edit their app-repository sources.
Promotion replaces destination settings rather than merging manual changes.
Platform-owned Argo CD projects and bootstrap configuration remain maintained here.

1. Update the trusted promotion pipeline, commit the app repository's `deploy/`
   configuration to the branch being built, and publish a new image. Its source
   commit must contain all four environments' Deployment and Service JSON files.
   No manually seeded target manifests are required. The Jenkins token needs
   read access to the app repository and write/PR access to this repository.
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
Each generated `release.json` includes `artifactId` and `configDigest`, a SHA-256 of all four
environments' rendered manifests, along with the image and source commit.
Higher environments require this exact release in the preceding environment;
its Deployment and Service must also match the source configuration. Legacy
release records without artifactId or this digest require a new `dev` promotion, followed by
the remaining environments in order. Jenkins checks declarations; operators
must also verify actual deployment health before promotion.

Deployments use UID/GID 10001, matching the application's Dockerfile, and
security settings compatible with the namespaces' restricted Pod Security
policy. Configure per-environment replica counts and resources in the
app repository's Deployment files; promotion copies them from the image commit.
