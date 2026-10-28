# Argo CD GitOps Assignment

This repository contains the Argo CD install notes, an Argo CD Application manifest, a Kustomize base and environment overlays, and the GitOps design document.

## Structure

```text
.
├── argocd-install/
│   ├── namespace.yaml
│   ├── argocd-app.yaml
│   └── README.md
├── gitops-config/
│   ├── base/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── kustomization.yaml
│   └── overlays/
│       ├── staging/kustomization.yaml
│       └── production/kustomization.yaml
└── GITOPS-DESIGN.md
```

1. Push this folder to a GitHub repository.
2. Replace `YOUR_GITHUB_USERNAME` in `argocd-install/argocd-app.yaml` with the actual repository owner and check the URL.
3. Validate the overlays with `kubectl kustomize gitops-config/overlays/staging` and `kubectl kustomize gitops-config/overlays/production`.
4. Follow `argocd-install/README.md` to install Argo CD and deploy the staging sample.

The Application manifest points to staging. The production overlay is included; create a separate Application pointing at `gitops-config/overlays/production` with destination namespace `production` when you are ready to deploy it.

