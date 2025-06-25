# `terraform-azure.yml`

This workflow allows you to easily run the Terraform `init` / `fmt` / `plan` / `apply` workflow when targeting Azure.

It runs using OIDC and requires you to use the same service principal for both running Terraform and accessing the backend state storage account.

Additionally two extra variables are included for use with GitHub configuration Terraform, `GH_OWNER` & `GH_TOKEN`.

It operates in two main modes:

## Dont Use Plan Outputs (Default)
`use-tfplan-output: false`

When running in this mode Terraform will run a fresh `plan` when applying configuration and `apply` based off that `plan`, this is the most simple way of operating.

Running in this way can be desired if you wish to ensure that your infrastructure *ALWAYS* aligns to the `main` branch.

However, this can mean `apply` runs are unpredictable, for example, if configuration was to change outside of Terraform between a pull request speculative `plan` running and a pull request being completed, the `apply` would run from a fresh `plan` "correcting" the changes made outside of Terraform, beyond those shown in the pull request speculative `plan`.

In many cases this may be desired to ensure that the `main` branch reflects truly your infrastructure (at the point of `apply`), however it does mean that changes outside Terraform occurring after the pull request `plan` and before the `apply` will be "corrected" beyond the changes shown in the pull request `plan`.

This method is also less efficient as dual plans are run, one for the pull request and another for the pull request completion when applying. This can become an issue with larger state files and lots of resources under management due to both time to `plan` but also potentially contributing to provider API throttling limits.

When running in this mode your calling workflow should use the following values for `on`:

```yaml
on:
  push:
    branches: 
      - main
  pull_request:
    branches: 
      - main
```

## Use Plan Outputs (Advanced)
`use-tfplan-output: true`

When running in this mode Terraform will use artifacts to pass a `plan` output file between the `plan` and `apply` runs.

This ensures a more predictable `apply` as the output from the pull request `plan` which is approved is *actually* what will be applied.

The trade off however is a far more complex pipeline which relies on the `octokit/request-action@v2.x` action.

Additionally any changes occurring outside Terraform after the pull request `plan` and before the `apply` will not be "corrected".

When running in this mode your calling workflow should use the following values for `on`:

```yaml
on:
  pull_request_target:
    types:
      - closed
    branches:
      - main  
  pull_request:
    branches: 
      - main
```

## Requirements
In order to use this workflow you should ensure the calling repository is allowed to use the `octokit/request-action@v2.x` action.

Additionally, the following base permissions are required when calling the workflow:

```yaml
permissions:
  id-token: write # This is required for requesting the JWT
  contents: read  # This is required for actions/checkout
```

And if using plan outputs the following permission is also required:

```yaml
permissions:
  actions: read # This is required to read previous actions workflow runs
```

## Example Usage

The following is an example calling workflow when using the default dont use plan outputs mode with an environment and working directory set, running specifically for one repo subdirectory:

```yaml
name: 'Terraform'

on:
  push:
    branches: 
      - main
    paths:
      - "prd-uksouth/**"
  pull_request:
    branches: 
      - main
    paths:
      - "prd-uksouth/**"
      
permissions:
  id-token: write # This is required for requesting the JWT
  contents: read  # This is required for actions/checkout
  
jobs:
  terraform:
    uses: ukhomeoffice/ccoe-cicd-github/.github/workflows/terraform-azure.yml@v1.0.0
    with:
      environment: "PRD UK South"
      working-directory: ./prd-uksouth
    secrets:
      TF_STATE_STORAGE_ACCOUNT_NAME: ${{ secrets.TF_STATE_STORAGE_ACCOUNT_NAME }}
      TF_ARM_CLIENT_ID: ${{ secrets.TF_ARM_CLIENT_ID }}
      TF_ARM_TENANT_ID: ${{ secrets.TF_ARM_TENANT_ID }}
```

The next example is using the advanced mode working with plan outputs and not specifying any additional parameters such as environment or working directory:

```yaml
name: 'Terraform'

on:
  pull_request_target:
    types:
      - closed
    branches:
      - main  
  pull_request:
    branches: 
      - main

permissions:
  id-token: write # This is required for requesting the JWT
  contents: read  # This is required for actions/checkout
  actions: read # This is required to read previous actions workflow runs

jobs:
  terraform:
    uses: ukhomeoffice/ccoe-cicd-github/.github/workflows/terraform-azure.yml@v1.0.0
    with:
      use-tfplan-output: true
    secrets:
      TF_STATE_STORAGE_ACCOUNT_NAME: ${{ secrets.TF_STATE_STORAGE_ACCOUNT_NAME }}
      TF_ARM_CLIENT_ID: ${{ secrets.TF_ARM_CLIENT_ID }}
      TF_ARM_TENANT_ID: ${{ secrets.TF_ARM_TENANT_ID }}
```