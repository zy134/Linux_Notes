---

---
# 总体流程：<br>
```mermaid
graph LR

init/main.c(start_kernel)-->arch/arm/kernel/setup.c(setup_arch)-->drivers/of/fdt.c(unflatten_device_tree)-->drivers/of/platform.c(of_platform_populate)
```
---
# 具体步骤：<br>
1. bootloader加载`DTB`到内存，并通过寄存器将DTB的地址传递给kernel。在ARM架构上可通过寄存器`R2`找到DTB地址。在kernel初始化时，调用到ARM专用初始化函数，从而开始解析DTB。
```c
// init/main.c
asmlinkage __visible void __init __no_sanitize_address start_kernel(void)
{
	......
	setup_arch(&command_line);
	......
}


// arch/arm/kernel/setup.c
void __init setup_arch(char **cmdline_p)
{
	......
	unflatten_device_tree();
	......
}

```
  如`arch/arm/kernel/head.S`里面所注释，内核启动时DTB的地址已经被存放到`R2`寄存器(ARM64则是X21)，然后将DTB地址继续传递，由`unflatten_device_tree()`函数进行节点的解析。<br>
2.  DTB被打包成二进制结构并作为镜像文件的一部分存放于磁盘，当系统启动时被bootloader读取到内存，然后在内核被重新解析成树状链表，并长期存放于内核堆中。可以先观察一下节点的结构：
```c
struct device_node {
	const char *name;
	phandle phandle;
	const char *full_name;
	struct fwnode_handle fwnode;

	struct	property *properties;
	struct	property *deadprops;	/* removed properties */
	struct	device_node *parent;
	struct	device_node *child;
	struct	device_node *sibling;
#if defined(CONFIG_OF_KOBJ)
	struct	kobject kobj;
#endif
	unsigned long _flags;
	void	*data;
#if defined(CONFIG_SPARC)
	unsigned int unique_id;
	struct of_irq_controller *irq_trans;
#endif
};

```
  可以看出这是个典型的二叉树节点。每个节点都拥有名字，指向父子兄弟的指针，以及对应的资源。此外每个节点都嵌入了kobject，这表示这些子节点也会被释放到`sysfs`中，作为可访问的文件而存在。

  接下来可以研究下节点的解析函数`__unflatten_device_tree()`。该函数负责各种check，并分配内存，然后将节点解析的工作交由`unflatten_dt_nodes()`。该函数会解析DTB所有的节点并生成device_node，并将解析的device_node置于已分配的内存中:
```c
// drivers/of/fdt.c
void *__unflatten_device_tree(const void *blob, // DTB文件的虚拟地址
			      struct device_node *dad, // 父节点，但针对of_root来说是NULL
			      struct device_node **mynodes, // 输出的节点链表
			      void *(*dt_alloc)(u64 size, u64 align), // 内存分配器
			      bool detached)
{
	int size;
	void *mem;

	pr_debug(" -> unflatten_device_tree()\n");

	if (!blob) {
		pr_debug("No device tree pointer\n");
		return NULL;
	}

	pr_debug("Unflattening device tree:\n");
	pr_debug("magic: %08x\n", fdt_magic(blob));
	pr_debug("size: %08x\n", fdt_totalsize(blob));
	pr_debug("version: %08x\n", fdt_version(blob));

	if (fdt_check_header(blob)) {
		pr_err("Invalid device tree blob header\n");
		return NULL;
	}

	/* First pass, scan for size */
	size = unflatten_dt_nodes(blob, NULL, dad, NULL);
	if (size < 0)
		return NULL;

	size = ALIGN(size, 4);
	pr_debug("  size is %d, allocating...\n", size);

	/* Allocate memory for the expanded device tree */
	mem = dt_alloc(size + 4, __alignof__(struct device_node));
	if (!mem)
		return NULL;

	memset(mem, 0, size);

	*(__be32 *)(mem + size) = cpu_to_be32(0xdeadbeef);

	pr_debug("  unflattening %p...\n", mem);

	/* Second pass, do actual unflattening */
	unflatten_dt_nodes(blob, mem, dad, mynodes);
	if (be32_to_cpup(mem + size) != 0xdeadbeef)
		pr_warn("End of tree marker overwritten: %08x\n",
			be32_to_cpup(mem + size));

	if (detached && mynodes) {
		of_node_set_flag(*mynodes, OF_DETACHED);
		pr_debug("unflattened tree is detached\n");
	}

	pr_debug(" <- unflatten_device_tree()\n");
	return mem;
}
```
<br>
3. 将通过树状的device_node链表生成kobject
  函数`unflatten_dt_nodes()`解析DTB的流程略过，有兴趣可以参考`scripts/dtc/libfdt/fdt.c`文件。当调用到该函数时，进而会调用到`populate_node()`，调用该函数后，每个device_node(主要是节点关系链以及每个节点的所有property)的内容都填充完毕：<br>
```c
// drivers/of/fdt.c
static int unflatten_dt_nodes(const void *blob,
			      void *mem,
			      struct device_node *dad,
			      struct device_node **nodepp)
{
	......
	// 解析二进制DTB后，调用populate_node()
		if (!populate_node(blob, offset, &mem, nps[depth],
				   &nps[depth+1], dryrun))
	......
}



```
<br>
  上述流程都是在`setup_arch()`中进行的，主要是一些平台硬件相关的初始化。这些硬件模块初始化完毕后，kernel就需要初始化各个软件模块了，譬如驱动子模块：
```mermaid
graph LR
start_kernel-->driver_init-->of_core_init
```
  `of_core_init`的主要逻辑很简单，上面也提过每个device_node结构都内嵌有kobject，因此通过已经创建的device_node树，可以将他们添加到sysfs中去。该函数调用之后，我们便可以通过`/sys/firmware/devicetree`来查看DTB的内容。通过查看sysfs中的devicetree，可以很容易的了解到各个硬件配置相关信息。<br>
```c
// drivers/of/base.c
void __init of_core_init(void)
{
	struct device_node *np;


	/* Create the kset, and register existing nodes */
	mutex_lock(&of_mutex);
	of_kset = kset_create_and_add("devicetree", NULL, firmware_kobj);
	if (!of_kset) {
		mutex_unlock(&of_mutex);
		pr_err("failed to register existing nodes\n");
		return;
	}
	for_each_of_allnodes(np) {
		__of_attach_node_sysfs(np);
		if (np->phandle && !phandle_cache[of_phandle_cache_hash(np->phandle)])
			phandle_cache[of_phandle_cache_hash(np->phandle)] = np;
	}
	mutex_unlock(&of_mutex);

	/* Symlink in /proc as required by userspace ABI */
	if (of_root)
		proc_symlink("device-tree", NULL, "/sys/firmware/devicetree/base");
}

// drivers/of/kobj.c
int __of_attach_node_sysfs(struct device_node *np)
{
	const char *name;
	struct kobject *parent;
	struct property *pp;
	int rc;

	if (!IS_ENABLED(CONFIG_SYSFS) || !of_kset)
		return 0;

	np->kobj.kset = of_kset;
	if (!np->parent) {
		/* Nodes without parents are new top level trees */
		name = safe_name(&of_kset->kobj, "base");
		parent = NULL;
	} else {
		name = safe_name(&np->parent->kobj, kbasename(np->full_name));
		parent = &np->parent->kobj;
	}
	if (!name)
		return -ENOMEM;

	rc = kobject_add(&np->kobj, parent, "%s", name);
	kfree(name);
	if (rc)
		return rc;

	for_each_property_of_node(np, pp)
		__of_add_property_sysfs(np, pp);

	of_node_get(np);
	return 0;
}

```
4. 在上述流程完成后，`of_platform_populate(NULL, NULL, NULL)`被调用，用于创建`platform_device`节点。只有拥有`compatible`的DT节点才会创建对应的`platform_device`结构。
```c
// of/platform.c
arch_initcall_sync(of_platform_default_populate_init);
```
<br>    `of_platform_default_populate_init`会在kernel初始化阶段调用，进而调用到上面说的`of_platform_populate()`。至于具体是哪个阶段，则可以对应到kernel初始化函数：
```c
static void __init do_basic_setup(void)
{
	cpuset_init_smp();
	driver_init();
	init_irq_proc();
	do_ctors();
	do_initcalls();
}
```
<br>    可以看到，`do_initcalls()`是晚于`driber_init()`调用的，也就是说在上述`of_platform_populate()`调用前，DTB的解析就已经完成了。