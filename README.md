# prj-ekc
This is the cloud-native-deploy repository for the application : engineering knowledge compiler. 
This repository is going to be responsible for containing all the application configuration related to 
it's deployment either in a local minikube k8s cluster or as a GKE application going forward.

## Operations

Use [OPERATIONS_RUNBOOK.md](OPERATIONS_RUNBOOK.md) for the complete ordered procedures covering Mac restart recovery, frontend and backend releases, Docker image publishing, Argo CD bootstrap, ingress and Cloudflare background processes, verification, rollback, and troubleshooting.

Normal startup:

```bash
minikube start
minikube addons enable ingress
kubectl wait --for=condition=Available deployment --all -n app-ekc-default --timeout=300s
```

Then keep these running in separate terminals:

```bash
kubectl port-forward -n ingress-nginx service/ingress-nginx-controller 8081:80
```

```bash
cloudflared tunnel run --url http://127.0.0.1:8081 minikube-tunnel
```

Links


Config repo: https://github.com/Soumik-Ghosh-2924/prj-ekc

Docker repo: https://hub.docker.com/r/dockersg6/engineering-knowledge-compiler

Frontend source: `engineering-knowledge-compiler/frontend`

Frontend image: `dockersg6/engineering-knowledge-compiler-ui`

The ingress routes `/api/v1/compiler` and `/health` to the Spring Boot API. `/home/` serves the React frontend. Before Argo CD syncs the frontend for the first time, build and push the image from the application repository:

```bash
docker build -t dockersg6/engineering-knowledge-compiler-ui:dev frontend
docker push dockersg6/engineering-knowledge-compiler-ui:dev
```

Install ArgoCD: https://argo-cd.readthedocs.io/en/stable/getting_started/#1-install-argo-cd

Login to ArgoCD: https://argo-cd.readthedocs.io/en/stable/getting_started/#4-login-using-the-cli

ArgoCD Configuration: https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/
