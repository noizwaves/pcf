# Deploying PAS onto PCF

How to deploy PAS onto GCP using mostly automation. The outcome will be a PCF Foundation that is running PAS and secured by a real Lets Encrypt cert

TODOS:
- update instructions to use official `GCP Terraform Templates` release from PivNet

### Requirements

- Google Cloud SDK: `brew cask install google-cloud-sdk`
- PivNet CLI: `brew install pivnet-cli`
- Ops Manager CLI: `brew install om`
- Certbot: `brew install certbot`

## 0. Certs (self signed)

Either:

1. Generate a self-signed cert whose Common Name is `*.dragonfruit.63r53rk54v0r.com` following https://devcenter.heroku.com/articles/ssl-certificate-self, or
1. Generate a self-signed cert covering multiple domains of `*.dragonfruit.63r53rk54v0r.com, *.apps.dragonfruit.63r53rk54v0r.com, *.sys.dragonfruit.63r53rk54v0r.com, *.login.sys.dragonfruit.63r53rk54v0r.com, *.uaa.sys.dragonfruit.63r53rk54v0r.com` following https://medium.com/@pubudu538/how-to-create-a-self-signed-ssl-certificate-for-multiple-domains-25284c91142b
1. Use Lets Encrypt to generate a cert (following https://medium.com/@mxplusb/lets-encrypt-with-pivotal-cloud-foundry-b128431c46b8 and adding a CAA DNS record to the domain) to finally run: 

```
sudo certbot --server https://acme-v02.api.letsencrypt.org/directory \
> -d *.dragonfruit.63r53rk54v0r.com \
> -d *.apps.dragonfruit.63r53rk54v0r.com \
> -d *.sys.dragonfruit.63r53rk54v0r.com \
> -d *.login.sys.dragonfruit.63r53rk54v0r.com \
> -d *.uaa.sys.dragonfruit.63r53rk54v0r.com \
> --manual --preferred-challenges dns-01 certonly
```

## 1. Create the infrastructure

Use the follow resources:
- https://docs.pivotal.io/pivotalcf/2-4/om/gcp/prepare-env-terraform.html
- https://github.com/pivotal-cf/terraforming-gcp/tree/v0.63.0

1. Run `terraform` init/plan/apply

Consume the output (`terraform output`):
1. Set env var `OM_TARGET` to `https://<ops_manager_dns>`
1. Set env var `OM_SKIP_SSL_VALIDATION` to `true`

### 1.1 Google Cloud SQL

Change the SQL Mode to `TRADITIONAL`

```
SET sql_mode = 'TRADITIONAL'
```

## 2. Configure Ops Manager & BOSH Director

Use the following resources:
- https://docs.pivotal.io/pivotalcf/2-4/om/gcp/config-terraform.html

1. `om configure-authentication -u <OPSMAN_USERNAME> -p <OPSMAN_PASSWORD>`, and then
    1. Set env var `OM_USERNAME` to `<OPSMAN_USERNAME>`
    1. Set env var `OM_PASSWORD` to `<OPSMAN_PASSWORD>`
1. `om configure-director -c config.yml` (more info at https://github.com/pivotal-cf/om/blob/master/docs/configure-director/gcp.md)
1. Replace SSL certs with signed LE certs via `om update-ssl-certificate --certificate-pem "$(cat path/to/fullchain.pem)" --private-key-pem "$(cat path/to/privkey.pem)"`

## 3. Deploy PAS

Useful constants:
- `PRODUCT_SLUG` = `elastic-runtime` for PAS.
- `VERSION` = `2.4.3`. Comes from `pivnet releases -p elastic-runtime`.

Use the following resources:

- https://docs.pivotal.io/pivotalcf/2-4/customizing/gcp-er-config-terraform.html

1. `pivnet login --api-token <API_TOKEN>` where:
    - `<API_TOKEN>` is the Legacy API Token from your [PivNet profile](https://network.pivotal.io/users/dashboard/edit-profile)
1. Accept PAS EULA (`pivnet accept-eula -p <PRODUCT_SLUG> -r <VERSION>`). The corresponding 
1. Download from PivNet (`om download-product -t $PIVNET_API_TOKEN -p <PRODUCT_SLUG> -v <VERSION> -f cf-*.pivotal -o ./`)
1. Upload to Ops Manager (`om upload-product -p ./cf-*.pivotal`). PAS should appear under the "Import a Product" button (`om available-products`).
1. Stage PAS for installation (`om stage-product -p cf -v 2.4.3`). The "Pivotal Application Service" tile should now appear in the Installation Dashboard (`om staged-products`) and require configuration.
1. Configure PAS Settings manually by clicking on tile and following [guide](https://docs.pivotal.io/pivotalcf/2-4/customizing/gcp-er-config-terraform.html) or `om configure-product -c cf-config.yml -l cf-variables.yml -l cf-secrets.yml`
    - "Networking"
        - Check `Disable SSL certificate verification for this environment` to trust self-signed certs
    - "Certs"
        - Use Ops Manager CA at https://pcf.dragonfruit.63r53rk54v0r.com/api/v0/certificate_authorities
    - `File Storage > External Google Cloud Storage with Service Account`
        - GCP Project ID: `terraform output pas_blobstore_service_account_project`
        - GCP Service Account Email: `terraform output pas_blobstore_service_account_email`
        - GCP Service Account Key JSON: `cat ~/Downloads/auth.json`
1. Stemcell via
    1. Navigate to https://network.pivotal.io/products/elastic-runtime/ "Depends On" to get the Stemcell version => Ubuntu Xenial
    1. Accept Stemcell EULA (`pivnet accept-eula -p stemcells-ubuntu-xenial -r 170.30`)
    1. Download Google specific Stemcell (`om download-product -t $PIVNET_API_TOKEN -p stemcells-ubuntu-xenial -v 170.30 -f light-bosh-stemcell-170.30-google-kvm-ubuntu-xenial-go_agent.tgz -o ./`)
    1. Upload Stemcell to Ops Manager (`om upload-stemcell -s light-bosh-stemcell-170.30-google-kvm-ubuntu-xenial-go_agent.tgz`). Stemcell should now be visible in [Stemcell Library](https://pcf.dragonfruit.63r53rk54v0r.com/stemcell_library)
    1. Assign Stemcell to PAS (`om assign-stemcell -p cf`)
    1. Preview pending changes (`om pending-changes`)
    1. Apply pending changes (`om apply-changes`) and watch the deployment at https://pcf.dragonfruit.63r53rk54v0r.com/change_log

Other useful commands:
- `pivnet product-files -p <PRODUCT_SLUG> -r <VERSION> --format=yaml` to list the available downloads for a PivNet release https://network.pivotal.io/products/elastic-runtime/#/releases/297394

# 4. Gain access to PAS

1. Navigate to `Ops Manager > PAS Tile > Credentials > UAA > Admin Credentials` and copy password (=> PASSWORD)
1. Visit https://login.sys.dragonfruit.63r53rk54v0r.com and log in with `admin:$PASSWORD`

# 5. Use PAS

`cf push` etc

# 6. Deleting all the things

1. Delete pushed apps
1. Stage deletion of PAS tile via `om unstage-product -p cf` (verify deletion is staged via `om pending-changes`)
1. Apply changes via `om apply-changes`

# Misc

Other useful commands:
- `pivnet product-files -p <PRODUCT_SLUG> -r <VERSION> --format=yaml` to list the available downloads for a PivNet release https://network.pivotal.io/products/elastic-runtime/#/releases/297394