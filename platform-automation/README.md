# Platform Automation

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