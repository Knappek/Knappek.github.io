# Install Concourse for Platform Automation

This guide follows the official documentation to [Installing Concourse for Platform Automation](https://docs.vmware.com/en/Concourse-for-VMware-Tanzu/7.0/vmware-tanzu-concourse/GUID-installation-platform-automation-index.html) but uses Ansible to automate the deployment on vCenter.

The Ansible playbook will:

- provision Opsman (VMware Operations Manager) in a VM
- configure BOSH director
- deploy Concourse as a BOSH release
- deploy MinIO as a Single-Node Single-Drive in a Ubuntu VM

The goal is to have a fully functioning Platform Automation Toolkit that can be used to install Tanzu Application Service or Tanzu Kubernetes Grid Integrated Edition. MinIO can be used to store the required artifacts.

## Prerequisites

- a running vCenter (["Management"](./index.md)): tested on 8.0.3 but should work on version 7 as well
- all required products need to be pre-downloaded from Broadcom Support Portal:
    - [Opsman](https://support.broadcom.com/group/ecx/productdownloads?subfamily=VMware%20Tanzu%20Operations%20Manager)
    - [all Concourse products](https://support.broadcom.com/group/ecx/productdownloads?subfamily=VMware%20Tanzu%20Operations%20Manager): Concourse Bosh deployment, Concourse BOSH Release, BPM, Postgres, UAA, Credhub, Backup and Restore SDK
    - [Ubuntu Jammy Stemcell](https://support.broadcom.com/group/ecx/productdownloads?subfamily=Stemcells%20(Ubuntu%20Jammy)) (you can also use any other stemcell)
- Ubuntu OVA downloaded, e.g. [noble-server-cloudimg](https://cloud-images.ubuntu.com/noble/current/noble-server-cloudimg-amd64.ova)

## Deployment

We will use [vmware-lab-builder](https://github.com/laidbackware/vmware-lab-builder) to bootstrap the infrastructure for the 
Platform Automation Toolkit using [this vars yaml](https://github.com/laidbackware/vmware-lab-builder/blob/main/var-examples/tanzu/platform-automation-toolkit/opinionated-not-nested.yml). This will provision all resources mentioned above non-nested, which means it will deploy the VMs on your hosting vCenter. See further details how to run the Ansible playbook to deploy this lab in my [Homelab Section](./../../homelab/index.md#nested-lab-setup).

Once, the Platform Automation Toolkit is running, you can access Concourse UI and MinIO UI to verify the deployment. Before we can actually create our Concourse pipelines to deploy TAS or TKGI, we first have to [Deploy vSphere + NSX-T for TAS and TKGI](./deploy-vsphere-with-nsxt-for-tas-tkgi.md).
