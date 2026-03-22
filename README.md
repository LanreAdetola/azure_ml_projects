# AzureML_Projects

This repository includes a GitHub Actions smoke test workflow for validating access to an Azure Machine Learning workspace.

## What This Setup Does

1. Creates a service principal for CI/CD.
2. Assigns Contributor access scoped to a single Azure ML workspace.
3. Stores credentials in GitHub Secrets.
4. Runs a workflow to verify Azure login and workspace access.

## Prerequisites

1. Azure CLI installed.
2. You are signed in to Azure CLI.
3. Azure ML extension installed for CLI.

```bash
az login
az extension add --name ml --yes
```

## Step 1: Set Your Azure Context

Use your subscription ID:

```bash
az account set --subscription <your-subscription-id>
```

Confirm workspace and resource group:

```bash
az ml workspace list --subscription <your-subscription-id> --query "[?name=='<your-aml-workspace-name>'].{name:name,resourceGroup:resourceGroup}" -o table
```

Set these values for your environment:

1. Subscription ID: `<your-subscription-id>`
2. Resource Group: `<your-resource-group>`
3. Workspace Name: `<your-aml-workspace-name>`

## Step 2: Create Service Principal For GitHub Actions

Run:

```bash
az ad sp create-for-rbac --json-auth --name github-aml-sp --role Contributor --scopes /subscriptions/<your-subscription-id>/resourceGroups/<your-resource-group>/providers/Microsoft.MachineLearningServices/workspaces/<your-aml-workspace-name>
```

This returns a JSON credential block with keys such as `clientId`, `clientSecret`, `tenantId`, and `subscriptionId`.

## Step 3: Add GitHub Secrets

In GitHub:

1. Open repository Settings.
2. Go to Secrets and variables > Actions.
3. Create these repository secrets:

1. `AZURE_CREDENTIALS`: Paste the full JSON output from Step 2.
2. `AZURE_SUBSCRIPTION_ID`: `<your-subscription-id>`
3. `AZURE_RESOURCE_GROUP`: `<your-resource-group>`
4. `AZURE_ML_WORKSPACE`: `<your-aml-workspace-name>`

Important:

1. Never commit credential JSON to source control.
2. If credentials are ever exposed, rotate immediately.

## Step 4: Workflow File

Workflow path:

1. [.github/workflows/azure-ml-smoke-test.yml](.github/workflows/azure-ml-smoke-test.yml)

What it does:

1. Checks out the repository.
2. Logs into Azure with `azure/login@v2` using `AZURE_CREDENTIALS`.
3. Installs Azure ML CLI extension.
4. Verifies access with `az ml workspace show`.

Note:

1. The workflow currently targets the GitHub environment `dev`. Create that environment in GitHub if needed.

## Step 5: Run Smoke Test

1. Open GitHub Actions tab.
2. Select `Azure ML Smoke Test`.
3. Click Run workflow.
4. Confirm job `smoke-test` completes successfully.

## Troubleshooting

1. `the following arguments are required: --resource-group/-g`
Use `az ml workspace list` to find the resource group, then pass it explicitly.

2. Login failures in workflow
Confirm `AZURE_CREDENTIALS` is valid JSON and not truncated.

3. Authorization failures
Confirm the service principal has role assignment at workspace scope:

```bash
az role assignment list --assignee <client-id> --scope /subscriptions/<your-subscription-id>/resourceGroups/<your-resource-group>/providers/Microsoft.MachineLearningServices/workspaces/<your-aml-workspace-name> -o table
```

## Security Recommendations

1. Keep scope narrow, as done here at workspace level.
2. Rotate service principal credentials regularly.
3. Prefer OpenID Connect later to avoid long-lived client secrets.

## Optional: Use OIDC Instead Of Client Secret

OIDC removes the need for storing `clientSecret` in GitHub.

### Why Use OIDC

1. No long-lived secret in GitHub.
2. Better security posture for CI/CD.
3. Recommended approach for GitHub Actions to Azure.

### Step 1: Create Or Reuse Service Principal

If you already created `github-aml-sp`, keep using it. Otherwise create one first.

Get the app (client) ID for an existing service principal:

```bash
az ad sp list --display-name github-aml-sp --query "[0].appId" -o tsv
```

Set it to a variable:

```bash
APP_ID=$(az ad sp list --display-name github-aml-sp --query "[0].appId" -o tsv)
echo "$APP_ID"
```

### Step 2: Add Federated Credential For GitHub

Create a federated identity credential tied to your repo and branch:

```bash
cat > federated-credential.json <<'JSON'
{
	"name": "github-main-branch",
	"issuer": "https://token.actions.githubusercontent.com",
	"subject": "repo:<org-or-user>/<repo-name>:ref:refs/heads/main",
	"description": "GitHub Actions OIDC for main branch",
	"audiences": [
		"api://AzureADTokenExchange"
	]
}
JSON

az ad app federated-credential create --id "$APP_ID" --parameters federated-credential.json
```

If you run workflows from environments, use this subject format instead:

```text
repo:<org-or-user>/<repo-name>:environment:<environment-name>
```

### Step 3: Keep RBAC Scope At Workspace Level

Ensure the service principal has Contributor access at AML workspace scope:

```bash
az role assignment create --assignee "$APP_ID" --role Contributor --scope /subscriptions/<your-subscription-id>/resourceGroups/<your-resource-group>/providers/Microsoft.MachineLearningServices/workspaces/<your-aml-workspace-name>
```

### Step 4: Add GitHub Secrets/Variables For OIDC

Create these repository secrets:

1. `AZURE_CLIENT_ID` = app ID from `github-aml-sp`
2. `AZURE_TENANT_ID` = `<your-tenant-id>`
3. `AZURE_SUBSCRIPTION_ID` = `<your-subscription-id>`

You can keep `AZURE_RESOURCE_GROUP` and `AZURE_ML_WORKSPACE` as secrets too.

### Step 5: Update Workflow Login Step

Use this login pattern in your workflow:

```yaml
permissions:
	id-token: write
	contents: read

jobs:
	smoke-test:
		runs-on: ubuntu-latest
		steps:
			- name: Checkout repository
				uses: actions/checkout@v4

			- name: Azure login (OIDC)
				uses: azure/login@v2
				with:
					client-id: ${{ secrets.AZURE_CLIENT_ID }}
					tenant-id: ${{ secrets.AZURE_TENANT_ID }}
					subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```

You can keep the same AML smoke test command after login.

### OIDC Troubleshooting

1. `AADSTS700213` or federation mismatch:
Check `subject` exactly matches branch or environment used by the workflow.

2. `Insufficient privileges`:
Verify role assignment exists at workspace scope.

3. Workflow cannot request token:
Ensure workflow has `permissions: id-token: write`.
