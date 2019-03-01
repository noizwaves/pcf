# Deploying PAS onto PCF

How to deploy PAS onto GCP using mostly automation. The outcome will be a PCF Foundation that is running PAS and secured by a real Lets Encrypt cert

### Requirements

- Google Cloud SDK: `brew cask install google-cloud-sdk`
    - `gcloud auth login`
    - `gcloud config set project cf-sandbox-aneumann`
- PivNet CLI: `brew install pivnet-cli`
- Ops Manager CLI: `brew install om`
- Certbot: `brew install certbot`
- direnv: `brew install direnv`

## Process

### 1. Certs

Lets Encrypt certs are preferred. Do this from the start.

#### Lets Encrypt

Use Lets Encrypt to generate a cert (following [this guide](https://medium.com/@mxplusb/lets-encrypt-with-pivotal-cloud-foundry-b128431c46b8) and adding a CAA DNS record to the domain) to finally run: 

```
sudo certbot --server https://acme-v02.api.letsencrypt.org/directory \
  -d dragonfruit.63r53rk54v0r.com \
  -d *.dragonfruit.63r53rk54v0r.com \
  -d *.apps.dragonfruit.63r53rk54v0r.com \
  -d *.sys.dragonfruit.63r53rk54v0r.com \
  -d *.login.sys.dragonfruit.63r53rk54v0r.com \
  -d *.uaa.sys.dragonfruit.63r53rk54v0r.com \
  --manual --preferred-challenges dns-01 certonly
```

TODO: Possible automated steps
```
gcloud dns --project=cf-sandbox-aneumann record-sets transaction start --zone=dragonfruit-zone

gcloud dns --project=cf-sandbox-aneumann record-sets transaction add VALUE --name=NAME.dragonfruit.63r53rk54v0r.com. --ttl=60 --type=TXT --zone=dragonfruit-zone
...

gcloud dns --project=cf-sandbox-aneumann record-sets transaction execute --zone=dragonfruit-zone
```

#### Self Signed

1. Generate a self-signed cert whose Common Name is `*.dragonfruit.63r53rk54v0r.com` following [this guide](https://devcenter.heroku.com/articles/ssl-certificate-self), or
1. Generate a self-signed cert covering multiple domains of `*.dragonfruit.63r53rk54v0r.com, *.apps.dragonfruit.63r53rk54v0r.com, *.sys.dragonfruit.63r53rk54v0r.com, *.login.sys.dragonfruit.63r53rk54v0r.com, *.uaa.sys.dragonfruit.63r53rk54v0r.com` following [this guide](https://medium.com/@pubudu538/how-to-create-a-self-signed-ssl-certificate-for-multiple-domains-25284c91142b)

### 2. Pave the infrastructure

Open the [official guide](https://docs.pivotal.io/pivotalcf/2-4/om/gcp/prepare-env-terraform.html).

Clone the corresponding git repo and check out an official tag:
- https://github.com/pivotal-cf/terraforming-gcp/tree/v0.63.0

Or, download the release from PivNet:
1. Set PivNet token `PIVNET_API_TOKEN` env var
1. `pivnet login --api-token $PIVNET_API_TOKEN`
1. Accept Elastic Runtime EULA (if required, `pivnet accept-eula -p $PAS_PRODUCT_SLUG -r $PAS_VERSION`)
1. `om download-product -t $PIVNET_API_TOKEN -p $PAS_PRODUCT_SLUG -v $PAS_VERSION -f terraforming-gcp-*.zip -o ./`
1. `unzip terraforming-gcp-*.zip`
1. `cd pivotal-cf-terraforming-gcp-*/terraforming-pas`

Then:
1. Edit `terraform.tfvars` with desired values
1. Run `terraform init`
1. Run `terraform plan -out=plan`
1. Run `terraform apply "plan"`

Consume Terraforms output:
1. Set env var `OM_TARGET` to `https://($terraform ops_manager_dns)`
1. Set env var `OM_SKIP_SSL_VALIDATION` to `true`

### 3. Configure Ops Manager & BOSH Director

Open the [official guide](https://docs.pivotal.io/pivotalcf/2-4/om/gcp/prepare-env-terraform.html)

1. Set env var `OM_USERNAME` to `admin`
1. Set env var `OM_PASSWORD` to `$(openssl rand -base64 10)`
1. `om configure-authentication -u $OPSMAN_USERNAME -p $OPSMAN_PASSWORD`, and then
1. Replace SSL certs with signed LE certs via `om update-ssl-certificate --certificate-pem "$(cat path/to/fullchain.pem)" --private-key-pem "$(cat path/to/privkey.pem)"`
1. Unset `OM_SKIP_SSL_VALIDATION`
1. `om configure-director -c bosh-config.yml -l bosh-variables.yml -l bosh-secrets.yml`
    - more info in [the docs](https://github.com/pivotal-cf/om/blob/master/docs/configure-director/gcp.md)

### 4. Configure BOSH Director

Open the [official guide](https://docs.pivotal.io/pivotalcf/2-4/om/gcp/config-terraform.html)

1. `cp bosh-variables.yml.example bosh-variables.yml` and edit values
1. `cp bosh-secrets.yml.example bosh-secrets.yml` and edit values
1. `om configure-director -c bosh-config.yml -l bosh-variables.yml -l bosh-secrets.yml -l gcp-constants.yml`
    - more info in [the docs](https://github.com/pivotal-cf/om/blob/master/docs/configure-director/gcp.md)

### 5. Deploy PAS

Useful environment variables:
- `PAS_PRODUCT_SLUG` = `elastic-runtime` for PAS.
- `PAS_VERSION` = `2.4.3`. Comes from `pivnet releases -p elastic-runtime`.

Open the [official guide](https://docs.pivotal.io/pivotalcf/2-4/customizing/gcp-er-config-terraform.html)

Configure the PAS tile:
1. If required, `pivnet login --api-token $PIVNET_API_TOKEN`
1. If required, accept PAS EULA (`pivnet accept-eula -p $PAS_PRODUCT_SLUG -r $PAS_VERSION`)
1. Download PAS Tile from PivNet via `om download-product -t $PIVNET_API_TOKEN -p $PAS_PRODUCT_SLUG -v $PAS_VERSION$ -f cf-*.pivotal -o ./`)
1. Upload to Ops Manager via `om upload-product -p ./cf-*.pivotal`
    - PAS should appear under the "Import a Product" button (or `om available-products`)
1. Stage PAS for installation (`om stage-product -p cf -v $PAS_VERSION`)
    - The "Pivotal Application Service" tile should now appear in the Installation Dashboard (`om staged-products`) and require configuration.
1. `cp cf-variables.yml.example cf-variables.yml` and edit values
1. `cp cf-secrets.yml.example cf-secrets.yml` and edit values
1. Apply PAS configuration via `om configure-product -c cf-config.yml -l cf-variables.yml -l cf-secrets.yml`

Configure the Stemcell:
1. Navigate to https://network.pivotal.io/products/elastic-runtime/ "Depends On" to get the Stemcell version => Ubuntu Xenial
1. Accept Stemcell EULA via `pivnet accept-eula -p stemcells-ubuntu-xenial -r 170.30`
1. Download Google specific Stemcell via `om download-product -t $PIVNET_API_TOKEN -p stemcells-ubuntu-xenial -v 170.30 -f light-bosh-stemcell-170.30-google-kvm-ubuntu-xenial-go_agent.tgz -o ./`
1. Upload Stemcell to Ops Manager via `om upload-stemcell -s light-bosh-stemcell-170.30-google-kvm-ubuntu-xenial-go_agent.tgz`
    - Stemcell should now be visible in [Stemcell Library](https://pcf.dragonfruit.63r53rk54v0r.com/stemcell_library)
1. Assign Stemcell to PAS (`om assign-stemcell -p cf`)

Apply tge changes
1. Preview pending changes via `om pending-changes`
1. Apply pending changes via `om apply-changes` and watch the deployment at https://pcf.dragonfruit.63r53rk54v0r.com/change_log
    - Prepare to wait for ~45 minutes while BOSH deploys all the things

### 6. Gain access to PAS

1. Navigate to `Ops Manager > PAS Tile > Credentials > UAA > Admin Credentials` and copy password (=> PASSWORD)
    - or run `om credentials -p cf -c .uaa.admin_credentials`
1. Visit https://login.sys.dragonfruit.63r53rk54v0r.com and log in with `admin:$PASSWORD`
1. `cf target https://api.sys.dragonfruit.63r53rk54v0r.com` & `cf login`

### 7. Use PAS

1. `cf create-space sandbox`
1. `cf target -s sandbox`
1. `cf push`

### 6. Deleting all the things

#### 6.1 Deleting PAS

1. TODO: Delete pushed apps?
1. Stage deletion of PAS tile via `om unstage-product -p cf`
    - verify deletion is staged via `om pending-changes`
1. Apply changes via `om apply-changes`

#### 6.2 Deleting BOSH Director

1. Stage deletion of BOSH via `om unstage-product -p p-bosh`
    - verify deletion is staged via `om pending-changes`
1. Apply changes via `om apply-changes`

#### 6.3 Unpave the infrastructure

1. `terraform destroy`

## Misc

Other useful commands:
- `pivnet product-files -p <PAS_PRODUCT_SLUG> -r <PAS_VERSION> --format=yaml` to list the available downloads for a PivNet release https://network.pivotal.io/products/elastic-runtime/#/releases/297394
- `om staged-config -p cf -c` to get the current PAS configuration
- `om interpolate -c cf-config.yml -l cf-secrets.yml -l cf-variables.yml` to get the expected PAS configuration
