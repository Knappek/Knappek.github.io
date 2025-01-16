# Using Cloud Foundry

I have built a lab environment for TAS using the Platform Automation Toolkit (Concourse) which is described [here](./../platform-automation-toolkit/index.md).

## Managing Orgs and Spaces from Concourse

To manage Cloud Foundry [Orgs and Spaces](https://docs.cloudfoundry.org/concepts/roles.html) you can use the [cf-mgmt](https://github.com/vmware-tanzu-labs/cf-mgmt) that enables you to declaratively create orgs and spaces and manage owners, permissions, quotas and more using Concourse.

You can find an example of this used in my lab setup mentioned above [here](https://github.com/Knappek/cf-mgmt).
