# Platform Automation

Walkthrough: deploy a foundation (named `grape`) from a control plane (named `eggplant`) on Azure.

## TODOs

- Use: https://github.com/pivotal-cloudops/azure-blobstore-concourse-resource
- Describe the contents and creation of the `config` and `variables` repositories
- Refactor variables so that Control Plane provides Minio storage and Credhub credentials

## Requirements

- Docker for Mac
- PivNet CLI: `brew install pivnet-cli`

## Running locally (optional)

Following [the docs](http://docs-platform-automation.cfapps.io/platform-automation/v2.1/reference/running-commands-locally.html)

1. `pivnet download-product-files -p platform-automation -r 2.1.1-beta.1 -g "*image*" -d ./ --accept-eula`
1. `docker import platform-automation-image-2.1.1-beta.1.tgz platform-automation-image`
1. `source aliases.sh`
1. Confirm working by running
    1. `p-automator -h` and confirm help is printed
    1. `pa-run om help` and confirm help is printed

## X. Create a client for Concourse jobs to speak to Credhub

1. SSH into Control Plane Ops Manager
1. Target CP UAA via `uaac target 10.0.8.11:8443 --skip-ssl-validation`
1. "Log in" by acquiring a token via `uaac token owner get login -s $UAA_LOGIN_PASSWORD` where
    - `UAA_LOGIN_PASSWORD` comes from [Uaa Login Client Credentials](https://pcf.eggplant.63r53rk54v0r.com/api/v0/deployed/director/credentials/uaa_login_client_credentials)
    - `User name` comes from [Uaa Admin User Credentials](https://pcf.eggplant.63r53rk54v0r.com/api/v0/deployed/director/credentials/uaa_admin_user_credentials)
    - `Password` comes from [Uaa Admin User Credentials](https://pcf.eggplant.63r53rk54v0r.com/api/v0/deployed/director/credentials/uaa_admin_user_credentials)
1. Set env var `CONCOURSE_CREDHUB_SECRET` = `$(openssl rand -base64 32)`
1. Create client via
    ```
    uaac client add credhub-grape \
        --authorized_grant_types client_credentials \
        --scope credhub.read,credhub.write \
        --authorities "credhub.write credhub.read" \
        --secret $CONCOURSE_CREDHUB_SECRET
    ```

## Deploy first pipeline

https://docs-platform-automation.cfapps.io/platform-automation/v2.1/reference/pipeline.html#retrieving-external-dependencies

## X. Prepare first pipeline

1. SSH onto CP Ops Manager
1. Create `pivnet-products` bucket via:
    1. `cd minio`
    1. `./mc mb controlplane/pivnet-products`

## X. Set first pipeline

1. `cp grape-download-vars.yml.example grape-download-vars.yml`
1. Fill in variables
1. `fly -t eggplant sp -p grape -c grape-download-pipeline.yml -l grape-download-vars.yml`
1. `fly -t eggplant unpause-pipeline -p grape-pipeline`

## X. Pave infrastructure for foundation

1. clone `terraforming-azure` repo as terraforming-((environment))
1. cd terraforming-((environment))/terraforming-pas
1. run the az-automation command to generate an account
1. copy the generated `creds.tfvars` file to `terraform.tfvars`
1. follow the [readme](./terraforming-grape/README.md#Var File) to fill in missing keys/values
1. set `ops_manager_vm = false` to prevent Terraform from creating an Ops Manager VM
1. expose the created opsman storage container name (or look it up from the Azure Portal) via
    1. modifying `terraforming-grape/modules/ops_manager/ops_manager.tf` to include:
        ```
        +output "ops_manager_storage_container" {
        +  value = "${azurerm_storage_container.ops_manager_storage_container.name}"
        +}
        ```
    1. modifying `terraforming-grape/terraforming-pas/outputs.tf` to include:
        ```
        +output "ops_manager_storage_container" {
        +  value = "${module.ops_manager.ops_manager_storage_container}"
        +}
        ```
1. `terraform init`
1. `terraform plan -out=plan`
1. `terraform apply plan`

## X. Create secrets

Create Ops Manager secrets via
1. SSH into Control Plane Ops Manager
1. Log into Credhub via `credhub login -u $UAA_ADMIN_USERNAME -p $UAA_ADMIN_PASSWORD`
1. `credhub n -n /pipeline/grape/opsman-admin-user -t user -z admin`
1. `credhub n -n /pipeline/grape/opsman-decryption-passphrase -t password`
1. `credhub n -n /pipeline/grape/pivnet-token -t value -v $PIVNET_API_TOKEN`
1. `credhub n -n /pipeline/grape/opsman_ssh -t ssh`

## X. Prepare the pipeline vars file

TODO: the current values in the vars file are near constants (and secret), maybe there is a better place for them?

1. `mkdir -p envs/grape`
1. `cp install-opsman.vars.yml.example envs/grape/install-opsman.vars.yml`
1. Fill in values unique to the foundation and those that are secrets

## X. Deploy the pipeline

1. `fly -t eggplant login -c https://plane.eggplant.63r53rk54v0r.com`
1. `fly -t eggplant sp -p grape -c install-opsman.pipeline.yml -l envs/grape/install-opsman.vars.yml`
1. `fly -t eggplant unpause-pipeline -p grape`
1. Trigger install Opsman job via `fly -t eggplant tj -j grape/install-opsman`
