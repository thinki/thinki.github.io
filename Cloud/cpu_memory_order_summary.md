# CPU memory order summary

# 1. Introduction

This article summarizes all the memory order related resouce since it is a hard to understand concept

# 2. Target Audience

Developer with basic cpu background

# 3. Reference

Weak vs. Strong Memory Models  <https://preshing.com/20120930/weak-vs-strong-memory-models/>

Memory Barriers Are Like Source Control Operations <https://preshing.com/20120710/memory-barriers-are-like-source-control-operations/>

This Is Why They Call It a Weakly-Ordered CPU <https://preshing.com/20121019/this-is-why-they-call-it-a-weakly-ordered-cpu/>

Memory Reordering Caught in the Act <https://preshing.com/20120515/memory-reordering-caught-in-the-act/>

Acquire and Release Semantics <https://preshing.com/20120913/acquire-and-release-semantics/>

Memory Ordering at Compile Time <https://preshing.com/20120625/memory-ordering-at-compile-time/>

The Significance of the x86 SFENCE Instruction <https://hadibrais.wordpress.com/2019/02/26/the-significance-of-the-x86-sfence-instruction/>

what is the effects of _mm_lfence and _mm_sfence <https://community.intel.com/t5/Intel-C-Compiler/what-is-the-effects-of-mm-lfence-and-mm-sfence/td-p/871883>

Intel® 64 and IA-32 Architectures Software Developer Manuals <https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html> Volum 3 Chapter 11 Memory Cache Control, Volume 3 Chapter 8.2 Memory Ordering

# 4. Example

In virtio pmd, barrier(rte_smp_wmb) is needed before the negotiation flag is commited to prevent the flag store is visible ahead of the vhost descriptor store operation.

```c
 228 static __rte_always_inline void
 229 vhost_shadow_dequeue_batch_packed(struct virtio_net *dev,
 230 ▸       ▸       ▸       ▸         struct vhost_virtqueue *vq,
 231 ▸       ▸       ▸       ▸         uint16_t *ids)
 232 {
 233 ▸       uint16_t flags;
 234 ▸       uint16_t i;
 235 ▸       uint16_t begin;
 236
 237 ▸       flags = PACKED_DESC_DEQUEUE_USED_FLAG(vq->used_wrap_counter);
 238
 239 ▸       if (!vq->shadow_used_idx) {
 240 ▸       ▸       vq->shadow_last_used_idx = vq->last_used_idx;
 241 ▸       ▸       vq->shadow_used_packed[0].id  = ids[0];
 242 ▸       ▸       vq->shadow_used_packed[0].len = 0;
 243 ▸       ▸       vq->shadow_used_packed[0].count = 1;
 244 ▸       ▸       vq->shadow_used_packed[0].flags = flags;
 245 ▸       ▸       vq->shadow_used_idx++;
 246 ▸       ▸       begin = 1;
 247 ▸       } else
 248 ▸       ▸       begin = 0;
 249
 250 ▸       vhost_for_each_try_unroll(i, begin, PACKED_BATCH_SIZE) {
 251 ▸       ▸       vq->desc_packed[vq->last_used_idx + i].id = ids[i];
 252 ▸       ▸       vq->desc_packed[vq->last_used_idx + i].len = 0;
 253 ▸       }
 254
 255 ▸       rte_smp_wmb();
 256 ▸       vhost_for_each_try_unroll(i, begin, PACKED_BATCH_SIZE)
 257 ▸       ▸       vq->desc_packed[vq->last_used_idx + i].flags = flags;
 258
 259 ▸       vhost_log_cache_used_vring(dev, vq, vq->last_used_idx *
 260 ▸       ▸       ▸       ▸          sizeof(struct vring_packed_desc),
 261 ▸       ▸       ▸       ▸          sizeof(struct vring_packed_desc) *
 262 ▸       ▸       ▸       ▸          PACKED_BATCH_SIZE);
 263 ▸       vhost_log_cache_sync(dev, vq);
 264
 265 ▸       vq_inc_last_used_packed(vq, PACKED_BATCH_SIZE);
 266 }
lib/librte_vhost/virtio_net.c(DPDK 19.11 release)
```

Normally, we write data and then write flag to tell consumer that it can consumes the data, accordingly, consumer need to firstly check flag and then check data, so barrier is also needed.

```c
1802 static __rte_always_inline int
1803 vhost_reserve_avail_batch_packed(struct virtio_net *dev,
1804 ▸       ▸       ▸       ▸        struct vhost_virtqueue *vq,
1805 ▸       ▸       ▸       ▸        struct rte_mempool *mbuf_pool,
1806 ▸       ▸       ▸       ▸        struct rte_mbuf **pkts,
1807 ▸       ▸       ▸       ▸        uint16_t avail_idx,
1808 ▸       ▸       ▸       ▸        uintptr_t *desc_addrs,
1809 ▸       ▸       ▸       ▸        uint16_t *ids)
1810 {
1811 ▸       bool wrap = vq->avail_wrap_counter;
...
1823
1824 ▸       vhost_for_each_try_unroll(i, 0, PACKED_BATCH_SIZE) {
1825 ▸       ▸       flags = descs[avail_idx + i].flags;
1826 ▸       ▸       if (unlikely((wrap != !!(flags & VRING_DESC_F_AVAIL)) ||
1827 ▸       ▸       ▸            (wrap == !!(flags & VRING_DESC_F_USED))  ||
1828 ▸       ▸       ▸            (flags & PACKED_DESC_SINGLE_DEQUEUE_FLAG)))
1829 ▸       ▸       ▸       return -1;
1830 ▸       }
1831
1832 ▸       rte_smp_rmb();
1833
1834 ▸       vhost_for_each_try_unroll(i, 0, PACKED_BATCH_SIZE)
1835 ▸       ▸       lens[i] = descs[avail_idx + i].len;
1836
1837 ▸       vhost_for_each_try_unroll(i, 0, PACKED_BATCH_SIZE) {
1838 ▸       ▸       desc_addrs[i] = vhost_iova_to_vva(dev, vq,
1839 ▸       ▸       ▸       ▸       ▸       ▸         descs[avail_idx + i].addr,
1840 ▸       ▸       ▸       ▸       ▸       ▸         &lens[i], VHOST_ACCESS_RW);
1841 ▸       }
1842
lib/librte_vhost/virtio_net.c(dpdk 19.11 release)
```

in x86 rte_smp_wmb is implemented as compile barrier(compile barrier is to prevent compiler reordering) because x86 implies strong memory order, which will gurantee StoreStore ordering. In architecture which is weakly-ordered(like arm), rte_smp_wmb is implemented as a store fence like instruction.

In iavf pmd/kernel driver, store fence (rtw_wmb() in dpdk, wmb() in kernel) is needed to force memory writes to complete before letting h/w know there are new descriptors to fetch.

iavf dpdk pmd

```c
1477 /* TX function */
1478 uint16_t
1479 iavf_xmit_pkts(void *tx_queue, struct rte_mbuf **tx_pkts, uint16_t nb_pkts)
1480 {
1481 ▸       volatile struct iavf_tx_desc *txd;
1482 ▸       volatile struct iavf_tx_desc *txr;
1483 ▸       struct iavf_tx_queue *txq;
...
1511 ▸       for (nb_tx = 0; nb_tx < nb_pkts; nb_tx++) {
1512 ▸       ▸       td_cmd = 0;
1513 ▸       ▸       td_tag = 0;
1514 ▸       ▸       td_offset = 0;
1515
1516 ▸       ▸       tx_pkt = *tx_pkts++;
1517 ▸       ▸       RTE_MBUF_PREFETCH_TO_FREE(txe->mbuf);
...
1645
1646 ▸       ▸       txd->cmd_type_offset_bsz |=
1647 ▸       ▸       ▸       rte_cpu_to_le_64(((uint64_t)td_cmd) <<
1648 ▸       ▸       ▸       ▸       ▸        IAVF_TXD_QW1_CMD_SHIFT);
1649 ▸       ▸       IAVF_DUMP_TX_DESC(txq, txd, tx_id);
1650 ▸       }
1651
1652 end_of_tx:
1653 ▸       rte_wmb();
1654
1655 ▸       PMD_TX_LOG(DEBUG, "port_id=%u queue_id=%u tx_tail=%u nb_tx=%u",
1656 ▸       ▸          txq->port_id, txq->queue_id, tx_id, nb_tx);
1657
1658 ▸       IAVF_PCI_REG_WRITE_RELAXED(txq->qtx_tail, tx_id);
1659 ▸       txq->tx_tail = tx_id;
1660
1661 ▸       return nb_tx;
1662 }
dpdk-master/drivers/net/iavf/iavf_rxtx.c
```

iavf kernel driver

```c
1728 static void
1729 ice_tx_map(struct ice_ring *tx_ring, struct ice_tx_buf *first,
1730 ▸          struct ice_tx_offload_params *off)
1731 {
1732 ▸       u64 td_offset, td_tag, td_cmd;
1733 ▸       u16 i = tx_ring->next_to_use;
1734 ▸       unsigned int data_len, size;
1735 ▸       struct ice_tx_desc *tx_desc;
1736 ▸       struct ice_tx_buf *tx_buf;
1737 ▸       struct sk_buff *skb;
1738 ▸       skb_frag_t *frag;
...
1761 ▸       for (frag = &skb_shinfo(skb)->frags[0];; frag++) {
1762 ▸       ▸       unsigned int max_data = ICE_MAX_DATA_PER_TXD_ALIGNED;
1763
1764 ▸       ▸       if (dma_mapping_error(tx_ring->dev, dma))
1765 ▸       ▸       ▸       goto dma_error;
1766
...
1836 ▸       /* Force memory writes to complete before letting h/w know there
1837 ▸        * are new descriptors to fetch.
1838 ▸        *
1839 ▸        * We also use this memory barrier to make certain all of the
1840 ▸        * status bits have been updated before next_to_watch is written.
1841 ▸        */
1842 ▸       wmb();
1843
1844 ▸       /* set next_to_watch value indicating a packet is present */
1845 ▸       first->next_to_watch = tx_desc;
1846
1847 ▸       tx_ring->next_to_use = i;
1848
1849 ▸       ice_maybe_stop_tx(tx_ring, DESC_NEEDED);
1850
1851 ▸       /* notify HW of packet */
1852 ▸       if (netif_xmit_stopped(txring_txq(tx_ring)) || !netdev_xmit_more())
1853 ▸       ▸       writel(i, tx_ring->tail);
1854
1855 ▸       return;
1856
1857 dma_error:
1858 ▸       /* clear DMA mappings for failed tx_buf map */
1859 ▸       for (;;) {
1860 ▸       ▸       tx_buf = &tx_ring->tx_buf[i];
1861 ▸       ▸       ice_unmap_and_free_tx_buf(tx_ring, tx_buf);
1862 ▸       ▸       if (tx_buf == first)
1863 ▸       ▸       ▸       break;
1864 ▸       ▸       if (i == 0)
1865 ▸       ▸       ▸       i = tx_ring->count;
1866 ▸       ▸       i--;
1867 ▸       }
1868
1869 ▸       tx_ring->next_to_use = i;
1870 }
drivers/net/ethernet/intel/ice/ice_txrx.c
```

