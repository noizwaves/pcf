# Control Plane on AWS

Following the [official documentation](https://control-plane-docs.cfapps.io/) to deploy a control plane to `cp.aws.63r53rk54v0r.com`

Specs include:
- Ops Manager 2.4.6

## Prerequisites

1. Terraform: `brew install terraform`
1. certbot: `brew install certbot`
1. postgresql: `brew install postgresql`
1. Om CLI

## TODO

- capture tfstate into blobstore
- cert creation via terraform
- investigate why RDS is postgres
- investigate what minimum set of alternative names the certs need

## X. Create service account

1. Log into AWS Console (`pcf-practice`)
1. Navigate to [IAM screen](https://console.aws.amazon.com/iam/home#/users)
1. Name user `$USERNAME-sa` with Programmatic access
1. Create Custom policy named `aneumann-kms-custom` from JSON definition at https://github.com/pivotal-cf/terraforming-aws
1. Attach appropriate policies and the custom policy created above
1. Capture created credentials as `$AWS_ACCESS_KEY` and `$AWS_SECRET_KEY`

## X. Generate certs

1. Ensure domain has a CAA record associated with it
1. Generate certs via:
    ```
    sudo certbot --server https://acme-v02.api.letsencrypt.org/directory \
    -d cp.aws.63r53rk54v0r.com \
    -d '*.cp.aws.63r53rk54v0r.com' \
    -d '*.apps.cp.aws.63r53rk54v0r.com' \
    -d '*.sys.cp.aws.63r53rk54v0r.com' \
    -d '*.login.sys.cp.aws.63r53rk54v0r.com' \
    -d '*.uaa.sys.cp.aws.63r53rk54v0r.com' \
    --manual --preferred-challenges dns-01 certonly
    ```
1. Copy certs locally via
    1. `mkdir certs`
    1. `sudo cp /etc/letsencrypt/live/cp.aws.63r53rk54v0r.com/* ./certs/`
    1. `sudo chown adam:staff certs/*`

## X. Pave IaaS

1. `git clone git@github.com:pivotal-cf/terraforming-aws.git`
1. `cd terraforming-aws/terraforming-control-plane`
1. `cp ../../terraform.tfvars.example ./terraform.tfvars`
1. Edit `terraform.tfvars` with appropriate values
    1. `access_key` as `$AWS_ACCESS_KEY`
    1. `secret_key` as `$AWS_SECRET_KEY`
    1. `ops_manager_ami` from AWS .yaml file from PivNet
    1. `ssl_cert` from `$(cat certs/cert.pem)`
    1. `ssl_private_key` from `$(cat certs/privkey.pem)`
1. Run `terraform init`
1. Run `terraform plan -out=plan`
1. Run `terraform apply plan`
1. Obtain the Ops Manager SSH key via `terraform output ops_manager_ssh_private_key > ../../opsman.pem`
1. `chmod 600 ../../opsman.pem`

## X. Configure root DNS

In the root DNS Zone (Azure) add a NS record with values `terraform output env_dns_zone_name_servers` for name `cp.aws`.

## X. Configure Ops Manager

1. Set env var `OM_TARGET` to `https://pcf.cp.aws.63r53rk54v0r.com`
1. Set env var `OM_USERNAME` to `admin`
1. Set env var `OM_PASSWORD` to `$(openssl rand -base64 10)`
1. Set env var `OM_DP` to `$(openssl rand -base64 32)`
1. Configure auth via `om -k configure-authentication -u $OM_USERNAME -p $OM_PASSWORD -dp $OM_DP`
1. Unset env var `OM_DP`
1. Specify certs via `om -k update-ssl-certificate --certificate-pem "$(cat certs/fullchain.pem)" --private-key-pem "$(cat certs/privkey.pem)"`

## X. Configure BOSH Director

Using the [documentation](https://docs.pivotal.io/pivotalcf/2-4/om/aws/config-terraform.html).

If a config is available, simply do:
1. `cp director-vars.yml.example director-vars.yml`
1. Update values in `director-vars.yml` to be correct
1. `om configure-director --config director-config.yml --vars-file director-vars.yml`
1. `om apply-changes`

### Step 1. Access Ops Manager

Visit [Ops Manager](https://pcf.cp.aws.63r53rk54v0r.com) and log in with `$OM_USERNAME` and `$OM_PASSWORD`.

### Step 2. AWS Config

1. Fill in values for
    1. Access Key ID from `terraform output ops_manager_iam_user_access_key`
    1. AWS Secret Key from `terraform output ops_manager_iam_user_secret_key`
    1. Security Group ID from `terraform output vms_security_group_id`
    1. Key Pair Name from `terraform output ops_manager_ssh_public_key_name`
    1. SSH Private Key from `terraform output ops_manager_ssh_private_key`
    1. Region from `terraform.tfvars`, `region` key

### Step 3. Director Config

1. Fill in values for
    1. NTP Servers with `0.amazon.pool.ntp.org,1.amazon.pool.ntp.org,2.amazon.pool.ntp.org,3.amazon.pool.ntp.org`
    1. Database location with `Internal Database`

### Step 4. Create Availability Zones

1. For each of the availability zones listed in `terraform output infrastructure_subnet_availability_zones`:
    1. Click add
    1. Enter in the AZ name

### Step 5. Networks

1. Do not check ICMP checks
1. Add a network for `infrastructure`
    1. `terraform output infrastructure_subnet_ids`
1. Add a network for `control-plane`
    1. `terraform output control_plane_subnet_ids`

### Step 6. Assign AZs and Networks

1. Singleton Availability Zone pick `us-west-1b`
1. Network pick `infrastructure`

### Step 7 - 10

1. Don't change anything

### Apply changes

1. `om apply-changes`

## X. Download releases to jumpbox

Using Ops Manager as a jumpbox.

1. `pivnet accept-eula -p p-control-plane-components -r 0.0.27`
1. `mkdir releases`
1. Download component releases via
    ```
    om download-product -t $PIVNET_API_TOKEN -p p-control-plane-components -v 0.0.27 -f 'uaa-release-*.tgz' -o ./releases/ && \
    om download-product -t $PIVNET_API_TOKEN -p p-control-plane-components -v 0.0.27 -f 'postgres-release-*.tgz' -o ./releases/ && \
    om download-product -t $PIVNET_API_TOKEN -p p-control-plane-components -v 0.0.27 -f 'garden-runc-release-*.tgz' -o ./releases/ && \
    om download-product -t $PIVNET_API_TOKEN -p p-control-plane-components -v 0.0.27 -f 'credhub-release-*.tgz' -o ./releases/ && \
    om download-product -t $PIVNET_API_TOKEN -p p-control-plane-components -v 0.0.27 -f 'control-plane-*.yml' -o ./releases/ && \
    om download-product -t $PIVNET_API_TOKEN -p p-control-plane-components -v 0.0.27 -f 'concourse-release-*.tgz' -o ./releases/ &&
    rm releases/download-file.json
    ```
1. `mkdir stemcells`
1. Download stemcell via
    ```
    om download-product -t $PIVNET_API_TOKEN -p stemcells-ubuntu-xenial -v 170.38 -f 'light-bosh-stemcell-*-aws-xen-hvm-ubuntu-xenial-go_agent.tgz' -o ./stemcells/ && \
    rm stemcells/download-file.json
    ```

## X. Configure release

1. `cp control-plane-vars.yml.example control-plane-vars.yml`
1. Populate cert values with real values from `certs/*`

## X. Upload assets to jumpbox

1. Create remote asset directory via  `ssh -i $OPS_MANAGER_KEY_PATH "ubuntu@${OPS_MANAGER_VM_URL}" mkdir control-plane-assets`
1. Copy all assets via 
    ```
    scp -r -i $OPS_MANAGER_KEY_PATH ./releases "ubuntu@${OPS_MANAGER_VM_URL}":~/control-plane-assets && \
    scp -r -i $OPS_MANAGER_KEY_PATH ./stemcells "ubuntu@${OPS_MANAGER_VM_URL}":~/control-plane-assets && \
    scp -r -i $OPS_MANAGER_KEY_PATH ./operations "ubuntu@${OPS_MANAGER_VM_URL}":~/control-plane-assets && \
    scp -i $OPS_MANAGER_KEY_PATH ./control-plane-vars.yml "ubuntu@${OPS_MANAGER_VM_URL}":~/control-plane-assets
    ```

## X. Upload releases to Bosh

1. Fetch Bosh Command Line credentials via `https://pcf.cp.aws.63r53rk54v0r.com/api/v0/deployed/director/credentials/bosh_commandline_credentials`
1. SSH into Ops Manager
1. `alias bosh="$BOSH_COMMAND_LINE_CREDENTIALS"`
1. `cd control-plane-assets`
1. Upload releases and stemcell via
    ```
    bosh upload-stemcell stemcells/*bosh-stemcell*.tgz && \
    bosh upload-release releases/concourse-release-*.tgz && \
    bosh upload-release releases/credhub-release-*.tgz && \
    bosh upload-release releases/garden-runc-release-*.tgz && \
    bosh upload-release releases/postgres-release-*.tgz && \
    bosh upload-release releases/uaa-release-*.tgz

## X. Adjust the cloud config

1. SSH into Opsman and set the Bosh Command line credentials as above
1. `bosh cloud-config > cc.yml`
1. Edit values (if `env_name != cp`) in `control-plane-assets/operations/add-vm-extension-to-cloud-config.yml` using output from `terraform output control_plane_lb_target_groups`
1. `bosh update-cloud-config cc.yml -o control-plane-assets/operations/add-vm-extension-to-cloud-config.yml`

## X. Deploy Concourse

1. Deploy via
    ```
    bosh deploy releases/control-plane-0.0.27-rc.1.yml \
    -d control-plane \
    -o operations/use-load-balancer.yml \
    -o operations/lets-encrypt-ssl.yml \
    -l control-plane-vars.yml
    ```

## X. Obtain admin credentials

1. Obtain (UUA Admin User Credentials)[https://pcf.cp.aws.63r53rk54v0r.com/api/v0/deployed/director/credentials/uaa_admin_user_credentials] and populate `$UAA_ADMIN_USERNAME` and `$UAA_ADMIN_PASSWORD`
1. Obtain (BOSH Command Line Credentials)[https://pcf.cp.aws.63r53rk54v0r.com/api/v0/deployed/director/credentials/bosh_commandline_credentials] and populate `$BOSH_ENVIRONMENT`
1. SSH into Opsman
1. `credhub api -s "$BOSH_ENVIRONMENT":8844 --ca-cert /var/tempest/workspaces/default/root_ca_certificate`
1. `credhub login -u $UAA_ADMIN_USERNAME -p $UAA_ADMIN_PASSWORD`
1. `credhub get -n $(credhub find | grep uaa_users_admin | awk '{print $3}')` -> `$CP_PASSWORD`
1. Visit [https://plane.cp.aws.63r53rk54v0r.com/]() and log in with `admin/$CP_PASSWORD`

## X. Use control plane

...

## 17. Turning off all the things

### Delete Control Plane deployment

1. SSH into Ops Manager
1. Delete Control Plane deployment via `bosh delete-deployment -d control-plane`

### Delete BOSH Director deployment

1. `om delete-installation`

### Unpave the IAAS

1. `cd terraforming-aws/terraforming-control-plane`
1. `terraform destroy`