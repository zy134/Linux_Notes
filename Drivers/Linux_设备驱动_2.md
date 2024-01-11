## 设备驱动核心：bus

## 1.bus基本结构

&emsp;&emsp;bus既可以表示物理总线(i2c bus, spi bus)，也可以表示虚拟总线(platform bus)。在设备驱动框架中，bus负责管理挂载在其上的device与driver。device与driver在初始化时，通过调用`bus_add_driver`和`bus_add_device`将自身注册到bus上，然后进行对应的match与probe操作。典型的bus流程就是：

```mermaid
graph LR

device_add-->driver_add-->match-->probe
```

&emsp;&emsp;bus的核心结构是`bus_type`。如其他设备驱动框架中的结构一样，bus也需要通过内嵌的kobject来联系其他组件。这个内嵌的kobject是存在于`subsys_private`成员中的。除此外可以看见bus_type主要成员就是各类函数指针：

+ match: 用于判断device与driver是否匹配，如果match返回1则是匹配，进而可以绑定device与driver
+ uevent: 用于过滤uevent
+ probe: 在device与driver绑定后调用，一般而言会在做一些与bus相关的工作后继续调用到driver中的probe成员函数。像是platform bus这样的虚拟总线，就只会去继续调用driver的probe。
+ remove: 与probe相对应，在device与driver解绑后调用，进而调用driver中的remove成员函数。

&emsp;&emsp;suspend和resume用于PM模块，如前所述，使用新的dev_pm_ops结构是更优的选择。

```cpp
struct bus_type {
	const char		*name;
	const char		*dev_name;
	struct device		*dev_root;
	const struct attribute_group **bus_groups;
	const struct attribute_group **dev_groups;
	const struct attribute_group **drv_groups;

	int (*match)(struct device *dev, struct device_driver *drv);
	int (*uevent)(struct device *dev, struct kobj_uevent_env *env);
	int (*probe)(struct device *dev);
	void (*sync_state)(struct device *dev);
	int (*remove)(struct device *dev);
	void (*shutdown)(struct device *dev);

	int (*online)(struct device *dev);
	int (*offline)(struct device *dev);

	int (*suspend)(struct device *dev, pm_message_t state);
	int (*resume)(struct device *dev);

	int (*num_vf)(struct device *dev);

	int (*dma_configure)(struct device *dev);

	const struct dev_pm_ops *pm;

	const struct iommu_ops *iommu_ops;

	struct subsys_private *p;
	struct lock_class_key lock_key;

	bool need_parent_lock;

};

```

&emsp;&emsp;subsys_private既是bus在sysfs中暴露的接口，也用于协助管理挂载在bus上device与driver。其中有三个kset成员。subsys表示bus在sysfs中的目录，即`/sys/bus/platform`，`/sys/bus/i2c`这样的目录。devices_kset与drivers_kset表示bus目录下的devices与drivers两个子目录，不难想象，当调用bus_add_driver或者bus_add_device时，对应device与driver的kobject节点也会被挂载或者链接到这两个目录下。两个链表klist_devices，klist_drivers，用于快速索引挂载到bus上的device与driver。drivers_autoprobe用于指明是否在绑定成功后自动调用probe函数。

```cpp
struct subsys_private {
	struct kset subsys;
	struct kset *devices_kset;
	struct list_head interfaces;
	struct mutex mutex;

	struct kset *drivers_kset;
	struct klist klist_devices;
	struct klist klist_drivers;
	struct blocking_notifier_head bus_notifier;
	unsigned int drivers_autoprobe:1;
	struct bus_type *bus;

	struct kset glue_dirs;
	struct class *class;
};
```

## 2.bus基本操作

&emsp;&emsp;bus的基本操作，包括bus的注册与销毁，device与driver的注册与销毁，以及控制bus上各个设备驱动的probe流程，比较核心的操作有以下：

```cpp
// bus的注册与销毁
int bus_register(struct bus_type *bus);
void bus_unregister(struct bus_type *bus);
// device的注册与销毁
int bus_add_device(struct device *dev);
void bus_remove_device(struct device *dev);
// driver的注册与销毁
int bus_add_driver(struct device_driver *drv);
void bus_remove_driver(struct device_driver *drv);

// 用于bus core和其他模块进行通信
int bus_register_notifier(struct bus_type *bus,
				 struct notifier_block *nb);
int bus_unregister_notifier(struct bus_type *bus,
				   struct notifier_block *nb);

// 以下都是设备驱动绑定流程相关的核心函数，后续说明
static inline int driver_match_device(struct device_driver *drv,
				      struct device *dev)
{
	return drv->bus->match ? drv->bus->match(dev, drv) : 1;
}
void bus_probe_device(struct device *dev);
int driver_probe_device(struct device_driver *drv, struct device *dev);
int device_driver_attach(struct device_driver *drv, struct device *dev);
void device_driver_detach(struct device *dev);
void device_block_probing(void);
void device_unblock_probing(void);

```

&emsp;&emsp;bus的根目录是/sys/bus，在系统启动时初始化，其实就是创建一个名为bus的kset。之后所有注册的bus，都会挂载到/sys/bus目录下面：

```cpp
int __init buses_init(void)
{
    // 创建根目录/sys/bus
	bus_kset = kset_create_and_add("bus", &bus_uevent_ops, NULL);
	if (!bus_kset)
		return -ENOMEM;

    // 为兼容性而保留的/sys/devices/system目录，不要在这个目录下面添加新的device节点
	system_kset = kset_create_and_add("system", NULL, &devices_kset->kobj);
	if (!system_kset)
		return -ENOMEM;

	return 0;
}
```

&emsp;&emsp;bus_register用于注册bus，主要都是sysfs相关的操作，以platform bus为例，调用bus_register时会创建/sys/bus/platform, /sys/bus/platform/devices, /sys/bus/platform/drivers等目录。后续向bus添加device时，会将device的kobject链接到/sys/bus/platform/devices目录下；如果是添加driver，那么driver对应的kobject会直接创建在/sys/bus/platform/drivers目录下面，因为driver是必须挂载在bus上面的，不然调用driver_register会直接报错退出；而device并非必须挂载在bus上。

```cpp
int bus_register(struct bus_type *bus)
{
	int retval;
	struct subsys_private *priv;
	struct lock_class_key *key = &bus->lock_key;

	priv = kzalloc(sizeof(struct subsys_private), GFP_KERNEL);
	if (!priv)
		return -ENOMEM;

	priv->bus = bus;
	bus->p = priv;

	BLOCKING_INIT_NOTIFIER_HEAD(&priv->bus_notifier);

	retval = kobject_set_name(&priv->subsys.kobj, "%s", bus->name);
	if (retval)
		goto out;

	priv->subsys.kobj.kset = bus_kset; // 该kset就系统启动时创建的/sys/bus目录
	priv->subsys.kobj.ktype = &bus_ktype;
	priv->drivers_autoprobe = 1; // 默认autoprobe，即driver和device匹配成功时自动调用probe

    // 在/sys/bus下面创建新的目录
	retval = kset_register(&priv->subsys);
	if (retval)
		goto out;

	retval = bus_create_file(bus, &bus_attr_uevent);
	if (retval)
		goto bus_uevent_fail;

    // 创建devices子目录，后续注册到bus的device会链接到这个目录
	priv->devices_kset = kset_create_and_add("devices", NULL,
						 &priv->subsys.kobj);
	if (!priv->devices_kset) {
		retval = -ENOMEM;
		goto bus_devices_fail;
	}

    // 创建driver的子目录，后续注册到bus的driver直接在这个目录下面创建新的目录项
	priv->drivers_kset = kset_create_and_add("drivers", NULL,
						 &priv->subsys.kobj);
	if (!priv->drivers_kset) {
		retval = -ENOMEM;
		goto bus_drivers_fail;
	}

	INIT_LIST_HEAD(&priv->interfaces);
	__mutex_init(&priv->mutex, "subsys mutex", key);
	klist_init(&priv->klist_devices, klist_devices_get, klist_devices_put);
	klist_init(&priv->klist_drivers, NULL, NULL);

	retval = add_probe_files(bus);
	if (retval)
		goto bus_probe_files_fail;

	retval = bus_add_groups(bus, bus->bus_groups);
	if (retval)
		goto bus_groups_fail;

	pr_debug("bus: '%s': registered\n", bus->name);
	return 0;

......
}

```

&emsp;&emsp;bus_unregister与bus_register操作相反，不再说明。bus_register_notifier这个函数是notifier的一个封装，用于子系统向设备驱动注册回调函数，当事件发生时，调用回调来处理事件，事件有以下几个：

```cpp
/* All 4 notifers below get called with the target struct device *
 * as an argument. Note that those functions are likely to be called
 * with the device lock held in the core, so be careful.
 */
#define BUS_NOTIFY_ADD_DEVICE		0x00000001 /* device added */
#define BUS_NOTIFY_DEL_DEVICE		0x00000002 /* device to be removed */
#define BUS_NOTIFY_REMOVED_DEVICE	0x00000003 /* device removed */
#define BUS_NOTIFY_BIND_DRIVER		0x00000004 /* driver about to be
						      bound */
#define BUS_NOTIFY_BOUND_DRIVER		0x00000005 /* driver bound to device */
#define BUS_NOTIFY_UNBIND_DRIVER	0x00000006 /* driver about to be
						      unbound */
#define BUS_NOTIFY_UNBOUND_DRIVER	0x00000007 /* driver is unbound
						      from the device */
#define BUS_NOTIFY_DRIVER_NOT_BOUND	0x00000008 /* driver fails to be bound */
```

&emsp;&emsp;以i2c dev为例，在初始化时，传入相关回调结构体，这样在device/driver状态变化时，可以对特定事件进行处理：

```cpp
// 处理device的add/delete事件
static int i2cdev_notifier_call(struct notifier_block *nb, unsigned long action,
			 void *data)
{
	struct device *dev = data;

	switch (action) {
	case BUS_NOTIFY_ADD_DEVICE:
		return i2cdev_attach_adapter(dev, NULL);
	case BUS_NOTIFY_DEL_DEVICE:
		return i2cdev_detach_adapter(dev, NULL);
	}

	return 0;
}

static struct notifier_block i2cdev_notifier = {
	.notifier_call = i2cdev_notifier_call,
};

static int __init i2c_dev_init(void)
{
	......
    // 注册回调
	res = bus_register_notifier(&i2c_bus_type, &i2cdev_notifier);
}
```

&emsp;&emsp;bus_add_device函数将device目录与bus目录相链接，并将device结构添加到bus的klist_devices链表上，这样bus可以快速的索引到挂载的device。此外每个bus都配有dev_groups，作为挂载在同一个bus上所有device的共有属性，在调用bus_add_device时dev_groups会被添加到device对应的目录下。

```cpp
int bus_add_device(struct device *dev)
{
	struct bus_type *bus = bus_get(dev->bus);
	int error = 0;

	if (bus) {
		pr_debug("bus: '%s': add device %s\n", bus->name, dev_name(dev));
        // 创建device的公共属性
		error = device_add_groups(dev, bus->dev_groups);
		if (error)
			goto out_put;
        // 将device目录与bus目录链接起来
		error = sysfs_create_link(&bus->p->devices_kset->kobj,
						&dev->kobj, dev_name(dev));
		if (error)
			goto out_groups;
		error = sysfs_create_link(&dev->kobj,
				&dev->bus->p->subsys.kobj, "subsystem");
		if (error)
			goto out_subsys;
        // 将device添加到bus的私有链表上，用于快速索引device
		klist_add_tail(&dev->p->knode_bus, &bus->p->klist_devices);
	}
	return 0;
}
```

&emsp;&emsp;bus_add_driver函数负责了driver内嵌kobject的初始化，driver对应的目录会创建在对应bus的drivers子目录下面(/sys/bus/xxx/drivers)。此外当drivers_autoprobe为1时，bus会尝试去匹配driver与device，继而调用probe函数，如前所述，drivers_autoprobe默认是1，所以一般添加driver时都会去进行match进而probe。

```cpp
int bus_add_driver(struct device_driver *drv)
{
	struct bus_type *bus;
	struct driver_private *priv;
	int error = 0;

	bus = bus_get(drv->bus);
	if (!bus)
		return -EINVAL;

	pr_debug("bus: '%s': add driver %s\n", bus->name, drv->name);

    // 创建driver的内嵌kobject节点
	priv = kzalloc(sizeof(*priv), GFP_KERNEL);
	if (!priv) {
		error = -ENOMEM;
		goto out_put_bus;
	}
	klist_init(&priv->klist_devices, NULL, NULL);
	priv->driver = drv;
	drv->p = priv;
    // 新创建的driver目录挂载在bus的drivers子目录下
	priv->kobj.kset = bus->p->drivers_kset;
	error = kobject_init_and_add(&priv->kobj, &driver_ktype, NULL,
				     "%s", drv->name);
	if (error)
		goto out_unregister;

    // 将driver结构添加到bus的klist_drivers链表上，方便bus索引driver结构
	klist_add_tail(&priv->knode_bus, &bus->p->klist_drivers);
    // 尝试绑定driver与device
	if (drv->bus->p->drivers_autoprobe) {
		error = driver_attach(drv);
		if (error)
			goto out_del_list;
	}
	module_add_driver(drv->owner, drv);

    // 为driver创建uevent属性
	error = driver_create_file(drv, &driver_attr_uevent);
	if (error) {
		printk(KERN_ERR "%s: uevent attr (%s) failed\n",
			__func__, drv->name);
	}
    // 创建driver的公共属性
	error = driver_add_groups(drv, bus->drv_groups);
	if (error) {
		/* How the hell do we get out of this pickle? Give up */
		printk(KERN_ERR "%s: driver_create_groups(%s) failed\n",
			__func__, drv->name);
	}

    // 为driver创建bind/unbind属性，用于手动bind/unbind device与driver
	if (!drv->suppress_bind_attrs) {
		error = add_bind_files(drv);
		if (error) {
			/* Ditto */
			printk(KERN_ERR "%s: add_bind_files(%s) failed\n",
				__func__, drv->name);
		}
	}

	return 0;
}

```

### 3.匹配与绑定device和driver

&emsp;&emsp;匹配与绑定是bus在设备驱动框架中所负责的主要工作，先说明两个函数：

```cpp
int driver_attach(struct device_driver *drv);
void bus_probe_device(struct device *dev);
```

&emsp;&emsp;这两个函数都在driver/device注册时被自动调用，进而去匹配和绑定操作，先看`driver_attach`函数。该函数被`bus_add_driver`所调用，其对于每一个注册到bus上的device调用`__driver_attach`，尝试去绑定device和driver

```cpp
int driver_attach(struct device_driver *drv)
{
	return bus_for_each_dev(drv->bus, NULL, drv, __driver_attach);
}
```

&emsp;&emsp;`__driver_attach`针对传入的device和driver，先尝试调用`driver_match_device`去匹配device和driver，匹配成功的情况下，调用`device_driver_attach`去进行绑定操作。匹配操作实际上是依赖于具体bus的，调用`bus_type`中的`match`方法。当match大于0时，匹配成功。

&emsp;&emsp;匹配失败存在两种情况，当返回**EPROBE_DEFER**时，说明虽然现在driver和device不能完成匹配，但是在未来某个节点device也许可以找到和他匹配的driver，所以将它添加到一个名为**deferred_probe_pending_list**的链表中，等待后续时刻重新开始probe。另一种情况就是匹配失败，直接返回。

```cpp
static int __driver_attach(struct device *dev, void *data)
{
	struct device_driver *drv = data;
	bool async = false;
	int ret;

	/*
	 * Lock device and try to bind to it. We drop the error
	 * here and always return 0, because we need to keep trying
	 * to bind to devices and some drivers will return an error
	 * simply if it didn't support the device.
	 *
	 * driver_probe_device() will spit a warning if there
	 * is an error.
	 */

    // driver_match_device会去调用bus的match方法
	ret = driver_match_device(drv, dev);
	if (ret == 0) {
		/* no match */
		return 0;
	} else if (ret == -EPROBE_DEFER) {
		dev_dbg(dev, "Device match requests probe deferral\n");
		driver_deferred_probe_add(dev);
		/*
		 * Driver could not match with device, but may match with
		 * another device on the bus.
		 */
		return 0;
	} else if (ret < 0) {
		dev_dbg(dev, "Bus failed to match device: %d\n", ret);
		return ret;
	} /* ret > 0 means positive match */

	if (driver_allows_async_probing(drv)) {
		/*
		 * Instead of probing the device synchronously we will
		 * probe it asynchronously to allow for more parallelism.
		 *
		 * We only take the device lock here in order to guarantee
		 * that the dev->driver and async_driver fields are protected
		 */
		dev_dbg(dev, "probing driver %s asynchronously\n", drv->name);
		device_lock(dev);
		if (!dev->driver) {
			get_device(dev);
			dev->p->async_driver = drv;
			async = true;
		}
		device_unlock(dev);
		if (async)
			async_schedule_dev(__driver_attach_async_helper, dev);
		return 0;
	}

	device_driver_attach(drv, dev);

	return 0;
}

```

&emsp;&emsp;匹配成功后，既可以直接调用`device_driver_attach`去同步的完成绑定device和driver的任务，也可以通过`async_schedule_dev`起一个异步任务去完成attach和probe，是否异步执行取决于驱动配置，当probe流程速度很慢会拖垮系统启动时，就应该配置成异步运行，类似mmc这样初始化较慢且与其他模块不存在过多依赖的外设，通常都使用异步的probe。通过将driver结构中的`probe_type`设置成**PROBE_PREFER_ASYNCHRONOUS**，就可以使驱动异步执行attach和probe。 

&emsp;&emsp;对于一些模块而言，需要等待设备初始化完毕，或者希望设备状态保持原子化，那么就可以调用`wait_for_device_probe`函数，等待所有异步任务与defered任务的完成。

```cpp

// 同步调用driver_probe_driver
int device_driver_attach(struct device_driver *drv, struct device *dev)
{
	int ret = 0;

	__device_driver_lock(dev, dev->parent);

	/*
	 * If device has been removed or someone has already successfully
	 * bound a driver before us just skip the driver probe call.
	 */
	if (!dev->p->dead && !dev->driver)
		ret = driver_probe_device(drv, dev);

	__device_driver_unlock(dev, dev->parent);

	return ret;
}

// 异步调用driver_probe_driver
static void __driver_attach_async_helper(void *_dev, async_cookie_t cookie)
{
	struct device *dev = _dev;
	struct device_driver *drv;
	int ret = 0;

	__device_driver_lock(dev, dev->parent);

	drv = dev->p->async_driver;

	/*
	 * If device has been removed or someone has already successfully
	 * bound a driver before us just skip the driver probe call.
	 */
	if (!dev->p->dead && !dev->driver)
		ret = driver_probe_device(drv, dev);

	__device_driver_unlock(dev, dev->parent);

	dev_dbg(dev, "driver %s async attach completed: %d\n", drv->name, ret);

	put_device(dev);
}

```

&emsp;&emsp;上述的asynchronous任务与defered任务是两个不同阶段的概念，首先defered任务指的是device和driver**匹配失败**，这也许是该device的依赖模块未完成初始化，因此并不简单的返回错误，而是将device添加到**deferred_probe_pending_list**中，等待后续某个时刻重新开始匹配device与driver。asynchrounous任务是在**匹配完成以后**，可以根据驱动配置同步的或者异步的去调用probe流程。

&emsp;&emsp;不管probe是同步进行的还是异步进行的，都会去调用`driver_probe_device`函数，该函数完成绑定设备驱动的任务，此外注意该函数不持有锁，必须在加锁后调用：

```cpp
int driver_probe_device(struct device_driver *drv, struct device *dev)
{
	int ret = 0;

	if (!device_is_registered(dev))
		return -ENODEV;

	pr_debug("bus: '%s': %s: matched device %s with driver %s\n",
		 drv->bus->name, __func__, dev_name(dev), drv->name);

    // 增加suppliers的pm计数
	pm_runtime_get_suppliers(dev);
    // 增加父节点的pm计数
	if (dev->parent)
		pm_runtime_get_sync(dev->parent);
    // 当pm计数为0时，对应驱动的resume函数会被调用，给父设备与suppliers上电
    // 不过在这里主要是为了防止probe过程中父设备或者suppliers的suspend被调用

	pm_runtime_barrier(dev);
    // really_probe，完成probe工作的函数体
	if (initcall_debug)
		ret = really_probe_debug(dev, drv);
	else
		ret = really_probe(dev, drv);
	pm_request_idle(dev);

	if (dev->parent)
		pm_runtime_put(dev->parent);

	pm_runtime_put_suppliers(dev);
	return ret;
}
```

&emsp;&emsp;really_probe比较长，这里主要注意几点，一是probe例程调用成功后，device和driver完成了绑定，device目录下面会新增有一个指向driver目录的链接，driver目录同理，会链接到绑定的device，这样可以通过sysfs快速检查probe是否成功。probe同样可以返回EPROBE_DEFER，来触发延迟初始化的流程。

&emsp;&emsp;其二是，really_probe函数并不是直接调用driver的probe，而是先调用bus的probe。像platform bus这样的虚拟bus，其probe实现一般只是driver probe的封装。类似i2c这样物理bus，其probe例程则需要做额外的工作，再去调用driver的probe。

```cpp
static int really_probe(struct device *dev, struct device_driver *drv)
{
	int ret = -EPROBE_DEFER;
	int local_trigger_count = atomic_read(&deferred_trigger_count);
	bool test_remove = IS_ENABLED(CONFIG_DEBUG_TEST_DRIVER_REMOVE) &&
			   !drv->suppress_bind_attrs;

    // 通过调用device_block_probing，推迟所有probe，用于实现pm的休眠唤醒机制
	if (defer_all_probes) {
		/*
		 * Value of defer_all_probes can be set only by
		 * device_block_probing() which, in turn, will call
		 * wait_for_device_probe() right after that to avoid any races.
		 */
		dev_dbg(dev, "Driver %s force probe deferral\n", drv->name);
		driver_deferred_probe_add(dev);
		return ret;
	}

	ret = device_links_check_suppliers(dev);
    // device的依赖设备未准备好，添加到defered链表中，后续再重新开始调用bus_probe_device例程
	if (ret == -EPROBE_DEFER)
		driver_deferred_probe_add_trigger(dev, local_trigger_count);
	if (ret)
		return ret;

	atomic_inc(&probe_count);
	pr_debug("bus: '%s': %s: probing driver %s with device %s\n",
		 drv->bus->name, __func__, drv->name, dev_name(dev));
	if (!list_empty(&dev->devres_head)) {
		dev_crit(dev, "Resources present before probing\n");
		ret = -EBUSY;
		goto done;
	}

re_probe:
    // 链接device和driver结构
	dev->driver = drv;

	/* If using pinctrl, bind pins now before probing */
	ret = pinctrl_bind_pins(dev);
	if (ret)
		goto pinctrl_bind_failed;

	if (dev->bus->dma_configure) {
		ret = dev->bus->dma_configure(dev);
		if (ret)
			goto probe_failed;
	}

    // 在driver的sysfs目录下面添加device的软链接
	ret = driver_sysfs_add(dev);
	if (ret) {
		pr_err("%s: driver_sysfs_add(%s) failed\n",
		       __func__, dev_name(dev));
		goto probe_failed;
	}

	if (dev->pm_domain && dev->pm_domain->activate) {
		ret = dev->pm_domain->activate(dev);
		if (ret)
			goto probe_failed;
	}

    // 调用probe，初始化device
	if (dev->bus->probe) {
		ret = dev->bus->probe(dev);
		if (ret)
			goto probe_failed;
	} else if (drv->probe) {
		ret = drv->probe(dev);
		if (ret)
			goto probe_failed;
	}

	ret = device_add_groups(dev, drv->dev_groups);
	if (ret) {
		dev_err(dev, "device_add_groups() failed\n");
		goto dev_groups_failed;
	}

	if (dev_has_sync_state(dev)) {
		ret = device_create_file(dev, &dev_attr_state_synced);
		if (ret) {
			dev_err(dev, "state_synced sysfs add failed\n");
			goto dev_sysfs_state_synced_failed;
		}
	}

	pinctrl_init_done(dev);

	if (dev->pm_domain && dev->pm_domain->sync)
		dev->pm_domain->sync(dev);

    // 绑定完成
	driver_bound(dev);
	ret = 1;
	pr_debug("bus: '%s': %s: bound device %s to driver %s\n",
		 drv->bus->name, __func__, dev_name(dev), drv->name);
	goto done;

dev_sysfs_state_synced_failed:
	device_remove_groups(dev, drv->dev_groups);
dev_groups_failed:
	if (dev->bus->remove)
		dev->bus->remove(dev);
	else if (drv->remove)
		drv->remove(dev);
probe_failed:
	if (dev->bus)
		blocking_notifier_call_chain(&dev->bus->p->bus_notifier,
					     BUS_NOTIFY_DRIVER_NOT_BOUND, dev);
pinctrl_bind_failed:
	device_links_no_driver(dev);
	devres_release_all(dev);
	arch_teardown_dma_ops(dev);
	kfree(dev->dma_range_map);
	dev->dma_range_map = NULL;
	driver_sysfs_remove(dev);
	dev->driver = NULL;
	dev_set_drvdata(dev, NULL);
	if (dev->pm_domain && dev->pm_domain->dismiss)
		dev->pm_domain->dismiss(dev);
	pm_runtime_reinit(dev);
	dev_pm_set_driver_flags(dev, 0);

	switch (ret) {
	case -EPROBE_DEFER:
		/* Driver requested deferred probing */
         // 一般来说是probe返回了EPROBE_DEFER，说明当前device有依赖未初始化完成
		dev_dbg(dev, "Driver %s requests probe deferral\n", drv->name);
		driver_deferred_probe_add_trigger(dev, local_trigger_count);
		break;
	case -ENODEV:
	case -ENXIO:
		pr_debug("%s: probe of %s rejects match %d\n",
			 drv->name, dev_name(dev), ret);
		break;
	default:
		/* driver matched but the probe failed */
		pr_warn("%s: probe of %s failed with error %d\n",
			drv->name, dev_name(dev), ret);
	}
	/*
	 * Ignore errors returned by ->probe so that the next driver can try
	 * its luck.
	 */
	ret = 0;
done:
	atomic_dec(&probe_count);
	wake_up_all(&probe_waitqueue);
	return ret;
}

```

&emsp;&emsp;在probe完成后device和driver完成绑定，并调用driver_bound完成后续工作：

```cpp
static void driver_bound(struct device *dev)
{
	if (device_is_bound(dev)) {
		pr_warn("%s: device %s already bound\n",
			__func__, kobject_name(&dev->kobj));
		return;
	}

	pr_debug("driver: '%s': %s: bound to device '%s'\n", dev->driver->name,
		 __func__, dev_name(dev));

	klist_add_tail(&dev->p->knode_driver, &dev->driver->p->klist_devices);
    
    // 更新devlink
	device_links_driver_bound(dev);

	device_pm_check_callbacks(dev);

	/*
	 * Make sure the device is no longer in one of the deferred lists and
	 * kick off retrying all pending devices
	 */
	driver_deferred_probe_del(dev);
	driver_deferred_probe_trigger();

	if (dev->bus)
		blocking_notifier_call_chain(&dev->bus->p->bus_notifier,
					     BUS_NOTIFY_BOUND_DRIVER, dev);

	kobject_uevent(&dev->kobj, KOBJ_BIND);
}
```

&emsp;&emsp;该函数会重新触发defer probe。也就是说，每当一个设备驱动完成绑定，就会去触发defer probe，触发新的probe流程。如果还是返回EPROBE_DEFER，那就把对应device加回defer队列，等候下一次被触发，直到defer list为空，所有的工作都已经完成。绑定成功后还会发送**KOBJ_BIND**事件到用户层，应用可以监听netlink来查看驱动状态。后续讲uevent时会再细讲。

&emsp;&emsp;再从device角度看待上述流程，因为基本是类似，只说明流程：

```cpp
void bus_probe_device(struct device *dev)
{
......
	if (bus->p->drivers_autoprobe)
		device_initial_probe(dev);
}

void device_initial_probe(struct device *dev)
{
	__device_attach(dev, true);
}

static int __device_attach(struct device *dev, bool allow_async)
{
// 同步调用，对于挂载在bus上的所有driver，尝试绑定device与driver
		ret = bus_for_each_drv(dev->bus, NULL, &data,
					__device_attach_driver);
......
// 异步调用
	if (async)
		async_schedule_dev(__device_attach_async_helper, dev);
	return ret;
}

static int __device_attach_driver(struct device_driver *drv, void *_data)
{
...
    // 匹配device与driver
	ret = driver_match_device(drv, dev);

    // 匹配成功后调用probe
	return driver_probe_device(drv, dev);
}


```



