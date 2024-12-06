# Deploy TAS and TKGI with Concourse

!!! info
    This page is in progress

This page explains how to deploy TAS, TKGI and some related products on vSphere with NSX-T networking in an automated way using the Platform Automation Toolkit (Concourse).

We will use Concourse Pipelines from [my repo](https://github.com/Knappek/concourse-pipelines) which is a fork of a [public repo provided by Broadcom](https://github.com/pivotal/docs-platform-automation-reference-pipeline-config) which is used in the official docs providing a [full pipeline and reference configurations](https://docs.vmware.com/en/Platform-Automation-Toolkit-for-VMware-Tanzu/5.2/vmware-automation-toolkit/GUID-docs-pipelines-resources.html#full-pipeline-and-reference-configurations). 

!!! info
    All steps below assume you have forked the [Concourse Pipelines Repo](https://github.com/Knappek/concourse-pipelines) to your own account

## Prerequisites

- Platform Automation Toolkit running: You can use [this guide](./install-concourse-for-platform-automation.md) to install Concourse for Platform Automation on vSphere
- a vSphere + NSX-T environment. NSX-T must be preconfigured to meet the requirements to [deploy TAS for VMs with NSX-T Networking](https://docs.vmware.com/en/VMware-Tanzu-Application-Service/5.0/tas-for-vms/vsphere-nsx-t.html) and to [deploy TKGI on vSphere with NSX](https://docs.vmware.com/en/VMware-Tanzu-Kubernetes-Grid-Integrated-Edition/1.20/tkgi/GUID-vsphere-nsxt-index-install.html). You can foollow [this guide](./deploy-vsphere-with-nsxt-for-tas-tkgi.md) to achieve this
- Concourse CLI (fly), Credhub CLI, OM ClI, BOSH CLI: best to install them with [asdf](https://github.com/vmware-tanzu/tanzu-plug-in-for-asdf)
- the S3 buckets in MinIO have Versioning enabled

## Download Products

The first Concourse pipeline will download all required products from Broadcom Support Portal and store it on the local MinIO S3 instance.
The purpose is that you can then deploy all products without requiring internet access.

1. Login to Minio (http://<MINIO-IP>:9092) and create Access Keys - you'll need them later

1. Retrieve the Tanzu API Token (aka Pivnet API Token) from Broadcom Support:
   1. Login to [Broadcom Support Portal](https://support.broadcom.com/)
   1. On the right navigation bar, click on Tanzu API Token, which will navigate you to the [tanzu-token page](https://support.broadcom.com/group/ecx/tanzu-token)

1. If you haven't configured to git clone/pull/push from Github via SSH, [add a new SSH Key to your Github account](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/adding-a-new-ssh-key-to-your-github-account)
1. login to Concourse (which will also log you in to Credhub)

    ```shell
    source ./login_to_concourse
    ```

  If the command tells you to sync fly, simply execute the provided command.

1. Create all required entries in Credhub

    ```shell
    credhub set -n /concourse/main/s3_secret_access_key -t password -w <minio-secret-access-key>
    credhub set -n /concourse/main/s3_access_key_id -t password -w  <minio-access-key>
    credhub set -n /concourse/main/s3_endpoint -t value -v http://<MINIO-IP>:9091 # port 9091 is the API port, which is different from the UI port 9092
    credhub set -n /concourse/main/s3_pivnet_products_bucket -t value -v products
    credhub set -n /concourse/main/pivnet_token -t password -w <pivnet-api-token-from>
    credhub set -n /concourse/main/pipeline-git-repo-key -t ssh -p ~/.ssh/id_rsa # path to your private SSH key that you use to interact with Github
    credhub set -n /concourse/main/pipeline-git-repo-uri -t value -v git@github.com:<your Github Handle>/concourse-pipelines.git
    ```

1. Set the Concourse Pipeline

    ```shell
    ./scripts/update-download-products-pipeline.sh
    ```

1. Unpause the pipeline

    ```shell
    fly -t ci unpause-pipeline -p download-products
    ```

1. Navigate to the Concourse UI (https://<CONCOURSE-IP>) and follow the `download-products` pipeline. 
      1. The `fetch-platform-automation` job starts automatically after a few seconds if the `platform-automation-pivnet` input resource has been checked successfully.
      1. all other jobs will not be triggered automatically: I have done this on purpose, because I don't want to stress my network and have control when the pipeline downloads and uploads lots of data. Hence, you need to trigger them manually. You can do this [either on the UI or using the CLI](https://concourse-ci.org/jobs.html#fly-trigger-job).

## Deploy Foundation

The next Concourse pipeline `deploy-foundation` will be used to deploy TAS & TKGI and other related products. This pipeline can ultimately been used for different environments
by providing different variables files. We will deploy the `sandbox` environment. Adding other environments will be self-explanatory.

### Adapt Variables Files

Most of the required information can be retrieved from [Deploy vSphere + NSX-T to be used for TAS and TKGI](./deploy-vsphere-with-nsxt-for-tas-tkgi.md) and [Install Concourse for Platform Automation](./install-concourse-for-platform-automation.md).

All Variables files can be found [here](https://github.com/Knappek/concourse-pipelines/tree/main/foundations/sandbox/vars). 

Some notes to some variables that might be unclear:

* `nsx_ca_certificate` in `director.yml`: can be retrieved with `openssl s_client -connect <nsx_address>:443 -showcerts </dev/null`
* `pks_ssl_certificate` & `pks_ssl_private_key`: Assuming your TKGI API will be `api.tkgi.example.com`, self-signed SSL certificates can be generated with:

    ```shell
    openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
      -keyout api.tkgi.example.com.key \
      -out api.tkgi.example.com.crt \
      -subj "/C=US/ST=California/L=CA/O=TKGi/CN=api.tkgi.example.com" \
      -extensions SAN \
      -config <(cat /etc/ssl/openssl.cnf \
        <(printf "\n[SAN]\nsubjectAltName=DNS:api.tkgi.example.com"))
    ```

* `tas_ssl_certificate` & `tas_ssl_private_key`: Assuming one of your Gorouter IPs is `172.30.5.126`, self-signed SSL certificates can be generated with:

    ```shell
    openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
      -keyout apps.172.30.5.126.nip.io.key \
      -out apps.172.30.5.126.nip.io.crt \
      -subj "/C=US/O=Pivotal/CN=*.apps.172.30.5.126.nip.io" \
      -extensions SAN \
      -config <(cat /etc/ssl/openssl.cnf \
        <(printf "\n[SAN]\nsubjectAltName=DNS:*.apps.172.30.5.126.nip.io,DNS:*.sys.172.30.5.126.nip.io,DNS:sys.172.30.5.126.nip.io,DNS:*.172.30.5.126.nip.io,DNS:172.30.5.126.nip.io"))
    ```

### Set Pipeline

1. Create required entries in Credhub

    ```shell
    credhub set -n /concourse/main/s3_installation_bucket -t value -v installation
    credhub set -n /concourse/main/s3_foundation_state_bucket -t value -v foundation-state
    credhub set -n /concourse/main/nsx_credentials -t user -z admin -w 'VMware1!VMware1!'
    credhub set -n /concourse/main/opsman_decryption_passphrase -t password -w 'VMware1!VMware1!'
    credhub set -n /concourse/main/opsman_user -t user -z admin -w 'VMware1!'
    credhub set -n /concourse/main/vcenter_credentials -t user -z administrator@vsphere.local -w 'VMware1!'
    ```

1. Set the Concourse Pipeline

    ```shell
    ./scripts/update-sandbox-foundation-pipeline.sh
    ```

1. Unpause the pipeline

    ```shell
    fly -t ci unpause-pipeline -p deploy-sandbox-foundation
    ```

1. Install Opsman

    ```shell
    fly -t ci trigger-job -j deploy-sandbox-foundation/install-opsman
    ```
