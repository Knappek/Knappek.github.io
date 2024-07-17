# Tanzu Platform Technical Details

## Questions

- who are the personas and who is responsible for what? where are the boundaries? What's the motivation behind the technical architecture - who is supposed to see what and what is intended to be abstracted away?
- when I have 3 availability targets (hence 3 clusters) and I choose to
  - deploy a space with 1 replica: will the namespace get deployed on one cluster only?
  - deploy a space with 2 replicas: will 2 namespaces get deployed on one cluster or 1 namespace on each cluster?
  - deploy a space with 3 replicas: will the namespace get deployed evenly, 1 namespace on each cluster?
- What happens when I don't enable `Tanzu Application Engine` when creating a cluster Group? Does it mean, the clusters in it are not managed by UCP?

## Concepts

There is a very good page explaining the concepts [here](https://docs.vmware.com/en/VMware-Tanzu-Platform/services/create-manage-apps-tanzu-platform-k8s/concepts-about-spaces.html). You have to be familiar with

- Spaces
- Capabilities
- Cluster Groups
- Availability Targets
- Profiles
- Traits

### Notes

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
- 
