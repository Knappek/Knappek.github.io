# Updating vSphere with Tanzu from 7 to 8 to Support TKG Clusters on Supervisor

We are following the official docs [here](https://docs.vmware.com/en/VMware-vSphere/8.0/vsphere-with-tanzu-tkg/GUID-F68DF779-F52E-4970-8460-6177BF601DC2.html).

:::warning
This page is still in progress.
:::

:::note
This page explains updating vCenter from 7u3 to 8u2c, and updating the vSphere with Tanzu environment (Supervisor cluster and guest clusters) to the latest version. We do not upgrade the underlying ESXi host There is also a guest cluster deployed with various Tanzu packages deployed which are updated as well.
:::

## Versions

| Component |  Before update | After update  |
|---|---|---|
| ESXi  | 7.0.3 22837322  | 7.0.3 22837322  |
| vCenter  |  7u3p (Build 22837322) |  8u2c (Build 23504390) |
| NSX-T  |  4.1.2.1.0.22667789   |  4.1.2.1.0.22667789  |
|  Supervisor cluster |  v1.24.9 |  v1.26.8 |
| Guest Cluster | v1alpha2, TKR 1.23.8  |   |
|  Harbor  | 2.3.3+vmware.1-tkg.1  |   |
|  Contour | 1.20.2+vmware.1-tkg.1  |   |
|  cert-manager | 1.7.2+vmware.1-tkg.1  |   |
|  kapp-controller | v0.30.0_vmware.1  |   |

## Procedure

### Upgrade Supervisor cluster

1. Update Supervisor cluster to `v1.25.6`
1. update vCenter from 7 to 8 by following the official docs [here](https://docs.vmware.com/en/VMware-vSphere/8.0/vsphere-vcenter-upgrade/GUID-30485437-B107-42EC-A0A8-A03334CFC825.html).
1. Update Supervisor cluster to `v1.25.13`. When upgrading the Supervisor cluster in vCenter 8u2c you have to apply a workaround to a [known issue](https://ikb.vmware.com/s/article/97660).
1. Update Supervisor cluster to `v1.26.8`.
