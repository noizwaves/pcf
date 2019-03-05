# Setup Control Plane using Terraform

Following the [offical documentation](https://control-plane-docs.cfapps.io/guide.html) (docs source [here](https://github.com/pivotal/control-plane-docs/)).

## Requirements

- Certbot: `brew install certbot`
- PivNet CLI: `brew install pivnet-cli`
- Ops Manager CLI: `brew install om`

## 1. Create service account

Using the [official steps](https://github.com/genevieve/az-automation)

1. `az login`
1. `az account list`
1. `az-automation --account <ACCOUNT_NAME_FROM_CLI> --identifier-uri http://eggplant.63r53rk54v0r.com --display-name eggplant-cp  --credential-output-file creds.tfvars`
    - where `ACCOUNT_NAME_FROM_CLI` is derived from running `az account list`

## 2. Pave infrastructure with Terraform scripts

1. `git clone https://github.com/pivotal-cf/terraforming-azure.git`
1. `cp creds.tfvars terraforming-azure/terraforming-control-plane/terraform.tfvars`
1. `cd terraforming-azure/terraforming-control-plane`
1. Copy example variables from terraforming-azure README.md
1. Extract Ops Manager image location from azure pdf on [Pivotal Network](https://network.pivotal.io/products/ops-manager/)
1. Fill in variables
1. `terraform init`
1. `terraform plan -out=plan`
1. `terraform apply plan`

## 3. Generate certs

TODO: Is the CAA record actually required?

1. Create CAA record via `az network dns record-set caa add-record -g eggplant -z eggplant.63r53rk54v0r.com -n "@" --flags 0 --tag "issue" --value "letsencrypt.org"`
1. Generate certs via:
    ```
    sudo certbot --server https://acme-v02.api.letsencrypt.org/directory \
    -d eggplant.63r53rk54v0r.com \
    -d *.eggplant.63r53rk54v0r.com \
    -d *.apps.eggplant.63r53rk54v0r.com \
    -d *.sys.eggplant.63r53rk54v0r.com \
    -d *.login.sys.eggplant.63r53rk54v0r.com \
    -d *.uaa.sys.eggplant.63r53rk54v0r.com \
    --manual --preferred-challenges dns-01 certonly
    ```
1. Copy certs locally via
    1. `mkdir certs`
    1. `sudo cp /etc/letsencrypt/live/eggplant.63r53rk54v0r.com/* ./certs/`
    1. `sudo chown adam:staff certs/*`
1. Prepare the certificate variables file by
    1. `cp lets-encrypt-vars.yml.example lets-encrypt-vars.yml`
    1. Fill in contents from each file in `./certs`

## 4. Configure Ops Manager

Following the [configuration guide](https://docs.pivotal.io/pivotalcf/2-4/om/azure/config-terraform.html)

1. Set env var `OM_USERNAME` to `admin`
1. Set env var `OM_PASSWORD` to `$(openssl rand -base64 10)`
1. `om configure-authentication -u $OM_USERNAME -p $OM_PASSWORD -dp <SOME_DECRYPTION_PASSPHRASE>`
1. `om update-ssl-certificate --certificate-pem "$(cat certs/fullchain.pem)" --private-key-pem "$(cat certs/privkey.pem)"`
1. Unset env var `OM_SKIP_SSL_VALIDATION`

## 5. Configure BOSH Director

Following the [configuration guide](https://docs.pivotal.io/pivotalcf/2-4/om/azure/config-terraform.html)

1. `om configure-director -c director-config.yml`
1. `om apply-changes -i`, where
    - `-i` ignores the warnings about ICMP checks failing

## 6. Download releases

1. Accept EULA via `pivnet accept-eula -p p-control-plane-components -r 0.0.25`
1. Download control plane component releases via 
    ```
    om download-product -t $PIVNET_API_TOKEN -p p-control-plane-components -v 0.0.25 -f uaa-release-60.9.tgz -o ./releases/ && \
    om download-product -t $PIVNET_API_TOKEN -p p-control-plane-components -v 0.0.25 -f postgres-release-35.tgz -o ./releases/ && \
    om download-product -t $PIVNET_API_TOKEN -p p-control-plane-components -v 0.0.25 -f garden-runc-release-1.18.2.tgz -o ./releases/ && \
    om download-product -t $PIVNET_API_TOKEN -p p-control-plane-components -v 0.0.25 -f credhub-release-2.1.2.tgz -o ./releases/ && \
    om download-product -t $PIVNET_API_TOKEN -p p-control-plane-components -v 0.0.25 -f control-plane-0.0.25-rc.1.yml -o ./releases/ && \
    om download-product -t $PIVNET_API_TOKEN -p p-control-plane-components -v 0.0.25 -f concourse-release-4.2.3.tgz -o ./releases/
    ```
1. Look up stemcell from [release manifest](https://network.pivotal.io/products/p-control-plane-components/). Use the major version to determine latest version of Stemcell to download.
1. `om download-product -t $PIVNET_API_TOKEN -p stemcells-ubuntu-xenial -v 170.30 -f bosh-stemcell-170.30-azure-hyperv-ubuntu-xenial-go_agent.tgz -o ./releases/`

## 7. Upload releases to jumpbox (aka Ops Manager)

Prep local environment variables:
1. Set env var `OPS_MANAGER_KEY_PATH` to `opsman.key`
1. `terraform output ops_manager_ssh_private_key > $OPS_MANAGER_KEY_PATH`
1. `chmod 0600 $OPS_MANAGER_KEY_PATH`
1. Set env var `OPS_MANAGER_VM_URL` to `$(terraform output ops_manager_dns)`

Copy files to `~/control-plane-assets` on Ops Manager
1. `ssh -i $OPS_MANAGER_KEY_PATH "ubuntu@${OPS_MANAGER_VM_URL}" mkdir control-plane-assets`
1. `scp -i $OPS_MANAGER_KEY_PATH releases/* "ubuntu@${OPS_MANAGER_VM_URL}":~/control-plane-assets`
1. `scp -i $OPS_MANAGER_KEY_PATH lets-encrypt-vars.yml "ubuntu@${OPS_MANAGER_VM_URL}":~/control-plane-assets`
1. `ssh -i $OPS_MANAGER_KEY_PATH "ubuntu@${OPS_MANAGER_VM_URL}" mkdir control-plane-assets/operations`
1. `scp -i $OPS_MANAGER_KEY_PATH operations/* "ubuntu@${OPS_MANAGER_VM_URL}":~/control-plane-assets/operations`

## 8. Upload releases to BOSH

1. Locate the [Bosh Commandlie Credentials](https://pcf.eggplant.63r53rk54v0r.com/api/v0/deployed/director/credentials/bosh_commandline_credentials)
1. SSH onto Opsman via `ssh -i $OPS_MANAGER_KEY_PATH "ubuntu@${OPS_MANAGER_VM_URL}"`
1. Set the following environment variables using `Bosh Commandlie Credentials`:
    ```
    export BOSH_CLIENT=<YOUR_CLIENT>
    export BOSH_CLIENT_SECRET=<YOUR_SECRET>
    export BOSH_CA_CERT=/path/to/your/ca/crt
    export BOSH_ENVIRONMENT=<YOUR_IP>
    export BOSH_DEPLOYMENT=control-plane
    ```
1. Navigate to assets dir via `cd ~/control-plane-assets`
1. Upload Stelcell via `bosh upload-stemcell *bosh-stemcell*.tgz`
1. Upload releases via
    ```
    bosh upload-release concourse-release-*.tgz && \
    bosh upload-release credhub-release-*.tgz && \
    bosh upload-release garden-runc-release-*.tgz && \
    bosh upload-release postgres-release-*.tgz && \
    bosh upload-release uaa-release-*.tgz
    ```

## 9. Update Cloud Config

Figure out the Azure VM Extensions using [CPI docs](https://bosh.io/docs/azure-cpi/) and [Azure console](https://portal.azure.com/#blade/HubsExtension/Resources/resourceType/Microsoft.Network%2FNetworkSecurityGroups):

1. Obtain current cloud config via `bosh cloud-config > cc.yml`
1. Add `control-plane-lb-cloud-properties` to existing `vm_extension` array via 
    ```
    vm_extensions:
    - cloud_properties:
        security_group: "eggplant-plane-security-group"
        load_balancer: "eggplant-lb"
    name: control-plane-lb-cloud-properties
    ```
1. Update cloud config via `bosh update-cloud-config cc.yml`

## 10. Deploy Control Plane

1. Deploy control plane deployment via
    ```
    bosh deploy control-plane-*.yml \
    -v "external_url=https://plane.eggplant.63r53rk54v0r.com" \
    -v "persistent_disk_type=10240" \
    -v "network_name=control-plane" \
    -v "azs=[\"null\"]" \
    -v "vm_type=Standard_F4s" \
    -o operations/azure-load-balancer.yml \
    -o operations/latest-stemcell.yml \
    -o operations/external-ssl-cert-lets-encrypt.yml
    -l lets-encrypt-vars.yml
    ```
1. Obtain [UAA Admin User Credentials](https://pcf.eggplant.63r53rk54v0r.com/api/v0/deployed/director/credentials/uaa_admin_user_credentials) -> `$UAA_ADMIN_USERNAME` & `$UAA_ADMIN_PASSWORD`
1. SSH into opsman
1. `credhub api -s "$BOSH_ENVIRONMENT":8844 --ca-cert /var/tempest/workspaces/default/root_ca_certificate`
1. `credhub login -u $UAA_ADMIN_USERNAME -p $UAA_ADMIN_PASSWORD$`
1. `credhub get -n $(credhub find | grep uaa_users_admin | awk '{print $3}')` -> `$CP_PASSWORD`

1. Visit [https://plane.eggplant.63r53rk54v0r.com/] and log in with `admin/$CP_PASSWORD`
