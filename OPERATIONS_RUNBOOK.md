# EKC Local Cloud Operations Runbook

This runbook explains how to build, release, restart, expose, verify, and troubleshoot the Engineering Knowledge Compiler (EKC) on the current local Minikube and Cloudflare setup.

Use it for:

- restarting everything after the Mac or Docker Desktop has stopped;
- releasing a frontend or backend feature;
- recovering Argo CD after complete cluster deletion;
- diagnosing public URL, image-pull, ingress, and synchronization failures.

## 1. Current architecture

```text
Internet user
    |
    v
https://ekc.sgsafesurf.com
    |
    v
Cloudflare DNS + named tunnel: minikube-tunnel
    |
    v
cloudflared on the Mac -> http://127.0.0.1:8081
    |
    v
kubectl port-forward -> ingress-nginx-controller:80
    |
    v
Kubernetes Ingress: app-ekc-ingress
    |-- /home/             -> ekc-frontend-service -> ekc-frontend pods
    |-- /health            -> ekc-app-service      -> ekc-app pods
    `-- /api/v1/compiler   -> ekc-app-service      -> ekc-app pods

Argo CD
    |
    `-- watches prj-ekc/k8s/dev/app_config
        and reconciles the Deployments, Services, Ingress, and RBAC
```

### Fixed project names

| Purpose | Value |
|---|---|
| Kubernetes context/profile | `minikube` |
| Kubernetes/Argo CD namespace | `app-ekc-default` |
| Argo CD Application | `ekc-application` |
| Backend Deployment / Service | `ekc-app` / `ekc-app-service` |
| Frontend Deployment / Service | `ekc-frontend` / `ekc-frontend-service` |
| Ingress | `app-ekc-ingress` |
| Ingress namespace | `ingress-nginx` |
| Cloudflare tunnel | `minikube-tunnel` |
| Tunnel UUID | `ebc3e710-dd97-41ae-a69d-7e77d050a545` |
| Public frontend | `https://ekc.sgsafesurf.com/home/` |
| Public health check | `https://ekc.sgsafesurf.com/health` |
| Argo CD UI (local only) | `https://127.0.0.1:8080` |

## 2. What must keep running

Kubernetes workloads run inside Minikube. Public access additionally depends on two long-running terminal processes on the Mac.

| Process | Must remain running? | Purpose |
|---|---:|---|
| Docker Desktop | Yes | Hosts the Minikube Docker node |
| Minikube | Yes | Runs Kubernetes, Argo CD, ingress, frontend, and backend |
| Ingress port-forward | Yes, for public access | Makes ingress available at `127.0.0.1:8081` |
| `cloudflared` tunnel | Yes, for public access | Connects Cloudflare to port `8081` |
| Argo CD UI port-forward | No | Only needed while viewing the local Argo CD UI |
| Vite and `mvn spring-boot:run` | No | Only needed for local development |

Keep the next two commands running in separate terminals whenever the application must be publicly reachable.

### Background terminal A: ingress origin

```bash
kubectl port-forward \
  -n ingress-nginx \
  service/ingress-nginx-controller \
  8081:80
```

Expected output:

```text
Forwarding from 127.0.0.1:8081 -> 80
```

### Background terminal B: Cloudflare tunnel

Start this after terminal A is listening:

```bash
cloudflared tunnel run \
  --url http://127.0.0.1:8081 \
  minikube-tunnel
```

Do not point this tunnel directly to `ekc-app-service`. It must target ingress so both frontend and backend routes work.

### Optional terminal C: Argo CD UI

```bash
kubectl port-forward \
  -n app-ekc-default \
  service/argocd-server \
  8080:443
```

Then open `https://127.0.0.1:8080`. A local certificate warning is expected. Do not publish Argo CD through the application tunnel.

## 3. Normal startup after restarting the Mac

Use this procedure after a Mac shutdown, Docker Desktop restart, or `minikube stop`.

### Step 1: start Docker Desktop

Open Docker Desktop and wait until the engine is running.

```bash
docker info
```

Do not continue until this succeeds.

### Step 2: resume Minikube

```bash
minikube start
kubectl config use-context minikube
```

The current profile uses the Docker driver, 2 CPUs, 4000 MB memory, a 20 GB disk, and the ingress addon.

```bash
minikube status
kubectl cluster-info
kubectl get nodes -o wide
```

### Step 3: ensure ingress is enabled

This command is idempotent:

```bash
minikube addons enable ingress
```

Wait for ingress:

```bash
kubectl rollout status \
  deployment/ingress-nginx-controller \
  -n ingress-nginx \
  --timeout=180s
```

### Step 4: wait for Argo CD and EKC

Kubernetes restarts existing workloads automatically. Do not reinstall Argo CD after an ordinary restart.

```bash
kubectl wait \
  --for=condition=Available \
  deployment \
  --all \
  -n app-ekc-default \
  --timeout=300s

kubectl get pods,deployments,services,ingress \
  -n app-ekc-default
```

Confirm Argo CD without opening its UI:

```bash
kubectl get application ekc-application \
  -n app-ekc-default \
  -o jsonpath='{.status.sync.status}{" / "}{.status.health.status}{"\n"}'
```

Expected: `Synced / Healthy`.

### Step 5: restore public access

Start background terminal A and then terminal B from section 2. The Cloudflare DNS route persists; do not recreate it after each restart.

```bash
cloudflared tunnel info minikube-tunnel
```

### Step 6: verify end to end

```bash
dig +short ekc.sgsafesurf.com
curl -I https://ekc.sgsafesurf.com/home/
curl https://ekc.sgsafesurf.com/health
```

Expected:

- DNS returns Cloudflare addresses;
- `/home/` returns HTTP `200` and HTML;
- `/health` returns HTTP `200` with `{"status":"UP"}`.

## 4. Frontend feature release

Use immutable Git-based tags. Avoid reusing `:dev`: Argo CD cannot detect an image change when the manifest text is unchanged, and mutable tags make rollback ambiguous.

### Step 1: test

```bash
cd /Users/g.soumik/IdeaProjects/engineering-knowledge-compiler/frontend
npm ci
npm run test
npm run lint
npm run build
```

Stop if any command fails.

### Step 2: commit the application feature

```bash
cd /Users/g.soumik/IdeaProjects/engineering-knowledge-compiler
git status
git add frontend
git commit -m "feature(frontend): describe the feature"
git push origin main

export EKC_UI_RELEASE_TAG=$(git rev-parse --short HEAD)
echo "$EKC_UI_RELEASE_TAG"
```

### Step 3: build and push the image

Current Apple Silicon/arm64 Minikube:

```bash
docker build \
  --platform linux/arm64 \
  -t dockersg6/engineering-knowledge-compiler-ui:$EKC_UI_RELEASE_TAG \
  frontend

docker push \
  dockersg6/engineering-knowledge-compiler-ui:$EKC_UI_RELEASE_TAG
```

Use `docker login` first if authentication expired.

For both typical cloud amd64 nodes and the current arm64 cluster:

```bash
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t dockersg6/engineering-knowledge-compiler-ui:$EKC_UI_RELEASE_TAG \
  --push \
  frontend
```

### Step 4: update GitOps

Edit `k8s/dev/app_config/frontend-deployment.yaml`:

```yaml
image: dockersg6/engineering-knowledge-compiler-ui:<EKC_UI_RELEASE_TAG>
```

Then:

```bash
cd /Users/g.soumik/IdeaProjects/prj-ekc
kubectl apply --dry-run=client \
  -f k8s/dev/app_config/frontend-deployment.yaml
git diff --check
git diff

git add k8s/dev/app_config/frontend-deployment.yaml
git commit -m "deploy(frontend): release $EKC_UI_RELEASE_TAG"
git push origin main
```

Do not use `kubectl edit` or `kubectl set image` for a GitOps release. Argo CD self-healing can revert live changes that are not in Git.

### Step 5: verify rollout

```bash
kubectl rollout status deployment/ekc-frontend \
  -n app-ekc-default \
  --timeout=300s

kubectl get application ekc-application \
  -n app-ekc-default \
  -o jsonpath='{.status.sync.status}{" / "}{.status.health.status}{"\n"}'

kubectl get deployment ekc-frontend \
  -n app-ekc-default \
  -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'

curl -I https://ekc.sgsafesurf.com/home/
curl https://ekc.sgsafesurf.com/health
```

Test the affected feature in the browser.

## 5. Backend feature release

### Step 1: test

```bash
cd /Users/g.soumik/IdeaProjects/engineering-knowledge-compiler
mvn clean test
```

Optional local functional check:

```bash
mvn spring-boot:run
```

In another terminal:

```bash
curl http://127.0.0.1:8080/health
```

### Step 2: commit the backend feature

```bash
git status
git add src pom.xml
git commit -m "feature(backend): describe the feature"
git push origin main

export EKC_API_RELEASE_TAG=$(git rev-parse --short HEAD)
echo "$EKC_API_RELEASE_TAG"
```

Only add files that belong to the feature. Preserve unrelated working-tree changes.

### Step 3: build and push with Jib

The Maven build targets `dockersg6/engineering-knowledge-compiler`.

```bash
mvn compile jib:build \
  -Dimage.tag=$EKC_API_RELEASE_TAG
```

If registry authentication fails, run `docker login` and retry. Confirm the tag exists before changing GitOps.

### Step 4: update GitOps

Edit `k8s/dev/app_config/deployment.yaml`:

```yaml
image: dockersg6/engineering-knowledge-compiler:<EKC_API_RELEASE_TAG>
```

```bash
cd /Users/g.soumik/IdeaProjects/prj-ekc
kubectl apply --dry-run=client \
  -f k8s/dev/app_config/deployment.yaml
git diff --check
git diff

git add k8s/dev/app_config/deployment.yaml
git commit -m "deploy(backend): release $EKC_API_RELEASE_TAG"
git push origin main
```

### Step 5: verify rollout

```bash
kubectl rollout status deployment/ekc-app \
  -n app-ekc-default \
  --timeout=300s

kubectl get deployment ekc-app \
  -n app-ekc-default \
  -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'

kubectl get application ekc-application \
  -n app-ekc-default \
  -o jsonpath='{.status.sync.status}{" / "}{.status.health.status}{"\n"}'

curl https://ekc.sgsafesurf.com/health
```

Test compilation with a small public repository:

```bash
curl -i \
  -X POST \
  -H 'Content-Type: application/json' \
  -d '{"repositoryUrl":"https://github.com/OWNER/REPOSITORY.git"}' \
  https://ekc.sgsafesurf.com/api/v1/compiler/compile
```

Use a repository you are authorized to compile.

## 6. Feature affecting frontend and backend

Use this order:

1. Finish and test backend contract changes.
2. Update and test the frontend against that contract locally.
3. Commit and push the application repository.
4. Build and push both images with the same Git SHA.
5. Update both image references in `prj-ekc` in one commit.
6. Push `prj-ekc`; let Argo CD roll out both Deployments.
7. Wait for both rollouts before testing publicly.

```bash
kubectl rollout status deployment/ekc-app \
  -n app-ekc-default --timeout=300s

kubectl rollout status deployment/ekc-frontend \
  -n app-ekc-default --timeout=300s
```

For an incompatible API change, use a backward-compatible transition or deploy the compatible backend first.

## 7. First bootstrap or recovery after `minikube delete`

Use this only if the profile was deleted, cluster state was lost, or the cluster is unrecoverable. Do not use it after an ordinary Mac restart.

### Step 1: create Minikube

```bash
minikube start \
  --driver=docker \
  --cpus=2 \
  --memory=4000 \
  --disk-size=20g \
  --addons=ingress

kubectl config use-context minikube
minikube status
```

### Step 2: install Argo CD in the project namespace

```bash
kubectl create namespace app-ekc-default

kubectl apply \
  -n app-ekc-default \
  --server-side \
  --force-conflicts \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

kubectl wait \
  --for=condition=Available \
  deployment \
  --all \
  -n app-ekc-default \
  --timeout=300s
```

If the namespace already exists, continue. For a long-lived environment, pin a tested Argo CD release instead of the moving `stable` branch.

### Step 3: create the Argo CD Application

```bash
cd /Users/g.soumik/IdeaProjects/prj-ekc
kubectl apply -f k8s/dev/applicationCRD.yaml
```

Argo CD reads `k8s/dev/app_config` from Git and recreates the Deployments, Services, Ingress, and RBAC.

```bash
kubectl get application ekc-application \
  -n app-ekc-default \
  -w
```

In a second terminal:

```bash
kubectl get pods -n app-ekc-default -w
```

### Step 4: recover Argo CD UI access if needed

Start optional terminal C from section 2.

```bash
argocd admin initial-password -n app-ekc-default
```

Without the Argo CD CLI:

```bash
kubectl get secret argocd-initial-admin-secret \
  -n app-ekc-default \
  -o jsonpath='{.data.password}' | base64 --decode
echo
```

Change the initial password after signing in.

### Step 5: restore Cloudflare

The named tunnel and DNS normally survive Minikube deletion because Cloudflare and the Mac store them separately.

```bash
cloudflared tunnel list
dig +short ekc.sgsafesurf.com
```

Only if DNS is missing:

```bash
cloudflared tunnel route dns \
  minikube-tunnel \
  ekc.sgsafesurf.com
```

Start background terminals A and B from section 2.

## 8. Troubleshooting decision guide

Work from the outside inward. Do not restart everything before identifying the failing layer.

### Safari: “Can't find the server”

```bash
dig +short ekc.sgsafesurf.com
```

No output means the Cloudflare DNS record is missing or not propagated:

```bash
cloudflared tunnel route dns minikube-tunnel ekc.sgsafesurf.com
```

### Cloudflare 1016 or tunnel unavailable

DNS exists but no connected tunnel can serve it.

```bash
cloudflared tunnel list
cloudflared tunnel info minikube-tunnel
```

Restart background terminal B. Confirm Minikube and terminal A are running.

### Cloudflare 502 / bad gateway

The tunnel is connected but its local origin is unavailable.

```bash
curl -I \
  -H 'Host: ekc.sgsafesurf.com' \
  http://127.0.0.1:8081/home/
```

If port `8081` refuses the connection, restart background terminal A. If it responds, inspect the `cloudflared` terminal.

### Nginx 404

```bash
kubectl describe ingress app-ekc-ingress \
  -n app-ekc-default
```

Confirm host `ekc.sgsafesurf.com` and paths `/home/`, `/health`, and `/api/v1/compiler`. Open the frontend with the trailing slash: `https://ekc.sgsafesurf.com/home/`.

### Kubernetes 503 / no upstream

```bash
kubectl get services,endpoints -n app-ekc-default
kubectl get pods -n app-ekc-default -o wide
```

Frontend endpoints use port `80`; backend endpoints use port `8080`.

### `ErrImagePull` or `ImagePullBackOff`

```bash
kubectl describe pod <POD_NAME> -n app-ekc-default

kubectl get nodes \
  -o custom-columns='NAME:.metadata.name,ARCH:.status.nodeInfo.architecture'
```

Confirm the repository/tag exists, registry access is public or authenticated, and the image supports the node architecture. Current Minikube is `arm64`.

### `CrashLoopBackOff`

```bash
kubectl logs <POD_NAME> -n app-ekc-default --previous
kubectl describe pod <POD_NAME> -n app-ekc-default
```

Read the previous container logs before restarting anything.

### Argo CD `OutOfSync`

```bash
kubectl describe application ekc-application \
  -n app-ekc-default

kubectl get application ekc-application \
  -n app-ekc-default \
  -o yaml
```

Check that the deployment commit is on `main`, the manifest uses the pushed image tag, the image was pushed first, and no YAML/RBAC/repository condition is reported.

### Argo CD UI does not open

Its port-forward is optional. Restart only terminal C. Argo CD continues syncing while its UI port-forward is stopped.

### Health works but frontend fails

```bash
curl https://ekc.sgsafesurf.com/health
curl -I https://ekc.sgsafesurf.com/home/
kubectl get pods -n app-ekc-default -l app=ekc-frontend
kubectl logs -n app-ekc-default deployment/ekc-frontend
```

Confirm the frontend was built with Vite base `/home/` and assets start with `/home/assets/`.

### Frontend works but compilation fails

```bash
curl https://ekc.sgsafesurf.com/health
kubectl logs -n app-ekc-default deployment/ekc-app --tail=200
```

Confirm requests use `/api/v1/compiler/compile` and ingress sends `/api/v1/compiler` to `ekc-app-service`.

## 9. Rollback

Rollback through Git rather than manually changing the cluster.

```bash
git log --oneline -- k8s/dev/app_config/deployment.yaml
git log --oneline -- k8s/dev/app_config/frontend-deployment.yaml
```

Change the relevant manifest to a previously working immutable tag, commit, push, and wait for Argo CD:

```bash
kubectl rollout status deployment/ekc-app \
  -n app-ekc-default --timeout=300s

kubectl rollout status deployment/ekc-frontend \
  -n app-ekc-default --timeout=300s
```

## 10. Daily health checklist

```bash
docker info >/dev/null && echo 'Docker: OK'
minikube status
kubectl get nodes
kubectl get pods -n ingress-nginx
kubectl get pods -n app-ekc-default
kubectl get application ekc-application -n app-ekc-default
cloudflared tunnel info minikube-tunnel
dig +short ekc.sgsafesurf.com
curl -I https://ekc.sgsafesurf.com/home/
curl https://ekc.sgsafesurf.com/health
```

Desired state:

- Docker and Minikube running;
- ingress controller ready;
- all Argo CD, frontend, and backend pods ready;
- Argo CD `Synced` and `Healthy`;
- tunnel connected and DNS resolving;
- frontend and health endpoints return HTTP `200`.

## 11. Operational rules

1. Build and push an image before referencing it in GitOps.
2. Use immutable Git SHA image tags, not reused `dev` or `latest` tags.
3. Let Argo CD deploy; do not fight self-healing with live edits.
4. Keep ingress port-forward and Cloudflare tunnel running for public access.
5. Keep Argo CD private.
6. Diagnose DNS -> tunnel -> local origin -> ingress -> service -> pod.
7. Use `minikube start`, not reinstallation, after normal Mac restarts.
8. Reinstall Argo CD and reapply `applicationCRD.yaml` only after cluster state is lost.

## 12. References

- Minikube start: <https://minikube.sigs.k8s.io/docs/commands/start/>
- Minikube ingress: <https://minikube.sigs.k8s.io/docs/tutorials/nginx_tcp_udp_ingress/>
- Argo CD getting started: <https://argo-cd.readthedocs.io/en/stable/getting_started/>
- Cloudflare local tunnel: <https://developers.cloudflare.com/tunnel/advanced/local-management/create-local-tunnel/>
- Cloudflare routing: <https://developers.cloudflare.com/tunnel/routing/>
