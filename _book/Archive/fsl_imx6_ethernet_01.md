# Freescale i.MX6 Linux Ethernet Driver驱动源码分析01

**本文最初发表于2014-07-31，网易博客**

最近需要在Freescale i.MX6上移植Ethernet AVB的内核patch，Ethernet AVB的Wiki：http://en.wikipedia.org/wiki/Audio_Video_Bridging，而Freescale原来已经在kernel 3.0.35 LTIB 4.0.0的基础上提供了patch：https://community.freescale.com/docs/DOC-95578，现在需要做的是把kernel 3.0.35上的patch移植到yocto kernel-3.10.17_ga上，第一次听上去就感觉像是代码的复制粘贴，不过首要的问题是Ethernet Driver都没有看过，还谈何移植，所以索性把Ethernet Driver也学习一遍，Google了一段时间，找到了几份不错的学习资料，特此共享之：http://free-electrons.com/doc/network-drivers-lab.pdf 

《Linux network driver development Training lab book》个人认为最经典的学习资料，从无到有地讲述了网卡驱动的编写过程，基于Atmel的MPU的Ethernet Controller。http://lwn.net/images/pdf/LDD3/ch17.pdf

《LDD3》的第17章节专门讲述了network driver，不过这里面着重介绍了一些技术细节，适合当工具书来看。

http://free-electrons.com/training/kernel/  这个页面是偶然找到的，不过对free-electrons印象一直不错，所以强烈推荐，这是专门讲内核与驱动开发的材料。

那几天我硬着头皮把《Linux network driver development Training lab book》看完了，后来又看了两遍，对Linux Ethernet Driver有了比较整体的认识，接下来的工作就是理解i.MX6 3.0.35内核的驱动源码。

这里不得不称赞的是，Freescale的驱动有相应的《i.MX 6Dual/6Quad Linux Reference
Manual》文档可以从官网上下到，它会给你一些简要的介绍。这对于很多初学者来说还是非常友好的。首先找到源码的位置：drviers/net/fec.c对应还有一个fec.h作头文件，同时现在大多数的MPU还集成了支持FEC1588的硬件控制器，从硬件上支持一些对时间有要求的应用，比如Ethernet AVB等等。这里以**3.0.35内核**目前最新release的**LTIB 4.1.0**环境的内核为参考。

```c
static int __init
fec_enet_module_init(void)
{
	printk(KERN_INFO "FEC Ethernet Driver\n");

	return platform_driver_register(&fec_driver);
}

static void __exit
fec_enet_cleanup(void)
{
	platform_driver_unregister(&fec_driver);
}

module_exit(fec_enet_cleanup);
module_init(fec_enet_module_init);

MODULE_LICENSE("GPL");
```

现在看到printk后面那个KERN_INFO一点都不觉得别扭，当初还被坑了。

printk有8个loglevel,定义在<linux/kernel.h>中，其中数值范围从0到7，数值越小，优先级越高。

```c
#define    KERN_EMERG      "<0>"      
#define    KERN_ALERT       "<1>"
#define    KERN_CRIT "<2>"      
#define    KERN_ERR    "<3>"      
#define    KERN_WARNING "<4>"      
#define    KERN_NOTICE     "<5>"      
#define    KERN_INFO "<6>"      
#define    KERN_DEBUG      "<7>"      
```

默认在Ubuntu rootfs下，优先级大于4才会显示出来，没有指定日志级别的printk语句默认采用的级别是 DEFAULT_ MESSAGE_LOGLEVEL（这个默认情况下是4，具体的解释见http://blog.sina.com.cn/s/blog_636a55070101i6sr.html）所以如果不设置优先级的情况下，在中断上是看不到printk打印的语句的，不过在dmesg中可以看到，当然也可以echo 8 > /proc/sys/kernel/printk来修改默认输出的最低优先级。

```c
static int fec_mac_addr_setup(char *mac_addr)
{
	char *ptr, *p = mac_addr;
	unsigned long tmp;
	int i = 0, ret = 0;

	while (p && (*p) && i < 6) {
		ptr = strchr(p, ':');
		if (ptr)
			*ptr++ = '\0';

		if (strlen(p)) {
			ret = strict_strtoul(p, 16, &tmp);
			if (ret < 0 || tmp > 0xff)
				break;
			macaddr[i++] = tmp;
		}
		p = ptr;
	}

	return 0;
}

__setup("fec_mac=", fec_mac_addr_setup);
```

这里的__setup是用来从uboot传给内核的启动参数中捕获fec_mac（即mac地址）参数，并将该参数传递给fec_mac_addr_setup(char *mac_addr)函数进行解析。具体内核参数解析的细节与内幕可以参考《Embedded Linux Primer: A Practical Real-World Approach》Christopher Hallinan （译本《嵌入式Linux基础教程》）。

接下来是fec_driver结构体：

```c
static struct platform_driver fec_driver = {
	.driver	= {
		.name	= DRIVER_NAME,
		.owner	= THIS_MODULE,
#ifdef CONFIG_PM
		.pm	= &fec_pm_ops,
#endif
	},
	.id_table = fec_devtype,
	.probe	= fec_probe,
	.remove	= __devexit_p(fec_drv_remove),
};
```

这里出了比较常见的.name,.owner,.probe,.remove以外，还多了.pm（power management）结构体。这里关于__devexit_p宏我不得不插嘴，每次都能见到类似地，但是从来没有想过到底有什么用，它定义在include/linux/init.h中：

```c
/* Functions marked as __devexit may be discarded at kernel link time, depending
   on config options.  Newer versions of binutils detect references from
   retained sections to discarded sections and flag an error.  Pointers to
   __devexit functions must use __devexit_p(function_name), the wrapper will
   insert either the function_name or NULL, depending on the config options.
 */
#if defined(MODULE) || defined(CONFIG_HOTPLUG)
#define __devexit_p(x) x
#else
#define __devexit_p(x) NULL
#endif
```

看完这一段应该不用我解释了，接下来要讲.id_table字段。指针指向的结构体如下：

```c
 static struct platform_device_id fec_devtype[] = {
	{
		.name = "enet",
		.driver_data = FEC_QUIRK_ENET_MAC | FEC_QUIRK_BUG_TKT168103,
	},
	{
		.name = "fec",
		.driver_data = 0,
	},
	{
		.name = "imx28-fec",
		.driver_data = FEC_QUIRK_ENET_MAC | FEC_QUIRK_SWAP_FRAME |
				FEC_QUIRK_BUG_TKT168103,
	},
	{ }
};
```

这里的fec_devtype[]数组主要是用来进行device与driver的匹配，在2.6模型中很重要的改进就是设备模型。默认情况下是以device的名字和driver的名字相同即匹配成功，但真正platform bus的匹配函数在drivers/base/platform.c中定义，首先可以看到platform_bus_type是bus_type的实例化。其中的.match函数即是用来匹配的：

```c
static int platform_match(struct device *dev, struct device_driver *drv)
{
	struct platform_device *pdev = to_platform_device(dev);
	struct platform_driver *pdrv = to_platform_driver(drv);

	/* Attempt an OF style match first */
	if (of_driver_match_device(dev, drv))
		return 1;

	/* Then try to match against the id table */
	if (pdrv->id_table)
		return platform_match_id(pdrv->id_table, pdev) != NULL;

	/* fall-back to driver name match */
	return (strcmp(pdev->name, drv->name) == 0);
}
```

可以看到匹配有三个条件：

1）如果内核开启了设备树支持的话优先匹配设备树，这种情况下，内核会寻找.driver 中的.of_match_table匹配表是否与设备树中设备结点的.compatible值相同，是则匹配成功。

2）使用驱动中的.id_table列表匹配，在id_table数组中寻找与device的名字相匹配的成员。

3）最后单纯地比较device的名字和driver的名字是否相同，是则匹配成功返回。

看完driver看device，接下来看看板级信息是在哪里添加的吧（我找的比较纠结）。

在arch/arm/mach-mx6/board-mx6q_sabresd.c文件中的mx6_sabresd_board_init()函数中，可以找到imx6_init_fec(fec_data);这一行代码即为添加以太网device的代码。而mx6_sabresd_board_init函数在MACHINE_START()函数中被初始化：

```c
/*
 * initialize __mach_desc_MX6Q_SABRESD data structure.
 */
MACHINE_START(MX6Q_SABRESD, "Freescale i.MX 6Quad/DualLite/Solo Sabre-SD Board")
	/* Maintainer: Freescale Semiconductor, Inc. */
	.boot_params = MX6_PHYS_OFFSET + 0x100,
	.fixup = fixup_mxc_board,
	.map_io = mx6_map_io,
	.init_irq = mx6_init_irq,
	.init_machine = mx6_sabresd_board_init,
	.timer = &mx6_sabresd_timer,
	.reserve = mx6q_sabresd_reserve,
MACHINE_END
```

ARM启动初始化部分的代码以后有时间再慢慢分析，这里主要是针对Sabre-SD Board进行初始化，调用一系列函数。下面分析imx6_init_fec(fec_data);

定义在arch/arm/mach-mx6/mx6_fec.c

```c
void __init imx6_init_fec(struct fec_platform_data fec_data)
{
	fec_get_mac_addr(fec_data.mac);
	if (!is_valid_ether_addr(fec_data.mac))
		random_ether_addr(fec_data.mac);


	if (cpu_is_mx6sl())
		imx6sl_add_fec(&fec_data);
	else
		imx6q_add_fec(&fec_data);
}
```

这里struct fec_platform_data fec_data是设备的私有数据，定义在include/linux/fec.h：

```c
struct fec_platform_data {
	int (*init) (struct phy_device *);
	int (*power_hibernate) (struct phy_device *);
	phy_interface_t phy;
	unsigned char mac[ETH_ALEN];
	int gpio_irq;
};
```

而这里的fec_data是已经在arch/arm/mach-mx6/board-mx6q_sabresd.c静态初始化好的：

```c
static struct fec_platform_data fec_data __initdata = {
	.init = mx6q_sabresd_fec_phy_init,
	.phy = PHY_INTERFACE_MODE_RGMII,
	.gpio_irq = MX6_ENET_IRQ,
};
```

可以看到imx6_init_fec()最后是执行imx6q_add_fec(&fec_data)添加设备，定义在arch/arm/mach-mx6/devices-imx6q.h：

```c
extern const struct imx_fec_data imx6q_fec_data __initconst;
#define imx6q_add_fec(pdata)	\
	imx_add_fec(&imx6q_fec_data, pdata)
```

这里引用了外部的imx6q_fec_data，定义在arch/arm/plat-mxc/devices/platform-fec.c：

```c
#ifdef CONFIG_SOC_IMX6Q
const struct imx_fec_data imx6q_fec_data __initconst =
	imx_fec_data_entry_single(MX6Q, "enet");


const struct imx_fec_data imx6sl_fec_data __initconst =
	imx_fec_data_entry_single(MX6DL, "fec");
#endif
```

看到“enet”字段就知道离最终代码不远了，而imx_fec_data_entry_single同样定义在该文件内：

```c
#define imx_fec_data_entry_single(soc, _devid)				\
	{								\
		.iobase = soc ## _FEC_BASE_ADDR,			\
		.irq = soc ## _INT_FEC,					\
		.devid = _devid,					\
	}
```

这里的连续两个井号##用于把前后的字母连接起来组成一个变量名，井号左右的空格会被忽略。因此上面的定义展开以后就是

\#define imx_fec_data_entry_single(MX6Q, "enet")

​	{

​		.iobase = MX6Q_FEC_BASE_ADDR,

​		.irq = MX6Q_INT_FEC,

​		.devid = "enet",

​	}

这两个宏可以在arch/arm/plat-mxc/include/mach/mx6.h中找到它们具体的值。

下面继续分析imx_add_fec(&imx6q_fec_data, pdata),定义在arch/arm/plat-mxc/devices/platform-fec.c中：

```c
struct platform_device *__init imx_add_fec(
		const struct imx_fec_data *data,
		const struct fec_platform_data *pdata)
{
	struct resource res[] = {
		{
			.start = data->iobase,
			.end = data->iobase + SZ_4K - 1,
			.flags = IORESOURCE_MEM,
		}, {
			.start = data->irq,
			.end = data->irq,
			.flags = IORESOURCE_IRQ,
		},
	};


	if (!fuse_dev_is_available(MXC_DEV_ENET))
		return ERR_PTR(-ENODEV);


	return imx_add_platform_device_dmamask(data->devid, 0,
			res, ARRAY_SIZE(res),
			pdata, sizeof(*pdata), DMA_BIT_MASK(32));
}
```

可以看到imx_fec_data的值最终都被传递给了resource结构体，然后通过imx_add_platform_device_dmamask()函数来将resource和额外的pdata数据添加到device中，最后在driver中获取device的这些信息。

分析完device的添加与driver的加载之后就到了probe入口函数：

static int __devinit fec_probe(struct platform_device *pdev)

这个函数主要做的事情就是初始化，首先是获取之前在设备中添加的resource属性，然后再申请该段内存空间的占用：

```c
r = platform_get_resource(pdev, IORESOURCE_MEM, 0);
if (!r)
	return -ENXIO;


r = request_mem_region(r->start, resource_size(r), pdev->name);
if (!r)
	return -EBUSY;
```

下面调用内核提供的api函数alloc_etherdev()初始化网络设备，传入的参数是driver的私有数据尺寸，用于额外分配driver的私有数据。

```c
/* Init network device */
ndev = alloc_etherdev(sizeof(struct fec_enet_private));
if (!ndev) {
	ret = -ENOMEM;
	goto failed_alloc_etherdev;
}
```

接下来告诉内核我们的设备属于网络设备，由于netdevice不属于char或者block设备，因此不能用常规的驱动方法来设计：

```c
SET_NETDEV_DEV(ndev, &pdev->dev);
定义如下：
/* Set the sysfs physical device reference for the network logical device * if set prior to registration will cause a symlink during initialization. 
*/#define SET_NETDEV_DEV(net, pdev)	((net)->dev.parent = (pdev))	
```

分配完netdevice以后，通过下面的函数获取指向分配的私有数据的指针。

```c
/* setup board info structure */fep = netdev_priv(ndev);
```

下面用ioremap映射，建立内核到寄存器物理地址的页表，以及给platform_device指针赋值。

```c
fep->hwp = ioremap(r->start, resource_size(r));
fep->pdev = pdev;


if (!fep->hwp) {
	ret = -ENOMEM;
	goto failed_ioremap;
}
```

接着设置platform driver的私有数据

```c
platform_set_drvdata(pdev, ndev);
```

这里其实最终是作了pdev->dev->p->driver_data = ndev的指针指向。

然后是接收之前在mach-mx6文件夹下静态添加device的platform_data：

```c
pdata = pdev->dev.platform_data;
```

接着是对pdata的数据进行解析：

```c
if (pdata)	fep->phy_interface = pdata->phy;
```

这里的phy是用来判断MAC层和以太网物理层的接口的，这里MX6Q的接口是PHY_INTERFACE_MODE_RGMII，也就是RGMII，千兆以太网接口。然后是获取以太网控制器对应的

```c
if (pdata->gpio_irq > 0) {
	gpio_request(pdata->gpio_irq, "gpio_enet_irq");
	gpio_direction_input(pdata->gpio_irq);

	irq = gpio_to_irq(pdata->gpio_irq);
	ret = request_irq(irq, fec_enet_interrupt,
			IRQF_TRIGGER_RISING,
			 pdev->name, ndev);
	if (ret)
		goto failed_irq;
} else {
	/* This device has up to three irqs on some platforms */
	for (i = 0; i < 3; i++) {
		irq = platform_get_irq(pdev, i);
		if (i && irq < 0)
			break;
		ret = request_irq(irq, fec_enet_interrupt,
				IRQF_DISABLED, pdev->name, ndev);
		if (ret) {
			while (--i >= 0) {
				irq = platform_get_irq(pdev, i);
				free_irq(irq, ndev);
			}
			goto failed_irq;
		}
	}
}
```

这里用于判断Ethernet的中断使用gpio边沿出发的中断还是从platform device的resource传过来的中断号。之前已经看到传递给driver的私有数据fec_data已经被静态地初始化为

```c
static struct fec_platform_data fec_data __initdata = {
	.init = mx6q_sabresd_fec_phy_init,
	.phy = PHY_INTERFACE_MODE_RGMII,
	.gpio_irq = MX6_ENET_IRQ,
};
```

这里MX6_ENET_IRQ的值是6，但是在添加私有数据前可以找到下面这句话，

```c
if (enet_to_gpio_6)
		/* Make sure the IOMUX_OBSRV_MUX1 is set to ENET_IRQ. */
	mxc_iomux_set_specialbits_register(
		IOMUX_OBSRV_MUX1_OFFSET,
		OBSRV_MUX1_ENET_IRQ,
		OBSRV_MUX1_MASK);
else
	fec_data.gpio_irq = -1;
imx6_init_fec(fec_data);
```

用来判断是否对fec_data的gpio_irq字段赋值-1，而enet_to_gpio_6是一个全局变量，定义在arch/arm/mach-mx6/cpu.c中，默认为0，同样在该c文件中可以找到下面这段代码：

```c
static int __init set_enet_irq_to_gpio(char *p)
{
	enet_to_gpio_6 = true;
	return 0;
}


early_param("enet_gpio_6", set_enet_irq_to_gpio);
```

early_param用来解析uboot传递给内核的参数，只有当参数中出现“enet_gpio_6”字段时，才会调用set_enet_irq_to_gpio，那么实际上是否调用是取决于运行时uboot的配置，默认情况下MX6Q的uboot是不带有这个参数，因此实际执行时，gpio_irq = -1。所以上面的if(pdata->gpio_irq > 0) {}不会执行，而是执行else分支，即实际从platform device的resource参数传递的信息中获取irq号，从前面的分析可以知道IRQ号被定义成宏MX6Q_INT_FEC，即150号中断，而具体irq的初始化过程有时间我再写一篇文章分析一下吧。对应的中断注册函数是fec_enet_interrupt()。

中断处理完了之后就到了时钟了，linux内核有一组专门的clock api用来处理时钟，简单点来说就只要知道首先clk_get()然后再clk_enable()就可以了，driver remove的时候反之即clk_disable()再clk_put()

，具体的时钟框架我有时间再写一篇文章分析一下，现在linux定义了一套全新的CCF框架（Common Clock Framework），资料也大多数为英文。

```c
fep->clk = clk_get(&pdev->dev, "fec_clk");
if (IS_ERR(fep->clk)) {
	ret = PTR_ERR(fep->clk);
	goto failed_clk;
}
fep->mdc_clk = clk_get(&pdev->dev, "fec_mdc_clk");
if (IS_ERR(fep->mdc_clk)) {
	ret = PTR_ERR(fep->mdc_clk);
	goto failed_clk;
}
clk_enable(fep->clk);
```

这里fec_clk指的是Ethernet控制器的clock，而fec_mdc_clk指的是MAC层与物理层接口MDIO的Clock。紧接着就到了MAC层的初始化函数：

```c
ret = fec_enet_init(ndev);
if (ret)
	goto failed_init;
```

再接着就是MAC层与物理层接口的初始化函数：

```c
ret = fec_enet_mii_init(pdev);
if (ret)
	goto failed_mii_init;
```

再接着就是对IEEE 1588 时钟同步协议的初始化：

```c
if (fec_ptp_malloc_priv(&(fep->ptp_priv))) {
	if (fep->ptp_priv) {
		fep->ptp_priv->hwp = fep->hwp;
		ret = fec_ptp_init(fep->ptp_priv, pdev->id);
		if (ret)
			printk(KERN_WARNING "IEEE1588: ptp-timer is unavailable\n");
		else
			fep->ptimer_present = 1;
	} else
		printk(KERN_ERR "IEEE1588: failed to malloc memory\n");
}
```

然后把内核网络层的传输队列关闭（即禁止发送），关闭时钟，等一些列辅助操作：

```c
/* Carrier starts down, phylib will bring it up */
netif_carrier_off(ndev);
clk_disable(fep->clk);


INIT_DELAYED_WORK(&fep->fixup_trigger_tx, fixup_trigger_tx_func);
```

最后也是最重要步骤，即向内核注册之前分配的net_device：

```c
ret = register_netdev(ndev);
if (ret)
	goto failed_register;
```

接下来将要详细讲述MAC层，物理层接口以及1588协议支持的代码。
