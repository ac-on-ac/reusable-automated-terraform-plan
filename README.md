# Reusable Automated Terraform Plan

A reusable GitHub Actions workflow that runs `terraform plan` on pull requests and posts the results as a pull request comment.

## Purpose

This workflow runs automatically when a pull request is opened, reopened, or updated. It:

1. Checks out the repository
2. Logs into any cloud providers whose authentication secrets are supplied
3. Initialises Terraform, optionally configuring a remote backend
4. Validates the Terraform configuration
5. Runs `terraform plan`
6. Posts the plan output as a comment on the pull request, updating the comment on subsequent pushes rather than creating a new one each time

## Inputs

### General

| Input | Required | Default | Description |
|---|---|---|---|
| `working_directory` | No | `.` | Directory containing the Terraform configuration |
| `terraform_version` | No | `latest` | Version of Terraform to install (e.g. `1.11.0`) |
| `terraform_vars` | No | `''` | Newline-separated `TF_VAR_*=value` pairs to export before the plan runs. Only `TF_VAR_*` keys are permitted. |
| `aws_region` | No | `us-east-1` | AWS region used for AWS login and as the default S3 backend region |

### Azure OIDC authentication

Supply these three inputs **instead of** `azure_credentials` to authenticate via OIDC. The calling workflow must also declare `permissions: id-token: write`.

| Input | Required | Description |
|---|---|---|
| `azure_client_id` | No | Azure client (application) ID |
| `azure_tenant_id` | No | Azure tenant ID |
| `azure_subscription_id` | No | Azure subscription ID |

### Azure blob storage backend

All four inputs must be provided together to configure the azurerm backend.

| Input | Required | Description |
|---|---|---|
| `azure_backend_resource_group_name` | No | Resource group containing the Storage Account |
| `azure_backend_storage_account_name` | No | Storage Account name |
| `azure_backend_container_name` | No | Blob container name |
| `azure_backend_key` | No | Blob path for the state file |

### S3 backend

`s3_backend_bucket` and `s3_backend_key` must both be provided to configure the S3 backend.

| Input | Required | Description |
|---|---|---|
| `s3_backend_bucket` | No | S3 bucket name |
| `s3_backend_key` | No | S3 object key (path) for the state file |
| `s3_backend_region` | No | AWS region of the bucket. Defaults to `aws_region` when not set. |
| `s3_backend_dynamodb_table` | No | DynamoDB table for state locking (optional) |

## Secrets

All secrets are optional. Only supply the secrets required by the providers your module uses.

### Azure

| Secret | Description |
|---|---|
| `azure_credentials` | Azure service principal credentials JSON for `azure/login`. Use this **or** the OIDC inputs — not both. |
| `databricks_host` | Databricks workspace host URL |
| `databricks_azure_workspace_resource_id` | Databricks Azure workspace resource ID |

### Terraform Cloud / Enterprise

| Secret | Description |
|---|---|
| `terraform_token` | Terraform Cloud or Terraform Enterprise API token |

### AWS

| Secret | Description |
|---|---|
| `aws_access_key_id` | AWS access key ID (static credentials) |
| `aws_secret_access_key` | AWS secret access key (static credentials) |
| `aws_role_to_assume` | IAM role ARN to assume via OIDC or static credentials |

At least one of `aws_access_key_id` or `aws_role_to_assume` must be provided to trigger AWS login.

### GitHub

| Secret | Description |
|---|---|
| `gh_token` | GitHub personal access token or app token for the Terraform GitHub provider. Exposed as `GITHUB_TOKEN_PROVIDER` to avoid conflicting with the built-in `GITHUB_TOKEN`. |

### HashiCorp Vault

| Secret | Description |
|---|---|
| `vault_url` | Vault server URL |
| `vault_token` | Vault token (token auth method) |
| `vault_role_id` | Vault AppRole role ID (AppRole auth method) |
| `vault_secret_id` | Vault AppRole secret ID (AppRole auth method) |

`vault_url` must be provided alongside either `vault_token` (token auth) or both `vault_role_id` and `vault_secret_id` (AppRole auth) to trigger Vault login.

### Snowflake

| Secret | Description |
|---|---|
| `snowflake_account` | Snowflake account identifier |
| `snowflake_user` | Snowflake username |
| `snowflake_password` | Snowflake password |
| `snowflake_role` | Snowflake role |

Snowflake has no dedicated login action. Secrets are exposed as the `SNOWFLAKE_ACCOUNT`, `SNOWFLAKE_USER`, `SNOWFLAKE_PASSWORD`, and `SNOWFLAKE_ROLE` environment variables, which the [Terraform Snowflake provider](https://registry.terraform.io/providers/Snowflake-Labs/snowflake/latest/docs#authentication) reads natively.

### Grafana

| Secret | Description |
|---|---|
| `grafana_url` | Grafana instance URL |
| `grafana_auth` | Grafana API key or service account token |

Grafana has no dedicated login action. Secrets are exposed as the `GRAFANA_URL` and `GRAFANA_AUTH` environment variables, which the [Terraform Grafana provider](https://registry.terraform.io/providers/grafana/grafana/latest/docs) reads natively.

## Permissions

The calling workflow must grant these permissions:

| Permission | Reason |
|---|---|
| `contents: read` | Check out the repository |
| `pull-requests: write` | Post the plan as a pull request comment |
| `id-token: write` | Required only when using Azure OIDC authentication |

## Calling this workflow

### Minimal — Terraform Cloud backend

```yaml
name: Terraform Plan

on:
  pull_request:
    types:
      - opened
      - synchronize
      - reopened

permissions:
  contents: read
  pull-requests: write

jobs:
  plan:
    uses: ac-on-ac/reusable-automated-terraform-plan/.github/workflows/plan.yml@v1.0.0
    secrets:
      terraform_token: ${{ secrets.TF_TOKEN }}
```

### Azure — service principal with Azure blob backend

```yaml
permissions:
  contents: read
  pull-requests: write

jobs:
  plan:
    uses: ac-on-ac/reusable-automated-terraform-plan/.github/workflows/plan.yml@v1.0.0
    with:
      working_directory: infrastructure
      terraform_version: 1.11.0
      azure_backend_resource_group_name: rg-terraform-state
      azure_backend_storage_account_name: stterraformstate
      azure_backend_container_name: tfstate
      azure_backend_key: mymodule.tfstate
    secrets:
      azure_credentials: ${{ secrets.AZURE_CREDENTIALS }}
```

### Azure — OIDC with Azure blob backend

```yaml
permissions:
  contents: read
  pull-requests: write
  id-token: write

jobs:
  plan:
    uses: ac-on-ac/reusable-automated-terraform-plan/.github/workflows/plan.yml@v1.0.0
    with:
      azure_client_id: ${{ vars.AZURE_CLIENT_ID }}
      azure_tenant_id: ${{ vars.AZURE_TENANT_ID }}
      azure_subscription_id: ${{ vars.AZURE_SUBSCRIPTION_ID }}
      azure_backend_resource_group_name: rg-terraform-state
      azure_backend_storage_account_name: stterraformstate
      azure_backend_container_name: tfstate
      azure_backend_key: mymodule.tfstate
```

### AWS — static credentials with S3 backend and DynamoDB locking

```yaml
permissions:
  contents: read
  pull-requests: write

jobs:
  plan:
    uses: ac-on-ac/reusable-automated-terraform-plan/.github/workflows/plan.yml@v1.0.0
    with:
      aws_region: eu-west-1
      s3_backend_bucket: my-terraform-state
      s3_backend_key: mymodule/terraform.tfstate
      s3_backend_dynamodb_table: terraform-locks
    secrets:
      aws_access_key_id: ${{ secrets.AWS_ACCESS_KEY_ID }}
      aws_secret_access_key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
```

### Multiple providers with Terraform variables

```yaml
permissions:
  contents: read
  pull-requests: write

jobs:
  plan:
    uses: ac-on-ac/reusable-automated-terraform-plan/.github/workflows/plan.yml@v1.0.0
    with:
      terraform_vars: |
        TF_VAR_location=uksouth
        TF_VAR_environment=prod
    secrets:
      azure_credentials: ${{ secrets.AZURE_CREDENTIALS }}
      terraform_token: ${{ secrets.TF_TOKEN }}
      snowflake_account: ${{ secrets.SNOWFLAKE_ACCOUNT }}
      snowflake_user: ${{ secrets.SNOWFLAKE_USER }}
      snowflake_password: ${{ secrets.SNOWFLAKE_PASSWORD }}
      snowflake_role: ${{ secrets.SNOWFLAKE_ROLE }}
      grafana_url: ${{ secrets.GRAFANA_URL }}
      grafana_auth: ${{ secrets.GRAFANA_AUTH }}
```

## Backend validation

When using the Azure blob backend, all four inputs (`azure_backend_resource_group_name`, `azure_backend_storage_account_name`, `azure_backend_container_name`, `azure_backend_key`) must be supplied together. Providing only some of them will fail immediately with an error identifying the missing inputs.

When using the S3 backend, `s3_backend_key` must be provided alongside `s3_backend_bucket`.

## Backend selection

The backend is selected automatically based on which inputs are provided:

| Condition | Backend used |
|---|---|
| `azure_backend_resource_group_name` is set | Azure Blob Storage (`azurerm`) |
| `s3_backend_bucket` is set | AWS S3 (`s3`) |
| Neither is set | Terraform Cloud, local, or whatever is configured in the Terraform source |

Backend configuration is passed as `-backend-config` flags at `terraform init` time, so no `backend` block is required in your Terraform source — or you can use a partial backend block and let the workflow fill in the remaining values.

## `terraform_vars` security

Only keys prefixed with `TF_VAR_` are permitted in `terraform_vars`. Any other key will cause the workflow to fail immediately with an error identifying the disallowed key.

## Input validation

`working_directory` is validated against the repository root using `realpath` before any commands run. Path traversal values (e.g. `../../other-repo`) are rejected. The directory must also exist.

## Plan comment behaviour

The plan output is posted as a collapsible comment on the pull request. If a plan comment already exists (from a previous push to the same PR), it is updated in place rather than creating a new comment. The comment includes the triggering actor, workflow name, and commit SHA for traceability.

The comment header reflects the plan result:

| Status | Meaning |
|---|---|
| ✅ No changes | Plan succeeded with exit code 0 — infrastructure matches the configuration |
| ⚠️ Changes detected | Plan succeeded with exit code 2 — resources will be added, changed, or destroyed |

Plan output is truncated at 60,000 characters if the plan is very large.

## Releases

This repository uses the [reusable-manual-release](https://github.com/ac-on-ac/reusable-manual-release) workflow to create releases. Releases are triggered manually from the **Actions** tab.
