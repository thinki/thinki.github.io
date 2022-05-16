# Freescale i.MX6 Linux Ethernet Driver驱动源码分析02

**本文最初发表于2014-08-03，网易博客**

   上一篇文章分析了Freescale i.MX6 Linux Ethernet Driver的device的添加和driver的加载过程，接下来分析fec_enet_init()函数：

首先提一点这个函数的声明是static int fec_enet_init(struct net_device *ndev)，即传递参数为net_device，那么通过netdev_priv(ndev)即可以获取到之前alloc_etherdev()函数分配的指向私有数据的地址：

```c
ndev = alloc_etherdev(sizeof(struct fec_enet_private));
struct fec_enet_private *fep = netdev_priv(ndev);
```

这种通过结构体内部指针传递私有数据的方式在driver中非常常见。函数开头即为Ethernet Controller的DMA 控制器分配相应的buffer描述符：

```c
/* Allocate memory for buffer descriptors. */
cbd_base = dma_alloc_noncacheable(NULL, BUFDES_SIZE, &fep->bd_dma,
			GFP_KERNEL);
if (!cbd_base) {
	printk("FEC: allocate descriptor memory failed?\n");
	return -ENOMEM;
}
```

这里分配的缓冲区大小是（tx buffer个数+rx buffer个数）×buffer描述符大小：

```c
#define BUFDES_SIZE ((RX_RING_SIZE + TX_RING_SIZE) * sizeof(struct bufdesc))
```

由于buffer描述符会被CPU以及DMA控制器访问，因此会存在Cache一致性问题，这里采用了dma_alloc_noncacheable()函数，即DMA一致性映射。这里采用一致性映射是因为CPU或者DMA控制器会以不可预知的方式去访问这段内存区，在Linux Kernel中解决Cache一致性问题有两种方案：DMA流式映射和DMA一致性映射，关于这两者的区别在《Understanding Linux Kernel》以及《LDD3》中均有介绍，我个人也总结了一篇博文初步讲述了这两者的区别：http://blog.163.com/thinki_cao/blog/static/83944875201362142939337。

这里分析一下DMA控制器，i.MX6的DMA控制器采用了环形buffer描述符，这里buffer分为两种，Legacy buffer descriptor是为了保持对前代Freescale器件的兼容性，而Enhanced buffer descriptor则提供了更多的功能，引用i.MX6Q的reference manual中的图：

 Legacy buffer descriptor一共有8个字节，注意这里是采用大端存储模式的。(Picture Miss)

而Enhanced buffer descriptor一个有64字节，也是采用大端存储模式的，个人觉得这个Ethernet IP有点像是从PowerPC那边扣过来的。(Picture Miss)

 可以从fec.h文件中找到对这两个描述符的定义：

```c
struct bufdesc {	
	unsigned short cbd_datlen;	/* Data length */	
	unsigned short cbd_sc;	/* Control and status info */	
	unsigned long cbd_bufaddr;	/* Buffer address */
#ifdef CONFIG_ENHANCED_BD	
	unsigned long cbd_esc;	
	unsigned long cbd_prot;
	unsigned long cbd_bdu;
	unsigned long ts;	
	unsigned short res0[4];
#endif
```

如果定义了CONFIG_ENHANCED_BD宏，则开启Enhanced buffer descriptor的支持。不过纵观整个driver程序，3.0.35的内核并没有使用enhanced buffer descriptor使用的一些功能，比如Enhanced transmit buffer descriptor中的offset+8位置的PINS和IINS位，提供了采用MAC提供的IP accelerator进行硬件校验，提供对协议的校验和IP头的校验。而在yocto 3.10.17内核上，这些已经支持了！这也是为什么3.0.35上的Ethernet driver的性能不如3.10.17上的原因之一吧。下面继续分析代码：

```c
spin_lock_init(&fep->hw_lock); /* 初始化自旋锁 */
fep->netdev = ndev; /*把net_device的地址传给netdev*/
/* Get the Ethernet address */
fec_get_mac(ndev);
```

fec_get_mac会从多个地方获取mac地址：

```c
static void __inline__ fec_get_mac(struct net_device *ndev){	
	struct fec_enet_private *fep = netdev_priv(ndev);	
	struct fec_platform_data *pdata = fep->pdev->dev.platform_data;	
	unsigned char *iap, tmpaddr[ETH_ALEN];
	/*	 
	 * try to get mac address in following order:	 
	 *	 
	 * 1) module parameter via kernel command line in form	 
	 *    fec.macaddr=0x00,0x04,0x9f,0x01,0x30,0xe0	 
	 */	iap = macaddr;
	/*	 
	 * 2) from flash or fuse (via platform data)	 
	 */	
	 if (!is_valid_ether_addr(iap)) {		
	 	if (pdata)			
	 		memcpy(iap, pdata->mac, ETH_ALEN);	}
	/*	 
	 * 3) FEC mac registers set by bootloader	 
	 /	
	 if (!is_valid_ether_addr(iap)) {		
	 	*((unsigned long *) &tmpaddr[0]) =			
	 		be32_to_cpu(readl(fep->hwp + FEC_ADDR_LOW));		
	 	*((unsigned short *) &tmpaddr[4]) =			
	 		be16_to_cpu(readl(fep->hwp + FEC_ADDR_HIGH) >> 16);		
	 	iap = &tmpaddr[0];	
	 }
	memcpy(ndev->dev_addr, iap, ETH_ALEN);
	/* Adjust MAC if using macaddr */	
	if (iap == macaddr)		 
		ndev->dev_addr[ETH_ALEN-1] = macaddr[ETH_ALEN-1] + fep->pdev->id;}
```

1）首先是从全局变量macaddr获取ip地址，macaddr定义相关的代码如下：

```c
static unsigned char macaddr[ETH_ALEN];
module_param_array(macaddr, byte, NULL, 0);
MODULE_PARM_DESC(macaddr, "FEC Ethernet MAC address");
__setup("fec_mac=", fec_mac_addr_setup);
```

这里的__setup是用来从uboot传给内核的启动参数中捕获fec_mac（即mac地址）参数，并将该参数传递给fec_mac_addr_setup(char *mac_addr)函数进行解析的。如果uboot中没有传递mac参数，那么macaddr数组中的成员全是0。

2）检测1）中获取的mac地址是否合法，如果不合法，则从设备的私有数据结构（如果pdata指针不为空）struct fec_platform_data中获取mac数组的值。

3）检测2）中获取的mac地址是否合法，如果不合法，则读取Ethernet控制器的mac地址寄存器来获取mac地址。

最后把mac地址传递给内核中net_device结构体中的dev_addr字段。

接着继续分析代码：

```c
/* Set receive and transmit descriptor base. */
fep->rx_bd_base = cbd_base;
fep->tx_bd_base = cbd_base + RX_RING_SIZE;
```

设置tx_bd_base和rx_bd_base，即tx buffer descriptor base 和rx buffer descriptor base，示意图如下：(Picture Miss)

 接着就是net_device已经fec_enet_private等结构体的设置：

```c
/* The FEC Ethernet specific entries in the device structure */
ndev->watchdog_timeo = TX_TIMEOUT; /* watchdong定时器唤醒间隔 */
ndev->netdev_ops = &fec_netdev_ops;
ndev->ethtool_ops = &fec_enet_ethtool_ops;


fep->use_napi = FEC_NAPI_ENABLE;
fep->napi_weight = FEC_NAPI_WEIGHT;
if (fep->use_napi) {
	fec_rx_int_is_enabled(ndev, false);
	netif_napi_add(ndev, &fep->napi, fec_rx_poll, fep->napi_weight);
}
```

下面netdev_ops是一个比较重要的结构体指针，同样地ethtool_ops也是一个常用的结构体指针，我们在稍后讲解。先分析剩余的内容。这里涉及到了一个新的东西napi（理解为new api），它主要为了提升在网络负荷较大的情况下的性能。一般来说，网络接收数据包是通过中断来通知内核的，但是napi会判断在网络负荷较大的情况下改为轮询处理接收数据包，这样显得更加高效，同时napi也会在网络负荷较小的情况下改为中断接收数据包，这里napi_weight个人理解是用于判断是否开启轮询模式的权值，一般情况下64用的比较多。接着判断如果开启了napi，那么首先禁止rx的中断，接着再注册napi，同时提供一个轮询的函数fec_rx_poll()，这个在稍后和netdev_ops一起讲解。

接着初始化rx buffer descriptor（tx的原理相同，这里略去）

```c
/* Initialize the receive buffer descriptors. */
	bdp = fep->rx_bd_base;
	for (i = 0; i < RX_RING_SIZE; i++) {


		/* Initialize the BD for every fragment in the page. */
		bdp->cbd_sc = 0;
		bdp->cbd_bufaddr = 0;
		bdp++;
	}


	/* Set the last buffer to wrap */
	bdp--;
	bdp->cbd_sc |= BD_SC_WRAP;
```

默认情况下rx buffer的控制字段和rx buffer的地址都应该是0，最后在末尾的buffer处设置wrap，示意图如下：(Picture Miss)

 最后整个Ethernet restart：

```c
fec_restart(ndev, 0);
```

下面分分析这个函数，一开始是对寄存器进行操作，注释写得很明白：

```c
/* Whack a reset.  We should wait for this. */
writel(1, fep->hwp + FEC_ECNTRL);
udelay(10);


/* if uboot don't set MAC address, get MAC address
	* from command line; if command line don't set MAC
	* address, get from OCOTP; otherwise, allocate random
	* address.
	*/
memcpy(&temp_mac, dev->dev_addr, ETH_ALEN);
writel(cpu_to_be32(temp_mac[0]), fep->hwp + FEC_ADDR_LOW);
writel(cpu_to_be32(temp_mac[1]), fep->hwp + FEC_ADDR_HIGH);


/* Clear any outstanding interrupt. */
writel(0xffc00000, fep->hwp + FEC_IEVENT);


/* Reset all multicast.	*/
writel(0, fep->hwp + FEC_GRP_HASH_TABLE_HIGH);
writel(0, fep->hwp + FEC_GRP_HASH_TABLE_LOW);

 /* Set maximum receive buffer size. */
writel(PKT_MAXBLR_SIZE, fep->hwp + FEC_R_BUFF_SIZE);


/* Set receive and transmit descriptor base. */
writel(fep->bd_dma, fep->hwp + FEC_R_DES_START);
writel((unsigned long)fep->bd_dma + sizeof(struct bufdesc) * RX_RING_SIZE,
		fep->hwp + FEC_X_DES_START);
```

最后一句设置rx和tx描述符基地址，这里由于DMA控制器访问的是物理地址，因此需要我们把tx/tx descriptor base的物理地址写进寄存器，fep->bd_dma是在dma_alloc_noncacheable()函数中赋值的。同样地也是RX地址在前，TX地址在后。

```c
/* Reinit transmit descriptors */
fec_enet_txbd_init(dev);


fep->dirty_tx = fep->cur_tx = fep->tx_bd_base;
fep->cur_rx = fep->rx_bd_base;


/* Reset SKB transmit buffers. */
fep->skb_cur = fep->skb_dirty = 0;
for (i = 0; i <= TX_RING_MOD_MASK; i++) {
	if (fep->tx_skbuff[i]) {
		dev_kfree_skb_any(fep->tx_skbuff[i]);
		fep->tx_skbuff[i] = NULL;
	}
}
```

这里主要是针对descriptor指针的初始化，dirty_tx指向还没有释放的buffer对应的descriptor，cur_tx,cur_rx分别指向当前已经填充的buffer对应的descriptor。最后复位Socket Buffer 发送缓冲区，这里也对应了两个计数值，skb_cur,skb_dirty，用于指向还没有释放的buffer以及当前已经填充的buffer：(Picture Miss)

同时遍历整个 tx_skbuff[]指针数组，释放非空指针，socket buffer指针数组：struct sk_buff* tx_skbuff[TX_RING_SIZE]的示意图如下（请注意图中的红色字体！）：(Picture Miss)

接下来设置半双工或者全双工模式，默认情况下是半双工模式（即发送时不接受数据） 

```c
/* Enable MII mode */
if (duplex) {
	/* MII enable / FD enable */
	writel(OPT_FRAME_SIZE | 0x04, fep->hwp + FEC_R_CNTRL);
	writel(0x04, fep->hwp + FEC_X_CNTRL);
} else {
	/* MII enable / No Rcv on Xmit */
	writel(OPT_FRAME_SIZE | 0x06, fep->hwp + FEC_R_CNTRL);
	writel(0x0, fep->hwp + FEC_X_CNTRL);
}
fep->full_duplex = duplex;


/* Set MII speed */
writel(fep->phy_speed, fep->hwp + FEC_MII_SPEED);
```

下面用于设置物理层接口相关的部分省略过。接着是开启IEEE 1588的定时器：

```c
if (fep->ptimer_present) {
	/* Set Timer count */
	ret = fec_ptp_start(fep->ptp_priv);
	if (ret) {
		fep->ptimer_present = 0;
		reg = 0x0;
	} else
		reg = 0x0;
} else
	reg = 0x0;
```

下面与寄存器相关的一些接口这里省略，那么到了这里MAC层相关的driver就只剩下ndev->netdev_ops入口以及ndev->ethtool_ops入口了：

```c
static const struct net_device_ops fec_netdev_ops = {
	.ndo_open		= fec_enet_open,
	.ndo_stop		= fec_enet_close,
	.ndo_start_xmit		= fec_enet_start_xmit,
	.ndo_set_multicast_list = set_multicast_list,
	.ndo_change_mtu		= eth_change_mtu,
	.ndo_validate_addr	= eth_validate_addr,
	.ndo_tx_timeout		= fec_timeout,
	.ndo_set_mac_address	= fec_set_mac_address,
	.ndo_do_ioctl		= fec_enet_ioctl,
#ifdef CONFIG_NET_POLL_CONTROLLER
	.ndo_poll_controller    = fec_enet_netpoll,
#endif
};
```

这里重点介绍的是前面三个函数，即open,stop,start_xmit。

首先看open函数，那么open函数对应着用户空间程序ifconfig的up操作，当我们执行ifconfig eth0 up的时候即会调用open，同样地，down操作时也会对应stop。首先是napi：

```c
if (fep->use_napi)
	napi_enable(&fep->napi);
```

如果开启了napi，那么就要enable。接下来是clock的enable：

```c
clk_enable(fep->clk);
```

接着是为接收缓冲区分配buffer，前面说过tx buffer是由内核的网络子系统分配的， 而rx buffer是由Ethernet驱动分配。

```c
ret = fec_enet_alloc_buffers(ndev);
if (ret)
	return ret;
```

这个函数与之前初始化buffer descriptor的类似，这里需要注意的是对rx socket buffer使用了流式映射，我个人以前常常对于流式映射的方向搞不清楚，这里再多说一些：

DMA_FROM_DEVICE

​    传输方向是从device那边的往ram写数据，因此其实是接收外部发送的数据。

DMA_TO_DEVICE

​    传输方向是从ram往device那边传数据，因此其实是发送数据到外部。

接收缓冲区的示意图如下，这里要注意的是接收缓冲区是有Ethernet Driver分配的，不是由内核网络子系统分配的：（Picture Miss)

对于tx buffer descriptor部分来说，这里tx_bounce指向的分配的内存是用来进行buffer字节对齐的，对应的tx buffer descriptor相关的示意图：(Picture Miss)

然后是与物理层的相关的函数：

```c
/* Probe and connect to PHY when open the interface */
ret = fec_enet_mii_probe(ndev);
if (ret) {
	fec_enet_free_buffers(ndev);
	return ret;
}


phy_start(fep->phy_dev);
netif_start_queue(ndev);
fep->opened = 1;


ret = -EINVAL;
if (pdata->init && pdata->init(fep->phy_dev))
	return ret;


return 0
```

这部分等到MAC层相关的内容全部介绍完再讲解。对应fec_enet_close所作的事情反之，不再讲解。

**下面讲述fec_enet_start_xmit：**

首先关闭全局中断

```c
spin_lock_irqsave(&fep->hw_lock, flags);
```

接着判断link是否断开，这里个人不是太明白，发送过程中link是否会中断：

```c
if (!fep->link) {
	/* Link is down or autonegotiation is in progress. */
	netif_stop_queue(ndev);
	spin_unlock_irqrestore(&fep->hw_lock, flags);
	return NETDEV_TX_BUSY;
}
```

接着从发送队列中获取待发送buffer对应的buffer描述符指针，并获取当前描述符的状态：

```c
/* Fill in a Tx ring entry */
bdp = fep->cur_tx;
status = bdp->cbd_sc;
```

接着判断当前buffer描述符的状态：

```c
if (status & BD_ENET_TX_READY) {
	/* Ooops.  All transmit buffers are full.  Bail out.
	 * This should not happen, since ndev->tbusy should be set.
	 */
	printk("%s: tx queue full!.\n", ndev->name);
	netif_stop_queue(ndev);
	spin_unlock_irqrestore(&fep->hw_lock, flags);
	return NETDEV_TX_BUSY;
}
```

如果当前buffer描述状态的BD_ENET_TX_READY位还没有被DMA控制器清零，那么说明当前的buffer还没有被发送出去，也就是发送队列满，因此需要我们需要停止内核网络系统的发送队列，并且恢复中断现场，返回BUSY状态。

下面接着是队列没有满的情况下执行：

```c
/* Clear all of the status flags */
status &= ~BD_ENET_TX_STATS;


/* Set buffer length and buffer pointer */
bufaddr = skb->data;
bdp->cbd_datlen = skb->len;
```

很简单，把从buffer描述符中获取的status清零，并且从内核网络层传过来的socket buffer结构体中获取要发送的buffer的逻辑地址（注意，这里是逻辑地址，后面会有物理地址）和长度。接着是字节对齐：

```c
/*
 * On some FEC implementations data must be aligned on
 * 4-byte boundaries. Use bounce buffers to copy data
 * and get it aligned. Ugh.
 */
if (((unsigned long) bufaddr) & FEC_ALIGNMENT) {
	unsigned int index;
	index = bdp - fep->tx_bd_base;
	bufaddr = PTR_ALIGN(fep->tx_bounce[index], FEC_ALIGNMENT + 1);
	memcpy(bufaddr, (void *)skb->data, skb->len);
}
```

Ethernet driver单独分配了tx_bounce[]指针数组用来进行字节对齐，我常常在想有更高效的解决方法呢，毕竟还是占用了不少ram，不过能解决问题就好。接着是IEEE 1588协议的支持，即标记时间戳：

```c
	if (fep->ptimer_present) {
		if (fec_ptp_do_txstamp(skb)) {
			estatus = BD_ENET_TX_TS;
			status |= BD_ENET_TX_PTP;
		} else
			estatus = 0;
#ifdef CONFIG_ENHANCED_BD
		bdp->cbd_esc = (estatus | BD_ENET_TX_INT);
		bdp->cbd_bdu = 0;
#endif
	}
```

同时这里还提供了对ENHANCED BUFFER DESCRIPTOR的支持，在yocto 3.10.17-ga中会支持地更好。接着是大小端的转换，有一部分IP用的是大端模式：

```c
/*
 * Some design made an incorrect assumption on endian mode of
 * the system that it's running on. As the result, driver has to
 * swap every frame going to and coming from the controller.
 */
if (id_entry->driver_data & FEC_QUIRK_SWAP_FRAME)
	swap_buffer(bufaddr, skb->len);
```

接着就是把socket buffer指针放进之前分配好的TX指针数组：

```c
/* Save skb pointer */
fep->tx_skbuff[fep->skb_cur] = skb;


ndev->stats.tx_bytes += skb->len;
fep->skb_cur = (fep->skb_cur+1) & TX_RING_MOD_MASK;
```

同时对net_device结构体中的统计数据进行更新（个人觉得这一块放在发送完成的函数中更好），然后TX指针数组下标递增。接着就是对待发送的socket buffer进行DMA映射，映射的过程也就是刷新Data Cache的过程：

```c
/* Push the data cache so the CPM does not get stale memory
 * data.
 */
bdp->cbd_bufaddr = dma_map_single(&fep->pdev->dev, bufaddr,
		FEC_ENET_TX_FRSIZE, DMA_TO_DEVICE);


/* Send it on its way.  Tell FEC it's ready, interrupt when done,
 * it's the last BD of the frame, and to put the CRC on the end.
 */
status |= (BD_ENET_TX_READY | BD_ENET_TX_INTR
		| BD_ENET_TX_LAST | BD_ENET_TX_TC);
bdp->cbd_sc = status;
```

映射完毕后标记status变量READY,INTR,LAST,TC等标志位，最后写回当前socket buffer对应的buffer descriptor的状态字节。下面就是写寄存器触发硬件的发送操作：

```c
	/* Trigger transmission start */
	writel(0, fep->hwp + FEC_X_DES_ACTIVE);
```

下面这行代码本来应该是没有的，我只能理解为IC的BUG : (	，本质上就是DMA发送到某个位置并满足某些条件额情况下，调度工作队列来，工作队列中就是解决这个BUG的代码。

```c
bdp_pre = fec_enet_get_pre_txbd(ndev);
if ((id_entry->driver_data & FEC_QUIRK_BUG_TKT168103) &&
	!(bdp_pre->cbd_sc & BD_ENET_TX_READY))
	schedule_delayed_work(&fep->fixup_trigger_tx,
				 msecs_to_jiffies(1));
```



下面递增buffer描述符指针，如果当前处在环的末尾就跳到开头：

```c
/* If this was the last BD in the ring, start at the beginning again. */
	if (status & BD_ENET_TX_WRAP)
		bdp = fep->tx_bd_base;
	else
		bdp++;
```



如果cur_tx和dirty_tx指针指向同一块地方的话，表示软件还没有来得及释放已经发送完成的buffer：

```c
if (bdp == fep->dirty_tx) {
	fep->tx_full = 1;
	netif_stop_queue(ndev);
}


fep->cur_tx = bdp;
```

这里的full区别于一开始的full，一开始的full是由于DMA传送比发送函数的填充要慢，而这里的full是由socket buffer释放地比发送函数填充地慢导致的。最后cur_tx指针指向当前的描述符，并恢复中断上下文，返回发送成功：

```c
spin_unlock_irqrestore(&fep->hw_lock, flags);


return NETDEV_TX_OK;
```

如果实际硬件发送成功之后，会触发发送中断，而中断入口函数就是在fec_probe函数中注册的fec_enet_interrupt()函数，并且如果接受网络传送过了的数据包之后也会触发这个中断。

**下面来分析这个中断函数fec_enet_interrupt()：**

```c
do {
	int_events = readl(fep->hwp + FEC_IEVENT);
	writel(int_events, fep->hwp + FEC_IEVENT);
…………
…………
} while (int_events);
```

纵观整个函数非常短小（当然这是应该的，大家都知道顶半部应该做重要的事情，不重要的留给底半部），并且是一个do while循环，首先是读取FEC_IEVENT寄存器，然后再写相同的值清掉这些寄存器位，这里很明显是写1清零，然后再读有没有中断过来，有的话再次这处理。下面分析while循环里面的内容：

如果是接收中断的话：

```c
if (int_events & FEC_ENET_RXF) {
	ret = IRQ_HANDLED;
	spin_lock_irqsave(&fep->hw_lock, flags);


	if (fep->use_napi) {
		/* Disable the RX interrupt */
		if (napi_schedule_prep(&fep->napi)) {
			fec_rx_int_is_enabled(ndev, false);
			__napi_schedule(&fep->napi);
		}
	} else
		fec_enet_rx(ndev);


	spin_unlock_irqrestore(&fep->hw_lock, flags);
}
```

那么首先是进入临界段操作，然后使判断是不是用的napi，如果是的话就调用napi_schedule_prep()函数检测队列是否已经在调度，如果没有（即返回true）则标记napi开始运行。然后if条件，关闭rx中断并调度napi队列。如果不是napi的话那么就直接执行fec_enet_rx()函数。接着如果是发送中断的话：

```c
	/* Transmit OK, or non-fatal error. Update the buffer
	* descriptors. FEC handles all errors, we just discover
	* them as part of the transmit process.
	*/
if (int_events & FEC_ENET_TXF) {
	ret = IRQ_HANDLED;
	fec_enet_tx(ndev);
}
```

直接执行fec_enet_tx()函数，while循环后面还有两个中断，分别是timestamp定时器中断和MII中断，这里略过。下面我就要着重分析fec_enet_tx()和fec_enet_rx()函数。

**首先看fec_enet_tx()函数：**

首先获取dirty_tx指向的描述符：

```c
bdp = fep->dirty_tx;
```

然后就是一个大的while循环：

```c
while (((status = bdp->cbd_sc) & BD_ENET_TX_READY) == 0) {
…………
…………
	/* Update pointer to next buffer descriptor to be transmitted */
	if (status & BD_ENET_TX_WRAP)
		bdp = fep->tx_bd_base;
	else
		bdp++;
}
```

这里我调换了一下前后的顺序，但是不影响程序执行的结果，基本的循环就是一个不断递增dirty_tx指针的过程，递增一直递增到待发送buffer的描述符为止。下面分析while循环中的代码：

```c
if (bdp == fep->cur_tx && fep->tx_full == 0)
	break;
```

这段代码一开始比较难理解，但多看几遍就能想到这是整个队列空的情况，此时dirty_tx和cur_tx指向同一块区域，并且tx_full为0（这里存在两种情况，当dirty_tx==cur_tx时，既有可能是队列空也有可能是队列满，因此需要tx_full来区分，但是实际我觉得不应该这样做，因为这样会增加程序逻辑的复杂性，而到了3.10.17-ga内核中，队列为空时dirty_tx会指向cur_tx前一块描述符）。队列非空之后就开始释放发送成功的缓冲区：

```c
if (bdp->cbd_bufaddr)
	dma_unmap_single(&fep->pdev->dev, bdp->cbd_bufaddr,
		FEC_ENET_TX_FRSIZE, DMA_TO_DEVICE);
bdp->cbd_bufaddr = 0;
```

下面是DMA UNMAP操作，这里对应的是skb = fep->tx_skbuff[fep->skb_dirty];DMA_TO_DEVICE方向。下面是一些列错误检查：

```c
skb = fep->tx_skbuff[fep->skb_dirty];
if (!skb)
	break;
```

从skb_dirty下标获取dirty（这里dirty表示使用完还没有释放）的socket buffer，判断是否非空。接着是net_device结构体的数据统计：

```c
/* Check for errors. */
if (status & (BD_ENET_TX_HB | BD_ENET_TX_LC |
			BD_ENET_TX_RL | BD_ENET_TX_UN |
			BD_ENET_TX_CSL)) {
	ndev->stats.tx_errors++;
	if (status & BD_ENET_TX_HB)  /* No heartbeat */
		ndev->stats.tx_heartbeat_errors++;
	if (status & BD_ENET_TX_LC)  /* Late collision */
		ndev->stats.tx_window_errors++;
	if (status & BD_ENET_TX_RL)  /* Retrans limit */
		ndev->stats.tx_aborted_errors++;
	if (status & BD_ENET_TX_UN)  /* Underrun */
		ndev->stats.tx_fifo_errors++;
	if (status & BD_ENET_TX_CSL) /* Carrier lost */
		ndev->stats.tx_carrier_errors++;
} else {
	ndev->stats.tx_packets++;
}
```

下面的这段代码基本不可能发生，个人认为是为了Debug用的

```c
if (status & BD_ENET_TX_READY)
	printk("HEY! Enet xmit interrupt and TX_READY.\n");
```

接着依旧是数据统计

```c
/* Deferred means some collisions occurred during transmit,
 * but we eventually sent the packet OK.
 */
if (status & BD_ENET_TX_DEF)
	ndev->stats.collisions++;
```

下面用来标记发送的时间戳，个人还没有完全理解IEEE 1588协议及其内核驱动：

```c
#if defined(CONFIG_ENHANCED_BD)
		if (fep->ptimer_present) {
			if (bdp->cbd_esc & BD_ENET_TX_TS)
				fec_ptp_store_txstamp(fpp, skb, bdp);
		}
#elif defined(CONFIG_IN_BAND)
		if (fep->ptimer_present) {
			if (status & BD_ENET_TX_PTP)
				fec_ptp_store_txstamp(fpp, skb, bdp);
		}
#endif
```

接着就是释放socket buffer：

```c
/* Free the sk buffer associated with this last transmit */
dev_kfree_skb_any(skb);
fep->tx_skbuff[fep->skb_dirty] = NULL;
fep->skb_dirty = (fep->skb_dirty + 1) & TX_RING_MOD_MASK;
```

同时TX指针数组对应的成员指向NULL，skb_dirty下标递增。最后是处理tx_full为1的情况：

```c
/* Since we have freed up a buffer, the ring is no longer full
*/
if (fep->tx_full) {
	fep->tx_full = 0;
	if (netif_queue_stopped(ndev))
		netif_wake_queue(ndev);
}
```

当然这部分到了3.10.17也被精简了。跳出了while循环之后就更新dirty_tx指针：

```c
fep->dirty_tx = bdp;
```

**下面来看fec_enet_rx()函数：**

类似于fec_enet_tx()函数，也是一个主while循环：

```c
bdp = fep->cur_rx;


while (!((status = bdp->cbd_sc) & BD_ENET_RX_EMPTY)) {
………………
	/* Update BD pointer to next entry */
	if (status & BD_ENET_RX_WRAP)
		bdp = fep->rx_bd_base;
	else
		bdp++;


}
fep->cur_rx = bdp;
```

也就是获取当前rx buffer对应的buffer描述符指针cur_rx，然后不停地判断其BD_ENET_RX_EMPTY位是否为0，如果为0则表示这个buffer已经被DMA控制器处理过，接受到了数据，然后进入while循环处理，最后再递增指针并更新cur_rx指针。下面来看while循环里面的内容：

```c
/* Since we have allocated space to hold a complete frame,
 * the last indicator should be set.
 */
if ((status & BD_ENET_RX_LAST) == 0)
	printk("FEC ENET: rcv is not +last\n");
```

这条语句纯粹是用来debug的，一般情况下不会出现BD_ENET_RX_LAST位没有置1的情况。接着判断物理层连接是否打开：

```c
if (!fep->opened)
	goto rx_processing_done;
```

下面全是进行错误检测的以及数据统计的：

```c
/* Check for errors. */
if (status & (BD_ENET_RX_LG | BD_ENET_RX_SH | BD_ENET_RX_NO |
	   BD_ENET_RX_CR | BD_ENET_RX_OV)) {
	ndev->stats.rx_errors++;
	if (status & (BD_ENET_RX_LG | BD_ENET_RX_SH)) {
		/* Frame too long or too short. */
		ndev->stats.rx_length_errors++;
	}
	if (status & BD_ENET_RX_NO)	/* Frame alignment */
		ndev->stats.rx_frame_errors++;
	if (status & BD_ENET_RX_CR)	/* CRC Error */
		ndev->stats.rx_crc_errors++;
	if (status & BD_ENET_RX_OV)	/* FIFO overrun */
		ndev->stats.rx_fifo_errors++;
}
/* Report late collisions as a frame error.
 * On this error, the BD is closed, but we don't know what we
 * have in the buffer.  So, just drop this frame on the floor.
 */
if (status & BD_ENET_RX_CL) {
	ndev->stats.rx_errors++;
	ndev->stats.rx_frame_errors++;
	goto rx_processing_done;
}
```

接下来也是，统计接受的数据包和字节数：

```c
/* Process the incoming frame. */
ndev->stats.rx_packets++;
pkt_len = bdp->cbd_datlen;
ndev->stats.rx_bytes += pkt_len;
```

接着进行DMA去映射和大小端转换：

```c
data = (__u8 *)__va(bdp->cbd_bufaddr);


if (bdp->cbd_bufaddr)
	dma_unmap_single(&fep->pdev->dev, bdp->cbd_bufaddr,
		FEC_ENET_RX_FRSIZE, DMA_FROM_DEVICE);


if (id_entry->driver_data & FEC_QUIRK_SWAP_FRAME)
	swap_buffer(data, pkt_len);
```

下面进行16字节边界对齐，因为包头是14字节，所以NET_IP_ALIGN为2：

```c
/* This does 16 byte alignment, exactly what we need.
	* The packet length includes FCS, but we don't want to
	* include that when passing upstream as it messes up
	* bridging applications.
	*/
skb = dev_alloc_skb(pkt_len - 4 + NET_IP_ALIGN);


if (unlikely(!skb)) {
	printk("%s: Memory squeeze, dropping packet.\n",
			ndev->name);
	ndev->stats.rx_dropped++;
} else {
	skb_reserve(skb, NET_IP_ALIGN);
	skb_put(skb, pkt_len - 4);	/* Make room */
	skb_copy_to_linear_data(skb, data, pkt_len - 4);
	/* 1588 messeage TS handle */
	if (fep->ptimer_present)
		fec_ptp_store_rxstamp(fpp, skb, bdp);
	skb->protocol = eth_type_trans(skb, ndev);
	netif_rx(skb);
}
```

对齐完了以后还要处理1588的Timestamp标记。关于skb_reserve的对齐作用，找了一篇博客，有图有真相：http://cooliron.blog.163.com/blog/static/124703138201322074856169/

对齐处理完了以后再重新建立dma映射：

```c
bdp->cbd_bufaddr = dma_map_single(&fep->pdev->dev, data,
		FEC_ENET_TX_FRSIZE, DMA_FROM_DEVICE);
```

接下来再更新dma buffer描述符中的status字段：

```c
rx_processing_done:
		/* Clear the status flags for this buffer */
		status &= ~BD_ENET_RX_STATS;


		/* Mark the buffer empty */
		status |= BD_ENET_RX_EMPTY;
		bdp->cbd_sc = status;
#ifdef CONFIG_ENHANCED_BD
		bdp->cbd_esc = BD_ENET_RX_INT;
		bdp->cbd_prot = 0;
		bdp->cbd_bdu = 0;
#endif


		/* Update BD pointer to next entry */
		if (status & BD_ENET_RX_WRAP)
			bdp = fep->rx_bd_base;
		else
			bdp++;
```

while循环的最后：

```c
/* Doing this here will keep the FEC running while we process
 * incoming frames.  On a heavily loaded network, we should be
 * able to keep up at the expense of system resources.
 */
writel(0, fep->hwp + FEC_R_DES_ACTIVE);
```

处理输入包的同时也保持FEC运行，提高吞吐量。

跳出while循环以后再更新cur_rx指针：

```c
fep->cur_rx = bdp;
```

到这里，以太网控制器的发送和接受以及中断相关的函数介绍地差不多了，写完一遍个人发现有点混乱，对于比较大型的程序也没有太多的经验，所以可能不太好理解，以后要多分析总结！目前以太网的驱动分析暂时到这里就结束了，而对于物理层的操作分析，由于精力有限加上对此认识地不够透彻，只能暂时搁置。
