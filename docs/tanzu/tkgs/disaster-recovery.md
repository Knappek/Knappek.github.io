# Disaster Recovery

This page explores certain disaster recovery use cases, how Tanzu behaves and how to repair the environment.

!!! info
    This page is based on [vSphere IaaS Control Plane](https://docs.vmware.com/en/VMware-vSphere/8.0/vsphere-with-tanzu-concepts-planning/GUID-28B0AEA2-2947-4FDD-AA71-51E46E24BF53.html), formerly known as vSphere with Tanzu aka TKGS. You should first know about the basic concepts before reading this page.

## Crash of a Worker Node on a guest cluster

### Power off Worker node in vCenter

What happens if we manually power off a Virtual Machine in vCenter which is a worker node of a guest cluster?

- the corresponding Kubernetes node enters the state `NotReady`
- the Supervisor Cluster still shows the corresponding `Machine` (see [Cluster API Machine](https://cluster-api.sigs.k8s.io/developer/architecture/controllers/machine)) to be `Running` and `VirtualMachine` (a [CAPV](https://github.com/kubernetes-sigs/cluster-api-provider-vsphere) object) to be `poweredOn`
- pods that should be scheduled on that node are in `Pending` state
- logs in `capi-controller-manager` (from MachineSet Controller):

    ```text
    Waiting for the Kubernetes node on the machine to report ready state
    ```

- after 5 minutes, the Supervisor Cluster (actually the Cluster API controllers) deletes the Virtual Machine in vCenter and the Kubernetes nodes gets deleted
- finally a new Virtual Machine gets created and the node joins the cluster

### Delete Worker VM in vCenter (Delete from Disk)

The behavior is quite similar to powering off the VM:

- corresponding Kubernetes node marked as `NotReady`
- after a few minutes the nodes enter `SchedulingDisabled`, but the Supervisor Cluster still shows the `Machine` to be running
- after some time, logs in capi-controller-manager:

    ```log
    reason="UnhealthyNode" message="Condition Ready on node is reporting status Unknown for more than 5m0s"
    ```

- after some minutes the `Machine` enters `Deleting` state
- the Supervisor Cluster provisions a new node with the same name
- node persists in state `NoteReady,SchedulingDisabled` because it is not reachable (as kubelet is not running)
- the `Machine` is still in `Deleting` state
- after some time the VM gets successfully deleted by CAPI
- a new VM with new name gets provisioned and node joins the cluster and operating successfully
- we have a functioning cluster again 13 minutes after we have deleted the VM

## Crash of a Control Plane Node on a guest cluster

### Power off Control Plane node of guest cluster

- Kubernetes API Server not available for a few seconds. This is because we were connected to that node that we powered off, if we would have powered off another CP node, we wouldn't even have recognized it
- VM powered on immediately after a few seconds
- It is that quick that no pods get terminated

### Delete CP VM in vCenter (Delete from Disk)

- in less than a minute a new VM with the same name gets deployed in vCenter and powered on
- the node is still marked `NotReady` because kubelet is not running and printing logs:

    ```
    "command failed" err="failed to load kubelet config file, error: failed to load Kubelet config file /var/lib/kubelet/config.yaml, error failed to read kubelet config file \"/var/lib/kubelet/config.yaml\", error: open /var/lib/kubelet/config.yaml: no such file or directory, path: /var/lib/kubelet/config.yaml"
    ```

- this indicates that kubeadm does not join the control plane successfully to the cluster. Indeed, the `cloud-init-output.log` file prints

    ```
    -info ConfigMap does not yet contain a JWS signature for token ID "5j3q2p", will try again
    [2024-09-12 14:52:39] I0912 14:52:39.589511    1244 token.go:223] [discovery] The cluster-info ConfigMap does not yet contain a JWS signature for token ID "5j3q2p", will try again
    [2024-09-12 14:52:43] error execution phase preflight: couldn't validate the identity of the API Server: could not find a JWS signature in the cluster-info ConfigMap for token ID "5j3q2p"
    [2024-09-12 14:52:43] To see the stack trace of this error execute with --v=5 or higher
    [2024-09-12 14:52:43] !!! [2024-09-12T14:52:43+00:00] kubeadm reported failed action(s) for 'kubeadm join phase preflight --ignore-preflight-errors=DirAvailable--etc-kubernetes-manifests'
    [2024-09-12 14:52:58] +++ [2024-09-12T14:52:58+00:00] running 'kubeadm join phase preflight --ignore-preflight-errors=DirAvailable--etc-kubernetes-manifests'
    ```

- deploying pods is still working, although the control plane node is in `NotReady` state and other etcd pods are not able to connect to this etcd instance. The other two etcd instances are logging

    ```
    "dial tcp 10.244.0.34:2380: connect: connection refused"
    ```

- after a few minutes the node enters `SchedulingDisabled`
- after some more minutes, the `Machine` enters `Deleting` state, but  thge VirtualMachine is still `poweredOn`
- after some time the VM gets deleted and a new one with a new name gets provisioned and joined to the cluster successfully
- we have a functioning cluster again ~15 minutes after we have deleted the VM

## Kubelet Crashing

### Kubelet not running on a worker node

#### Stop kubelet

In this scenario, we simply stopped kubelet service on a node.

The behavior is same as if you [Power off a Worker node in vCenter](#power-off-worker-node-in-vcenter).

#### Let kubelet crash

In this scenario, we removed the kubelet config file.

The behavior is same as if you [Power off a Worker node in vCenter](#power-off-worker-node-in-vcenter).

## etcd not working properly

### only 1 out of 3 etcd instances running

- if you have only 2 etcd instances and you kill one of it, etcd tries to do a new leader election, also if you explicitly don't kill the leader
- because etcd doesn't reach a quorum, as there is only 1 out of 3 instances running, the remaining instance will never become a leader
- as a result, `kube-apiserver` is failing with the error `watch chan error: etcdserver: no leader` - apparently when apiserver receives this error, it terminates all watchers according to [this description](https://github.com/kubernetes/kubernetes/issues/111116)
- the apiserver is not able to recreate the watchers because there is no etcd leader
- Tanzu (Cluster API) does not recreate a node with a crashing etcd

As a result:

- you are not able to communicate with the cluster using `kubectl` (or else to the API Server)
- because etcd is in readonly mode, the self-healing mechanism in Kubernetes doesn't work anymore
- running apps on the cluster are still working
- ingress to web apps is still functioning
