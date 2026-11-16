# Required github / repo settings

```
# you might need to request additional scope, so you are allowed to push workflows
gh auth status
gh auth refresh --hostname github.com --scopes workflow

# enable dependency scanning for this repo
github repo > settings > advanced security > Dependency graph > enable
```

# Workflow CI
**ci.yml** — runs on every push to main and every PR to main

```
- Set up JDK 21 (Temurin) + Gradle (with dependency caching)
- Run unit tests         (./gradlew test)
- Run integration tests  (./gradlew integrationTest) (Testcontainers -> needs Docker, no docker-compose)
- Build                  (./gradlew build) (tests already ran, Gradle skips up-to-date tasks)
- Upload test reports as artifact on failure (7-day retention)
- Superseded PR runs are cancelled
- dependency-submission job -> dependency graph -> Dependabot alerts
```

# Workflow Publish image (CD)
**publish-image.yml** — runs on push to main (`latest`) and on `v*` tags (`x.y.z`)

```
Job publish:
- ./gradlew check — tests must pass before anything is published
- docker login to ghcr.io (GITHUB_TOKEN, permission packages:write)
- Compute image ref: ghcr.io/<owner>/rest-api-books:<latest|x.y.z>
  (GHCR only accepts lowercase image names)
- Gradle bootBuildImage --publishImage — Cloud Native Buildpacks image,
  pushed to GHCR (GitHub Packages) with an additional immutable tag
  :sha-<commit> (read by build.gradle via the IMAGE_SHA_TAG env var)

Job deploy (runs after publish):
- Checkout the GitOps repository Orb1979/cicd-kubernetes
  (uses the GITOPS_PAT secret — GITHUB_TOKEN cannot write to other repos)
- yq pins k8s/app/deployment.yaml to ghcr.io/<owner>/rest-api-books:sha-<commit>
- Commit "deploy: <image ref>" + push — the GitOps commit history is the deployment log
```

Note: the first push creates a *private* package; flip visibility in the
package settings if it should be pullable publicly (this repo does that,
so the cluster needs no imagePullSecret).

# Deployment pipeline (GitOps)

```
git push to main
   ↓
GitHub Actions (publish-image.yml)
   ↓
tests, build image :sha-<commit>, push to GHCR
   ↓
deploy job updates deployment.yaml: old sha → new sha
   ↓
commit + push to cicd-kubernetes
   ↓
Argo CD notices the Git change (polls every ~3 min)
   ↓
Argo CD applies the change to the local K3s (Rancher Desktop)
   ↓
K3s pulls the new image and creates a new Pod
   ↓
app reachable at http://localhost:8080/api/books
```

# Manual setup Deployment pipeline (Github + repo settings)
```
Ceate PAT (Personal Access token)
GitHub settings: github.com → click your avatar (top right)
  → Settings
  → Developer settings (bottom of the left menu)
  → Personal access tokens → Fine-grained tokens
  → Generate new token

Token name:	<app repo> cicd-app-with-gha deploy job
Expiration:	90 days
Resource owner:	<you> 
Repository access: Only select repositories → <kubernetes repo> 'cicd-kubernetes'
Permissions: Repository permissions → Contents	 -> change to: Read and write

Add the created token value to your <App Repo> settings
https://github.com/Orb1979/cicd-app-with-gha/settings/secrets/actions/
new Secret > GITOPS_PAT 
```

# Run ArgoCD Locally in kubernetes (Rancher Desktop)
```
# 1. if needed: Install Argo CD in your cluster (Rancher desktop)
kubectl create namespace argocd
kubectl apply -n argocd -k "github.com/argoproj/argo-cd/manifests/cluster-install?ref=stable"
kubectl apply -n argocd --server-side -k "github.com/argoproj/argo-cd/manifests/cluster-install?ref=stable"

kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d
<save this paswsowrd>

kubectl -n argocd port-forward svc/argocd-server 8088:443

# login ArgoCD Gui 
https://localhost:8088 
admin / <pasw you got prev step>
(optional: setings > user info > change password)

# add recursive direcotry scan, else it would only look for yaml files in k8s/
kubectl -n argocd patch application rest-api-books --type merge  -p '{"spec":{"source":{"directory":{"recurse":true}}}}'
k8s/
├── app/
│   ├── deployment.yaml 
│   └── service.yaml 
└── postgres/
    ├── deployment.yaml 
    └── ...

# 2. In the Argo CD UI create New Application (choose yaml)
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: rest-api-books
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/Orb1979/cicd-kubernetes
    targetRevision: HEAD
    path: k8s
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      selfHeal: true
```

# make sure local ports are available (argo cd gui and app)
```
Run the port forwarding
kubectl -n argocd port-forward svc/argocd-server 8088:443
kubectl port-forward svc/rest-api-books 8080:8080

kubectl -n argocd get application rest-api-books    # should show Synced / Healthy
kubectl get pods,pvc -n default                     # postgres + rest-api-books Running, pvc Bound
curl http://localhost:8080/api/books                # the app answers

update e.g the Readme.MD and it should get redeployed
```
