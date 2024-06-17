# Homelab

When I joined VMware in February 2021 I have built a homelab to be able to quickly spin up test environments using VMware products with a primary focus on Tanzu Labs product portfolio.

## BOM

| Component   | Item                                                                               |
|-------------|------------------------------------------------------------------------------------|
| CPU         | HP/AMD EPYC 7551 PS7551BDVIHAF 2GHz 64MB 32 Core Processor                         |
| Motherboard | Supermicro H11SSL-i Socket SP3 ATX                                                 |
| RAM         | Samsung 4x 64GB 256GB DDR4 ECC RAM 2933 Mhz RDIMM                                  |
| SSD         | Crucial MX500 2TB                                                                  |
| NVMe        | Samsung MZ-V7S2T0BW SSD 970 EVO Plus 2 TB M.2 Internal NVMe SSD (up to 3.500 MB/s) |
| Cooler      | Noctua NH U12s TR4 SP3                                                             |
| GPU         | Asus GeForce GT 710 1GB                                                            |
| Case        | Fractal Design Core 2500                                                           |
| PSU         | Kolink Enclave 500W                                                                |

## Setup

There is a "Management vCenter" (vcenter-mgmt) VM deployed on the physical host that manages the physical host. Additionally, there is a VyOS router deployed as a VM that is responsible for the entire homelab networking & routing. My home router has a static IP route configured to forward requests to `172.20.0.0/22` and `172.30.0.0/22` to this VyOS router.

As a result, the Management vCenter looks like this:

![Management vCenter](vcenter-mgmt.png)

Here you can see:

- `192.168.178.100`: the physical ESXi host
- `jumpbox01`: an Ubuntu jumpbox for testing purposes
- VMs prefixed with `tkgs-`: a nested lab environment (more details see [Nested Lab Setup](#nested-lab-setup))
- `vcenter-mgmt`: Management vCenter
- `vrli`: vRealize Log Insight
- `vrops`: vRealize Operations Manager
- `vyos`: the VyOS Router
- `windows`: a Windows VM

### Networking & Routing

TODO

### Nested Lab Setup

My physical host is running ESXi. To bootstrap nested lab environments I am using [vmware-lab-builder](https://github.com/laidbackware/vmware-lab-builder), Kudos to Matt :clap:.
