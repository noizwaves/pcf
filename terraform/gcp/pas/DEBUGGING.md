# Debugging

For `dragonfruit` environment name at `63r53rk54v0r.com` DNS.

## Accessing BOSH (SSH into opsman)

1. SSH into opsman VM (following https://docs.pivotal.io/pivotalcf/2-3/customizing/trouble-advanced.html#ssh) via:
    - `gcloud compute ssh dragonfruit-ops-manager`
1. Export the BOSH director command line settings to the environment (from https://pcf.dragonfruit.63r53rk54v0r.com/api/v0/deployed/director/credentials/bosh_commandline_credentials)

## Accessing `uaac`

1. SSH into opsman
1. `uaac target uaa.sys.dragonfruit.63r53rk54v0r.com --skip-ssl-validation`
1. `uaac token client get`
1. Get credentials from https://pcf.dragonfruit.63r53rk54v0r.com/api/v0/deployed/products/cf-cc5af124b5e3c479f442/credentials/.uaa.admin_client_credentials