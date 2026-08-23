# prj-ekc
This is the cloud-native-deploy repository for the application : engineering knowledge compiler. 
This repository is going to be responsible for containing all the application configuration related to 
it's deployment either in a local minikube k8s cluster or as a GKE application going forward.

**Commands**

# install ArgoCD in k8s
kubectl create namespace argocd
kubectl apply -n app-ekc-default -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# access ArgoCD UI
kubectl get svc -n app-ekc-default
kubectl port-forward svc/argocd-server 8080:443 -n app-ekc-default

# login with admin user and below token (as in documentation):
kubectl -n app-ekc-default get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 --decode && echo

# you can change and delete init password

Links


Config repo: https://github.com/Soumik-Ghosh-2924/prj-ekc

Docker repo: https://hub.docker.com/r/dockersg6/engineering-knowledge-compiler

Frontend source: `engineering-knowledge-compiler/frontend`

Frontend image: `dockersg6/engineering-knowledge-compiler-ui`

The ingress routes `/api/v1/compiler` and `/health` to the Spring Boot API. The `/` route serves the React frontend. Before Argo CD syncs the frontend for the first time, build and push the image from the application repository:

```bash
docker build -t dockersg6/engineering-knowledge-compiler-ui:dev frontend
docker push dockersg6/engineering-knowledge-compiler-ui:dev
```

Install ArgoCD: https://argo-cd.readthedocs.io/en/stable/getting_started/#1-install-argo-cd

Login to ArgoCD: https://argo-cd.readthedocs.io/en/stable/getting_started/#4-login-using-the-cli

ArgoCD Configuration: https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/
