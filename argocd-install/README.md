# Install Argo CD and access the UI

These steps assume a running Kubernetes cluster and a working `kubectl` context.

## 1. Check your cluster

```bash
kubectl cluster-info
kubectl get nodes
```

## 2. Create the Argo CD namespace

Run from the repository root:

```bash
kubectl apply -f argocd-install/namespace.yaml
```

## 3. Install Argo CD

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl wait --for=condition=Available deployment --all -n argocd --timeout=300s
kubectl get pods -n argocd
```

## 4. Access the UI

Run this command and keep the terminal open:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Open https://localhost:8080 in your browser. A certificate warning may appear in a local learning setup.

## 5. Get the initial admin password

In a second terminal:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
echo
```

Username: `admin`. Use the password printed by the command. In Windows PowerShell, if `base64 -d` is unavailable:

```powershell
$secret = kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}"
[Text.Encoding]::UTF8.GetString([Convert]::FromBase64String($secret))
```

Change the initial password and configure appropriate access before using this beyond a local learning cluster. Never commit passwords or credentials to Git.

## 6. Deploy the sample app

1. Push this repository to GitHub.
2. Replace `YOUR_GITHUB_USERNAME` in `argocd-app.yaml` with the repository owner's username and confirm the repository URL.
3. Apply the Application resource:

```bash
kubectl apply -f argocd-install/argocd-app.yaml
kubectl get applications -n argocd
kubectl get deployments,services -n staging
```

The Application tracks `gitops-config/overlays/staging`, which sets nginx to one replica.

## Notes

- `argocd-app.yaml` is an Argo CD `Application` resource, not the Argo CD installation manifest.
- The example uses the `default` Argo CD Project for simplicity. For a real team, create scoped Projects and RBAC.
- For private repositories, configure repository credentials securely in Argo CD. Do not put tokens in this manifest.
