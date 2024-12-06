# Deploy TAS and TKGI with Concourse

This page uses Concourse Pipelines from [my repo](https://github.com/Knappek/concourse-pipelines) which is a fork of a [public repo provided by Broadcom](https://github.com/pivotal/docs-platform-automation-reference-pipeline-config).

!!! info
    All steps below assume you have forked the [Concourse Pipelines Repo](https://github.com/Knappek/concourse-pipelines) to your own account

## Prerequisites

- Platform Automation Toolkit running: You can use [this guide](./install-concourse-for-platform-automation.md) to install Concourse for Platform Automation on vSphere
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
