# Set up an airgapped TKGS homelab environment

We will create a fully airgapped TKGS (aka vSphere with Tanzu) environment. This is a common setup of lots of our customers who run vSphere with Tanzu without any internet connection, not even via a proxy.

As described in [Nested Lab Setup](./index.md#nested-lab-setup) we will use [vmware-lab-builder](https://github.com/laidbackware/vmware-lab-builder) to bootstrap nested lab environments. This Ansible playbook is not (yet) implemented to work in airgapped environments as it creates a [subscribed content library](https://docs.vmware.com/en/VMware-vSphere/8.0/vsphere-with-tanzu-tkg/GUID-C8096867-6C18-4CB2-8803-C95100D54F8B.html). Hence, we will first run `vmware-lab-builder` with internet access to bootstrap a TKGS Supervisor Cluster and afterwards, we create a [Local Content Library](https://docs.vmware.com/en/VMware-vSphere/8.0/vsphere-with-tanzu-tkg/GUID-19E8E034-5256-4EFC-BEBF-D4F17A8ED021.html) to provision guest clusters without internet access.

## Initial Setup with internet access

As mentioned in the [lab network setup](./index.md#networking-routing) we use a virtualized VyOS router to route all traffic in our homelab. For the TKGS with NSX-T environment we configure the following interface in VyOS

```shell
set interfaces ethernet eth1 vif 16 address 172.20.16.1/22
set interfaces ethernet eth1 vif 16 description tanzu-without-dhcp
set interfaces ethernet eth1 vif 16 mtu 9000
set interfaces ethernet eth1 vif 20 address 172.20.20.1/22
set interfaces ethernet eth1 vif 20 description nsxt-tep
set interfaces ethernet eth1 vif 20 mtu 9000
commit 
save
```

The result is:

```shell
vyos@vyos# show interfaces
 ethernet eth0 {
     address 192.168.178.101/24
     hw-id 00:0c:29:85:a5:3b
     offload {
         gro
         gso
         sg
         tso
     }
 }
 ethernet eth1 {
     hw-id 00:0c:29:85:a5:45
     mtu 9000
     vif 16 {
         address 172.20.16.1/22
         description tanzu-without-dhcp
         mtu 9000
     }
     vif 20 {
         address 172.20.20.1/22
         description nsxt-tep
         mtu 9000
     }
 }
 loopback lo {
 }
```

where `vif 16` is used for all Virtual Machines and `vif 20` for the [NSX-T TEP network infrastructure](https://docs.vmware.com/en/VMware-vSphere/8.0/vsphere-with-tanzu-installation-configuration/GUID-E1C7E1DD-8D5D-4BEF-BBB0-61A7F7EBFB7A.html).

For the Supervisor Management Network, for the Egress and Ingress range, we create a VyOS protocol (static route)

```shell
set protocols static route 172.30.4.0/24 next-hop 172.20.16.103
commit
save
```

The result is:

```shell
vyos@vyos# show protocols
 static {
     route 172.30.4.0/24 {
         next-hop 172.20.16.103 {
         }
     }
 }
```

here, `172.20.16.103` will be the Tier-0 Gateway sitting on `vif 16`.

My `vmware-lab-builder` config looks like this:

```yaml
---
# SOFTWARE_DIR must contain all required software
vc_iso: "{{ lookup('env', 'SOFTWARE_DIR') }}/VMware-VCSA-all-7.0.3-19717403.iso"
esxi_ova: "{{ lookup('env', 'SOFTWARE_DIR') }}/Nested_ESXi7.0u3c_Appliance_Template_v1.ova"
nsxt_ova: "{{ lookup('env', 'SOFTWARE_DIR') }}/nsx-unified-appliance-4.1.2.1.0.22667794.ova"

environment_tag: "tkgs-nsxt"  # Used to prepend object names in hosting vCenter
dns_server: "192.168.178.1"
dns_domain: "home.local"
ntp_server_ip: "192.168.178.1"  # Must be set to an IP address!
disk_mode: thin  # How all disks should be deployed
nested_host_password: "{{ opinionated.master_password }}"

hosting_vcenter:  # This is the vCenter which will be the target for nested vCenters and ESXi hosts
  ip: "192.168.178.102"
  username: "{{ lookup('env', 'PARENT_VCENTER_USERNAME') }}"
  password: "{{ lookup('env', 'PARENT_VCENTER_PASSWORD') }}"
  datacenter: "Home"  # Target for all VM deployment

# This section describes what will be created
opinionated:
  master_password: "VMware1!"
  nested_hosts:
    cpu_cores: 12  # CPU count per nested host
    ram_in_gb: 128  # memory per nested host
    local_disks:
      - size_gb: 500
        datastore_prefix: "datastore"
  hosting_cluster: Physical
  hosting_datastore: NVME
  hosting_network:
    base:
      port_group: tanzu-without-dhcp
      cidr: "172.20.16.0/22"
      gateway: "172.20.16.1"
      # A NSX-T deployment requires 4 IPs, plus 1 per esxi host. They MUST be contiguous.
      starting_addr: "172.20.16.100"
    # nsxt tep pool will not be routed, but should not clash with routeable ranges
    nsxt_tep:
      port_group: nsxt-tep
      vlan_id: 0
      cidr: "172.20.20.0/22"  # Should be at least a 29 which supports up to 5 hosts and 1 edge
  tanzu_vsphere:
    # This network must be minimum of a /25
    # 1/8 of the network will be used for the supervisor
    # 3/8 of the network will be used for egress IP range
    # 1/2 of the network will be used for ingress IP range
    routeable_super_net: 172.30.4.0/24
    # This network is private and should not overlap with any routable networks
    internal_pod_network: 172.32.0.0/22
    # This network is private and should not overlap with any routable networks
    internal_kubernetes_services_network: 172.32.4.0/22

#####################################################################
### No need to edit below this line for an opinionated deployment ###
#####################################################################

nested_vcenter:  # the vCenter appliance that will be deployed
  ip: "{{ opinionated.hosting_network.base.starting_addr }}"  # vCenter ip address
  mask: "{{ opinionated.hosting_network.base.cidr.split('/')[1] }}"
  gw: "{{ opinionated.hosting_network.base.gateway }}"
  host_name: "{{ opinionated.hosting_network.base.starting_addr }}"  # FQDN if there is working DNS server, otherwise put the ip as a name
  username: "administrator@vsphere.local"
  password: "{{ opinionated.master_password }}"
  datacenter: "Lab"  # DC to create after deployment
  # Below are properties of parent cluster
  hosting_network: "{{ opinionated.hosting_network.base.port_group }}"  # Parent port group where the vCenter VM will be deployed
  hosting_cluster: "{{ opinionated.hosting_cluster }}"  # Parent cluster where the vCenter VM will be deployed
  hosting_datastore: "{{ opinionated.hosting_datastore }}"  # Parent datastore where the vCenter VM will be deployed

nested_clusters:  # You can add clusters in this section by duplicating the existing cluster
  compute:  # This will be the name of the cluster in the nested  vCenter. Below are the minimum settings.
    enable_drs: true
    enable_ha: true
    # Below are properties of the hosting cluster
    hosting_cluster: "{{ opinionated.hosting_cluster }}"  # The nested ESXi VMs will be deployed here
    hosting_datastore: "{{ opinionated.hosting_datastore }}"  # Datastore target for nested ESXi VMs
    # Settings below are assigned to each host in the cluster
    vswitch0_vm_port_group_name: vm-network
    vswitch0_vm_port_group_vlan: "0"
    cpu_cores: "{{ opinionated.nested_hosts.cpu_cores }}"  # CPU count
    ram_in_gb: "{{ opinionated.nested_hosts.ram_in_gb }}"  # memory
    # In order list of disks to assign to the nested host. All will be marked as SSD.
    # Datastore names will be automatically be pre-pended with the hostname. E.g esx1
    # If the datastore_prefix property is removed the disk will not be set as a datastore
    # To leave the default OVA disks in place, delete this section.
    nested_hosts_disks: "{{ opinionated.nested_hosts.local_disks | default(omit) }}"
    # Added in vmnic order, these port groups must exist on the physical host
    # Must specify at least 2 port groups, up to a maximum of 10
    vmnic_physical_portgroup_assignment:
      - name: "{{ opinionated.hosting_network.base.port_group }}"
      - name: "{{ opinionated.hosting_network.nsxt_tep.port_group }}"

nested_hosts:
  - name: "esx1"
    ip: "{{ opinionated.hosting_network.base.starting_addr | ansible.utils.ipmath(4) }}"
    mask: "{{ opinionated.hosting_network.base.cidr | ansible.utils.ipaddr('netmask') }}"
    gw: "{{ opinionated.hosting_network.base.gateway }}"
    nested_cluster: compute

distributed_switches:  # (optional) - section can be removed to not create any distributed switches
  - vds_name: nsxt-vds
    mtu: 1600
    vds_version: 7.0.0  # Should be 7.0.0, 6.7.0
    clusters:  # distributed switch will be attached to all hosts in the clusters listed
      - compute
    uplink_quantity: 1
    vmnics:
      - vmnic1


tspbm:  # Tag-based Storage Policy Based Management
  tag_categories:
    - category_name: tkgs-storage-category
      description: "TKGS tag category"
      tags:
        - tag_name: tkgs-storage-tag
          description: "Tag for datastores used by TKGS"
  datastore_tags:
    - datastore_name: "{{ opinionated.nested_hosts.local_disks[0].datastore_prefix }}-esx1"
      tag_names:
        - tkgs-storage-tag
  vm_storage_policies:
    - storage_policy_name: tkgs-storage-policy
      description: "TKGS storage performance policy"
      tag_name: tkgs-storage-tag
      tag_category: tkgs-storage-category

tanzu_vsphere:
  services_cidr: "{{ opinionated.tanzu_vsphere.internal_kubernetes_services_network }}"  # This is private within each cluster
  content_library_datastore: "{{ opinionated.nested_hosts.local_disks[0].datastore_prefix }}-esx1"
  content_library_name: tkgs-library
  content_library_url: "http://wp-content.vmware.com/v2/latest/lib.json"
  default_content_library: tkgs-library
  dns_server_list: ["{{ dns_server }}"]
  ephemeral_storage_policy: "{{ tspbm.vm_storage_policies[0].storage_policy_name }}"
  fluentbit_enabled: false
  image_storage_policy: "{{ tspbm.vm_storage_policies[0].storage_policy_name }}"
  ntp_server_list: ["{{ ntp_server_ip }}"]
  management_dns_servers: ["{{ dns_server }}"]
  management_port_group: supervisor-seg
  management_gateway: "{{ opinionated.tanzu_vsphere.routeable_super_net | ansible.utils.ipmath(1) }}"
  management_netmask: >-
    {{ opinionated.tanzu_vsphere.routeable_super_net |
    ansible.utils.ipsubnet((opinionated.tanzu_vsphere.routeable_super_net.split('/')[1] |int)+3, 0) |
    ansible.utils.ipaddr('netmask') }}
  management_starting_address: "{{ opinionated.tanzu_vsphere.routeable_super_net | ansible.utils.ipmath(2) }}"
  master_storage_policy: "{{ tspbm.vm_storage_policies[0].storage_policy_name }}"
  network_provider: NSXT_CONTAINER_PLUGIN
  supervisor_size: tiny
  vsphere_cluster: compute
  workload_dns_servers: ["{{ dns_server }}"]

  nsxt:
    cluster_distributed_switch: "{{ distributed_switches[0].vds_name }}"
    egress_cidrs:
      - >-
        {{ opinionated.tanzu_vsphere.routeable_super_net |
        ansible.utils.ipsubnet((opinionated.tanzu_vsphere.routeable_super_net.split('/')[1] |int)+3, 1) }}
      - >-
        {{ opinionated.tanzu_vsphere.routeable_super_net |
        ansible.utils.ipsubnet((opinionated.tanzu_vsphere.routeable_super_net.split('/')[1] |int)+2, 1) }}
    ingress_cidrs:
      - >-
        {{ opinionated.tanzu_vsphere.routeable_super_net |
        ansible.utils.ipsubnet((opinionated.tanzu_vsphere.routeable_super_net.split('/')[1] |int)+1, 1) }}
    nsx_edge_cluster: "{{ nsxt.edge_clusters[0].edge_cluster_name}}"
    pod_cidrs: "{{ opinionated.tanzu_vsphere.internal_pod_network }}"
    # This is used by the task which checks if the supervisor network is online
    t0_uplink_ip: "{{ opinionated.hosting_network.base.starting_addr | ansible.utils.ipmath(3) }}"

nsxt:  # (optional) - section can be removed to not create any nsxt objects
  manager:
    hostname: "{{ opinionated.hosting_network.base.starting_addr | ansible.utils.ipmath(1) }}"
    ip: "{{ opinionated.hosting_network.base.starting_addr | ansible.utils.ipmath(1) }}"
    netmask: "{{ opinionated.hosting_network.base.cidr | ansible.utils.ipaddr('netmask') }}"
    gateway: "{{ opinionated.hosting_network.base.gateway }}"
    username: admin  # this cannot be changed
    password: "{{ opinionated.master_password }}{{ opinionated.master_password }}"
    hosting_vcenter_ip: "{{ hosting_vcenter.ip }}"
    hosting_vcenter_username: "{{ hosting_vcenter.username }}"
    hosting_vcenter_password: "{{ hosting_vcenter.password }}"
    hosting_datacenter: "{{ hosting_vcenter.datacenter }}"
    hosting_datastore: "{{ opinionated.hosting_datastore }}"
    hosting_network: "{{ opinionated.hosting_network.base.port_group }}"
    hosting_cluster: "{{ opinionated.hosting_cluster }}"
    license_key: "{{ lookup('env', 'NSXT_LICENSE_KEY') }}"

  # If the section below is defined, the playbook will wait for the IP to become pingable
  # For TKG service deployments this is the default gateway of the supervisor network
  routing_test:
    ip_to_ping: "{{ opinionated.tanzu_vsphere.routeable_super_net | ansible.utils.ipmath(1) }}"
    # The playbook will present a message using the params below
    # A static route must be made to the router uplink for nsxt_supernet
    router_uplink: "{{ opinionated.hosting_network.base.starting_addr | ansible.utils.ipmath(3) }}"
    nsxt_supernet: "{{ opinionated.tanzu_vsphere.routeable_super_net }}"

  policy_ip_pools:
    - display_name: tep-pool  # This is a non-routable range which is used for the overlay tunnels.
      pool_static_subnets:
        - id: tep-pool-1
          state: present
          allocation_ranges:
            - start: "{{ opinionated.hosting_network.nsxt_tep.cidr | ansible.utils.ipmath(1) }}"
              end: "{{ opinionated.hosting_network.nsxt_tep.cidr | ansible.utils.ipaddr('-2') | ansible.utils.ipaddr('address') }}"
          cidr: "{{ opinionated.hosting_network.nsxt_tep.cidr }}"
          do_wait_till_create: true

  uplink_profiles:
    - display_name: host-tep-profile
      teaming:
        active_list:
          - uplink_name: "uplink-1"
            uplink_type: PNIC
        policy: FAILOVER_ORDER
      transport_vlan: "{{ opinionated.hosting_network.nsxt_tep.vlan_id }}"
    - display_name: edge-tep-profile
      mtu: 9000
      teaming:
        active_list:
          - uplink_name: "uplink-1"
            uplink_type: PNIC
        policy: FAILOVER_ORDER
      transport_vlan: "{{ opinionated.hosting_network.nsxt_tep.vlan_id }}"
    - display_name: edge-uplink-profile
      mtu: 1500
      teaming:
        active_list:
          - uplink_name: "uplink-1"
            uplink_type: PNIC
        policy: FAILOVER_ORDER
      transport_vlan: 0

  transport_zones:
    - display_name: tz-overlay
      transport_type: OVERLAY
      # host_switch_name: "{{ distributed_switches[0].vds_name }}"
      nested_nsx: true  # Set this to true if you use NSX-T for your physical host networking
      description: "Overlay Transport Zone"
      # - display_name: nsx-vlan-transportzone
      #   transport_type: VLAN
      #   # host_switch_name: sw_vlan
      #   description: "Uplink Transport Zone"

  transport_node_profiles:
    - display_name: tnp1
      host_switches:
        - host_switch_profiles:
            - name: host-tep-profile
              type: UplinkHostSwitchProfile
          host_switch_name: "{{ distributed_switches[0].vds_name }}"
          host_switch_type: VDS
          host_switch_mode: STANDARD
          ip_assignment_spec:
            resource_type: StaticIpPoolSpec
            ip_pool_name: "tep-pool"
          transport_zone_endpoints:
            - transport_zone_name: "tz-overlay"
            - transport_zone_name: "nsx-vlan-transportzone"
          uplinks:
            - uplink_name: "uplink-1"
              vds_uplink_name: "Uplink 1"
      description: "Cluster node profile"

  cluster_attach:
    - display_name: "tnc1"
      description: "Transport Node Collections 1"
      compute_manager_name: "vCenter"
      cluster_name: "compute"
      transport_node_profile_name: "tnp1"

  edge_nodes:
    - display_name: edge-node-1
      size: MEDIUM
      mgmt_ip_address: "{{ opinionated.hosting_network.base.starting_addr | ansible.utils.ipmath(2) }}"
      mgmt_prefix_length: "{{ opinionated.hosting_network.base.cidr.split('/')[1] }}"
      mgmt_default_gateway: "{{ opinionated.hosting_network.base.gateway }}"
      network_management_name: vm-network
      network_uplink_name: vm-network
      network_tep_name: edge-tep-seg
      datastore_name: datastore-esx1
      cluster_name: compute
      host_switches:
        tep:
          uplink_profile_name: edge-tep-profile
          ip_assignment_spec:
            resource_type: StaticIpPoolSpec
            ip_pool_name: tep-pool
          transport_zone_endpoints:
            - transport_zone_name: "tz-overlay"
        uplink:
          host_switch_name: "sw_vlan"
          uplink_profile_name: edge-uplink-profile
          transport_zone_endpoints:
            - transport_zone_name: "nsx-vlan-transportzone"
      transport_zone_endpoints:
        - transport_zone_name: "tz-overlay-2"
        - transport_zone_name: "nsx-vlan-transportzone"

  edge_clusters:
    - edge_cluster_name: edge-cluster-1
      edge_cluster_members:
        - transport_node_name: edge-node-1

  vlan_segments:
    - display_name: t0-uplink
      vlan_ids: [0]
      transport_zone_display_name: nsx-vlan-transportzone
    - display_name: edge-tep-seg
      vlan_ids: [0]
      transport_zone_display_name: nsx-vlan-transportzone

  # For full spec see - https://github.com/laidbackware/ansible-for-nsxt/blob/vmware-lab-builder/library/nsxt_policy_tier0.py
  tier_0:
    display_name: "tkgs-t0"
    ha_mode: ACTIVE_STANDBY
    uplink_ip: "{{ opinionated.hosting_network.base.starting_addr | ansible.utils.ipmath(3) }}"
    disable_firewall: true
    static_routes:
      - state: present
        display_name: default-route
        network: "0.0.0.0/0"
        next_hops:
          - ip_address: "{{ opinionated.hosting_network.base.gateway }}"
    locale_services:
      - state: present
        display_name: "tkgs-t0-ls"
        edge_cluster_info:
          edge_cluster_display_name: edge-cluster-1
        interfaces:
          - display_name: "test-t0-t0ls-iface"
            state: present
            subnets:
              - ip_addresses: ["{{ opinionated.hosting_network.base.starting_addr | ansible.utils.ipmath(3) }}"]
                prefix_len: "{{ opinionated.hosting_network.base.cidr.split('/')[1] | int }}"
            segment_id: t0-uplink
            edge_node_info:
              edge_cluster_display_name: edge-cluster-1
              edge_node_display_name: edge-node-1
            mtu: 1500

  overlay_segments:
    - display_name: supervisor-seg
      transport_zone_display_name: tz-overlay
      tier1_display_name: supervisor-t1
      subnets:
        - gateway_address: >-
            {{ opinionated.tanzu_vsphere.routeable_super_net |
            ansible.utils.ipmath(1) }}/{{ opinionated.tanzu_vsphere.routeable_super_net.split('/')[1] |int +3 }}

  tier_1_gateways:
    - display_name: supervisor-t1
      route_advertisement_types:
        - "TIER1_CONNECTED"
      tier0_display_name: tkgs-t0
```

## Restrict internet access in VyOS interfaces

We will create an environment that doesn't have outbound internet connection but ingress connection from the internet is allowed and also all traffic within the network is allowed. We don't restrict internal communications on a per-port level.

1. Create a `Firewall address-group` called `ALLOWED-IPS` that will collect all IP ranges to and from which communication is allowed using the command

    ```shell
    set firewall group address-group ALLOWED-IPS address 172.20.16.1-172.20.16.255
    ```

    Do this for every IP range you are using (don't forget the internal Kubernetes pod and service range). We get the following result:

    ```shell
    vyos@vyos# show firewall group
    address-group ALLOWED-IPS {
        address 192.168.178.1-192.168.178.255
        address 172.20.16.1-172.20.16.255
        address 172.20.17.1-172.20.17.255
        address 172.20.18.1-172.20.18.255
        address 172.20.19.1-172.20.19.255
        address 172.20.20.1-172.20.20.255
        address 172.20.21.1-172.20.21.255
        address 172.20.22.1-172.20.22.255
        address 172.20.23.1-172.20.23.255
        address 172.30.4.1-172.30.4.255
        address 172.32.0.1-172.32.0.255
        address 172.32.1.1-172.32.1.255
        address 172.32.2.1-172.32.2.255
        address 172.32.3.1-172.32.3.255
        address 172.32.4.1-172.32.4.255
        address 172.32.5.1-172.32.5.255
        address 172.32.6.1-172.32.6.255
        address 172.32.7.1-172.32.7.255
    }
    ```

1. Create a firewall rule to allow inbound traffic from everywhere

    ```shell
    set firewall name INBOUND-ALL default-action drop
    set firewall name INBOUND-ALL description "Allow all incoming connections"
    set firewall name INBOUND-ALL enable-default-log
    set firewall name INBOUND-ALL rule 20 action accept
    set firewall name INBOUND-ALL rule 20 log enable
    set firewall name INBOUND-ALL rule 20 source address 0.0.0.0/0
    commit 
    save
    ```

    The result is:

    ```shell
    vyos@vyos# show firewall name INBOUND-ALL
    default-action drop
    description "Allow all incoming connections"
    enable-default-log
    rule 20 {
        action accept
        log enable
        source {
            address 0.0.0.0/0
        }
    }
    ```

1. Create a firewall rule for outbound communication

    ```shell
    set firewall name OUTBOUND-RESTRICT default-action 'drop'
    set firewall name OUTBOUND-RESTRICT description 'Restrict outbound connections'
    set firewall name OUTBOUND-RESTRICT enable-default-log
    set firewall name OUTBOUND-RESTRICT rule 10 action 'accept'
    set firewall name OUTBOUND-RESTRICT rule 10 description 'Allow DNS'
    set firewall name OUTBOUND-RESTRICT rule 10 destination port '53'
    set firewall name OUTBOUND-RESTRICT rule 10 protocol 'tcp_udp'
    set firewall name OUTBOUND-RESTRICT rule 20 action 'accept'
    set firewall name OUTBOUND-RESTRICT rule 20 description 'Allow SSH'
    set firewall name OUTBOUND-RESTRICT rule 20 destination port '22'
    set firewall name OUTBOUND-RESTRICT rule 20 log 'enable'
    set firewall name OUTBOUND-RESTRICT rule 20 protocol 'tcp'
    set firewall name OUTBOUND-RESTRICT rule 30 action 'accept'
    set firewall name OUTBOUND-RESTRICT rule 30 description 'Allow specific outbound traffic'
    set firewall name OUTBOUND-RESTRICT rule 30 log 'enable'
    set firewall name OUTBOUND-RESTRICT rule 30 protocol 'all'
    set firewall name OUTBOUND-RESTRICT rule 30 source group address-group 'ALLOWED-IPS'
    commit 
    save
    ```

    The result is:

    ```shell
    vyos@vyos# show firewall name OUTBOUND-RESTRICT
    default-action drop
    description "Restrict outbound connections"
    enable-default-log
    rule 10 {
        action accept
        description "Allow DNS"
        destination {
            port 53
        }
        protocol tcp_udp
    }
    rule 20 {
        action accept
        description "Allow SSH"
        destination {
            port 22
        }
        log enable
        protocol tcp
    }
    rule 30 {
        action accept
        description "Allow specific outbound traffic"
        log enable
        protocol all
        source {
            group {
                address-group ALLOWED-IPS
            }
        }
    }
    ```

1. Finally, apply the firewall rules on the respective interfaces:

    ```shell
    set interfaces ethernet eth1 vif 20 firewall in name INBOUND-ALL
    set interfaces ethernet eth1 vif 20 firewall out name OUTBOUND-RESTRICT
    set interfaces ethernet eth1 vif 16 firewall in name INBOUND-ALL
    set interfaces ethernet eth1 vif 16 firewall out name OUTBOUND-RESTRICT
    commit
    save
    ```

    The result is:

    ```shell
    vyos@vyos# show interfaces ethernet eth1
    hw-id 00:0c:29:85:a5:45
    mtu 9000
    vif 16 {
        address 172.20.16.1/22
        description tanzu-without-dhcp
        firewall {
            in {
                name INBOUND-ALL
            }
            out {
                name OUTBOUND-RESTRICT
            }
        }
        mtu 9000
    }
    vif 20 {
        address 172.20.20.1/22
        description nsxt-tep
        firewall {
            in {
                name INBOUND-ALL
            }
            out {
                name OUTBOUND-RESTRICT
            }
        }
        mtu 9000
    }
    ```
