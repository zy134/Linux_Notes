## 设备驱动核心：device与driver

&emsp;&emsp;整个设备驱动架构核心由Device，Driver，Bus，Class四个部分组成。这四个结构体都从kobject“继承”而来，因而其生命周期与引用计数是相关的，需要通过kobject的get/put操作来控制。四个结构体的大致分工如下：

+ device： 位于/sys/devices目录下，表示某个设备，通常与硬件相关的资源(中断，IO物理地址，PM)等也会保存在其中
+ driver：位于/sys/bus/*/drivers目录下，表示驱动，即对某个device的操作。device与driver，可以简单的看作是数据与接口的关系，当device与driver绑定后，才可执行driver中的一系列操作。driver是挂载在bus上的，driver与device的绑定操作，也必须通过bus的协助来进行。
+ bus：位于/sys/bus目录下。在linux驱动架构上，driver必须挂载在某个总线上，bus可以绑定device与driver，从而使能device。这个总线可以是物理意义上的总线(I2C, SPI)，也可以是虚拟总线(platform bus)。通过总线可以很好的将设备驱动分为不同的层次。此外相同的物理总线上，设备一般遵循相同的specific，就可以将相同的属性与操作提取出来，提升软件的复用程度。
+ class：位于/sys/class目录下。顾名思义，表示分类。这个分类很杂，一般来说取决于实现，可以是物理性质上的分类，譬如I2C，gpio，也可以是功能分类，像是video4linux。/sys/class下的目录项一般都是到/sys/devices各个目录的链接。

## 1.device

#### (1) 概述

&emsp;&emsp;`struct device`是表示设备属性的结构体，包含诸多底层信息。先看定义:

```c
struct device {
	struct kobject kobj;
	struct device		*parent;

	struct device_private	*p;

	const char		*init_name; /* initial name of the device */
	const struct device_type *type;

	struct bus_type	*bus;		/* type of bus device is on */
	struct device_driver *driver;	/* which driver has allocated this
					   device */
	void		*platform_data;	/* Platform specific data, device
					   core doesn't touch it */
	void		*driver_data;	/* Driver data, set and get with
					   dev_set_drvdata/dev_get_drvdata */
#ifdef CONFIG_PROVE_LOCKING
	struct mutex		lockdep_mutex;
#endif
	struct mutex		mutex;	/* mutex to synchronize calls to
					 * its driver.
					 */

	struct dev_links_info	links;
	struct dev_pm_info	power;
	struct dev_pm_domain	*pm_domain;

#ifdef CONFIG_ENERGY_MODEL
	struct em_perf_domain	*em_pd;
#endif

#ifdef CONFIG_GENERIC_MSI_IRQ_DOMAIN
	struct irq_domain	*msi_domain;
#endif
#ifdef CONFIG_PINCTRL
	struct dev_pin_info	*pins;
#endif
#ifdef CONFIG_GENERIC_MSI_IRQ
	struct list_head	msi_list;
#endif
#ifdef CONFIG_DMA_OPS
	const struct dma_map_ops *dma_ops;
#endif
	u64		*dma_mask;	/* dma mask (if dma'able device) */
	u64		coherent_dma_mask;/* Like dma_mask, but for
					     alloc_coherent mappings as
					     not all hardware supports
					     64 bit addresses for consistent
					     allocations such descriptors. */
	u64		bus_dma_limit;	/* upstream dma constraint */
	const struct bus_dma_region *dma_range_map;

	struct device_dma_parameters *dma_parms;

	struct list_head	dma_pools;	/* dma pools (if dma'ble) */

#ifdef CONFIG_DMA_DECLARE_COHERENT
	struct dma_coherent_mem	*dma_mem; /* internal for coherent mem
					     override */
#endif
#ifdef CONFIG_DMA_CMA
	struct cma *cma_area;		/* contiguous memory area for dma
					   allocations */
#endif
	/* arch specific additions */
	struct dev_archdata	archdata;

	struct device_node	*of_node; /* associated device tree node */
	struct fwnode_handle	*fwnode; /* firmware device node */

#ifdef CONFIG_NUMA
	int		numa_node;	/* NUMA node this device is close to */
#endif
	dev_t			devt;	/* dev_t, creates the sysfs "dev" */
	u32			id;	/* device instance */

	spinlock_t		devres_lock;
	struct list_head	devres_head;

	struct class		*class;
	const struct attribute_group **groups;	/* optional groups */

	void	(*release)(struct device *dev);
	struct iommu_group	*iommu_group;
	struct dev_iommu	*iommu;

	bool			offline_disabled:1;
	bool			offline:1;
	bool			of_node_reused:1;
	bool			state_synced:1;
#if defined(CONFIG_ARCH_HAS_SYNC_DMA_FOR_DEVICE) || \
    defined(CONFIG_ARCH_HAS_SYNC_DMA_FOR_CPU) || \
    defined(CONFIG_ARCH_HAS_SYNC_DMA_FOR_CPU_ALL)
	bool			dma_coherent:1;
#endif
#ifdef CONFIG_DMA_OPS_BYPASS
	bool			dma_ops_bypass : 1;
#endif
};

```

&emsp;&emsp;这个庞大的结构体可以分成三个部分看待，一是device的基本属性，如：

+ kobject： 内嵌的kobject，用于注册sysfs中的文件项，以及管理引用计数
+ device_private：设备私有数据，在Linux设备驱动架构里面，主要用于存储device所属的bus、class、driver的引用，以及该device的父亲、兄弟节点等。
+ type：设备关联的一些操作，其实就是对应于底层的ktype，和sysfs操作有关
+ id： 设备号

&emsp;&emsp;二是表示device在设备驱动架构中的关系的各个指针，如：

+ driver：该设备所绑定的驱动，每个设备必须绑定驱动后才可用
+ bus：该设备所绑定的总线，初始化时传入，必须先于device初始化完成
+ class：该设备所属的class
+ dev_links_info：表示该device与其他device的comsumer/supplier关系，用于设备的初始化顺序，PM管理等等。这是个大话题，留给后续讨论。

&emsp;&emsp;第三部分主要是各个硬件资源模块，如DMA，IRQ，PM，IOMMU，DTS等等，这块留到对应模块时继续讨论。

#### (2) device主要接口

&emsp;&emsp;device的核心接口有以下几个：

```cpp
int __must_check device_register(struct device *dev);
void device_initialize(struct device *dev);
int __must_check device_add(struct device *dev);

void device_unregister(struct device *dev);
void device_del(struct device *dev);
```

&emsp;&emsp;`device_register`与`device_ungister`成对使用。其中`device_register`又是先后调用了`device_initialize`与`device_add`。`device_initialize`的作用主要是调用`kobject_initialize`初始化device结构体内嵌的kobject，包括赋予对应的kset、ktype：

```cpp
void device_initialize(struct device *dev)
{
    // 所有的struct device类型内嵌的kobject，使用的kset、ktype相同
    // kset对应的是目录/sys/devices/，在系统初始化时device_kset会先被创建完成
    // ktype主要用于释放对应的device结构体
	dev->kobj.kset = devices_kset;
	kobject_init(&dev->kobj, &device_ktype);
	INIT_LIST_HEAD(&dev->dma_pools);
	mutex_init(&dev->mutex);
	lockdep_set_novalidate_class(&dev->mutex);
	spin_lock_init(&dev->devres_lock);
	INIT_LIST_HEAD(&dev->devres_head);
    // 初始化pm，devlink，numa等其他组件
	device_pm_init(dev);
	set_dev_node(dev, NUMA_NO_NODE);
	INIT_LIST_HEAD(&dev->links.consumers);
	INIT_LIST_HEAD(&dev->links.suppliers);
	INIT_LIST_HEAD(&dev->links.defer_sync);
	dev->links.status = DL_DEV_NO_DRIVER;
#if defined(CONFIG_ARCH_HAS_SYNC_DMA_FOR_DEVICE) || \
    defined(CONFIG_ARCH_HAS_SYNC_DMA_FOR_CPU) || \
    defined(CONFIG_ARCH_HAS_SYNC_DMA_FOR_CPU_ALL)
	dev->dma_coherent = dma_default_coherent;
#endif
#ifdef CONFIG_SWIOTLB
	dev->dma_io_tlb_mem = &io_tlb_default_mem;
#endif
}
```

&emsp;&emsp;`device_add`完成了device注册的主要工作，首先可以想到的是既然`device_initialize`时调用了`kobject_initialize`，那么对应的就要去调用`kobject_add`了。但是要做的工作不止于此，像是绑定bus，绑定class，绑定driver，通知用户层等工作都是在该函数中完成的：

```cpp
int device_add(struct device *dev)
{
	struct subsys_private *sp;
	struct device *parent;
	struct kobject *kobj;
	struct class_interface *class_intf;
	int error = -EINVAL;
	struct kobject *glue_dir = NULL;

    // get_kobject的包装函数，增加引用计数
	dev = get_device(dev);
	if (!dev)
		goto done;

    // 分配device私有内存
	if (!dev->p) {
		error = device_private_init(dev);
		if (error)
			goto done;
	}

	/*
	 * for statically allocated devices, which should all be converted
	 * some day, we need to initialize the name. We prevent reading back
	 * the name, and force the use of dev_name()
	 */
	if (dev->init_name) {
		error = dev_set_name(dev, "%s", dev->init_name);
		dev->init_name = NULL;
	}

	if (dev_name(dev))
		error = 0;
	/* subsystems can specify simple device enumeration */
	else if (dev->bus && dev->bus->dev_name)
		error = dev_set_name(dev, "%s%u", dev->bus->dev_name, dev->id);
	else
		error = -EINVAL;
	if (error)
		goto name_error;

	pr_debug("device: '%s': %s\n", dev_name(dev), __func__);

    // 获取device的父节点，同时增加其引用计数
    // 拥有层次结构的device，会在调用device_register前设置就parent属性，使得注册的子设备可以挂载到父设备的目录下
    // 如果传入的device的父节点为空，即dev->parent未设置，那么就选择class或者bus上的某个根节点作为父节点
    // 一般来说，device会注册到sysfs的/sys/devices/xxxx/目录下面，比如/sys/devices/platform,/sys/devices/virtual等
	parent = get_device(dev->parent);
	kobj = get_device_parent(dev, parent);
	if (IS_ERR(kobj)) {
		error = PTR_ERR(kobj);
		goto parent_error;
	}
	if (kobj)
		dev->kobj.parent = kobj;

	/* use parent numa_node */
	if (parent && (dev_to_node(dev) == NUMA_NO_NODE))
		set_dev_node(dev, dev_to_node(parent));

    // 成功获取device的父亲节点后，创建device的文件节点，链接到父节点的目录下
	/* first, register with generic layer. */
	/* we require the name to be set before, and pass NULL */
	error = kobject_add(&dev->kobj, dev->kobj.parent, NULL);
	if (error) {
		glue_dir = kobj;
		goto Error;
	}

	/* notify platform of device entry */
	device_platform_notify(dev);

    // 在device对应目录下创建uevent文件
	error = device_create_file(dev, &dev_attr_uevent);
	if (error)
		goto attrError;

    // 在device对应目录下，创建of_node、susbsystem等节点
    // of_node对应dts，susbsystem对应该device所属的bus或者class
	error = device_add_class_symlinks(dev);
	if (error)
		goto SymlinkError;
    // 在device对应目录下，根据device、class、bus中的属性，创建sysfs节点
	error = device_add_attrs(dev);
	if (error)
		goto AttrsError;
    // 将device添加到对应的bus的sysfs节点上
	error = bus_add_device(dev);
	if (error)
		goto BusError;
    // PM相关操作，先跳过
	error = dpm_sysfs_add(dev);
	if (error)
		goto DPMError;
	device_pm_add(dev);

	if (MAJOR(dev->devt)) {
		error = device_create_file(dev, &dev_attr_dev);
		if (error)
			goto DevAttrError;

		error = device_create_sys_dev_entry(dev);
		if (error)
			goto SysEntryError;

		devtmpfs_create_node(dev);
	}

	/* Notify clients of device addition.  This call must come
	 * after dpm_sysfs_add() and before kobject_uevent().
	 */
    // 通知bus，调用notifier上注册的回调函数老响应BUS_NOTIFY_ADD_DEVICE事件
	if (dev->bus)
		blocking_notifier_call_chain(&dev->bus->p->bus_notifier,
					     BUS_NOTIFY_ADD_DEVICE, dev);
    // 通过netlink发送信息到用户态，通知device已经add完毕
	kobject_uevent(&dev->kobj, KOBJ_ADD);

	/*
	 * Check if any of the other devices (consumers) have been waiting for
	 * this device (supplier) to be added so that they can create a device
	 * link to it.
	 *
	 * This needs to happen after device_pm_add() because device_link_add()
	 * requires the supplier be registered before it's called.
	 *
	 * But this also needs to happen before bus_probe_device() to make sure
	 * waiting consumers can link to it before the driver is bound to the
	 * device and the driver sync_state callback is called for this device.
	 */
	if (dev->fwnode && !dev->fwnode->dev) {
		dev->fwnode->dev = dev;
		fw_devlink_link_device(dev);
	}

    // 开始匹配driver与probe，如果匹配成功，那么driver的probe函数就会被调用
	bus_probe_device(dev);

	/*
	 * If all driver registration is done and a newly added device doesn't
	 * match with any driver, don't block its consumers from probing in
	 * case the consumer device is able to operate without this supplier.
	 */
	if (dev->fwnode && fw_devlink_drv_reg_done && !dev->can_match)
		fw_devlink_unblock_consumers(dev);

	if (parent)
		klist_add_tail(&dev->p->knode_parent,
			       &parent->p->klist_children);

    // 如果赋予了device以class，则绑定device与class
	sp = class_to_subsys(dev->class);
	if (sp) {
		mutex_lock(&sp->mutex);
		/* tie the class to the device */
        // 将device添加到class的链表上记录起来
		klist_add_tail(&dev->p->knode_class, &sp->klist_devices);

		/* notify any interfaces that the device is here */
        // 调用class中的add_dev自定义函数
		list_for_each_entry(class_intf, &sp->interfaces, node)
			if (class_intf->add_dev)
				class_intf->add_dev(dev);
		mutex_unlock(&sp->mutex);
		subsys_put(sp);
	}
done:
	put_device(dev);
	return error;
}
```

&emsp;&emsp;通过`device_add`函数，将device结构体与bus，class这样的基础结构串联在一起，并且调用了`bus_probe_device`，该函数的主要功能就是遍历bus上的驱动，试图去匹配device与driver，进而调用driver的probe函数，完成设备与驱动的绑定。也就是说，每次调用`device_register`或者`device_add`，都会触发match操作，进而有可能触发driver的probe操作，这是需要去注意的点。关于设备驱动是如何绑定的，后续会在bus模块中说明。

&emsp;&emsp;device_del与device_add的行为是相反的，调用该函数时会从bus以及driver上移除device，并调用bus与driver上的remove函数，完成反初始化操作。

## 2. driver

&emsp;&emsp;device与driver的关系可以大致当作是数据与接口的关系。

```cpp
struct device_driver {
	const char		*name;
	struct bus_type		*bus;

	struct module		*owner;
	const char		*mod_name;	/* used for built-in modules */

	bool suppress_bind_attrs;	/* disables bind/unbind via sysfs */
	enum probe_type probe_type;

	const struct of_device_id	*of_match_table;
	const struct acpi_device_id	*acpi_match_table;

	int (*probe) (struct device *dev);
	void (*sync_state)(struct device *dev);
	int (*remove) (struct device *dev);
	void (*shutdown) (struct device *dev);
	int (*suspend) (struct device *dev, pm_message_t state);
	int (*resume) (struct device *dev);
	const struct attribute_group **groups;
	const struct attribute_group **dev_groups;

	const struct dev_pm_ops *pm;
	void (*coredump) (struct device *dev);

	struct driver_private *p;
};

```

&emsp;&emsp;driver的私有数据结构体如下：

```cpp
struct driver_private {
	struct kobject kobj;
	struct klist klist_devices;
	struct klist_node knode_bus;
	struct module_kobject *mkobj;
	struct device_driver *driver;
};
```

&emsp;&emsp;该结构体首先自带一个kobject，因此driver可以被显示在sysfs的某个目录下(/sys/bus/xxxx/drivers)。该私有结构记录了所有绑定的device同时，也作为节点被记录在对应的bus上，这样bus才能在添加device时，尝试匹配并绑定device和driver。

&emsp;&emsp;一般来说，主要关注driver以下三个属性：

+ probe：device和driver绑定后会调用该函数，一般用于初始化设备信息。
+ remove：与probe相反，device与driver解绑后调用。对于不支持系统运行时bind\unbind的驱动而言，就并不是很有必要去实现了。
+ of_match_table：用于匹配DTS节点中compatible节点的属性，会被bus的match流程所使用。

&emsp;&emsp;suspend, shutdown, resume主要用于runtime power manager，但是新的power设计是建议将runtime power manager与device power manager等电源管理都整合到了pm结构体中，所以建议不要使用上述三个接口，统一配置pm指针即可。pm也提供的方便的宏用于设置接口：

```cpp
// 用于设置rpm接口
SET_RUNTIME_PM_OPS(suspend_fn, resume_fn, idle_fn)
// 用于设置dpm接口
SET_SYSTEM_SLEEP_PM_OPS(suspend_fn, resume_fn) 
SET_LATE_SYSTEM_SLEEP_PM_OPS(suspend_fn, resume_fn)
SET_NOIRQ_SYSTEM_SLEEP_PM_OPS(suspend_fn, resume_fn)
```

&emsp;&emsp;driver的主要函数有两个，主要关注前者，即register操作，unregister操作基本是register操作的相反操作：

```cpp
int __must_check driver_register(struct device_driver *drv);
void driver_unregister(struct device_driver *drv);
```

&emsp;&emsp;代码如下：

```cpp
int driver_register(struct device_driver *drv)
{
	int ret;
	struct device_driver *other;

    // driver所属的bus必须先于driver初始化完成
	if (!drv->bus->p) {
		pr_err("Driver '%s' was unable to register with bus_type '%s' because the bus was not initialized.\n",
			   drv->name, drv->bus->name);
		return -EINVAL;
	}

    // 判断driver的操作是否合法
	if ((drv->bus->probe && drv->probe) ||
	    (drv->bus->remove && drv->remove) ||
	    (drv->bus->shutdown && drv->shutdown))
		pr_warn("Driver '%s' needs updating - please use "
			"bus_type methods\n", drv->name);

    // 判断是否重复注册driver
	other = driver_find(drv->name, drv->bus);
	if (other) {
		pr_err("Error: Driver '%s' is already registered, "
			"aborting...\n", drv->name);
		return -EBUSY;
	}

    // 核心操作，往bus添加新的driver
	ret = bus_add_driver(drv);
	if (ret)
		return ret;
    // 在sysfs添加driver对应目录，实际上是添加的目录是driver_private中内嵌的kobject所对应的节点属性
	ret = driver_add_groups(drv, drv->groups);
	if (ret) {
		bus_remove_driver(drv);
		return ret;
	}
    // 通过uevent通知用户层相关信息
	kobject_uevent(&drv->p->kobj, KOBJ_ADD);
	deferred_probe_extend_timeout();

	return ret;
}

int bus_add_driver(struct device_driver *drv)
{
	struct bus_type *bus;
	struct driver_private *priv;
	int error = 0;

    // 添加bus的引用计数
	bus = bus_get(drv->bus);
	if (!bus)
		return -EINVAL;

	pr_debug("bus: '%s': add driver %s\n", bus->name, drv->name);

    // 创建driver_private
	priv = kzalloc(sizeof(*priv), GFP_KERNEL);
	if (!priv) {
		error = -ENOMEM;
		goto out_put_bus;
	}
    // 初始化kobject
	klist_init(&priv->klist_devices, NULL, NULL);
	priv->driver = drv;
	drv->p = priv;
	priv->kobj.kset = bus->p->drivers_kset;
	error = kobject_init_and_add(&priv->kobj, &driver_ktype, NULL,
				     "%s", drv->name);
	if (error)
		goto out_unregister;

 	// 将driver记录到bus的链表上，用于快速查找bus上所挂载的driver
	klist_add_tail(&priv->knode_bus, &bus->p->klist_drivers);
    
    // !!!!!!!
    // 尝试绑定driver与device
	if (drv->bus->p->drivers_autoprobe) {
		error = driver_attach(drv);
		if (error)
			goto out_del_list;
	}
	module_add_driver(drv->owner, drv);

    // 创建driver的文件节点，位于/sys/bus/xxxx/drivers
	error = driver_create_file(drv, &driver_attr_uevent);
	if (error) {
		printk(KERN_ERR "%s: uevent attr (%s) failed\n",
			__func__, drv->name);
	}
	error = driver_add_groups(drv, bus->drv_groups);
	if (error) {
		/* How the hell do we get out of this pickle? Give up */
		printk(KERN_ERR "%s: driver_add_groups(%s) failed\n",
			__func__, drv->name);
	}

	if (!drv->suppress_bind_attrs) {
        // 创建bind/unbind属性
		error = add_bind_files(drv);
		if (error) {
			/* Ditto */
			printk(KERN_ERR "%s: add_bind_files(%s) failed\n",
				__func__, drv->name);
		}
	}

	return 0;

out_del_list:
	klist_del(&priv->knode_bus);
out_unregister:
	kobject_put(&priv->kobj);
	/* drv->p is freed in driver_release()  */
	drv->p = NULL;
out_put_bus:
	bus_put(bus);
	return error;
}
```

&emsp;&emsp;driver的核心操作基本如此。到现在我们可以发现，当`device_register`或者`driver_register`调用时，其挂载的bus都会去尝试绑定driver和device，并进一步的去调用probe操作。这个概念是整个设备驱动框架的核心。

&emsp;&emsp;driver的目录下面，通常都有bind和unbind属性，这是通过调用`add_bind_files`实现的，其对应的节点操作为：

```cpp
/* Manually detach a device from its associated driver. */
static ssize_t unbind_store(struct device_driver *drv, const char *buf,
			    size_t count)
{
	struct bus_type *bus = bus_get(drv->bus);
	struct device *dev;
	int err = -ENODEV;

	dev = bus_find_device_by_name(bus, NULL, buf);
	if (dev && dev->driver == drv) {
		device_driver_detach(dev);
		err = count;
	}
	put_device(dev);
	bus_put(bus);
	return err;
}
static DRIVER_ATTR_IGNORE_LOCKDEP(unbind, S_IWUSR, NULL, unbind_store);

static ssize_t bind_store(struct device_driver *drv, const char *buf,
			  size_t count)
{
	struct bus_type *bus = bus_get(drv->bus);
	struct device *dev;
	int err = -ENODEV;

	dev = bus_find_device_by_name(bus, NULL, buf);
	if (dev && dev->driver == NULL && driver_match_device(drv, dev)) {
		err = device_driver_attach(drv, dev);

		if (err > 0) {
			/* success */
			err = count;
		} else if (err == 0) {
			/* driver didn't accept device */
			err = -ENODEV;
		}
	}
	put_device(dev);
	bus_put(bus);
	return err;
}
static DRIVER_ATTR_IGNORE_LOCKDEP(bind, S_IWUSR, NULL, bind_store);

```

&emsp;&emsp;可以看见当对bind进行写操作时，会调用到`device_driver_attach()`，进而让bus尝试去绑定device和driver，对unbind的写操作是类似的。通过写driver的bind与unbind属性，可以实现在用户态手动控制设备驱动的bind/unbind行为，为此需要将driver结构体中的suppress_bind_attrs设置为false(默认就是false)。如果不想提供手动bind/unbind操作，则应该将suppress_bind_attrs设为true。

