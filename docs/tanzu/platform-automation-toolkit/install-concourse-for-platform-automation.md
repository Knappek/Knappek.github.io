# Install Concourse for Platform Automation

!!! info
    This page is in Progress

This guide follows the official documentation to [Installing Concourse for Platform Automation](https://docs.vmware.com/en/Concourse-for-VMware-Tanzu/7.0/vmware-tanzu-concourse/GUID-installation-platform-automation-index.html) but uses Ansible to automate the deployment on vCenter.

The Ansible playbook will:
- provision Opsman (VMware Operations Manager) in a single VM
- configure BOSH director
- deploy Concourse as a BOSH release
- deploy MinIO as a Single-Node Single-Drive in a Ubuntu VM

The goal is to have a fully functioning Platform Automation Toolkit that can be used to install Tanzu Application Service or Tanzu Kubernetes Grid Integrated Edition. MinIO can be used to store the required artifacts.

## Prerequisites

- vCenter: tested on 8.0.3 but should work on version 7 as well
- all required products need to be pre-downloaded from Broadcom Support Portal:
  - [Opsman](https://support.broadcom.com/group/ecx/productdownloads?subfamily=VMware%20Tanzu%20Operations%20Manager)
  - [all Concourse products](https://support.broadcom.com/group/ecx/productdownloads?subfamily=VMware%20Tanzu%20Operations%20Manager): Concourse Bosh deployment, Concourse BOSH Release, BPM, Postgres, UAA, Credhub, Backup and Restore SDK
  - [Ubuntu Jammy Stemcell](https://support.broadcom.com/group/ecx/productdownloads?subfamily=Stemcells%20(Ubuntu%20Jammy)) (you can also use any other stemcell)
- Ubuntu OVA, e.g. [noble-server-cloudimg](https://cloud-images.ubuntu.com/noble/current/)

