# GitOps Deployment Design

## 1. Goal and workflow

Eight engineers currently deploy with manual `kubectl apply`, creating drift and making it difficult to audit changes. Git should become the reviewed source of truth, and Argo CD should be the normal deployment mechanism.

1. Engineers propose Kubernetes changes in pull requests.
2. CI validates YAML/Kustomize output and runs policy and application checks.
3. Required reviewers approve the change; production changes require designated approval.
4. The approved change is merged into a protected branch.
5. Argo CD observes the new Git revision and syncs the application.
6. Git history, pull request reviews, Argo CD history, and Kubernetes audit logs provide the audit trail.

Application CI builds and scans images separately. Once an image is ready, the GitOps repository is updated with the new immutable image tag through a pull request.

## 2. Prevent direct production `kubectl apply`

Argo CD alone cannot stop a cluster administrator from running `kubectl apply`; prevention requires Kubernetes access controls and team process.

- **Restrict production write access:** Developers should have read-only production access for routine troubleshooting. Avoid granting individual engineers broad write permissions or `cluster-admin`.
- **Use Kubernetes RBAC:** Limit write permissions for managed production resources to the Argo CD controller/service account and a small, audited break-glass role. Keep ordinary developer groups out of production write roles.
- **Use individual identity:** Integrate with the organisation's identity provider where possible and require MFA. Avoid shared admin credentials.
- **Protect Git branches:** Require pull requests, status checks, reviews, and CODEOWNERS approval for production paths. Limit merge and branch-protection bypass rights.
- **Keep audit records:** Enable Kubernetes API audit logging and retain Git provider and Argo CD audit/history records.
- **Add admission policy:** Where practical, enforce approved registries, resource limits, labels, and other production policies as defence in depth.
- **Make Git the normal path:** Engineers request deployments through reviewed Git changes, not manual commands.

Do not rely on policy statements alone: remove routine production write capability wherever possible. A person with cluster-admin privileges can bypass many safeguards, so that role must be tightly controlled.

## 3. Emergency hotfix / break-glass process

Emergencies may require action before a normal pull request can complete, but a direct change must not become an undocumented permanent deployment path.

1. Declare the incident and identify the incident commander or authorised approver.
2. Prefer a fast-tracked Git change with the smallest safe patch and a second-person review when feasible. Let Argo CD deploy it.
3. If GitOps cannot act quickly enough, an authorised break-glass operator may use a separate, time-limited credential. Record the reason, approver, commands, affected resources, and timestamps in the incident ticket or channel.
4. Avoid disabling self-heal casually. If reconciliation must be paused to prevent a known-bad sync, record the approval and expiry/restore step.
5. After stabilisation, commit the intended final state to Git through a reviewed pull request. Verify live state matches Git.
6. Restore normal sync/self-heal if paused, revoke or rotate temporary credentials, and review audit logs.
7. Conduct a post-incident review and add tests or policy checks to reduce recurrence.

Git remains the source of truth. A direct hotfix is a temporary exception, not a second deployment process.

## 4. Self-healing: what it does and when to disable it

With automated sync and `selfHeal: true`, Argo CD compares Git's desired configuration with live Kubernetes resources. If someone manually changes a field managed by the Application, Argo CD can restore the Git-declared value.

For example, if Git declares three replicas but someone manually scales the Deployment to one, Argo CD can restore three replicas.

Self-healing helps prevent drift, but it may need to be paused for a specific, approved reason:
- A controlled migration or maintenance operation where reconciliation would interfere.
- Investigation of a known-bad Git change while a corrected commit is prepared.
- An approved emergency where a temporary live change is essential.

Before disabling it, record why, limit who can do it, set an expiry, and restore it afterward. Prefer fixing or reverting Git over leaving undocumented live changes.

`prune` is separate from self-heal. It allows Argo CD to delete managed resources removed from Git, so enable it only with appropriate review and testing.

## 5. RBAC: developers sync their own apps, not others

Use both **Argo CD RBAC** and **AppProjects**. RBAC controls which actions a user can take; AppProjects constrain approved source repositories, destination clusters/namespaces, and resource kinds.

### Suggested roles

- **Platform administrators:** Manage Argo CD configuration, Projects, repository credentials, destinations, and global RBAC.
- **Application developers:** View and sync Applications assigned to their team; cannot sync other teams' Applications or alter global settings.
- **Production approvers:** Approve production changes through protected Git branches. Sync permission must not bypass required Git review.
- **Break-glass operators:** A small, audited group with temporary elevated access during declared incidents.

Create separate Projects such as `team-a`, `team-b`, and `platform`. Restrict each Project to approved source repositories and destination namespaces. Team A should not be able to deploy into Team B's namespace or production unless explicitly authorised.

A conceptual RBAC policy (adapt group names and project/application names, then test it against your Argo CD version) could be:

```csv
p, role:team-a, applications, get, team-a/*, allow
p, role:team-a, applications, sync, team-a/*, allow
p, role:team-a, applications, get, team-b/*, deny
p, role:team-a, applications, sync, team-b/*, deny
g, team-a-developers, role:team-a
```

The exact subject names depend on the identity provider or local account setup. Configure the default policy to be minimally privileged. Do not grant developers permission to create or modify Applications if that could let them point an Application at an unapproved repository or destination.

### Additional safeguards

- Use non-default Projects for teams and production.
- Restrict each Project's `sourceRepos`, `destinations`, and permitted resource kinds.
- Keep repository and cluster credentials under platform-team control.
- Test both allowed and denied cases: Team A can sync its own Application, cannot sync Team B's Application, and cannot deploy to production without authorisation.
- Review group membership and access regularly.

## 6. Validation and acceptance checks

Run these commands from the repository root:

```bash
kubectl kustomize gitops-config/overlays/staging
kubectl kustomize gitops-config/overlays/production
```

Expected replica counts:
- Staging: `1`
- Production: `3`

After installing Argo CD, updating the repository URL, and deploying the staging Application, verify:

```bash
kubectl get applications -n argocd
kubectl get deployments,services -n staging
```

To deploy production, create a separate Application pointing at `gitops-config/overlays/production` with destination namespace `production`, and apply your production approval process first.

This repository demonstrates the requested configuration structure. Before production use, also configure identity integration, least-privilege RBAC, protected branches, audit retention, and the emergency procedure.
