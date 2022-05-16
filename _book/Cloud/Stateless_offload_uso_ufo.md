---
typora-root-url: ..
---

# Stateless offload: USO and UFO

# 1. Introduction

This summarize USO, UFO and related segmentation offload technology.

# 2. Target Audience

Developer with basic background about networking

# 3. USO

## 3.1 Intro

​    The USO is “UDP GSO” and it is to divide large UDP packet into several segments where each of these smaller packets has UDP header.

## 3.2 Compared with UFO

​    The major difference between UFO and USO is that:

​    UFO divide large UDP packet at IP layer but USO divide large UDP packet at UDP layer.(check below picture)

![ufo_vs_uso_01](/images/ufo_vs_uso_01.PNG)

​    UFO was deprecated in kernel support. In reality, fragmentation doesn't work due to some reason:

1. RX side will take extra effort to reassemble the fragmented IP packets
2. RSS issue. The first fragment and subsequent fragments are likely to be hashed to different queue/core(typical hash input is 5-tuple), which is troublesome.
3. Many firewalls simply discard the IP fragments

​    Compared to UFO, USO is transparent to RX side

## 3.3 Why USO

​    One of the motivation to improve UDP stack performance is the QUIC which is running on UDP. Refer to https://blog.cloudflare.com/accelerating-udp-packet-transmission-for-quic/ for more benchmark details.

## 3.4 Design Philosophy

​    The key idea behind USO and UFO, which is similar to [TCP GSO](https://lwn.net/Articles/188489/), is to postpone the segmentation/fragmentation as late as possible. This can reduce the syscall frequency and stack traversal which makes performance improved. Refer to https://www.youtube.com/watch?v=ccUeG1dAhbw for more ideas behind.

## 3.5 NIC offload support

   Intel NIC support USO hardware offload. Take ixgbe hardware spec for example:

![ufo_vs_uso_02](/images/ufo_vs_uso_02.PNG)

   UDP GSO has been widely supported in intel NIC drivers(igb/ixgbe/i40e/ice) and other vendor drivers in Linux kernel.

  ![ufo_vs_uso_03](/images/ufo_vs_uso_03.PNG)

