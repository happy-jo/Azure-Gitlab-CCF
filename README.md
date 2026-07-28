# GitLab Audit Events — Microsoft Sentinel CCF Connector

A one-click deployable [Codeless Connector Framework](https://learn.microsoft.com/en-us/azure/sentinel/create-codeless-connector) (CCF) connector that pulls GitLab **instance audit events** (`GET /api/v4/audit_events`) into Microsoft Sentinel. Built for Azure Government and Azure commercial.

`azuredeploy.json` provisions everything in a single deployment:

1. **Custom table** — `GitLabAudit_CL`
2. **Data Collection Endpoint** — ingestion endpoint for the poller
3. **Data Collection Rule** — flattens each event (promotes `ip_address`, `author_email`, `target_details`, `custom_message`, etc.) and keeps the full `details` object
4. **Connector definition** — the connect/disconnect tile in Sentinel → Data connectors
5. **RestApiPoller** — polls GitLab with keyset pagination and incremental time filtering

## Deploy to Azure

> Replace the buttons below with ones pointing at **your** repo by running:
> `python generate-deploy-buttons.py <OWNER> <REPO> [BRANCH]`
> and pasting the output here. The examples below assume `punchcyber/gitlab-sentinel-ccf@main`.

### Azure Government (portal.azure.us)

[![Deploy to Azure Gov](https://aka.ms/deploytoazuregovbutton)](https://portal.azure.us/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fhappy-jo%2FAzure-Gitlab-CCF%2Fmain%2Fazuredeploy.json/createUIDefinitionUri/https%3A%2F%2Fraw.githubusercontent.com%2Fhappy-jo%2FAzure-Gitlab-CCF%2Fmain%2FcreateUiDefinition.json)

### Azure Commercial (portal.azure.com)

[![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fhappy-jo%2FAzure-Gitlab-CCF%2Fmain%2Fazuredeploy.json/createUIDefinitionUri/https%3A%2F%2Fraw.githubusercontent.com%2Fhappy-jo%2FAzure-Gitlab-CCF%2Fmain%2FcreateUiDefinition.json)

The form asks for: **workspace name**, **GitLab endpoint**, and your **PAT** (masked). Deploy it into the resource group that holds your Sentinel workspace.

## Repo layout

```
gitlab-audit-ccf/
├── azuredeploy.json            # the single deployable template (button target)
├── createUiDefinition.json     # portal form (workspace, endpoint, masked PAT)
├── generate-deploy-buttons.py  # prints button Markdown for your repo
├── README.md
└── src/                        # the same resources split out, for reference/editing
    ├── table.json
    ├── dataCollectionEndpoint.json
    ├── dcr.json
    ├── connectorDefinition.json
    └── poller.json
```

`src/` is for readability and maintenance. The deployment only uses `azuredeploy.json`; if you change a component in `src/`, mirror the change into `azuredeploy.json` (that's the file the button deploys).

## Parameters

| Parameter | Required | Default | Notes |
|-----------|----------|---------|-------|
| `workspaceName` | yes | — | Sentinel/Log Analytics workspace name |
| `gitlabPAT` | yes | — | securestring; sent as `PRIVATE-TOKEN` |
| `gitlabApiEndpoint` | no | PunchCyber URL | your `/api/v4/audit_events` URL |
| `location` | no | resource group location | table/DCE/DCR region |
| `connectorName` | no | `GitLabAuditConnector` | names the connector, DCE, and DCR |
| `initialCheckpointTimeUtc` | no | deploy time (`utcNow`) | first-poll start; set earlier to backfill |
| `queryWindowInMin` | no | `10` | poll window length |
| `tableRetentionInDays` | no | `90` | table analytics retention |

## About the GitLab query parameters

You asked whether all of these are needed:

```
?created_after=...&pagination=keyset&order_by=id&sort=asc&per_page=100
```

Recommendation — **keep all of them**, and here's why:

- **`pagination=keyset`** — enables GitLab's `Link: ...; rel="next"` header, which the poller follows automatically (`pagingType: LinkHeader`). Keyset is GitLab's recommended method for large/audit datasets and avoids deep-offset slowdowns.
- **`order_by=id`** — **required** whenever `pagination=keyset`; GitLab only supports keyset ordering by `id`.
- **`sort=asc`** — returns oldest→newest, which lines up with how the connector advances its checkpoint. Keep it.
- **`per_page=100`** — GitLab's max page size; fewer round trips.
- **`created_after`** — do **not** hard-code this. The poller injects it each cycle from its checkpoint (`startTimeAttributeName: created_after`), so it stays incremental with no gaps.

These live in the poller's `request.queryParameters` (all except `created_after`, which is dynamic). If you ever want to test the simplest possible call, you can drop `pagination`, `order_by`, and `sort` and rely on GitLab's default offset pagination — `LinkHeader` still works — but keyset is the better default. If your GitLab version doesn't emit a `Link` header under keyset, that's the one thing worth verifying on your instance; tell me and I'll switch the paging config.

## Manual deploy (CLI)

Use this instead of the button if you want to backfill history via `initialCheckpointTimeUtc`.

```bash
az cloud set --name AzureUSGovernment      # omit for commercial
az login
az account set --subscription "<sub-id>"

az deployment group create \
  --resource-group <rg-with-your-workspace> \
  --template-file azuredeploy.json \
  --parameters workspaceName="<workspace>" \
               gitlabPAT="<PAT>" \
               initialCheckpointTimeUtc="2026-07-01T00:00:00Z"
```

## Verify

Give it up to ~30 minutes after the first successful poll, then:

```kql
GitLabAudit_CL
| sort by TimeGenerated desc
| take 50
```

You should see events parsed into columns (`EventName`, `AuthorEmail`, `IpAddress`, `TargetDetails`, …) with the full payload preserved in `Details`.

## Notes & gotchas

- **PAT scope:** instance-wide audit events require an admin token. A non-admin PAT returns only events that user can see — confirm it matches your `curl` output.
- **Endpoints:** the poller reads the DCE ingestion URL and DCR immutable ID via ARM `reference()`, so Gov (`.monitor.azure.us`) vs commercial (`.monitor.azure.com`) resolves automatically. Nothing to change per cloud.
- **Duplicate DCE/DCR:** this template creates its own `GitLabAuditConnector-dce` and `-dcr`. If you already made `GitLab-DCE` / `gitlabauditdcr` by hand, you can delete those after this deploys, or set `connectorName` so the names line up.
- **Connect tile:** the poller deploys with `isActive: true`, so it starts polling immediately. The gallery tile will also let you disconnect/reconnect from the portal.
- **`connectorDefinitionName` is required** and must match a definition that exists at deploy time — the template guarantees this by deploying the definition first and making the poller `dependsOn` it. (This is what caused the earlier standalone-poller failures.)
