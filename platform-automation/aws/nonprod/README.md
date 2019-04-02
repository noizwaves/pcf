# Deploy PAS with Platform Automation to AWS

Using (Platform Automation)[http://docs.pivotal.io/platform-automation/v2.1/index.html]

/pipeline/nonprod

## X. Generate SSL certificates

1. Generate certs using
    ```
    sudo certbot --server https://acme-v02.api.letsencrypt.org/directory \
    -d nonprod.aws.63r53rk54v0r.com \
    -d '*.nonprod.aws.63r53rk54v0r.com' \
    -d '*.apps.nonprod.aws.63r53rk54v0r.com' \
    -d '*.sys.nonprod.aws.63r53rk54v0r.com' \
    -d '*.login.sys.nonprod.aws.63r53rk54v0r.com' \
    -d '*.uaa.sys.nonprod.aws.63r53rk54v0r.com' \
    --manual --preferred-challenges dns-01 certonly
    ```

## X. Pave the IaaS

Terraform IaaS by
1. `cd terraform-aws/terraform-pas`
1. `terraform plan -out=plan`
1. `terraform apply plan`
1. Capture opsman ssh key by
    1. `terraform output ops_manager_ssh_private_key > ../../opsman.pem`
    1. `chmod 600 ../../opsman.pem`

## X. Create a client for Concourse jobs to speak to Credhub (Opsman's Credhub)

Note: This is not yet connecting to the Control Plane deployed Credhub (instead it's using Ops Man's Credhub)

1. SSH into Control Plane Ops Manager
1. Target CP UAA via `uaac target $BOSH_ENVIRONMENT:8443 --skip-ssl-validation`
1. "Log in" by acquiring a token via `uaac token owner get login -s $UAA_LOGIN_PASSWORD` where
    - `UAA_LOGIN_PASSWORD` comes from [Uaa Login Client Credentials](https://pcf.cp.aws.63r53rk54v0r.com/api/v0/deployed/director/credentials/uaa_login_client_credentials)
    - `User name` comes from [Uaa Admin User Credentials](https://pcf.cp.aws.63r53rk54v0r.com/api/v0/deployed/director/credentials/uaa_admin_user_credentials)
    - `Password` comes from [Uaa Admin User Credentials](https://pcf.cp.aws.63r53rk54v0r.com/api/v0/deployed/director/credentials/uaa_admin_user_credentials)
1. Set env var `NONPROD_CREDHUB_SECRET` = `$(openssl rand -base64 32)`
1. Create client via
    ```
    uaac client add nonprod_credhub_client \
        --authorized_grant_types client_credentials \
        --scope credhub.read,credhub.write \
        --authorities "credhub.write credhub.read" \
        --secret $NONPROD_CREDHUB_SECRET
    ```

## X. Add existing secrets to Credhub (Opsman's)

Log into Credhub via
1. `credhub api -s $BOSH_ENVIRONMENT:8844 --skip-tls-validation`
1. `credhub login -s $BOSH_ENVIRONMENT:8844 --client-name=nonprod_credhub_client --client-secret=$NONPROD_CREDHUB_SECRET --skip-tls-validation`

Store PivNet token (Pipelines)
1. `credhub set -n /pipeline/nonprod/pivnet-token -t value -v $PIVNET_API_TOKEN`

Store Root AWS credentials (Ops Manager)
1. `credhub set -n /pipeline/nonprod/opsman-aws-access-key -t value -v $ROOT_AWS_ACCESS_KEY`
    - `terraform input access_key`
1. `credhub set -n /pipeline/nonprod/opsman-aws-secret-key -t value -v "$ROOT_AWS_SECRET_KEY"`
    - `terraform input secret_key`

Store IAM AWS credentials (BOSH, PAS)
1. `credhub set -n /pipeline/nonprod/opsman-iam-access-key -t value -v $IAM_AWS_ACCESS_KEY`
    - `terraform output ops_manager_iam_user_access_key | pbcopy`
1. `credhub set -n /pipeline/nonprod/opsman-iam-secret-key -t value -v "$IAM_AWS_SECRET_KEY"`
    - `terraform output ops_manager_iam_user_secret_key | pbcopy`

Store Ops Manager auth credentials (Ops Manager)
1. `credhub set -n /pipeline/nonprod/opsman-admin-password -t value -v $OM_PASSWORD`
1. `credhub set -n /pipeline/nonprod/opsman-decryption-passphrase -t value -v $OM_DP`

Store Ops Manager SSH Key via (BOSH)
1. `credhub set -n /pipeline/nonprod/opsman-ssh -t rsa --private="$PRIVATE" --public="$PUBLIC"`
    - `terraform output ops_manager_ssh_private_key` -> `$PRIVATE`
    - `terraform output ops_manager_ssh_public_key` -> `$PUBLIC`

Store PAS Credhub Encryption Secret (PAS)
1. `credhub set -n /pipeline/nonprod/pas-credhub-encryption-secret -t value -v $PAS_CREDHUB_SECRET`
    - `$(openssl rand -base64 32)` -> `$PAS_CREDHUB_SECRET`

Store the Lets Encrypt certificates (PAS)
1. `credhub set -n /pipeline/nonprod/lets-encrypt-cert -t value -v "$CERT"`
    - `cat certs/cert.pem`
1. `credhub set -n /pipeline/nonprod/lets-encrypt-privkey -t value -v "$PRIVKEY"`
    - `cat certs/privkey.pem`
1. `credhub set -n /pipeline/nonprod/lets-encrypt-chain -t value -v "$CHAIN"`
    - `cat certs/chain.pem`

## X. Download deps

1. `fly login -t nonprod -c https://plane.cp.aws.63r53rk54v0r.com/ -n main`
1. `fly -t nonprod sp -p external-dependencies -c external-dependencies-pipeline.yml -l secrets.yml`
1. `fly -t nonprod up -p external-dependencies`


## X. Install all things

1. `fly -t nonprod sp -p install-all-things -c install-all-things-pipeline.yml -l secrets.yml`
1. `fly -t nonprod up -p install-all-things`

## Use nonprod

...

## stop all the things

Delete the installation (products, BOSH Director, and Ops Manager) via:
1. `fly -t nonprod sp -p destroy-foundation -c destroy-foundation-pipeline.yml -l secrets.yml`
1. `fly -t nonprod up -p destroy-foundation`
1. `fly -t nonprod tj -j destroy-foundation/destroy-foundation`

Unpave the IaaS via
1. `terraform destroy`

# TODOS

- create S3 buckets in control plane terraforming
- common descriptions/creation of Github repos and keys
- change to use CP credhub (not ops man credhub)
- use RDS for database instead of internal
- update certificates
- base pas configuration, with operations capturing features?
- use credhub for pipeline secrets