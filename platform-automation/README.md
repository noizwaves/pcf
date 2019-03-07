# Platform Automation

## TODOs

- https://github.com/pivotal-cloudops/azure-blobstore-concourse-resource


## Requirements

- Docker for Mac
- PivNet CLI: `brew install pivnet-cli`

## Running locally

Following [the docs](http://docs-platform-automation.cfapps.io/platform-automation/v2.1/reference/running-commands-locally.html)

1. `pivnet download-product-files -p platform-automation -r 2.1.1-beta.1 -g "*image*" -d ./ --accept-eula`
1. `docker import platform-automation-image-2.1.1-beta.1.tgz platform-automation-image`
1. `source aliases.sh`
1. Confirm working by running
    1. `p-automator -h` and confirm help is printed
    1. `pa-run om help` and confirm help is printed

## Create a client for Concourse jobs to speak to Credhub

1. SSH into Ops Manager
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

cat files/credhub_ca_cert | sed -E 's/\s+/\r\n/g'
cat files/credhub_ca_cert | sed -E 's/\s+/\r\n/g' | sed '/^\-/ d' | sed '/-$/d' | sed '/$CERT/d'