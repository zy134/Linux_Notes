# 基本内嵌结构：kobject与kset
&emsp;&emsp;kobject作为Linux设备驱动框架的底层实现，本身并不具备特定的含义。要说的话他类似于Java中的Object类，是Linux设备驱动中所有结构体的"基类"。当然C语言本身没有继承这回事，所以Linux内核是通过结构体内嵌以及`container_of`宏来达成类似的效果。`kobject`在架构上作为基类，实现上则是以引用计数为核心，详细结构如下：
```cpp
struct kobject {
	const char		*name;
	struct list_head	entry;
	struct kobject		*parent;
	struct kset		*kset;
	const struct kobj_type	*ktype;
	struct kernfs_node	*sd; /* sysfs directory entry */
	struct kref		kref;
#ifdef CONFIG_DEBUG_KOBJECT_RELEASE
	struct delayed_work	release;
#endif
	unsigned int state_initialized:1;
	unsigned int state_in_sysfs:1;
	unsigned int state_add_uevent_sent:1;
	unsigned int state_remove_uevent_sent:1;
	unsigned int uevent_suppress:1;
};
```
&emsp;&emsp;从kobject的结构体就能了解到有关他的绝大部分内容，这个道理对于Linux驱动框架同样适用——Linux驱动模块的绝大部分功能都可以通过阅读头文件得到。kobjet首先有着自己的名字，这个名字也对应着`sysfs`中目录项的名字。`parent`与`kset`则指示了kobject间的联系，parent指示的是树状关系的kobject，kset则是集合关系的kobject(entry是kobject在kset中的节点)。`kref`是kobject的核心，用于管理引用计数。`ktype`则是指示kobject所拥有的一组操作，详见后文。`sd`为kobject在sysfs中的文件节点。

&emsp;&emsp;`ktype`具体结构如下。在Linux内核的许多架构中，都体现了这种**数据与操作分离**的特征，设备驱动框架尤为明显。ktype中的操作我们现在主要关注两个，一是`release`，当引用计数为0时，release会被调用。二是`sysfs_ops`，对于sysfs中文件的读写实现就是通过填充这个函数集指针实现。

```cpp
struct kobj_type {
	void (*release)(struct kobject *kobj);
	const struct sysfs_ops *sysfs_ops;
	const struct attribute_group **default_groups;
	const struct kobj_ns_type_operations *(*child_ns_type)(struct kobject *kobj);
	const void *(*namespace)(struct kobject *kobj);
	void (*get_ownership)(struct kobject *kobj, kuid_t *uid, kgid_t *gid);
};
```
&emsp;&emsp;关于kobject的核心操作主要是下面五个。前三个是kobject的创建/销毁，后面两个则是kobject的引用计数操作。
```cpp
extern void kobject_init(struct kobject *kobj, const struct kobj_type *ktype);
extern int kobject_add(struct kobject *kobj, struct kobject *parent,
		const char *fmt, ...);
extern void kobject_del(struct kobject *kobj);

// 增加引用计数
extern struct kobject *kobject_get(struct kobject *kobj);
// 减少引用计数
extern void kobject_put(struct kobject *kobj);
```
&emsp;&emsp;kobject_init会做一些检查，并初始化结构体和引用计数。从代码中也可以看到，初始化一个kobject时，ktype必须不为空，否则这个kobject就是不能操作的无效对象。

```cpp
void kobject_init(struct kobject *kobj, const struct kobj_type *ktype)
{
	char *err_str;

	if (!kobj) {
		err_str = "invalid kobject pointer!";
		goto error;
	}
	if (!ktype) {
		err_str = "must have a ktype to be initialized properly!\n";
		goto error;
	}
	if (kobj->state_initialized) {
		/* do not error out as sometimes we can recover */
		pr_err("kobject (%p): tried to init an initialized object, something is seriously wrong.\n",
		       kobj);
		dump_stack();
	}

	kobject_init_internal(kobj);
	kobj->ktype = ktype;
	return;

error:
	pr_err("kobject (%p): %s\n", kobj, err_str);
	dump_stack();
}

static void kobject_init_internal(struct kobject *kobj)
{
	if (!kobj)
		return;
	// 初始化引用计数为1
	kref_init(&kobj->kref);
	INIT_LIST_HEAD(&kobj->entry);
	// 还未添加到sysfs中
	kobj->state_in_sysfs = 0;
	kobj->state_add_uevent_sent = 0;
	kobj->state_remove_uevent_sent = 0;
	// 已经初始化完成
	kobj->state_initialized = 1;
}
```
  在初始化完成后需要调用`kobject_add`。该函数会连接kobject与parent、kset节点，由于parent与kset被kobject引用到了，所以他们的引用计数也要一并增加
  。然后在sysfs创建kobject中创建对应的节点。需要注意的是，如果没有设定kobject的父节点，那么会将kset作为该kobject的父节点。如果kset也没有设置，
  那kobject在文件系统中的父节点就是`/sys`这个根节点。另外，kobject_add不会修改kobject自身的引用计数。
```cpp
static int kobject_add_internal(struct kobject *kobj)
{
	int error = 0;
	struct kobject *parent;

	if (!kobj)
		return -ENOENT;

	if (!kobj->name || !kobj->name[0]) {
		WARN(1,
		     "kobject: (%p): attempted to be registered with empty name!\n",
		     kobj);
		return -EINVAL;
	}
    // 增加父节点的引用计数
	parent = kobject_get(kobj->parent);

	/* join kset if set, use it as parent if we do not already have one */
	if (kobj->kset) {
		if (!parent)
			parent = kobject_get(&kobj->kset->kobj);
		kobj_kset_join(kobj);
		kobj->parent = parent;
	}

	pr_debug("kobject: '%s' (%p): %s: parent: '%s', set: '%s'\n",
		 kobject_name(kobj), kobj, __func__,
		 parent ? kobject_name(parent) : "<NULL>",
		 kobj->kset ? kobject_name(&kobj->kset->kobj) : "<NULL>");
    // 在sysfs中创建对应的文件节点
	error = create_dir(kobj);
	if (error) {
		kobj_kset_leave(kobj);
		kobject_put(parent);
		kobj->parent = NULL;

		/* be noisy on error issues */
		if (error == -EEXIST)
			pr_err("%s failed for %s with -EEXIST, don't try to register things with the same name in the same directory.\n",
			       __func__, kobject_name(kobj));
		else
			pr_err("%s failed for %s (error: %d parent: %s)\n",
			       __func__, kobject_name(kobj), error,
			       parent ? kobject_name(parent) : "'none'");
	} else
		kobj->state_in_sysfs = 1;

	return error;
}
```
  kobject_del将kobject从parent、kset中移除，并减少parent、kset的引用计数。该函数与kobject_add一样，不会修改自身的引用计数，除开一开始的kobject_init
  外，要修改kobject的引用计数，需要显示的调用kobject_get与kobject_put。
```cpp
void kobject_del(struct kobject *kobj)
{
	struct kobject *parent;

	if (!kobj)
		return;

	parent = kobj->parent;
	__kobject_del(kobj);
	kobject_put(parent);
}
```
<br>   kset顾名思义，是一个集合，具体的说就是kobject的集合。kset本身也是一个kobject，所以它也是通过引用计数管理的。从kset的结构中可以看见，他维护
  着一个kobject的链表，以及一个uevent_ops。uevent的作用是，当从kset中添加/删除kobject时，将该信息通知到userspace。
```cpp
struct kset {
	struct list_head list;
	spinlock_t list_lock;
	struct kobject kobj;
	const struct kset_uevent_ops *uevent_ops;
} __randomize_layout;
```
  kset的主要操作`kset_register`，`kset_unregister`都比较简单，主要是对内部kobject的操作。`kset_create_and_add`这个函数比较有意思，从这个函数也可
  以了解到kobject的使用方式：
```cpp
struct kset *kset_create_and_add(const char *name,
				 const struct kset_uevent_ops *uevent_ops,
				 struct kobject *parent_kobj)
{
	struct kset *kset;
	int error;

    // 分配新的kset
	kset = kset_create(name, uevent_ops, parent_kobj);
	if (!kset)
		return NULL;
	// 内部调用kobject_init，kobject_add,以初始化kobject
	error = kset_register(kset);
	if (error) {
		kfree(kset);
		return NULL;
	}
	return kset;
}
```
   kset_create会创建新的kset，包括kobject，这个kobject是内存中分配的，那么他是什么时候怎么被释放的呢？答案是引用计数。还记得之前说过，`ktype`
   对于一个`kobject`是必要的么，见下：
```cpp
static struct kset *kset_create(const char *name,
				const struct kset_uevent_ops *uevent_ops,
				struct kobject *parent_kobj)
{
	struct kset *kset;
	int retval;

	kset = kzalloc(sizeof(*kset), GFP_KERNEL);
	if (!kset)
		return NULL;
	retval = kobject_set_name(&kset->kobj, "%s", name);
	if (retval) {
		kfree(kset);
		return NULL;
	}
	kset->uevent_ops = uevent_ops;
	kset->kobj.parent = parent_kobj;

    // 见注释
	/*
	 * The kobject of this kset will have a type of kset_ktype and belong to
	 * no kset itself.  That way we can properly free it when it is
	 * finished being used.
	 */
	kset->kobj.ktype = &kset_ktype;
	kset->kobj.kset = NULL;

	return kset;
}

static struct kobj_type kset_ktype = {
	.sysfs_ops	= &kobj_sysfs_ops,
	.release	= kset_release, // release函数
	.get_ownership	= kset_get_ownership,
};
```
  这里跳回kobject_put，看看引用计数减少时的操作：
```cpp
void kobject_put(struct kobject *kobj)
{
	if (kobj) {
		if (!kobj->state_initialized)
			WARN(1, KERN_WARNING
				"kobject: '%s' (%p): is not initialized, yet kobject_put() is being called.\n",
			     kobject_name(kobj), kobj);
		kref_put(&kobj->kref, kobject_release);
	}
}
```
  当引用计数为0时，`kobject_release`函数会被调用，再看看这个函数：
```cpp
static void kobject_release(struct kref *kref)
{
	struct kobject *kobj = container_of(kref, struct kobject, kref);
#ifdef CONFIG_DEBUG_KOBJECT_RELEASE
// 可无视这部分
	unsigned long delay = HZ + HZ * prandom_u32_max(4);
	pr_info("kobject: '%s' (%p): %s, parent %p (delayed %ld)\n",
		 kobject_name(kobj), kobj, __func__, kobj->parent, delay);
	INIT_DELAYED_WORK(&kobj->release, kobject_delayed_cleanup);

	schedule_delayed_work(&kobj->release, delay);
#else
	kobject_cleanup(kobj);
#endif
}

static void kobject_cleanup(struct kobject *kobj)
{
	struct kobject *parent = kobj->parent;
	const struct kobj_type *t = get_ktype(kobj);
	const char *name = kobj->name;

	pr_debug("kobject: '%s' (%p): %s, parent %p\n",
		 kobject_name(kobj), kobj, __func__, kobj->parent);

	if (t && !t->release)
		pr_debug("kobject: '%s' (%p): does not have a release() function, it is broken and must be fixed. See Documentation/core-api/kobject.rst.\n",
			 kobject_name(kobj), kobj);

	/* remove from sysfs if the caller did not do it */
	if (kobj->state_in_sysfs) {
		pr_debug("kobject: '%s' (%p): auto cleanup kobject_del\n",
			 kobject_name(kobj), kobj);
		__kobject_del(kobj);
	} else {
		/* avoid dropping the parent reference unnecessarily */
		parent = NULL;
	}

    // ktype中的release被调用！
	if (t && t->release) {
		pr_debug("kobject: '%s' (%p): calling ktype release\n",
			 kobject_name(kobj), kobj);
		t->release(kobj);
	}

	/* free name if we allocated it */
	if (name) {
		pr_debug("kobject: '%s': free name\n", name);
		kfree_const(name);
	}

	kobject_put(parent);
}
```
  可以看见ktype中的release函数在引用计数为0时被调用了，再看看kset设置的这个release是什么：
```cpp
static void kset_release(struct kobject *kobj)
{
	struct kset *kset = container_of(kobj, struct kset, kobj);
	pr_debug("kobject: '%s' (%p): %s\n",
		 kobject_name(kobj), kobj, __func__);
	kfree(kset);
}
```
 kfree，这正好对应前面的kzalloc，自此资源释放完成，总结一下调用路径：
    kobject_put() -> kobject_release() -> kobject_cleanup() ->kset_release() ->kfree()
 其中`kset_release()`实际上是ktype中的`release`函数指针，也就是说，当一个结构体内嵌了kobject，可以将定制的函数赋给release指针，以实现该结构的
 自动销毁。如下：
```cpp
struct my_kobj {
    struct kobject kobj;
    char arr[256];
};

static void my_release(struct kobject *kobj)
{
	struct my_kobj *p = container_of(kobj, struct my_kobj, kobj);
	kfree(p);
}

static struct kobj_type my_ktype = {
	.sysfs_ops	= &my_sysfs_ops,
	.release	= my_release, // release函数
	.get_ownership	= THIS_MODULE,
};

// test code
void test() {
    struct my_kobj *p = kzalloc(sizeof(my_kobj));
    kobject_init(p->kobj, my_ktype);
    kobject_add(p->kobj, NULL);
}
```