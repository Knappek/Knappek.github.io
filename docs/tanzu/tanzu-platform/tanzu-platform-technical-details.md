# Tanzu Platform Technical Details

## Goal of this page

The goal of this page is to add additional information around [Tanzu Platform for Kubernetes](https://docs.vmware.com/en/VMware-Tanzu-Platform/services/create-manage-apps-tanzu-platform-k8s/index.html), what you can't easily find in the official documentation, but what you would learn when working hands-on with it. This page tries to go into all necessary technical details required to better understand the concepts. By providing this, it should be easier to manage, own and troubleshoot.

!!! warning
    This page is in progress. Currently it just contains notes, there is no clear format or structure. It's mainly used by myself while studying TP4K8s and I will bring it into a good shape once I understand it better.

## Questions

- What happens when I don't enable `Tanzu Application Engine` when creating a cluster Group? Does it mean, the clusters in it are not managed by UCP?
- when interacting via kubectl with UCP (using kubeconfig `~/.config/tanzu/kube/config`), then you see different things depending which context you use (`project`, `clustergroup` or `space` context). Which objects do you see using which context?

## Concepts

There is a very good page explaining the concepts [here](https://docs.vmware.com/en/VMware-Tanzu-Platform/services/create-manage-apps-tanzu-platform-k8s/concepts-about-spaces.html). You have to be familiar with

- Spaces
- Capabilities
- Cluster Groups
- Availability Targets
- Profiles
- Traits

## Notes

- groupings (cluster groups to group clusters, availability targets to also group clusters, spaces to group namespaces) is all based on labels
- because of label selectors, a cluster might belong to one or many Availability Targets
- because of label selectors, a cluster might belong to one or many Cluster Groups
- capabilities are provided by Tanzu Packages (Carvel Packages) installed to every cluster belonging to the cluster group that provides this capability
- if a capability is *provided* to a cluster group, the corresponding Tanzu Package installed on all clusters in that cluster group have the `capability.tanzu.vmware.com/provides: <capability-name>` annotation
- Capabilities vs Traits
    - Capabilities provides the set of CRDs (with controllers)
    - Traits creates an instance of the CRD (CR - Custom Resource)
    - example:
        - the `cert-manager` capability deploys the `cert-manager` pods and installs the cert-manager CRDs, like `ClusterIssuer`, `Certificate` etc.
        - there is a `multicloud-certmanager` trait, which actually creates certificate issuers, so custom resources of `kind: ClusterIssuer` etc. and actually makes use of the `cert-manager` issuer

- Profiles is just a configuration but you don't really deploy anything by creating a profile. A space is ultimately combining everything to provision something with the information from a profile. This means, that all available capabilities are provided on the cluster group, but you can only use capabilities in your spaces if you have provided them in your space via the profile
- when creating a space you can select an Availability Target and with it how many replicas of the AT - when you configure 3 replicas, the UCP will create 3 managed namespaces across 3 clusters
- namespaces/pods added by Hub (UCP)
    - tanzu-cluster-group-system: used to deploy `PackageInstalls` of all capabilities installed on the cluster group
    - tanzu system
        - `aria-k8s-collector`
        - `syncer`
        - `tanzu-capabilities-controller-manager`
    - vmware-system-tmc
- Hub adds 4 Package Repositories that bundle all capabilities
- when you create a space, it will
    - create a `ManagedNamespaceSet` on UCP
    - which will create a `ManagedNamespace` on UCP
    - which will create two `Namespace`'s on the target cluster, called `<space-name>-<hash>` and `<space-name>-<hash>-internal`
        - the internal namespace is used to deploy `PackageInstalls` of the traits, referenced in the associated profiles
        - the "normal" namespace is used to deploy end applications
- a `Space`and a `ManagedNamespace` (and the final `Namespace` on the target cluster(s) - which is essentially a 1:n mirror from a `ManagedNamespace`) can be compared to a `Deployment` and a `Pod`
    - each time you update the Space, it will create a new `ManagedNamespaceSet`
- `Spaces` are reconciled by a `Space Controller`
- there is a dedicated ManagedNamespace for each AvailabilityTarget of a ManagedNamespaceSet
- once a ManagedNamespace has been created successfully on UCP, there is the `Space Scheduler` that provisions a namespace on a target cluster and installs the traits into that namespace

## FAQ

### When I have 1 availability target with 3 clusters and I choose to

- deploy a space with 1 replica: will the namespace get deployed on one cluster only? => yes
- deploy a space with 2 replicas: will 2 namespaces get deployed on one cluster or 1 namespace on each cluster? => 2 namespaces on one cluster is possible
- deploy a space with 3 replicas: will the namespace get deployed evenly, 1 namespace on each cluster? => no. 3 namespaces on one cluster is possible
- deploy a space with more replicas than available clusters => explained by above behaviour

### Who are the personas and who is responsible for what?

Where are the boundaries? What's the motivation behind the technical architecture - who is supposed to see what and what is intended to be abstracted away? => https://docs.google.com/document/d/1Rmt-eskKBo2mNDsSdHeVaIWuZU3zhKznEDcRVN9BRRs/edit#heading=h.ixfosuy76boz

### Is there is a dedicated ManagedNamespace for each AvailabilityTarget of a ManagedNamespaceSet

Yes. If you deploy a space to two Availability Targets with one replica each, there is one `ManagedNamespaceSet` on UCP, and two `ManagedNamespace`'s, one for each cluster.
