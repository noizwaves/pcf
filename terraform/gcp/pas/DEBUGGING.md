# Debugging

For `dragonfruit` environment name at `63r53rk54v0r.com` DNS.

## Accessing BOSH (SSH into opsman)

1. SSH into opsman VM (following https://docs.pivotal.io/pivotalcf/2-3/customizing/trouble-advanced.html#ssh) via:
    - `gcloud compute ssh dragonfruit-ops-manager`
1. Export the BOSH director command line settings to the environment (from https://pcf.dragonfruit.63r53rk54v0r.com/api/v0/deployed/director/credentials/bosh_commandline_credentials)

## Accessing `uaac`

1. SSH into opsman
1. `uaac target uaa.sys.dragonfruit.63r53rk54v0r.com`
1. `uaac token client get`
1. Get credentials from https://pcf.dragonfruit.63r53rk54v0r.com/api/v0/deployed/products/cf-cc5af124b5e3c479f442/credentials/.uaa.admin_client_credentials

## Diffing deployments

Unfortunately `om interpolate` and `om staged-config` produce differently ordered YAML. To compare equivalents, we must sort first (using [yml-sorter](https://www.npmjs.com/package/yml-sorter)).

Setup
1. `npm install -g yml-sorter`

Generate the expected configuration (what the code says)
1. `om interpolate -c cf-config.yml -l cf-secrets.yml -l cf-variables.yml > expected.yml`
1. `yml-sorter --input expected.yml`

Generate the actual configurtion (what is live)
1. `om staged-config -p cf -c > actual.yml`
1. `yml-sorter --input actual.yml`

Now compare the actual and expected configurations
1. `diff actual.yml expected.yml`
