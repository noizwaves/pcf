# Control Plane on Azure with BOSH Bootloader

From the official [BBL guide](https://github.com/cloudfoundry/bosh-bootloader/blob/master/docs/getting-started-azure.md)

## Requirements

1. Azure CLI via `brew update && brew install azure-cli`

## Inputs

- `$DISPLAY_NAME` = user made up, `berserk-savor-cp`
- `$IDENTIFIER_URI` = user made up, `http://cp.63r53rk54v0r.com`
- `$BBL_AZURE_CLIENT_ID` = generated in step below
- `$BBL_AZURE_CLIENT_SECRET` = user name up, `$(random string)`
- `$BBL_AZURE_SUBSCRIPTION_ID` = from `az account list | jq .id`
- `$BBL_AZURE_TENANT_ID` = from `az account list | jq .tenantId`

## X. Create Service Account

TODO: Use `az-automation` as described in the [official docs](https://github.com/cloudfoundry/bosh-bootloader/blob/master/docs/getting-started-azure.md)

1. Create application via `az ad app create --display-name $DISPLAY_NAME --homepage $IDENTIFIER_URI --identifier-uris $IDENTIFIER_URI --password $BBL_AZURE_CLIENT_SECRET` 
    - record the generated `appId` from JSON result as `$BBL_AZURE_CLIENT_ID`
1. Create Service Principle `az ad sp create --id $BBL_AZURE_CLIENT_ID`
1. Assign Contributor role via `az role assignment create --role Contributor --assignee $BBL_AZURE_CLIENT_ID`
1. `cp creds.tfvars.example creds.tfvars` and update placeholders in `creds.tfvars`
1. `cp .envrc.example .envrc` and fill in empty values

## X. Stand up Ops Manager

1. `bbl up`

## X. Use Ops Manager

TODO

## X. Shut down Ops Manager

1. `bbl down`
