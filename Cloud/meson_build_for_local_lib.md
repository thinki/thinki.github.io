# Meson build guide for local installed library

# 1 build Pkt-gen dependency

## 1.1 build dpdk

in meson use prefix env variable to specify install directory, but you should config it before build

```sh
$ meson -Dprefix=${LOCAL_DPDK_PATH} release && ninja -C release
```

# 1.2 install library into a local directory

```shell
$ meson install
```

# 2 build Pkt-gen

## 2.1 pkg-config path

meson use pkg-config to detect dpdk library installed or not

Since we install pkg-config in local directory, we need to tell pkg-config where to find the local pgk-config ".pc" file

let's support we installed dpdk into the /home/jim/bin/ folder

```shell
$ export PKG_CONFIG_PATH=$PKG_CONFIG_PATH:/home/jim/bin/lib/x86_64-linux-gnu/pkgconfig
```

You can check if dpdk can be detected by pkg-config by:

```sh
$ pkg-config --libs libdpdk
-L/home/jim/bin/lib/x86_64-linux-gnu -Wl,--as-needed -lrte_node -lrte_graph -lrte_bpf -lrte_flow_classify -lrte_pipeline -lrte_table -lrte_port -lrte_fib -lrte_ipsec -lrte_vhost -lrte_stack -lrte_security -lrte_sched -lrte_reorder -lrte_rib -lrte_regexdev -lrte_rawdev -lrte_pdump -lrte_power -lrte_member -lrte_lpm -lrte_latencystats -lrte_kni -lrte_jobstats -lrte_ip_frag -lrte_gso -lrte_gro -lrte_eventdev -lrte_efd -lrte_distributor -lrte_cryptodev -lrte_compressdev -lrte_cfgfile -lrte_bitratestats -lrte_bbdev -lrte_acl -lrte_timer -lrte_hash -lrte_metrics -lrte_cmdline -lrte_pci -lrte_ethdev -lrte_meter -lrte_net -lrte_mbuf -lrte_mempool -lrte_rcu -lrte_ring -lrte_eal -lrte_telemetry -lrte_kvargs -lbsd
```

## 2.2 add include path for c compiler

since all the dpdk header is installed into local directory, the compile can't find there headers by default search path, so we need to tell meson to add extra header search path by c_args

```sh
$ meson -D c_args=-I/home/jim/bin/include  build
```

But this is not enough since you'll still get symbol error at compile link stage

```sh

pktgen-cmds.c:(.text+0x7254): undefined reference to `rte_eth_bond_8023ad_conf_get'
/usr/bin/ld: pktgen-cmds.c:(.text+0x7272): undefined reference to `rte_eth_bond_slaves_get'
/usr/bin/ld: pktgen-cmds.c:(.text+0x7290): undefined reference to `rte_eth_bond_active_slaves_get'
/usr/bin/ld: pktgen-cmds.c:(.text+0x7307): undefined reference to `rte_eth_bond_8023ad_ext_distrib'
/usr/bin/ld: pktgen-cmds.c:(.text+0x732f): undefined reference to `rte_eth_bond_8023ad_ext_collect'
/usr/bin/ld: pktgen-cmds.c:(.text+0x73a2): undefined reference to `rte_eth_bond_8023ad_ext_distrib'
/usr/bin/ld: app/a172ced@@pktgen@exe/pktgen-cmds.c.o: in function `show_bonding_mode':
pktgen-cmds.c:(.text+0x7453): undefined reference to `rte_eth_bond_mode_get'
/usr/bin/ld: pktgen-cmds.c:(.text+0x7499): undefined reference to `rte_eth_bond_slaves_get'
/usr/bin/ld: pktgen-cmds.c:(.text+0x74c5): undefined reference to `rte_eth_bond_active_slaves_get'
/usr/bin/ld: pktgen-cmds.c:(.text+0x7659): undefined reference to `rte_eth_bond_8023ad_slave_info'
/usr/bin/ld: pktgen-cmds.c:(.text+0x7960): undefined reference to `rte_eth_bond_primary_get'
/usr/bin/ld: pktgen-cmds.c:(.text+0x79d4): undefined reference to `rte_eth_bond_8023ad_agg_selection_get'
/usr/bin/ld: pktgen-cmds.c:(.text+0x7a24): undefined reference to `rte_eth_bond_xmit_policy_get'
/usr/bin/ld: app/a172ced@@pktgen@exe/cli-functions.c.o: in function `bonding_cmd':
cli-functions.c:(.text+0x2101): undefined reference to `rte_eth_bond_8023ad_conf_get'
/usr/bin/ld: cli-functions.c:(.text+0x224d): undefined reference to `rte_eth_bond_8023ad_conf_ge
```

## 2.3 add link library path for c linker

### 2.3.1 add system library search path for c compiler

```
$ sudo vim /etc/ld.so.conf.d/user.conf
```

add your local path into the file:

/home/jim/bin/lib/x86_64-linux-gnu

Then reload the file by

```sh
$ sudo ldconfig
```

Note: using LD_LIBRARY_PATH is another alternative, it is only useful at link time library search path but not applicable for runtime library search path.

### 2.3.2 add meson linker args for compiler

```sh
$  meson -D c_args=-I/home/jim/bin/include -D c_link_args="-L/home/jim/bin/lib/x86_64-linux-gnu/ -lrte_net_bond" build
```

see more options on https://mesonbuild.com/Builtin-options.html#compiler-options

## 2.4 build

```
$ ninja -C build
```

## 2.5 launch

there is run time library search path dependency, but since we have specify it in /etc/[ld.so](http://ld.so/).conf.d/user.conf, it is enough for us to run Pkt-gen. I

You can check if the library search path has been cached successfully by

```sh
$  ldconfig -v
...
/home/jim/bin/lib/x86_64-linux-gnu:
        librte_net_thunderx.so.21 -> librte_net_thunderx.so.21.2
        librte_raw_ifpga.so.21 -> librte_raw_ifpga.so.21.2
        librte_ring.so.21 -> librte_ring.so.21.2
        librte_net_dpaa.so.21 -> librte_net_dpaa.so.21.2
        librte_net_vhost.so.21 -> librte_net_vhost.so.21.2
        librte_raw_dpaa2_qdma.so.21 -> librte_raw_dpaa2_qdma.so.21.2
        librte_lpm.so.21 -> librte_lpm.so.21.2
        librte_kni.so.21 -> librte_kni.so.21.2
        librte_net_mlx5.so.21 -> librte_net_mlx5.so.21.2
        librte_crypto_nitrox.so.21 -> librte_crypto_nitrox.so.21.2
        librte_gro.so.21 -> librte_gro.so.21.2
        librte_acl.so.21 -> librte_acl.so.21.2
        librte_bus_fslmc.so.21 -> librte_bus_fslmc.so.21.2
        librte_event_cnxk.so.21 -> librte_event_cnxk.so.21.2
        librte_net_ionic.so.21 -> librte_net_ionic.so.21.2
        librte_power.so.21 -> librte_power.so.21.2
        librte_net_igc.so.21 -> librte_net_igc.so.21.2
        librte_rawdev.so.21 -> librte_rawdev.so.21.2
        librte_mempool_cnxk.so.21 -> librte_mempool_cnxk.so.21.2
        librte_compress_zlib.so.21 -> librte_compress_zlib.so.21.2
        librte_bpf.so.21 -> librte_bpf.so.21.2
        librte_net_pcap.so.21 -> librte_net_pcap.so.21.2
        librte_net_bond.so.21 -> librte_net_bond.so.21.2
        librte_net_bnxt.so.21 -> librte_net_bnxt.so.21.2
        librte_ipsec.so.21 -> librte_ipsec.so.21.2
        librte_bus_ifpga.so.21 -> librte_bus_ifpga.so.21.2
        librte_common_mlx5.so.21 -> librte_common_mlx5.so.21.2
        librte_baseband_acc100.so.21 -> librte_baseband_acc100.so.21.2
        librte_mempool_octeontx2.so.21 -> librte_mempool_octeontx2.so.21.2
        librte_pci.so.21 -> librte_pci.so.21.2
        librte_crypto_ccp.so.21 -> librte_crypto_ccp.so.21.2
        librte_net_i40e.so.21 -> librte_net_i40e.so.21.2
        librte_compressdev.so.21 -> librte_compressdev.so.21.2
        librte_net_virtio.so.21 -> librte_net_virtio.so.21.2
```

Launch pktgen

```sh
$  sudo ./build/app/pktgen -l 28-32 -m 512 -a 16:00.0 -a 49:00.0 --file-prefix=avf -- -P -m "[29:30].0, [31:32].1"
```



# 3 Reference

http://code.dpdk.org/pktgen-dpdk/pktgen-21.05.0/source/INSTALL.md

http://mesonbuild.com/Builtin-options.html
