# ppm-gitops

GitOps repository for the PPM (Project Portfolio Management) platform.

## Layout

```
charts/
├── ppm-frontend/         Helm chart for the Angular SPA (nginx)
│   ├── Chart.yaml
│   ├── templates/
│   ├── values.yaml          base values
│   ├── values-test.yaml     test env overrides    ← image.tag auto-updated by frontend CI
│   ├── values-staging.yaml  staging env overrides
│   └── values-prod.yaml     prod env overrides
└── ppm-backend/          Helm chart for the Spring Boot API
    ├── Chart.yaml
    ├── templates/
    ├── values.yaml
    ├── values-test.yaml     ← image.tag auto-updated by backend CI
    ├── values-staging.yaml
    └── values-prod.yaml

apps/
├── argocd/
│   └── app-of-apps.yaml  Root Argo CD Application — bootstraps everything
└── environments/
    ├── test/             auto-sync enabled (CI promotes here on every main push)
    │   ├── frontend.yaml
    │   └── backend.yaml
    ├── staging/          auto-sync (manual promotion from test)
    └── prod/             manual sync only (gated by human review)

namespaces/               Kubernetes Namespace manifests per environment
```

## Promotion flow

1. Commit lands on `main` in `sbouiwael/ppm-frontend` or `sbouiwael/ppm-backend`.
2. CI builds, tests, scans (Trivy), signs (cosign keyless), generates SBOM and SLSA
   provenance attestations, then pushes the image to GHCR.
3. CI's `promote-test` job clones this repo, bumps `image.tag` in the matching
   `values-test.yaml`, and pushes a `[skip ci]` commit.
4. Argo CD detects the change and syncs the test cluster automatically.
5. Promotion to staging/prod is done by humans via PRs against this repo.

## Bootstrap

```bash
kubectl apply -f apps/argocd/app-of-apps.yaml -n argocd
```

Argo CD picks up `apps/environments/**` and creates every child Application.
