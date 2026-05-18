# Azure Load Test — CI/CD gate

This repo proves you can run **Azure Load Testing as a pass/fail quality gate** from GitHub Actions, with **OIDC federated credentials** (no client secret stored anywhere).

## What's in here

```
.
├── .github/workflows/
│   └── loadtest.yml          GitHub Actions workflow (build-deploy-loadtest skipped — gate-only)
└── loadtest/
    ├── api-load.jmx          JMeter test plan (JWT auth → GET/POST /api/permits)
    ├── api-config.yaml       Azure Load Testing config (engines, env, failureCriteria)
    └── users.csv             500 seeded test users
```

## How it works

On every push to `main` the workflow:

1. `azure/login@v2` — exchanges the GitHub OIDC token for an Azure token (federated cred). No secrets in the repo.
2. `azure/load-testing@v1` — uploads `api-load.jmx` + `users.csv` to ALT, runs the test (~5 min, 100 VU), and evaluates `failureCriteria` from `api-config.yaml`.
3. Workflow exit code = pass/fail of the load test → green/red badge.

## Required GitHub config (one-time)

**Settings → Secrets and variables → Actions:**

| Type | Name | Value |
|---|---|---|
| Secret | `AZURE_CLIENT_ID` | App registration `appId` |
| Secret | `AZURE_TENANT_ID` | `az account show --query tenantId -o tsv` |
| Secret | `AZURE_SUBSCRIPTION_ID` | Subscription id |
| Variable | `ALT` | `cgov-alt-fovgoutjv2ah6` |
| Variable | `RG` | `rg-cgov-workshop` |

The App registration needs:
- Federated credential with subject `repo:dcucereavii-ms/azure-load-test-cicd:ref:refs/heads/main`, audience `api://AzureADTokenExchange`, issuer `https://token.actions.githubusercontent.com`.
- RBAC: **Load Test Contributor** on the ALT resource.

## Trigger a run

Any push to `main` triggers it. Or **Actions → load-test-gate → Run workflow**.

## The "money shot" — make the gate fail on purpose

Tighten `failureCriteria` in `loadtest/api-config.yaml`:

```yaml
failureCriteria:
  - p95(response_time_ms) > 200   # impossibly tight
```

Push → workflow goes red → in a real pipeline this would block the next stage (prod deploy).

Revert with `git revert HEAD --no-edit && git push`.
