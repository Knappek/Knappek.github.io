# Customize Kubernetes Auditing

This page describes how you can customize Kubernetes auditing for TKG Service Clusters deployed with [vSphere IaaS Control Plane](https://techdocs.broadcom.com/us/en/vmware-cis/vsphere/vsphere-supervisor/8-0.html).

## Prerequisites

- a running Supervisor Cluster (Workload Management is enabled in vCenter)
- using [TKG Service with vSphere Supervisor](https://techdocs.broadcom.com/us/en/vmware-cis/vsphere/vsphere-supervisor/8-0/using-tkg-service-with-vsphere-supervisor.html)

    !!! note
        This was tested using

        - TKG Service 3.2.0
        - Supervisor Cluster version 1.28.3
        - vCenter version 8.0.3 Build 24022515

## General

TKG Service Clusters deployed with the Supervisor Cluster are provisioned with [Kubernetes Auditing](https://kubernetes.io/docs/tasks/debug/debug-cluster/audit/) by default.

The audit policy used is

```yaml
kind: Policy
rules:
  # The following requests were manually identified as high-volume and low-risk,
  # so drop them.
  - level: None
    users: ["system:serviceaccount:kube-system:kube-proxy"]
    verbs: ["watch"]
    resources:
      - group: "" # core
        resources: ["endpoints", "services", "services/status"]
  - level: None
    userGroups: ["system:nodes"]
    verbs: ["get"]
    resources:
      - group: "" # core
        resources: ["nodes", "nodes/status"]
  - level: None
    users:
      - system:kube-controller-manager
      - system:kube-scheduler
      - system:serviceaccount:kube-system:endpoint-controller
    verbs: ["get", "update"]
    namespaces: ["kube-system"]
    resources:
      - group: "" # core
        resources: ["endpoints"]
  - level: None
    users: ["system:apiserver"]
    verbs: ["get"]
    resources:
      - group: "" # core
        resources: ["namespaces", "namespaces/status", "namespaces/finalize"]
  # Don't log HPA fetching metrics.
  - level: None
    users:
      - system:kube-controller-manager
    verbs: ["get", "list"]
    resources:
      - group: "metrics.k8s.io"
  # Don't log these read-only URLs.
  - level: None
    nonResourceURLs:
      - /healthz*
      - /version
      - /swagger*
  # Don't log events requests.
  - level: None
    resources:
      - group: "" # core
        resources: ["events"]
  # Don't log TMC service account performing read operations because they are high-volume.
  - level: None
    userGroups: ["system:serviceaccounts:vmware-system-tmc"]
    verbs: ["get", "list", "watch"]
  # Don't log read requests from garbage collector because they are high-volume.
  - level: None
    users: ["system:serviceaccount:kube-system:generic-garbage-collector"]
    verbs: ["get", "list", "watch"]
  # node and pod status calls from nodes are high-volume and can be large, don't log responses for expected updates from nodes
  - level: Request
    userGroups: ["system:nodes"]
    verbs: ["update","patch"]
    resources:
      - group: "" # core
        resources: ["nodes/status", "pods/status"]
    omitStages:
      - "RequestReceived"
  # deletecollection calls can be large, don't log responses for expected namespace deletions
  - level: Request
    users: ["system:serviceaccount:kube-system:namespace-controller"]
    verbs: ["deletecollection"]
    omitStages:
      - "RequestReceived"
  # Secrets, ConfigMaps, and TokenReviews can contain sensitive & binary data,
  # so only log at the Metadata level.
  - level: Metadata
    resources:
      - group: "" # core
        resources: ["secrets", "configmaps"]
      - group: authentication.k8s.io
        resources: ["tokenreviews"]
    omitStages:
      - "RequestReceived"
  # Get responses can be large; skip them.
  - level: Request
    verbs: ["get", "list", "watch"]
    resources:
      - group: "" # core
      - group: "admissionregistration.k8s.io"
      - group: "apiextensions.k8s.io"
      - group: "apiregistration.k8s.io"
      - group: "apps"
      - group: "authentication.k8s.io"
      - group: "authorization.k8s.io"
      - group: "autoscaling"
      - group: "batch"
      - group: "certificates.k8s.io"
      - group: "extensions"
      - group: "metrics.k8s.io"
      - group: "networking.k8s.io"
      - group: "policy"
      - group: "rbac.authorization.k8s.io"
      - group: "settings.k8s.io"
      - group: "storage.k8s.io"
    omitStages:
      - "RequestReceived"
  # Default level for known APIs
  - level: RequestResponse
    resources:
      - group: "" # core
      - group: "admissionregistration.k8s.io"
      - group: "apiextensions.k8s.io"
      - group: "apiregistration.k8s.io"
      - group: "apps"
      - group: "authentication.k8s.io"
      - group: "authorization.k8s.io"
      - group: "autoscaling"
      - group: "batch"
      - group: "certificates.k8s.io"
      - group: "extensions"
      - group: "metrics.k8s.io"
      - group: "networking.k8s.io"
      - group: "policy"
      - group: "rbac.authorization.k8s.io"
      - group: "settings.k8s.io"
      - group: "storage.k8s.io"
    omitStages:
      - "RequestReceived"
  # Default level for all other requests.
  - level: Metadata
    omitStages:
      - "RequestReceived"
```

## Customize Audit Policy

You can customize the audit policy by using a custom cluster class as described [here](https://techdocs.broadcom.com/us/en/vmware-cis/vsphere/vsphere-supervisor/8-0/using-tkg-service-with-vsphere-supervisor/provisioning-tkg-service-clusters/using-the-cluster-v1beta1-api/v1beta1-example-cluster-based-on-a-custom-clusterclass-vsphere-8-u2-and-later-workflow.html#GUID-EFE7DB40-8748-42B5-9694-DBC21F9FB76A-en_TITLE_77784069-5CE5-47E3-AB00-C32CC5C27912). 

To use this minimal policy file that logs all requests on at the Metadata level:

```yaml
# Log all requests at the Metadata level.
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
- level: Metadata
```

you can use [this custom cluster class](https://gist.github.com/Knappek/b6c7f931ef4c709d22d5bab66c00c5f8).

## Explore Audit Logs

### by SSH'ing into one of the Control Plane nodes

1. SSH to a TKG Service Cluster as described [here](https://docs.vmware.com/en/VMware-vSphere/8.0/vsphere-with-tanzu-tkg/GUID-10FA9D07-2928-4262-AA03-530D28ACFCC5.html)
1. switch to root user: `sudo su`
1. tail the logs: `tail -f /var/log/kubernetes/kube-apiserver.log`

### Ship to an external audit backend

TODO
