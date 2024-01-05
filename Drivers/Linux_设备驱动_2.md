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

