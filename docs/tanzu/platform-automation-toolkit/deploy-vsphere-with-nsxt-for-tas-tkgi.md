# Deploy vSphere + NSX-T for TAS and TKGI

This page explains how to deploy a Nested Lab with a "naked" vCenter plus NSX Manager pre-configured that can be used to [Deploy TAS and TKGI with Concourse](deploy-tas-tkgi-with-concourse.md).

## Configure Routing for Nested Lab

Read about my routing configuration in my [homelab page](./../../homelab/index.md#networking-routing). As it is a NSX backed environment where my Tier0 will have the IP `172.20.16.13`, I have the following static routes:

- `route 172.30.5.0/24 next-hop 172.20.16.13`
- `route 172.30.6.0/26 next-hop 172.20.16.13`

Additionally I will deploy one [TAS Isolation Segment](https://docs.vmware.com/en/VMware-Tanzu-Application-Service/6.0/tas-for-vms/installing-pcf-is.html) for which I create another static route:

- `route 172.30.7.0/27 next-hop 172.20.16.13`

They all point to the same Tier0 which is shared between TAS and TKGI. This example can be easily expanded to have multiple Tier0 if desired.

## Deploy Nested ESXi Lab

We will use [vmware-lab-builder](https://github.com/laidbackware/vmware-lab-builder) to bootstrap a nested lab environment with

* a Nested ESXi host
* vCenter
* NSX Manager

We pre-configure some NSX resources required for TAS & TKGi (like a T0 Router, T1 Routers, IP Pools etc.) using the following opinionated vars yaml (see more info in [how to deploy labs](https://github.com/laidbackware/vmware-lab-builder?tab=readme-ov-file#deploying)):

```yaml
---
# SOFTWARE_DIR must contain all required software
vc_iso: "{{ lookup('env', 'SOFTWARE_DIR') }}/VMware-VCSA-all-8.0.2-23319993.iso"
esxi_ova: "{{ lookup('env', 'SOFTWARE_DIR') }}/Nested_ESXi7.0u3c_Appliance_Template_v1.ova"
nsxt_ova: "{{ lookup('env', 'SOFTWARE_DIR') }}/nsx-unified-appliance-4.2.0.0.0.24105821.ova"

environment_tag: "tas-tkgi-nsx"  # Used to prepend object names in hosting vCenter
dns_server: "192.168.178.1"
dns_domain: "lab.knappster.de"
ntp_server_ip: "192.168.178.1"  # Must be set to an IP address!
disk_mode: thin  # How all disks should be deployed
nested_host_password: "{{ opinionated.master_password }}"

hosting_vcenter:  # This is the vCenter which will be the target for nested vCenters and ESXi hosts
  ip: "192.168.178.102"
  username: "{{ lookup('env', 'PARENT_VCENTER_USERNAME') }}"
  password: "{{ lookup('env', 'PARENT_VCENTER_PASSWORD') }}"
  datacenter: "Home"  # Target for all VM deployment

# This section is only referenced by other variables in this file
opinionated:
  master_password: "VMware1!"
  number_of_hosts: 1  # number of ESXi VMs to deploy
  nested_hosts:
    cpu_cores: 16  # CPU count per nested host
    ram_in_gb: 128  # memory per nested host
    local_disks:  # (optional) this section can be removed to not modify local disks
      - size_gb: 1000
        datastore_prefix: "datastore"  # omit this to not have a datastore created
  hosting_cluster: Physical
  hosting_datastore: NVME
  hosting_network:
    base:
      port_group: tanzu-without-dhcp
      cidr: "172.20.16.0/22"
      gateway: "172.20.16.1"
      # A NSX-T deployment requires 4 IPs, plus 1 per esxi host. They MUST be contiguous.
      starting_addr: "172.20.16.10"
    nsxt_tep:
      port_group: nsxt-tep
      vlan_id: 0
      cidr: "172.20.20.0/22" # IP calculation depends on this being a /24
  nsxt:
    tier0_gateway_name: tas-tkgi-t0
    tier1_gateways:
      - display_name: tkgi-mgmt-t1
        route_advertisement_types:
          - "TIER1_CONNECTED"
          - "TIER1_NAT"
        segments:
          - display_name: tkgi-mgmt-seg
            default_gateway_cidr: "172.30.6.1/26"
      - display_name: tas-t1
        route_advertisement_types:
          - "TIER1_CONNECTED"
          - "TIER1_NAT"
        segments:
          - display_name: tas-seg
            default_gateway_cidr: "172.30.5.1/24"
          - display_name: tas-iso-seg-1
            default_gateway_cidr: "172.30.7.1/27"

  tas:
    nsx_principal_identity:
      public_key: |-
        -----BEGIN CERTIFICATE-----
        MIIDaDCCAlCgAwIBAgIUbz/tPuQrBt2uIAkFIP51c+TybecwDQYJKoZIhvcNAQEL
        BQAwHzELMAkGA1UEBhMCVVMxEDAOBgNVBAoMB1Bpdm90YWwwHhcNMjQxMDAxMTE1
        NjIxWhcNMjYxMDAyMTE1NjIxWjA3MQswCQYDVQQGEwJVUzEQMA4GA1UECgwHUGl2
        b3RhbDEWMBQGA1UEAwwNMTcyLjIwLjE2LjEwMTCCASIwDQYJKoZIhvcNAQEBBQAD
        ggEPADCCAQoCggEBAMcdwWmzvzKL14+4dzUSX7T/hBHbvKLdMBfqU7WOsm9wBpdd
        rYNFtElSI56MTgyYA9U94PyX/+IkpFznTT56U77TuFgrRBZlmleqDa4MV3+W0s+J
        UfurdRLzW2vyIJNlNK4urub3YQV6F002GUa7PqA8ILf55CKRhaDYq+9LLMUq7KFG
        Myl/nuKTfuRrSiKL67A9NElcPnaZTeE8yv3Qs70mv22una13xYqjU9zOZbXMLUxf
        nuFlsqjzm521uISEK5G9nSilsyMMTfc1WksjSG2qlbBwWCtKp94oaiFJNuk2Pwdz
        8n4+jYDJpnOTC8JSA6QqCZZWkL6IIgqzYRG7ywcCAwEAAaOBgzCBgDAdBgNVHQ4E
        FgQUqX1jSYfVxHz/LX96LaI3K8nZEEMwHwYDVR0jBBgwFoAUiXTpjmsD5sGj10LC
        1V0oRXu2ERIwHQYDVR0lBBYwFAYIKwYBBQUHAwIGCCsGAQUFBwMBMA4GA1UdDwEB
        /wQEAwIFoDAPBgNVHREECDAGhwSsFBBlMA0GCSqGSIb3DQEBCwUAA4IBAQCJIIik
        nagwmntydYA/m3gHe5xPApysxjUDw6qSKHgNxzu9y7Pdnfh6XZf6puPFci4xH5QE
        QvEI5t6GrzfuI0OLO89YRQyaK5w/KTRFNhQ0sqvXgETn5i9sxzrv7MPgMlrkPj7Y
        KcO4Aav9Is2l62kCTpKw4z5L1Q7k8CaJhsjRXmYnrFLFynyXCaCa9oxlGacQltl1
        VBn7PMM05rrR+s0fFgmKvgH9l0aM2A59qSeSYA1A3F05CZ7nuuKQTw2/mkNuLZKe
        HUB6jbp935hbOcew6bTUUrJu6T1ek+DQnb4RlNaO5Rnm0xxGabS0Pc0LHw4DlJiH
        rFAge4ATExlV0+iV
        -----END CERTIFICATE-----
      private_key: |-
        -----BEGIN RSA PRIVATE KEY-----
        MIIEpAIBAAKCAQEAxx3BabO/MovXj7h3NRJftP+EEdu8ot0wF+pTtY6yb3AGl12t
        g0W0SVIjnoxODJgD1T3g/Jf/4iSkXOdNPnpTvtO4WCtEFmWaV6oNrgxXf5bSz4lR
        +6t1EvNba/Igk2U0ri6u5vdhBXoXTTYZRrs+oDwgt/nkIpGFoNir70ssxSrsoUYz
        KX+e4pN+5GtKIovrsD00SVw+dplN4TzK/dCzvSa/ba6drXfFiqNT3M5ltcwtTF+e
        4WWyqPObnbW4hIQrkb2dKKWzIwxN9zVaSyNIbaqVsHBYK0qn3ihqIUk26TY/B3Py
        fj6NgMmmc5MLwlIDpCoJllaQvogiCrNhEbvLBwIDAQABAoIBADwx4ilW5jvdK98u
        iJc6RUW+I0qUz+u6i5IHTKQsDgSDbPKwpsZzOaQa2VrSlrvW7v211cD3IKvYoPnX
        ETKMn6mmbun0toJA2A6dgcI2x/LyASwtmuPG+z8t49r32WJF682mnkiDy8hwlv/I
        FY8dBztAwjFsMcxDiw7Lwfq3EsNOBImEpFCm5QWASYGADtW3su1mRy/nYGrRvfzL
        nGDUVy7X2RHDd+h3JlXGXmkmTHpWEgjiHtbsiejWgjUP4UUtlERtsQ3hDGw8eo1W
        o+YTglTZ5yJkVMGFs2wgMbYA+zrEYcqLoYqWegxqfIwnCf7m6//rXyLIFJ/QiU4r
        3upPFW0CgYEA1qthNarOGlrHicC3Y8vsHw4hYYzH062bu+TX8uW32SkLgZ/rN1rZ
        6PaAfDn7qd7035p+S9yhjumtVVUNWhVK8a/jAnE/lSLkI4Zb6WA5VF2FaSoSzIUV
        fon6TSBkBENv2qhtgjQXUPy4FNjUwLpsPFm0k0yELHVizRcc/KAisE0CgYEA7XPK
        DULnlX2kpsGShSerGnz/HH6dmZOUNRUcXFsLjSVxZOvKoHmjQj/k5WCeePogxlVo
        dqyF2VenjqQp/vJPds4n+5khOxAimXl4xL9GIKLmg1g5d8CqC/fA9SibHjJXsinw
        jjmgfI+WXcAKAKhVe6PAg4YqzkrFla1g0flQsqMCgYEAoU+RMcHTNGyo6rO9Wyme
        mkuE/AfNFRytHQk+2RCUEYRNWC+ykhscCnpJXJA5s5GN0wUGCL2XTYv9K1VJPjsn
        4Ou5m1k8XTYl1ygcowcirWnFWZw7GiKbX0YRp6lCXw3J3LaZ67B3IO126ntxjA3K
        TaNfFRz3aW0gPFs09gTjbDUCgYEAhqfMJDsVs0u+DKbnXUWCnZHW5iTTYN01BelD
        3QfwhAmAxZeFn/163L35Iy7oj3hhD7gtdmcdvIQdzCFCg4aME7aTK/XJx4G97UTa
        fNBvh2B50nA8nrGOfRzxutVdKgGog6uO9EivvxN6VQ3rXjYXy/av3KZALh5u8BOT
        PV/iKHsCgYBQ/nSZeh8DNOkUxpNmC5mh4ZEjJ1dX8NWzomA2K5tkMqC53vrJLJx/
        T3GH0STE7JwsFJrwIepKV85BotO/no4OOUyWFcpmu7f+0msKbae8FroRXaN8a5YI
        lwOMrTuT7Caol9by5h8CT0pZbvzEPCsWJE1npT4QyPDcDI+tya0nBA==
        -----END RSA PRIVATE KEY-----
  tkgi:
    node_ip_block: 100.69.0.0/16
    pod_ip_block: 100.70.0.0/16
    floating_ip_pool: 172.30.6.64/26
  # principal identity can be kept as default unless there is a security reason to change
    nsx_principal_identity:
      public_key: |-
        -----BEGIN CERTIFICATE-----
        MIIDADCCAeigAwIBAgIUJdyN7pMx9a5rFluqWIdBu05GGIcwDQYJKoZIhvcNAQEL
        BQAwHjEcMBoGA1UEAwwTcGtzLW5zeC10LXN1cGVydXNlcjAeFw0yMzA0MDQxNDIy
        NDlaFw0yNTA0MDMxNDIyNDlaMB4xHDAaBgNVBAMME3Brcy1uc3gtdC1zdXBlcnVz
        ZXIwggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAwggEKAoIBAQDXUJoCuRXNvfG24gCB
        H367P3BjeCJ04utTCwgsj0fnk/MbUQagXidL/0azIoCvEJ6eA8n1SfD14kErnrOi
        9TwOvy9E2MRoFYwpLcjv1oKm0SUsldFfEU0TWTIt2zJcintIWZkEzb63YK2gud0k
        g/snCcggK1rH91SwUgi3qlXzIdsrxxaKyL5G/u2P9zUKIfYIq+S4K5hh6PNM/7Pd
        BaSeu1Tv9AKTUrRMnporO1iopcZhcwX43YkSnSHcZqvOIOsi8cry3iaFgRbhxtEE
        QCG0IMmtFWTFrtoNBhtjLCIQvje2pWCM5dRRHos+LN1IigtcoZAORAwtN+05z6+M
        tZ7RAgMBAAGjNjA0MBMGA1UdJQQMMAoGCCsGAQUFBwMCMB0GA1UdDgQWBBTT5Fot
        BYKTSO49UJHpIKZusO0ZezANBgkqhkiG9w0BAQsFAAOCAQEAw3afcTvhnbCwlzeY
        cWOiRK+gJ3YWN+4pzY04y3A+mFhHDFZGy01eUuW1qPnV9HWyfB1GpnEtnu3aw89h
        F0bBIJ9Tvx8bXCVnSIn/+Zea39/hOYqXHewVrennpMyCCiDu6lxadHKJVUCBDcqV
        e9uUhEcPupNs6f8P8FflHf5jCQ95464d9sZ5caM19vfomg3bC7iLlo/Owo23nZlw
        X2gpUu8hJFDbAeeW6Sx6ZbtVikTUFaiwTzMmSAt+xLeGmYqXNxUPDP8ORxAEcrKN
        J9fWUWc/BhKzlXGqOf/EGjYs41OiAY/2ok1gODPZW/2nwPlFNrrq1Qv72jJ0GaWW
        onRhDA==
        -----END CERTIFICATE-----
      private_key: |-
        -----BEGIN PRIVATE KEY-----
        MIIEvAIBADANBgkqhkiG9w0BAQEFAASCBKYwggSiAgEAAoIBAQDXUJoCuRXNvfG2
        4gCBH367P3BjeCJ04utTCwgsj0fnk/MbUQagXidL/0azIoCvEJ6eA8n1SfD14kEr
        nrOi9TwOvy9E2MRoFYwpLcjv1oKm0SUsldFfEU0TWTIt2zJcintIWZkEzb63YK2g
        ud0kg/snCcggK1rH91SwUgi3qlXzIdsrxxaKyL5G/u2P9zUKIfYIq+S4K5hh6PNM
        /7PdBaSeu1Tv9AKTUrRMnporO1iopcZhcwX43YkSnSHcZqvOIOsi8cry3iaFgRbh
        xtEEQCG0IMmtFWTFrtoNBhtjLCIQvje2pWCM5dRRHos+LN1IigtcoZAORAwtN+05
        z6+MtZ7RAgMBAAECggEAAeLouS23mq9XfvODmfNVhWdyC8phfDusCx9gHsp8kJ3+
        YGwOePjfiENqx6aoO6Bo0K1A0jTSdx0DLCd+Sbxd89UfTA+dcjn/bzGW/pRBsxtx
        o26GleTNaOYm/I7ckJdSqtjExe2AMRQYQVMPicKG60R4gTZQBnYiGE9dAzBAri9x
        d2CMUYv+u8MgIz4UcNlOGT2m9pvOC/CX5YbDKP7GX3xuJ3Zg/XHmjvuEyBfKEGx2
        9D7KKI9d8ra+KMJVqAR8NzubVVz0VDALXpFumbL2jJQb1jccyDbSTx619n4kRnIO
        GtJqLWjuSAuRFk+PV6Vb48gmaKWSo5uXpvqeat/cdwKBgQDxq5J8xlXz/rzPLUS4
        fraFTlR+MnwmN65FQyqHPYaGI4o9/DJJO7DuuUbj7fNta2FDuzR+ZQZgZhiBNqiq
        RYL0sL6AHSnduka5XSbXRCDiwl7nfO6i4+iy52VeKdacrhV3YVDydx6hfxGPHDPo
        F8pMpYC2mizFYPvTu1httfISgwKBgQDkFPgGSvWZt5BpkofTJfNNtQcg2eiFhnXn
        bVaKScVmvk7AWpZ/FkmQRc1gXGKszDp1Zx7EgUBhCiN8YgxK8bQT2aVBViIaqG7N
        mkN1RUIisBIaOPFURt1T1XAaZJxLPFD30ufv1dKhoNm50/SBUp6DITnhpNjNcT/s
        DvQSIm+5GwKBgGQUbUGG0SmOIJqbYI4Wy3dBDPSF66vX+y9rtTz0WbVLGoC45Ao3
        0fnKeHUDoX96rHjkGcUOCSn6ncNE42xABQ9X8kwTx7au4YL59I/JAuVlIPA0aI7E
        WyVbdjsckGeqH/GkN2VxtxmiCZ9+SnCfCYPcNgVoq4nBtAfm2aP1aR4JAoGAHZay
        zm4vCnAL5gZCZJwJwkz3zcU3KwtUhF9k2K/VUgziPoYB/B6yEGtdx2B01KHx+4UT
        Mr7p0Sz1iY9WtOpCSEj17VH1PqwXI8kdczs25zUcRBabCCnhUJzh3CqtM/1xK5VK
        zYxZtOofFMJwd852DeDjl2hBT/WfK0qNU0TwZX0CgYAm+wjS++HJBv9f2oZM2PjF
        GVM9Bp/uEiG1z/4O7Rpsv9qJHCgVaZ3q2Qebppq4D4o7ELLcV15djClmv7qkaR9B
        ZauGoYXF2AwcdX8JO4bWHc985IwdAfZohGEqYSOIDkwgOoDXbrtK3LE/R8XtmJJk
        MQTQCVfQMqNNIHcVkgWBNg==
        -----END PRIVATE KEY-----

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

opinionated_host_ip_ofset: 4
# See the custom example for host to build hosts out manually
nested_hosts: >-
  [
    {% for host_number in range(opinionated.number_of_hosts) %}
    {
      "name": "esx{{ host_number + 1 }}",
      "ip": "{{ opinionated.hosting_network.base.starting_addr | ansible.utils.ipmath(opinionated_host_ip_ofset + host_number) }}",
      "mask": "{{ opinionated.hosting_network.base.cidr | ansible.utils.ipaddr('netmask') }}",
      "gw": "{{ opinionated.hosting_network.base.gateway }}",
      "nested_cluster": "compute"
    },
    {% endfor %}
  ]

distributed_switches:  # (optional) - section can be removed to not create any distributed switches
  - vds_name: nsxt-vds
    mtu: 1600
    vds_version: 7.0.0  # Should be 7.0.0, 6.7.0
    clusters:  # distributed switch will be attached to all hosts in the clusters listed
      - compute
    uplink_quantity: 1
    vmnics:
      - vmnic1

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

  principal_identities:
    - display_name: tkgi-super-user
      role: "enterprise_admin"
      public_key: |-
        {{ opinionated.tkgi.nsx_principal_identity.public_key }}
    - display_name: tas-super-user
      role: "enterprise_admin"
      public_key: |-
        {{ opinionated.tas.nsx_principal_identity.public_key }}

  policy_ip_pools:
    - display_name: tep-pool  # This is a non-routable range which is used for the overlay tunnels.
      pool_static_subnets:
        - id: tep-pool-1
          state: present
          allocation_ranges:
            - start: "{{ opinionated.hosting_network.nsxt_tep.cidr | ansible.utils.ipmath(1) }}"
              end: "{{ opinionated.hosting_network.nsxt_tep.cidr | ansible.utils.ipaddr('-2') |ansible.utils.ipaddr('address') }}"
          cidr: "{{ opinionated.hosting_network.nsxt_tep.cidr }}"
          do_wait_till_create: true

    - display_name: tkgi-floating-ips
      pool_static_subnets:
        - id: tkgi-floating-ips-1
          state: present
          allocation_ranges:
            - start: "{{ opinionated.tkgi.floating_ip_pool | ansible.utils.ipaddr('1') | ansible.utils.ipv4('address') }}"
              end: "{{ opinionated.tkgi.floating_ip_pool | ansible.utils.ipaddr('-2') | ansible.utils.ipv4('address') }}"
          cidr: "{{ opinionated.tkgi.floating_ip_pool }}"

  ip_blocks:
    - display_name: tkgi-nodes
      cidr: "{{ opinionated.tkgi.node_ip_block }}"
    - display_name: tkgi-pods
      cidr: "{{ opinionated.tkgi.pod_ip_block }}"

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
      # transport_type: OVERLAY_BACKED
      transport_type: OVERLAY
      # host_switch_name: "{{ distributed_switches[0].vds_name }}"
      nested_nsx: true  # Set this to true if you use NSX-T for your physical host networking
      description: "Overlay Transport Zone"
    - display_name: tz-vlan
      # transport_type: VLAN_BACKED
      transport_type: VLAN
      # host_switch_name: sw_vlan
      description: "Uplink Transport Zone"

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
            - transport_zone_name: "tz-vlan"
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
      size: LARGE
      mgmt_ip_address: "{{ opinionated.hosting_network.base.starting_addr | ansible.utils.ipmath(2) }}"
      mgmt_prefix_length: "{{ opinionated.hosting_network.base.cidr.split('/')[1] }}"
      mgmt_default_gateway: "{{ opinionated.hosting_network.base.gateway }}"
      network_management_name: vm-network
      network_uplink_name: vm-network
      network_tep_name: edge-tep-seg
      datastore_name: "{{ opinionated.nested_hosts.local_disks[0].datastore_prefix }}-esx1"
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
            - transport_zone_name: "tz-vlan"
      transport_zone_endpoints:
        - transport_zone_name: "tz-overlay"
        - transport_zone_name: "tz-vlan"

  edge_clusters:
    - edge_cluster_name: edge-cluster-1
      edge_cluster_members:
        - transport_node_name: edge-node-1

  vlan_segments:
    - display_name: t0-uplink
      vlan_ids: [0]
      transport_zone_display_name: tz-vlan
    - display_name: edge-tep-seg
      vlan_ids: [0]
      transport_zone_display_name: tz-vlan

  # For full spec see - https://github.com/laidbackware/ansible-for-nsxt/blob/vmware-lab-builder/library/nsxt_policy_tier0.py
  tier_0:
    display_name: "{{ opinionated.nsxt.tier0_gateway_name }}"
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
        display_name: "{{ opinionated.nsxt.tier0_gateway_name }}-locale"
        edge_cluster_info:
          edge_cluster_display_name: edge-cluster-1
        interfaces:
          - display_name: "{{ opinionated.nsxt.tier0_gateway_name }}-t0ls-iface"
            state: present
            subnets:
              - ip_addresses: ["{{ opinionated.hosting_network.base.starting_addr | ansible.utils.ipmath(3) }}"]
                prefix_len: "{{ opinionated.hosting_network.base.cidr.split('/')[1] | int }}"
            segment_id: t0-uplink
            edge_node_info:
              edge_cluster_display_name: edge-cluster-1
              edge_node_display_name: edge-node-1
            mtu: 1500

  # build list of segments from opinionated section. See custom deployment for example of how to configurre
  overlay_segments: >-
    [
      {% if "overlay_segment" in  opinionated.nsxt %}
      {% for overlay_segment in opinionated.nsxt.standalone_overlay_segments %}
      {
        "display_name": "{{ overlay_segment.display_name }}",
        "transport_zone_display_name": "tz-overlay"
      },
      {% endfor %}
      {% endif %}
      {% for tier1_gateway in opinionated.nsxt.tier1_gateways %}
      {% if "segments" in  tier1_gateway %}
      {% for segment in tier1_gateway.segments %}
      {
        "display_name": "{{ segment.display_name }}",
        "transport_zone_display_name": "tz-overlay",
        "tier1_display_name": "{{ tier1_gateway.display_name }}",
        "subnets": [
          {"gateway_address": "{{ segment.default_gateway_cidr }}"}
        ]
      },
      {% endfor %}
      {% endif %}
      {% endfor %}
    ]

  tier_1_gateways: >-
    [
      {% for tier1_gateway in opinionated.nsxt.tier1_gateways %}
      {
        "display_name": "{{ tier1_gateway.display_name }}",
        "route_advertisement_types": "{{ tier1_gateway.route_advertisement_types }}",
        "tier0_display_name": "{{ opinionated.nsxt.tier0_gateway_name }}"
      },
      {% endfor %}
    ]
```

Now we are ready to finally [Deploy TAS and TKGI with Concourse](./deploy-tas-tkgi-with-concourse.md).
