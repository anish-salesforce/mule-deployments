Mule Deployments (Reusable GitHub Actions)
=========================================

This repo hosts reusable GitHub Actions workflows and composite actions for MuleSoft CI/CD. It is designed to be consumed by Mule application repos, e.g. `salesforce-app-api`.

Provided workflows
------------------

- PR verification: `.github/workflows/reusable-pr.yml`
  - Build + run MUnit
  - Resolve version from `pom.xml` if not provided
  - Check Anypoint Exchange to see if the version already exists

- Release: `.github/workflows/reusable-release.yml`
  - Build + run MUnit
  - Publish to Exchange if the version does not already exist
  - Deploy to CloudHub 2

Required repository variables and secrets (consumer repo)
-------------------------------------------------------

Set these on the consumer repo (e.g., `salesforce-app-api`). Prefer GitHub Environments for prod.

- Secrets
  - `ANYPOINT_CLIENT_ID`: Connected App client id
  - `ANYPOINT_CLIENT_SECRET`: Connected App client secret
- Variables
  - `ANYPOINT_GROUP_ID`: Business group/org UUID (Exchange groupId)
  - `ANYPOINT_ENV_ID`: Environment UUID for deployment
  - `CLOUDHUB2_TARGET_ID`: Runtime target id (RTF/CH2 org target)
  - `CLOUDHUB_REGION`: Region like `us-east-2`

Exchange Version Check
----------------------

Composite action: `.github/actions/exchange-version-check`
- Gets token via Accounts API v2
- Queries Exchange v2 for `groupId`, `assetId`, `version`
- Outputs `exists=true/false`

CloudHub 2 Deployment
---------------------

Release workflow calls CloudHub v2 API to deploy using Exchange artifact coordinates.

Minimal fields used:
- `groupId`, `assetId`, `version`
- `deploymentTargetId`, `environmentId`, `replicas`, `region`

Adjust `runtimeVersion` as needed.

Security considerations
-----------------------

- Use OIDC-based federated auth for GitHub Actions if your org supports a short-lived token broker. Otherwise restrict Connected App scopes to Exchange read/publish and CH2 deploy only.
- Store secrets only in GitHub Secrets or Environments. Never in code.
- Enable branch protection with required PR checks (this PR workflow) and required review.
- Pin actions to immutable SHAs in regulated environments. This sample uses tags for brevity.
- Enable dependency caching with checksum keys; avoid uploading target artifacts as cache.
- Use least-privilege scopes for the Connected App (no org admin).

Post-deployment alerts (Monitoring)
-----------------------------------

You can automate alert creation via the Anypoint Monitoring APIs:
- Metric alert API: `POST /monitoring/alerts/api/v1/environments/{envId}/alerts`
- Example approach in a follow-up job:

```bash
TOKEN=... # from the same OAuth token step
ANYPOINT_ENV_ID=...
curl -sS -X POST "https://anypoint.mulesoft.com/monitoring/alerts/api/v1/environments/$ANYPOINT_ENV_ID/alerts" \
  -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{
    "name": "High error rate",
    "severity": "CRITICAL",
    "query": "-5m:application.metrics:response.errors:error:sum > 50",
    "channels": [ { "type": "EMAIL", "addresses": ["ops@example.com"] } ]
  }'
```

Alternatively, create alert templates in the UI and export/import via API for consistency.

Unified Monitoring and Management
---------------------------------

- Enable Anypoint Monitoring for the environment; ensure visualization is on for apps.
- Ship logs to Anypoint Logging; use log4j2 JSON layouts; set correlation IDs.
- Use dashboards: create custom dashboards and export via Monitoring APIs for infra-as-config.
- Leverage API Manager policies and Analytics for managed APIs.
- Consider external SIEM by forwarding logs via Log Forwarding.

Using from app repo
-------------------

In the consumer app repo add:
- `.github/workflows/pr.yml` and `.github/workflows/release.yml` that `uses:` these reusable workflows (see sample in `salesforce-app-api`).

Local testing
-------------

- Validate tokens with curl and jq locally before running pipeline.
- Use `mvn -Pmunit` to ensure tests pass locally.


