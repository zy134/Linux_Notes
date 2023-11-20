# COREDUMP文件解析
---
1. 生成core文件
    通过命令行`ulimit -c`命令可以指定生成core文件的大小，若为0则不生成core文件。
    通过`cat /proc/sys/kernel/core_pattern`指定生成coredump的路径以及coredump文件格式。
2. 需要准备的环境
    解析core文件时，需要提供以下前提：
    + core对应的可执行文件。该可执行文件编译时，需要添加编译选项`-fno-omit-frame-pointer`，以生成帧指针
    + 可执行文件所加载的动态库
        上述文件需要带符号表，否则core解析出的结果往往是一堆`???`或者`broken stack frame`。通过`readelf --section-headers`查看二进制文件是否存在符号表。
        一般来说没有strip的可执行文件和库，section headers里面会有名为**symtab**的段，即为符号段，若该段的大小不为0，那么符号表应该就是正常的。如果没有
        符号表，通过`-g -ggdb -no-omit-frame-pointer`建议重新编译一份库/可执行文件。如果debug文件过大，可以通过`strip ----strip-debug`，仅仅只去掉
        debug相关的段，而保留符号表。
        此外，可以在GDB中手动设定加载动态库路径，以加载带符号的动态库环境，用于解析core文件：

```
(gdb) core /path/to/coredump.elf
(gdb) symbol-file /path/to/symbols.elf
```
<br>           也可在GDB中直接加载符号表，符号表可以从带有符号的动态库中分离出来，通过`objcopy --only-keep-debug foo foo.debug`生成符号文件。
```
# 命令加载带符号的可执行文件
$ gdb --se /path/to/symbols.elf
# 命令加载符号文件
$ gdb --symbols /path/to/symbols.elf

# 运行时加载符号表
(gdb) set solib-search-path /path/to/lib
(gdb) symbol-file /path/to/symbols.elf
```

3. 打印backtracie，直接在GDB中用命令`bt`即可。通过`f NUMBER`可以打印对应栈帧的数据，f表示frame，后面跟的数字是backtrace中的第几帧的意思
# 调用栈解析
---
    没有符号表时可按照下述流程解析调用栈：
1. 找到crash发生的虚拟地址。
2. 在gdb下使用`info proc mappings`可以获取共享库和可执行文件在内存中的映射，从而获取程序运行时的基址。
3. 通过 **crash virtual addr - process base addr**，获取crash点的相对地址，然后通过**addr2line**直接解析该地址定位到源文件的具体行，解析失败那就继续往下走
4. 找到crash的库/可执行文件，通过`objdump -d`，反编译二进制文件从而获取汇编文件
5. 通过在汇编文件中定位上面获取的crash在库中的相对地址，找到crash发生的行，并定位crash发生的函数
6. 通过**c++filt**函数复原函数名字，然后对着源代码和汇编代码综合分析

    Tips:
    + 当出现`0x00000000 ??`这样的结果，一般是调用函数指针为空，可以通过`lr`寄存器中的地址，判断空指针发生在进程哪个库，进而打trace定位问题