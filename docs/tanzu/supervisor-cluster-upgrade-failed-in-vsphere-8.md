# Supervisor Cluster upgrade failed in vSphere 8u2b

## Issue Description

When I upgraded the Supervisor cluster from 1.25.6 to 1.26.8 on vCenter version 8u2b which has been recently upgraded from 7u3p, I faced an issue that the upgrade process was stuck at 50% in vCenter and the `vsphere-csi-controller` pods in namespace `vmware-system-csi` were in `CrashLoopBackOff` resulting in the error

```markdown
failed to init controller. Error: could not find any AvailabilityZone
```

and `kubectl get az` on the Supervisor Clusters doesn't  show any output.

When [ssh'ing into a Supervisor Cluster Control Plane node](https://mappslearning.wordpress.com/2021/12/01/ssh-login-into-vsphere-with-tanzu-supervisor-cluster-nodes/) you can execute the following script to find more details of the upgrade process with

```shell
/usr/lib/vmware-wcp/upgrade/upgrade-ctl.py get-status \
  | jq '.progress | to_entries | .[] | "\(.value.status) - \(.key)"' \
  | sort
```

This gave me the following result:

```markdown
"failed - ImageRegistryUpgrade"
"pending - CapvUpgrade"
"pending - CertManagerAdditionalUpgrade"
"pending - LicenseOperatorControllerUpgrade"
"pending - NamespaceOperatorControllerUpgrade"
"pending - PinnipedUpgrade"
"pending - PspOperatorUpgrade"
"pending - TkgUpgrade"
"pending - TMCUpgrade"
"pending - VmOperatorUpgrade"
"processing - UtkgControllersUpgrade"
"skipped - HarborUpgrade"
"skipped - LoadBalancerApiUpgrade"
"upgraded - AKOUpgrade"
"upgraded - AppPlatformOperatorUpgrade"
"upgraded - CapwUpgrade"
"upgraded - CertManagerUpgrade"
"upgraded - CsiControllerUpgrade"
"upgraded - ExternalSnapshotterUpgrade"
"upgraded - ImageControllerUpgrade"
"upgraded - KappControllerUpgrade"
"upgraded - NetOperatorUpgrade"
"upgraded - NSXNCPUpgrade"
"upgraded - RegistryAgentUpgrade"
"upgraded - SchedextComponentUpgrade"
"upgraded - SphereletComponentUpgrade"
"upgraded - TelegrafUpgrade"
"upgraded - UCSUpgrade"
"upgraded - UtkgClusterMigration"
"upgraded - VMwareSystemLoggingUpgrade"
"upgraded - WCPClusterCapabilities"
```

where we can see that the `ImageRegistryUpgrade` failed. Looking further into `var/log/VMware/upgrade-ctl-compupgrade.log` we could see the following error:

```text
2024-06-18T05:33:08.437Z DEBUG comphelper: ret=0 out={
    "apiVersion": "v1",
    "items": [
        {
            "apiVersion": "imageregistry.vmware.com/v1alpha1",
            "kind": "ContentLibrary",
            "metadata": {
                "annotations": {
                    "kubectl.kubernetes.io/last-applied-configuration": "{\"apiVersion\":\"imageregistry.vmware.com/v1alpha1\",\"kind\":\"ContentLibrary\",\"metadata\":{\"annotations\":{},\"name\":\"cl-fe4e8c74d59491be7\",\"namespace\":\"test-vsphere-namespace\"},\"spec\":{\"uuid\":\"5f773a1c-5aa3-4268-871a-359401c55950\",\"writable\":false}}\n"
2024-06-18T05:33:08.438Z ERROR compupgrade: {"error": "TypeError", "message": "argument of type 'NoneType' is not iterable", "backtrace": ["  File \"/usr/lib/vmware-wcp/upgrade/compupgrade.py\", line 362, in do_upgrade_with_out_resume_failed_support\n    comp.doUpgrade(upCtx)\n", "  File \"/usr/lib/vmware-wcp/objects/image-registry-operator/imageregistry_upgrade.py\", line 323, in doUpgrade\n    self.updateV1alpha1ImageRegistryResources()\n", "  File \"/usr/lib/vmware-wcp/objects/image-registry-operator/imageregistry_upgrade.py\", line 171, in updateV1alpha1ImageRegistryResources\n    self.updateV1alpha1Resource('contentlibraries', True)\n", "  File \"/usr/lib/vmware-wcp/objects/image-registry-operator/imageregistry_upgrade.py\", line 193, in updateV1alpha1Resource\n    patch_list = self.getResourcePatchBody(status, resource_kind)\n", "  File \"/usr/lib/vmware-wcp/objects/image-registry-operator/imageregistry_upgrade.py\", line 218, in getResourcePatchBody\n    if 'UTC' in creation_time:\n"]}
```

The error `argument of type 'NoneType' is not iterable` first indicated that the root cause was because of the missing Availability Zone as mentioned earlier. But digging into the logs of the `imageregistry-controller-manager` we could see the following errors:

```text
"msg"="Reconciler error" "error"="The underlying content library with ID 5f773a1c-5aa3-4268-871a-359401c55950 does not exist in vSphere"
```

## Root Cause

There is an operator introduced in vCenter 8.x called `ImageRegistryOperator` that takes over some of the responsibilities of `VMoperator`. Part of the upgrade script is to migrate VMoperator's `ContentSource` and `ContentSourceBindings` to newer CRDs (`ContentLibrary`, `ClusterContentLibrary`, `ContentLibraryItems` etc.).

There was a bug in the upgrade script (already fixed in newer versions of vCenter) that is exposed when a Supervisor namespace references a content library that does not exist, e.g. because it was removed from vCenter. Other component upgrades stalled out because they depend on `ImageRegistryUpgrade` to be finished - this also caused the `CrashLoopBackOff` of the `csi-controller` because `VmOperatorUpgrade` hasn't started yet and has not yet created the missing `AvailabilityZone` custom resource on the Supervisor Cluster.

In this case, the content library `5f773a1c-5aa3-4268-871a-359401c55950` has been already deleted in vCenter, but it's still associated on some namespaces. This can be seen using the [VMware Datacenter CLI (dcli)](https://www.techcrumble.net/2019/01/how-to-use-vmware-datacenter-cli-dcli/):

```shell
dcli> namespaces instances get --namespace <vsphere-namespace-name>
```

or when executing `kubectl get contentsourcebinding -A` on the Supervisor Cluster. Also, the orphaned `ContentLibrary` resource is still present on the Supervisor cluster when executing `kubectl get contentlibrary`.

## Resolution

1. delete the content library reference on all vSphere namespaces using dcli:

    ```shell
    dcli> namespaces instances update --namespace <vsphere-namespace-name> --content-libraries '[]'
    ```

1. delete all `contentsourcbindings` related to the orphaned Content Library with

    ```shell
    kubectl delete contentsourcebinding -n <vsphere-namespace-name> 5f773a1c-5aa3-4268-871a-359401c55950
    ```

1. delete the orphaned `contentlibrary` reference on the Supervisor cluster with

    ```shell
    kubectl delete contentlibrary <name>
    ```

1. ensure the last step also deleted the corresponding `contentsource` with `kubectl get contentsource`.

The Supervisor Cluster upgrade process will be retried automatically. You can follow the progress again with

```shell
/usr/lib/vmware-wcp/upgrade/upgrade-ctl.py get-status \
  | jq '.progress | to_entries | .[] | "\(.value.status) - \(.key)"' \
  | sort
```

and you should see that `ImageRegistryUpgrade` process should complete successfully. After `VmOperatorUpgrade` has been completed you should also see the missing `AvailabilityZone` with `kubectl get az` and the `csi-controller` pods running successfully.
