# Linux switchdev and DPDK port representor

# 1. Linux Switchdev

## 1.1 Intro

Initially, [switchdev](https://www.kernel.org/doc/Documentation/networking/switchdev.txt) was introduced into Linux community due to below reasons:
https://plvision.eu/rd-lab/blog/linux-kernel/switchdev-power-of-open-linux

## 1.2 Use case

Mellanox was the first vendor who adopt switchdev in their product, some sharing from mlx and their customer:
[Introduction to switchdev SR-IOV offloads](https://netdevconf.info/1.2/slides/oct6/04_gerlitz_efraim_introduction_to_switchdev_sriov_offloads.pdf)

[Linux Switchdev the Mellanox way](https://blog.qrator.net/en/linux-switchdev-mellanox-way_104/)

One of the typical use case for switchdev is OVS exceptional path(OVS Re-injection).

# 2. DPDK port representor

## 2.1 Intro

Dpdk community was planned to use switchdev, but later introduced a similar but simpler thing called [port representor](https://doc.dpdk.org/guides/prog_guide/switch_representation.html).

Mellanox was one of the main contributor for port representor:
https://www.dpdk.org/wp-content/uploads/sites/35/2018/12/RonyEfraim_Using-New-DPDK-Port-Representor-by-Switch-Application-like-OVS_DPDK_Summit18_v3.pdf

## 2.2 Use case

Except OVS exceptional path, another use case for port representor is[ vlan filter](https://doc.dpdk.org/dts/test_plans/port_representor_test_plan.html), for example, if customer wants to config vlan encap/decap for VF, port representor for VF can be leveraged for configuration.

AFAIK, there is no other interface to config vlan stripping in DPDK. So VF vlan filtering maybe the only choice. This is recently merged in 21.02 as ice dcf port representor.



Recently, CVL Linux driver adds support for switchdev support.
