# Deploying PAS onto PCF

How to deploy PAS onto GCP using mostly automation

TODOS:
- update instructions to use official `GCP Terraform Templates` release from PivNet

## 1. Create the infrastructure

Use the follow resources:
- https://docs.pivotal.io/pivotalcf/2-4/om/gcp/prepare-env-terraform.html
- https://github.com/pivotal-cf/terraforming-gcp/tree/v0.63.0

1. Run `terraform` init/plan/apply

Consume the output (`terraform output`):
1. Set env var `OM_TARGET` to `https://<ops_manager_dns>`
1. Set env var `OM_SKIP_SSL_VALIDATION` to `true`

## 2. Configure Ops Manager & BOSH Director

Use the following resources:
- https://docs.pivotal.io/pivotalcf/2-4/om/gcp/config-terraform.html

1. `om configure-authentication -u <OPSMAN_USERNAME> -p <OPSMAN_PASSWORD>`, and then
    1. Set env var `OM_USERNAME` to `<OPSMAN_USERNAME>`
    1. Set env var `OM_PASSWORD` to `<OPSMAN_PASSWORD>`
1. `om configure-director -c config.yml` (more info at https://github.com/pivotal-cf/om/blob/master/docs/configure-director/gcp.md)

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
1. Configure PAS Settings manually by clicking on tile and following [guide](https://docs.pivotal.io/pivotalcf/2-4/customizing/gcp-er-config-terraform.html)
    - "Certs"
        - Use Ops Manager CA at https://pcf.dragonfruit.63r53rk54v0r.com/api/v0/certificate_authorities
    - `File Storage > External Google Cloud Storage with Service Account`
        - GCP Project ID: `terraform output` > `pas_blobstore_service_account_project`
        - GCP Service Account Email: `terraform output` > `pas_blobstore_service_account_email`
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
