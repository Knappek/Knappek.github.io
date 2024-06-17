# Setup

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

## Nested Lab Setup

My physical host is running ESXi. To bootstrap nested lab environments I am using [vmware-lab-builder](https://github.com/laidbackware/vmware-lab-builder), Kudos to Matt :clap:.

### Networking & Routing

I have deployed a VyOS router as a virtual machine on my physical host. My home router has a static IP route configured to forward requests to `172.20.0.0/22` and `172.30.0.0/22` to the VyOS router.
