http://docs.pivotal.io/platform-automation/v2.1/index.html

/pipeline/nonprod

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
1. Terraform IaaS
1. Create `credhub-automation` user in credhub via Azure steps

1. Add secrets to credhub at `/pipeline/nonprod` (`pivnet-token`) via
    1. SSH into opsman
    1. `credhub api 10.0.16.5:8844 --ca-cert=/var/tempest/workspaces/default/root_ca_certificate`
    1. `credhub login --client-name=credhub-automation --client-secret=$CONCOURSE_CREDHUB_SECRET`
    1. `credhub set -n /pipeline/nonprod/pivnet-token -t value -v $PIVNET_API_TOKEN`

TODOS

- credhub for secrets
- create S3 buckets in control plane terraforming
- common descriptions/creation of Github repos and keys
- change to use CP credhub (not ops man credhub)
- use RDS for database instead of internal
- upload certificates