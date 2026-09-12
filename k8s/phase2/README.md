# EKC Phase 2 preview

This overlay is deliberately isolated from the current MVP:

- its Argo CD application is manual-sync only;
- it deploys into the dedicated `app-ekc-phase2` namespace;
- its services and workload names are unique;
- it defines no Ingress and therefore receives no public or Cloudflare traffic.

Before a preview sync, build the UI with `VITE_PHASE2_ENABLED=true` and
`NGINX_CONFIG=nginx.phase2.conf`, publish both
images with the commit-based tag `phase2-iteration2-f10dc15`. Validate through a
local port-forward first. Deleting the `ekc-phase2-preview` Argo application and the
`app-ekc-phase2` namespace rolls back the preview without touching the MVP.
