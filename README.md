# mid_project_cd

GitOps / CD repository for [`opsera-mid_project`](https://github.com/mhmdmstfa2010/opsera-mid_project).
**ArgoCD watches this repo** — CI never talks to the cluster directly.

```
push to opsera-mid_project
        │
        ▼
CI: build → SBOM → Grype → GHCR → ACS → Docker Hub → Scout → cosign sign
        │
        ▼
update-manifest job: sed newTag → commit → push HERE
        │
        ▼
ArgoCD detects the diff → syncs the cluster
```

## Layout

```
base/                          # environment-agnostic manifests
├── backend/                   # Deployment + Service (:8080)
├── frontend/                  # Deployment + Service (:3000)
└── mongodb/                   # Deployment + Service + PVC (hostname: mongodb)
overlays/
└── production/
    ├── kustomization.yaml     # namespace: student-app + common labels
    ├── backend/kustomization.yaml    # ← CI rewrites newTag here
    └── frontend/kustomization.yaml   # ← CI rewrites newTag here
argocd/application.yaml        # ArgoCD Application (apply once)
```

## How the version bump works

The `update-manifest` job in the CI pipeline runs:

```bash
sed -i "s#newTag: .*#newTag: ${GITHUB_SHA}#" overlays/production/backend/kustomization.yaml
sed -i "s#newTag: .*#newTag: ${GITHUB_SHA}#" overlays/production/frontend/kustomization.yaml
git commit -am "bump backend+frontend to ${GITHUB_SHA}"
git push
```

So each file must keep **exactly one** `newTag:` line — don't add extra
images entries to those two files without adjusting the CI script.

## One-time setup

1. **Replace the image namespace** in both overlay files:
   `DOCKERHUB_NAMESPACE` → your Docker Hub namespace.
   *(Same value as the `DOCKERHUB_NAMESPACE` repository variable.)*

2. **CI repository variable**
   - `MANIFEST_REPO` = `mhmdmstfa2010/mid_project_cd`

3. **CI secret** — a fine-grained PAT with *Contents: Read & write* on this
   repo only:
   - `MANIFEST_REPO_TOKEN` = `<token>`

4. **ArgoCD** — point it at this repo (or apply the included Application):
   ```bash
   kubectl apply -f argocd/application.yaml
   ```

## Render locally

```bash
kustomize build overlays/production
# or
kubectl kustomize overlays/production
```

## Verify the images ArgoCD is about to deploy

Images are signed in CI with cosign; the public key lives in the app repo:

```bash
cosign verify --key cosign.pub docker.io/<DOCKERHUB_NAMESPACE>/backend:<git-sha>
cosign verify --key cosign.pub docker.io/<DOCKERHUB_NAMESPACE>/frontend:<git-sha>
```

## Roll back

Revert the `newTag` commit here and ArgoCD rolls back, or pin a known-good
SHA:

```bash
git revert HEAD && git push
```
