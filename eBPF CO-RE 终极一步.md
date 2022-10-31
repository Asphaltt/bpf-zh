# eBPF CO-RE 的终极一步

本文翻译自 [eBPF BTF GENERATOR: The road to truly portable CO-RE eBPF programs](https://github.com/aquasecurity/btfhub/blob/main/docs/btfgen-internals.md)。

> 译者水平有限，如有疑问，请查阅原文。
>
> 本文为翻译初稿，未校对，请谅解。

---

## 简介

CO-RE 需要描述类型的 BTF 信息去做到重定位（relocations）。这通常由配置了 CONFIG_DEBUG_INFO_BTF 选项的内核自己提供的。然而，该配置并不是在所有的发行版 Linux 系统里开启，也不存在于旧版本内核里。

对于不支持 CONFIG_DEBUG_INFO_BTF 配置的内核，通过外部 BTF 信息也能够做到 CO-RE。[BTFHub](https://github.com/aquasecurity/btfhub/) 包含了不支持 BTF 的发行版 Linux 系统的 BTF 信息。

使用这些 BTF 文件会带来如下问题：

1. 每个 BTF 文件有几 MB 大小。所以不可能将所有内核 BTF 文件跟随着 eBPF 程序一起分发。（如果为了跑在大量内核里，这些 BTF 文件会达到 GB 大小）
2. 在程序启动时再下载对应的 BTF 文件；但并不是任何时候都可以从外网下载 BTF 文件。

只提供 eBPF 程序所需要的 BTF 信息，这样可以绝杀大部分场景。通常 eBPF 程序只需要固定几个内核数据结构（kernel fields）。

我们的目的就是：拓展 libbpf，提供 API 去生成仅仅包含 eBPF 程序（eBPF object）所需要的类型信息的 BTF 文件。相比包含所有内核类型信息的 BTF 文件，这样生成的 BTF 文件通常都非常小。这就可以将生成的 BTF 文件跟随着 eBPF 程序一起分发了。

在 Linux Plumbers 2021 大会上，在 [Towards truly portable eBPF](https://www.youtube.com/watch?v=igJLKyP1lFk&t=2418s) 技术分享中讨论了这个想法。我们在 [BTFGen](https://github.com/kinvolk/btfgen) 仓库中放了个使用该 API 的例子。我们计划**将该想法合入 bpftool 中** （support in bpftool once it's merged in libbpf）。

这有个 [例子](https://github.com/aquasecurity/btfhub/tree/main/tools) 能够示例如何使用 BTFGen 和 BTFHub 去给多个内核生成多个 BTF 文件，并内嵌到一个应用中。譬如：一个复杂的 eBPF 程序（bpf object）仅仅需要 1.5 MB 的 BTF 文件就能够支持近 400 个内核。

> 如有时间，请观看 **Linux Plumbers 2021** 的技术分享，里面讲述了写可移植的 eBPF 代码的困难：[Towards truly portable eBPF](https://www.youtube.com/watch?v=igJLKyP1lFk&t=734s)。

> 在技术分享的末尾，在 [演示了给 eBPF 程序支持多个内核的所有障碍](https://t.co/e5YjqdxQDU) 之后，有个关于 [如何使用 BTFHub](https://youtu.be/igJLKyP1lFk?t=2315) 和 [接下来的步骤的讨论](https://youtu.be/igJLKyP1lFk?t=2418) 的演示。

## 作者与声明

为创造 **btfgen** 工具而感谢：

Mauricio Vasquez Bernal (Kinvolk/Microsoft) - 主作者
Rafael David Tinoco (Aqua Security) - 修复与检视（fixes and review）
Lorenzo Fontana (Elastic) - 修复与检视（fixes and review）
Itay Shakury (Aqua Security) - 主意、支持与管理
Marga Manterola (Kinvolk/Microsoft) - 支持与管理

**原文档** 由以下人员编写和检视（created and reviewed）：

Rafael David Tinoco (Aqua Security) - 主作者
Mauricio Vasquez Bernal (Kinvolk/Microsoft) - 检视与修复
Lorenzo Fontana (Elastic) - 检视与修复
Yaniv Agman (Aqua Security) - 检视

> 代码并未合入上游，代码仓库如下：
>
> https://github.com/kinvolk/libbpf/tree/btfgen
> https://github.com/kinvolk/btfgen
>
> [合入到 libbpf](https://x-lore.kernel.org/bpf/20211027203727.208847-1-mauricio@kinvolk.io/) 是为了将 **bpfgen** 变成 bpftool 的一个子功能。

## eBPF CO-RE: eBPF 可移植性背后的引擎

诚如大家所知，eBPF 可移植性高度依赖于 [代码重定位（code relocation）](https://en.wikipedia.org/wiki/Relocation_(computing))。通常，内存重定位（memory relocation）发生在 [链接或加载阶段](https://stffrdhrn.github.io/hardware/embedded/openrisc/2019/11/29/relocs.html)。BPF 里亦是如此。

> 如果你对 "链接器、加载器与重定位（linkers & loaders and relocations）" 的概念感兴趣，请查阅 [这篇文章](https://rafaeldtinoco.com/toolchain-relocations-overview/)。

## eBPF 重定位（简介）

引用 Brendan Gregg 的文章来解释为什么 eBPF 需要重定位：

> 不是仅仅将 BPF 字节码保存到 ELF 中，然后发给内核。许多 BPF 程序使用的内核结构体（kernel structs）在不同内核中会有所变更。你的 BPF 字节码依然可以跑在不同的内核中，但可能读取到错误的结构体字段并打印出一些垃圾（reading the wrong struct offsets and printing garbage output）。

> 这是个关于重定位的问题，也是关于为 BPF 程序（BPF binaries）而生的 BTF 和 CO-RE 的问题。BTF 提供的类型信息，从而可以按需获取结构体偏移（struct offsets）和其他更多细节，并且 CO-RE 可以记录需要重写的部分 BPF 程序和如何重写（and how）。

eBPF ELF 对象不以普通 ELF 对象的方式进行组织，且不可执行；所以它不包含 **程序头部表项**（program header table entries），如前一节所表述。总之，当处理不同架构（other architectures）时，相比 GCC/CLANG 生成的 ELF 文件，eBPF ELF 文件拥有 _不一样的片段_（different sections）。

在同一个 eBPF 程序文件里，里面可能有 _多个不同的_ （multiple different）eBPF 程序。每个非内联函数（non-inline function）都可以是不同的程序。每个 eBPF 程序都有其各自的 ELF 片段（ELF section）（TEXT 片段），也有独属的重定位表项（relocation table section）（REL），这就区别于只有单一 .text 片段（.text section）的 ELF 可执行文件。eBPF 对象里所有声明的 map 都保存在 .maps （DATA）片段（section）。

如下示例：

![eBPF Object File](eBPF%20CO-RE%20%E7%BB%88%E6%9E%81%E4%B8%80%E6%AD%A5.assets/image03.png)

上图就是 [BTFHub 例子](https://github.com/aquasecurity/btfhub/tree/main/example) eBPF 程序的样例。看下 [源代码](https://github.com/aquasecurity/btfhub/blob/main/example/example.bpf.c) ，就可以看到这里有 2 个_内联函数_（inlined functions）和 1 个 _非内联函数_（non-inlined one）。诚如读者所知（As the reader probably knows），内联函数会成为调用者的一部分（编译器不会在栈中使用新的栈帧）（the compiler won't arrange the stack with a new frame），所以最终我们得到仅有的一个 eBPF 程序。：

1. 由 `do_sys_openat2` 内核函数触发的一个 kprobe eBPF 程序。

> 注：这仅用来做个示范（for education purposes only），一个简单的 `open` 调用就会触发该 eBPF 程序的执行。这只是用来描述不同的程序类型和他们的 ELF 片段（ELF sections）是怎样的。

对于每个 eBPF 程序 ELF 片段（ELF section），这简单例子里只有一个，都有包含所有重定位信息的对应的片段（correspondent section）：

![eBPF Object File](./eBPF%20CO-RE%20%E7%BB%88%E6%9E%81%E4%B8%80%E6%AD%A5.assets/image04.png)

用于该 eBPF 对象的 **类型**、**函数** 等信息都 **包含在如下两个 ELF 片段中**：

.BTF 和 .BTF.ext（稍候将解释它们）。

![eBPF Object File](./eBPF%20CO-RE%20%E7%BB%88%E6%9E%81%E4%B8%80%E6%AD%A5.assets/image05.png)

在此简介之后，本文将关注于这些结构，并解释 **BTF 生成器**（BTF GENERATOR）是什么、使用 eBPF CO-RE 寻求代码可移植性的 eBPF 开发者将从中受益。

## BPF 类型格式（或 BTF）

尽管 [Andrii's - BTF deduplication and Linux Kernel BTF](https://nakryiko.com/posts/btf-dedup/) 才是讲解 BTF 的最佳文档，但这里才是讲解脱离 BTF 而 eBPF CO-RE 寸步难行的地方（without BTF, eBPF CO-RE would be very hard (or impossible) to be achieved）。

那份文档里有如下图表：

![BTF sample](./eBPF%20CO-RE%20%E7%BB%88%E6%9E%81%E4%B8%80%E6%AD%A5.assets/image02.png)

用来描述 **BTF 类型图**（BTF type graph）。如你所见，由 **类型描述符**（type descriptors）组成的 BTF 使用 **BTF 类型**（BTF types）来描述 eBPF 对象中使用的类型。每个 **BTF 类型** 都有特定 **类别** （KIND），可能指向其他 **BTF 类型**，也可能不指向。

你可以想像 BTF 有如下内存集合（memory chunk）：

- 一个头部
- 一个以 null 结束的字符串
- 两两相连的 **BTF 类型**

### BTF 类别（BTF KINDS）

本文编写时就存在以下 BTF 类别：

- [BTF_KIND_INT](https://www.kernel.org/doc/html/latest/bpf/btf.html#btf-kind-int)： 整数。
- [BTF_KIND_FLOAT](https://www.kernel.org/doc/html/latest/bpf/btf.html#btf-kind-float)： 浮点数。
- [BTF_KIND_PTR](https://www.kernel.org/doc/html/latest/bpf/btf.html#btf-kind-ptr)：指向其他类型的指针。
- [BTF_KIND_ARRAY](https://www.kernel.org/doc/html/latest/bpf/btf.html#btf-kind-array)： 指定类型的数组，使用同样/其他的类型当作索引。
- [BTF_KIND_STRUCT](https://www.kernel.org/doc/html/latest/bpf/btf.html#btf-kind-struct)：有字段成员的指定类型。
- [BTF_KIND_UNION](https://www.kernel.org/doc/html/latest/bpf/btf.html#btf-kind-union)：跟结构体雷同。
- [BTF_KIND_ENUM](https://www.kernel.org/doc/html/latest/bpf/btf.html#btf-kind-union)：指定类型的枚举。
- [BTF_KIND_FWD](https://www.kernel.org/doc/html/latest/bpf/btf.html#btf-kind-fwd)：其他类型的前向声明（forward-declaration）。
- [BTF_KIND_TYPEDEF](https://www.kernel.org/doc/html/latest/bpf/btf.html#btf-kind-typedef)：其他类型的 typedef。
- [BTF_KIND_VOLATILE](https://www.kernel.org/doc/html/latest/bpf/btf.html#btf-kind-volatile)：volatile。
- [BTF_KIND_CONST](https://www.kernel.org/doc/html/latest/bpf/btf.html#btf-kind-const)：常量。
- [BTF_KIND_RESTRICT](https://www.kernel.org/doc/html/latest/bpf/btf.html#btf-kind-restrict)：restrict。
- [BTF_KIND_FUNC](https://www.kernel.org/doc/html/latest/bpf/btf.html#btf-kind-func)：函数，不是一种类型。定义一段子程序/函数。
- [BTF_KIND_FUNC_PROTO](https://www.kernel.org/doc/html/latest/bpf/btf.html#btf-kind-func-proto)：函数原型/签名类型。
- [BTF_KIND_VAR](https://www.kernel.org/doc/html/latest/bpf/btf.html#btf-kind-var)：变量。（指向变量类型）。
- [BTF_KIND_DATASEC](https://www.kernel.org/doc/html/latest/bpf/btf.html#btf-kind-datasec)：数据片段（elf data section）。

BTF 类别编码为二进制。它们一个接着另一个，也可能依赖于其他类别，结构体就会附带更多信息（an addended structure containing more information）（比如结构体别类（STRUCTS）附带其它结构体成员）。

下图示例了主要 BTF 类型类别的内存组织（memory organization）：

![memory organization of BTF type kinds](./eBPF%20CO-RE%20%E7%BB%88%E6%9E%81%E4%B8%80%E6%AD%A5.assets/image06.png)

### BTF ELF 片段

eBPF 程序中的每个变量、函数都在 ELF 片段 **.BTF** 中使用 BTF 类别去描述。

`.BTF` 和 `.BTF.ext` ELF 片段可以用两种方式进行编码：

1. [pahole](https://lwn.net/Articles/762847/) 工具（[dwarves package](https://github.com/acmel/dwarves)）：使用非修剪（non-stripped）的 ELF 文件 [DWARF debug data](https://en.wikipedia.org/wiki/DWARF)。该 ELF 文件既可以是内核（查看下一小节），也可以是普通的 ELF 文件。这两个额外的 ELF 片段将会加到原 ELF 文件中，或者生成一个独立的 BTF 文件（more recently, to an external RAW BTF file）（用于 libbpf）。
2. LLVM：自从 [支持 BTF 生成](https://reviews.llvm.org/D53261)，在编译 eBPF 程序时，这两片段由 LLVM 自动创建，用于提供 eBPF 程序类型的 BTF 信息。

诚如之前所说，这两 ELF 片段的区别如下：

1. .BTF - 包含了当前对象所使用的所有 BTF 类型信息
2. .BTF.ext - 包含了函数原型、行号等调试信息，和 LLVM 编码的用于加载 eBPF 程序所使用的重定位信息（information about needed relocations to load the ELF object）。

你可以执行 **bpftool** 命令来查看这些信息（[BTFHUB example](https://github.com/aquasecurity/btfhub/tree/main/example) 中使用的对象）：

```c
$ bpftool btf dump file ./example.bpf.o

[1] PTR '(anon)' type_id=2
[2] STRUCT 'pt_regs' size=168 vlen=21
        'r15' type_id=3 bits_offset=0
        'r14' type_id=3 bits_offset=64
        'r13' type_id=3 bits_offset=128
        'r12' type_id=3 bits_offset=192
        'bp' type_id=3 bits_offset=256
        'bx' type_id=3 bits_offset=320
        'r11' type_id=3 bits_offset=384
        'r10' type_id=3 bits_offset=448
        'r9' type_id=3 bits_offset=512
        'r8' type_id=3 bits_offset=576
        'ax' type_id=3 bits_offset=640
        'cx' type_id=3 bits_offset=704
        'dx' type_id=3 bits_offset=768
        'si' type_id=3 bits_offset=832
        'di' type_id=3 bits_offset=896
        'orig_ax' type_id=3 bits_offset=960
        'ip' type_id=3 bits_offset=1024
        'cs' type_id=3 bits_offset=1088
        'flags' type_id=3 bits_offset=1152
        'sp' type_id=3 bits_offset=1216
        'ss' type_id=3 bits_offset=1280
[3] INT 'long unsigned int' size=8 bits_offset=0 nr_bits=64 encoding=(none)
[4] FUNC_PROTO '(anon)' ret_type_id=5 vlen=1
        'ctx' type_id=1
[5] INT 'int' size=4 bits_offset=0 nr_bits=32 encoding=SIGNED
[6] FUNC 'do_sys_openat2' type_id=4 linkage=global
[7] STRUCT 'task_struct' size=9472 vlen=229
        'thread_info' type_id=8 bits_offset=0
        'state' type_id=12 bits_offset=192
        'stack' type_id=14 bits_offset=256
        'usage' type_id=15 bits_offset=320
        'flags' type_id=11 bits_offset=352
        'ptrace' type_id=11 bits_offset=384
        'on_cpu' type_id=5 bits_offset=416
        'wake_entry' type_id=19 bits_offset=448
        'cpu' type_id=11 bits_offset=576
...
[8] STRUCT 'thread_info' size=24 vlen=3
        'flags' type_id=3 bits_offset=0
        'syscall_work' type_id=3 bits_offset=64
        'status' type_id=9 bits_offset=128
[9] TYPEDEF 'u32' type_id=10
[10] TYPEDEF '__u32' type_id=11
[11] INT 'unsigned int' size=4 bits_offset=0 nr_bits=32 encoding=(none)
[12] VOLATILE '(anon)' type_id=13
[13] INT 'long int' size=8 bits_offset=0 nr_bits=64 encoding=SIGNED
[14] PTR '(anon)' type_id=0
[15] TYPEDEF 'refcount_t' type_id=16
[16] STRUCT 'refcount_struct' size=4 vlen=1
        'refs' type_id=17 bits_offset=0
[17] TYPEDEF 'atomic_t' type_id=18
[18] STRUCT '(anon)' size=4 vlen=1
        'counter' type_id=5 bits_offset=0
[19] STRUCT '__call_single_node' size=16 vlen=4
        'llist' type_id=20 bits_offset=0
        '(anon)' type_id=22 bits_offset=64
        'src' type_id=23 bits_offset=96
        'dst' type_id=23 bits_offset=112
[20] STRUCT 'llist_node' size=8 vlen=1
        'next' type_id=21 bits_offset=0
...
```

**bpftool** 命令可以展示 `.BTF`  ELF 片段中的所有类型信息。它获取所有 **BTF 类型** 并解析这些类型，和字符串结合（string chunk）中所需的字符串（因为有的 BTF 类型中的 "name_offset" 指向了字符串缓存（string buffer）中的偏移量）。

如果你还记得前面的图表，可以从具体的 BTF 类型声明中解析得到。举例，一个具体的原始的类型（一个结构体）：

```c
The BTF_TYPE id 311 is a BTF_KIND_STRUCT and describes a STRUCT
called  "swregs_state":

        [311] STRUCT 'swregs_state' size=136 vlen=16
                'cwd' type_id=9 bits_offset=0
                'swd' type_id=9 bits_offset=32
                'twd' type_id=9 bits_offset=64
                'fip' type_id=9 bits_offset=96
                'fcs' type_id=9 bits_offset=128
                'foo' type_id=9 bits_offset=160
                'fos' type_id=9 bits_offset=192
                'st_space' type_id=302 bits_offset=224
                'ftop' type_id=58 bits_offset=864
                'changed' type_id=58 bits_offset=872
                'lookahead' type_id=58 bits_offset=880
                'no_update' type_id=58 bits_offset=888
                'rm' type_id=58 bits_offset=896
                'alimit' type_id=58 bits_offset=904
                'info' type_id=312 bits_offset=960
                'entry_eip' type_id=9 bits_offset=1024

The struct 'swregs_state' has a field 'entry_eip'. This field
is a BTF_MEMBER of the BTF_TYPE id 311 (struct). The member
points to BTF_TYPE id 9.

        [9] TYPEDEF 'u32' type_id=10

The BTF_TYPE id 9 is a BTF_KIND_TYPEDEF and describes a typedef
called 'u32'. It points to the BTF_TYPE id 10.

        [10] TYPEDEF '__u32' type_id=11

The BTF_TYPE id 10 is a BTF_KIND_TYPEDEF and describes a typedef
called '__u32'. It points to the BTF_TYPE id 11.

        [11] INT 'unsigned int' size=4 bits_offset=0 nr_bits=32 encoding=(none)

The BTF_TYPE id 11 is a BTF_KIND_INT and describes an integer of
type "unsigned int". It is not a modifier BTF type, so it does not
point to anything more.

The BTF_TYPE id 11 is a BTF_KIND_INT and describes a “32 bits signed integer”
called "unsigned int". It is not a modifier BTF type, so it does not point to
anything more.
```

### BTF 独立文件

前面已说到，[pahole](https://lwn.net/Articles/762847/) 工具（[dwarves package](https://github.com/acmel/dwarves)）可以为 ELF 生成 BTF 信息。因为 GCC 编译器还不能生成 BTF 信息，[这是当前 Linux 内核得到 BTF 片段的方法](https://elixir.bootlin.com/linux/v5.14.14/source/scripts/link-vmlinux.sh#L218)。

> 如果你想知道为什么 Linux 内核的 ELF 文件也有 BTF 信息，本文就进入一个重要阶段（reached into an important mark）。当 libbpf 将 eBPF 对象装载进内核时，eBPF 重定位（eBPF relocations）将使用来自内核的和来自 eBPF 对象的 BTF 信息，用来计算、推测对象中 eBPF 程序所需的重定位，为了在内核 eBPF VM 中运行。

> 在 [这次提交](https://lwn.net/Articles/790177/) 之后，BTF 信息也用于创建 [eBPF maps](https://ebpf.io/what-is-ebpf#maps)。BPF 对象拥有关于 map （`maps` ELF 片段）和 map 类型（`.BTF` ELF 片段）的足够的信息，所以能被 eBPF 程序和用户态代码同时用来创建 eBPF map。

在 [这次提交](https://github.com/acmel/dwarves/commit/89be5646a03435bfc6d2b3f208989d17f7d39312) 之前，pahole 只支持将 BTF 信息编码进 ELF 文件里。这次提交增加了将 BTF 信息编码进一个独立文件（detached file）（我们称之为 _外部的_（独立的）BTF 文件、_原始的_ BTF 文件）里的功能。

需要独立 BTF 文件的原因是，libbpf [需要加载外部的 BTF 文件](https://lore.kernel.org/bpf/1626180159-112996-2-git-send-email-chengshuyi@linux.alibaba.com/) 用来描述当前不支持 BTF 的内核。不幸的是，[有些旧的发行版本（甚至最近的一些）的内核不带有 BTF 信息](https://bugs.launchpad.net/ubuntu/+source/linux/+bug/1926330)；对于这些情况，非常有必要从现有的 DWARF 信息（DWARF symbols）中生成 BTF 信息，这就是 pahole 干的事情。

## BTFHub

有了前面关于 BTF 独立文件的概念之后，我们暂停深挖 **BTFHub 是什么、从何而来**：BTF 独立文件应该用于没有（在 ELF 片段（ELF sections）中）内嵌用来保存调试信息（可从 DWARF 格式转为 BTF 格式）的 BTF 信息的内核。

因此，[BTFHUB](https://github.com/aquasecurity/btfhub/) 出现了。它包含了每个（发行版本的）内核的 BTF 文件。它的 [README.md](https://github.com/aquasecurity/btfhub/blob/main/README.md) 讲解了创建 BTF 独立文件的过程，也列举了大部分发行的 Linux （或过时的（some EOL））内核是否支持 BTF。

> 当前，以下系统需要 BTF 独立文件：CentOS7 (7.5.1804, kernel: 3.10.0-862), CentoOS 8 (8.1.1911, kernel: 4.18.0-147), Fedora 29 (kernel: 4.18), Fedora 30 (kernel: 5.0), Fedora 31 (kernel: 5.3), Ubuntu Bionic (with HWE kernels: 5.4 and 5.8) and Ubuntu Focal (kernel: 5.4 and HWE kernels: 5.8 and 5.11)。如果你使用了这些发行系统的新版本，很可能你不需要 BTF 独立文件，因为那些系统的内核可能已有 ELF .BTF 片段（ELF .BTF section）信息。你可以通过命令`bpftool btf dump file /sys/kernel/btf/vmlinux format raw` 检查是否已支持 BTF。

> _注：_ BTFHub 欢迎提交 PR（opened to contributions），所以如果你的项目需要 BTF 独立文件，你可以提交请求让 BTFHub 去收录你的 BTF 文件。

如果你想试试使用 BTF 独立文件，如下：

```bash
$ git clone --recurse-submodules git@github.com:aquasecurity/btfhub.git
$ cd ./btfhub/example
$ make
```

然后执行 `example-c-static` ，默认使用 **已有内核 BTF 信息** 、或者 **提供 BTF 独立文件**，所以 （`example`所使用的）libbpf 可以为需要在当前内核中运行的 eBPF 对象/程序计算出所需要的重定位（needed relocations）。

首先演示一下如何给二进制 **提供 BTF 独立文件**。先使用当前内核的 BTF 文件， libbpf 默认也是这样，来测试一下：

```bash
$ sudo EXAMPLE_BTF_FILE=/sys/kernel/btf/vmlinux ./example-c-static

Foreground mode...<Ctrl-C> or or SIG_TERM to end it.
libbpf: loading example.bpf.o
...
```

现在，在另一个更老的内核中，比如 Ubuntu Bionic，**使用 BTFHub 提供的 BTF 文件**，我们可以运行同样的二进制：

```bash
$ uname -a
Linux bionic 5.4.0-87-generic #98~18.04.1-Ubuntu SMP Wed Sep 22 10:45:04 UTC 2021 x86_64 x86_64 x86_64 GNU/Linux

# Copy compressed BTF file and uncompress it:

$ cp /sources/ebpf/aquasec-btfhub/ubuntu/bionic/x86_64/$(uname -r).btf.tar.xz .
$ tar xvfJ ./$(uname -r).btf.tar.xz
5.4.0-87-generic.btf

# Execute ./example-c-static with the BTF for the current kernel

$ sudo EXAMPLE_BTF_FILE=./5.4.0-87-generic.btf ./example-c-static
Foreground mode...<Ctrl-C> or or SIG_TERM to end it.
libbpf: loading example.bpf.o
..
```

### BTFHub 作为解决方案和一个问题

敏感的读者可能已有疑问：OK，是否损耗太重了（isn't that too expensive）？那些 BTF 文件的大小呢？为了做到完全的可移植，项目中需要多少 BTF 独立文件呢？为了在任何一个内核里运行，是否要将所有 BTFHub 里的 BTF 独立文件都跟 eBPF 程序一起打包？

> 让我们回到 Linux Plumbers 2021 分享：[Towards truly portable eBPF](https://www.youtube.com/watch?v=igJLKyP1lFk&t=1520s)，但现在只需要关注 [discussion about next steps](https://youtu.be/igJLKyP1lFk?t=2825)。

没错，这就是最大的问题，也是 **btfgen** 诞生的最主要的原因。

```bash
[user@aquasec-btfhub]$ ls -1 ./ubuntu/bionic/x86_64 | head -10
4.15.0-20-generic.btf.tar.xz
4.15.0-22-generic.btf.tar.xz
4.15.0-23-generic.btf.tar.xz
4.15.0-24-generic.btf.tar.xz
4.15.0-29-generic.btf.tar.xz
4.15.0-30-generic.btf.tar.xz
4.15.0-32-generic.btf.tar.xz
4.15.0-33-generic.btf.tar.xz
4.15.0-34-generic.btf.tar.xz
4.15.0-36-generic.btf.tar.xz

[user@aquasec-btfhub]$ du -sh .
1.5G	.
```

现今 BTFHub 已保存有 1.5GB 的压缩 BTF 文件（平均每个 1MB）。不幸的是，将它们都跟 eBPF 软件一起打包是不可行的。我们的想法，诚如 Linux Plumbers 大会上讨论的，是 **最小化每个所需的 BTF 文件，使得它们仅包含 eBPF 应用所需要的 BTF 信息**。

> 如果不够清晰：并非使用所有的内核类型，每个 BTF 独立文件仅仅包含了代码执行时 libbpf 计算 eBPF 重定位真正需要的类型。

> 那次分享之后，来自 Kinvolk/Microsoft 的 Mauricio 联系我们，Aqua Security 开源团队，说他们早有这想法，并邀请我们一起合作完成这件杰作。

下一小节将会讲解 **btfgen** 带来的结果：修剪可移植的 eBPF 应用程序的大小。

但让我们先学习一下 BPF 重定位。

## eBPF CO-RE 和 BPF 重定位（续集）

> 注：本小节来源于 Andrii's blog (bpf-core-reference-guide)。

现今你已熟悉 BTF 信息是如何组织起来的，所以我们可以继续探讨 [BPF LLVM relocations](https://www.kernel.org/doc/html/latest/bpf/llvm_reloc.html)。对于 eBPF 对象而言，[libbpf](https://github.com/libbpf/libbpf) 就像是一个 _动态链接器_。有了从 **eBPF 对象** 和 **目标内核** 得到的 BTF 信息，libbpf 就可以在装载 eBPF 对象进内核之前解决所有的重定位（relocations）。

Andrii 在 [this patchset](https://x-lore.kernel.org/bpf/20190724192742.1419254-1-andriin@fb.com/) 中给 libbpf 增加了 CO-RE (Compile Once - Run Everywhere) 的支持。在 [presented in 2019](http://vger.kernel.org/bpfconf2019_talks/bpf-core.pdf) 中，对 BPF-CORE 总结如下：

1. 自描述的内核（BTF）
2. clang 生成重定位（relocations）信息（__builtin_preserve_access_index() 特性）
3. libbpf 当作重定位装载器（relocating loader）

> 关于本文档的编写，你应该和 Andrii 写的 [eBPF CO-RE 参考指引](https://nakryiko.com/posts/bpf-core-reference-guide/) 一起阅读。它能提升你对 eBPF CO-RE 特性的理解，且帮助理解 BTF 生成器（BTF generator）为什么诞生背后的主意。

**总而言之，有了 CO-RE，同一个 eBPF 对象就能够在没有重新编译的情况下运行在多个目标内核中。**

像普通的 ELF 重定位（ELF relocations），eBPF 字节码也许要在装载进内核之前完成 TEXT（指令）和 DATA 片段（segments）中的重定位处理，因为内核 BPF JIT VM 会 JIT 处理 eBPF 字节码，并且当作当前内核架构的指令去执行（executed as instructions of the architecture you're running your kernel on）。

libbpf 当前支持的重定位类型（kinds of relocations）：

1. 本地重定位（Local Relocations）
2. 基于属性的重定位（Field Based Relocations）
3. 基于类型的重定位（Type Based Relocations）
4. 基于枚举值的重定位（ENUM Value Based Relocations）

> 不要将这里所说的 _重定位类别（kinds of relocations）_ 跟架构（architecture）（对于 x64 架构：`R_X86_NONE`, `R_X86_64`, `R_X86_64_PC32` 等等）支持的重定位类型（types of relocations）搞混了。

### 1. 本地重定位（Local Relocatioins）

本地重定位源于编译器/架构的工作机制（how the compiler/architecture works）。譬如，那些处理全局变量（包括 eBPF MAPs）和函数符号名称（function symbol names）的重定位。它们就是 **架构** 所支持的 _重定位类型（types of relocations）_（不是 **libbpf** 所支持的

重定位类别（kinds of relocations））。

eBPF 架构支持以下 **6 种重定位类型**：

```
Enum  ELF Reloc Type     Description      BitSize  Offset        Calculation
0     R_BPF_NONE         None
1     R_BPF_64_64        ld_imm64 insn    32       r_offset + 4  S + A
2     R_BPF_64_ABS64     normal data      64       r_offset      S + A
3     R_BPF_64_ABS32     normal data      32       r_offset      S + A
4     R_BPF_64_NODYLD32  .BTF[.ext] data  32       r_offset      S + A
10    R_BPF_64_32        call insn        32       r_offset + 4  (S + A) / 8 - 1
```

并且那些重定位类型（relocations types）将用于 eBPF 重定位表（relocation tables），用来指导 JIT 编译器去重定位（对象）的本地类型。

在我们的 BTFHub 代码例子中，我们能看到这些本地重定位：

```
$ llvm-readelf -r ./example.bpf.o

Relocation section '.relkprobe/do_sys_openat2' at offset 0x6f88 contains 1 entries:
    Offset             Info             Type               Symbol's Value  Symbol's Name
0000000000000290  0000000500000001 R_BPF_64_64            0000000000000000 events

Relocation section '.rel.BTF' at offset 0x6f98 contains 2 entries:
    Offset             Info             Type               Symbol's Value  Symbol's Name
0000000000003a40  0000000300000000 R_BPF_NONE             0000000000000000 LICENSE
0000000000003a58  0000000500000000 R_BPF_NONE             0000000000000000 events

Relocation section '.rel.BTF.ext' at offset 0x6fb8 contains 42 entries:
    Offset             Info             Type               Symbol's Value  Symbol's Name
000000000000002c  0000000200000000 R_BPF_NONE             0000000000000000 kprobe/do_sys_openat2
0000000000000040  0000000200000000 R_BPF_NONE             0000000000000000 kprobe/do_sys_openat2
0000000000000050  0000000200000000 R_BPF_NONE             0000000000000000 kprobe/do_sys_openat2
0000000000000060  0000000200000000 R_BPF_NONE             0000000000000000 kprobe/do_sys_openat2
...
```

这就跟普通的 ELF 装载时/链接时的重定位（load-time/link-time relocations）很像；链接器能读到符号表：

```
$ llvm-readelf --symbols ./example.bpf.o

Symbol table '.symtab' contains 6 entries:
   Num:    Value          Size Type    Bind   Vis       Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT   UND
     1: 00000000000002c8     0 NOTYPE  LOCAL  DEFAULT     2 LBB0_2
     2: 0000000000000000     0 SECTION LOCAL  DEFAULT     2 kprobe/do_sys_openat2
     3: 0000000000000000     4 OBJECT  GLOBAL DEFAULT     4 LICENSE
     4: 0000000000000000   728 FUNC    GLOBAL DEFAULT     2 do_sys_openat2
     5: 0000000000000000    20 OBJECT  GLOBAL DEFAULT     3 events
```

差别在于，在普通 ELF 文件中所有重定位（relocations）通常放在 .rela.text ELF 片段中（ELF section .rela.text），而在 eBPF 对象 ELF 文件中则会放在不同的（名字以 .relXXX 开头的）片段（sections）中。

### eBPF CO-RE 相关的重定位

在讲解了本地重定位（local relocations）之后，剩下的重定位类别都是 eBPF 专属的，使用 BTF 信息的，并且它们在加载 eBPF 对象之前就被 libbpf 处理完毕。它们是专为 eBPF 对象的 **加载时（load-time）** 重定位。通过遍历 eBPF ELF 对象，指令和数据都能够被那些对应类别的重定位修改。

#### 2. 基于属性的重定位（Field Based Relocations）

经过本地重定位（local relocations）之后，eBPF 对象还不能被加载。因为有其它重定位需要处理：基于类型属性的重定位。程序员会在源代码中明确枚举它们。基于 **builtin_preserve_access_index** 特性，出现了更好使用 eBPF 帮助函数的宏（MACROs）。

当编译 CO-RE（一次编译，到处运行）BPF 架构的对象时，LLVM BPF 后端将每个重定位记录在仅包含重定位信息的 **ELF 结构体（ELF structure）** 中。那些结构体存放在对应的 RELO 片段（.BTF.ext）中，所以它们就能够在加载的时候被 libbpf 所使用。

> 感谢 **[builtin_preserve_access_index](https://clang.llvm.org/docs/LanguageExtensions.html#builtin-preserve-access-index)** 特性（用于 bpf_core_read() 帮助函数），这才能做到。当访问一个内核指针时，通过使用该关键字，这就是告知 LLVM 去 **将重定位信息保存到 ELF 文件中**，从而使得 **BPF 对象所在的内核可以知道如何去重定位符号（relocate the symbols）**。

举例：

```c
#pragma clang attribute push (__attribute__((preserve_access_index)), apply_to = record)

struct task_struct {
        pid_t pid;
        pid_t tgid;
}
```

如何使得 LLVM 去保存重定位信息：

```c
pid_t pid = __builtin_preserve_access_index(({ task->pid; }));
```

基本想法：告诉编译器需要通过帮助函数访问的类型和属性。它将使用 LLVM 内部特性去跟踪所有能够重定位（比如 BTF 信息）的信息，因此无论什么时候去加载生成的对象，都能够在运行代码之前处理重定位。

#### 3. 基于类型的重定位（Type Based Relocations）

> 在 BPF 程序众多需要处理的问题中最常见的是特性检测（feature detection）。譬如，检测宿主系统内核中是否支持新的、可选的特性，以此用来获取更多信息、或者提升效率。

基于类型的重定位就是 eBPF 程序用于发现更多运行环境信息的手段。有了基于类型的重定位，运行中的 eBPF 代码就可以做到：

- **BPF\_TYPE\_ID\_LOCAL** - 使用本地 BTF 信息去获取指定 BTF 类型的类型 ID。
- **BPF\_TYPE\_ID\_TARGET** - 使用目标 BTF 信息去获取指定 BTF 类型的类型 ID。
- **BPF\_TYPE\_EXISTS** - 检查所提供的命名类型（struct/union/enum/typedef）是否在目标当中。
- **BPF\_TYPE\_SIZE** - 在目标内核中获取所提供的命名类型（struct/union/enum/typedef）的字节大小。

> **BTF 生成器**目前还不支持这种类型的重定位。

#### 4. 基于枚举值的重定位（Enum Value Based Relocations）

> 一个有趣的挑战是，一些 BPF 程序需要运行在“不稳定的”内部内核枚举的内核中。这是因为这些枚举没有固定的常量而且/或者整数值。

基于枚举值的重定位就是 eBPF 程序用来（通过一个 eBPF 帮助函数）从所运行的内核中获取精确枚举值的手段。
> **BTF 生成器** 目前还不支持这种类型的重定位。

### eBPF 重定位相关的失败

> 在某些内核中，一些属性缺失的情况并非少见。如果 BPF 程序尝试使用 `BPF_CORE_READ()` 去读取缺失的属性，会导致一个 BPF 检验错误。类似地，当获取宿主系统内核中不存在的枚举（或者类型）的枚举值（或者类型大小）时，CO-RE 重定位会失败。

诚如前面所说，无论什么时候当重定位不能在加载期间（load time）完成时，[libbpf 将使用毒指令](https://nakryiko.com/posts/bpf-core-reference-guide/#guarding-potentially-failing-relocations) 去处理不支持的重定位。结果如下：

```
1: (85) call unknown#195896080
invalid func unknown#195896080
```

> 195896080 就是十六进制值 0xbad2310 （“bad relo”），是 libbpf 用来标记 CO-RE 重定位失败的指令的常量。libbpf 并不立即报告该错误，是因为缺失的字段、类型、枚举和对应失败的 CO-RE 重定位能够按需被 BPF 应用程序优雅地处理掉。这就使得仅仅单一一个 BPF 应用程序适应内核类型的急剧变化成为可能（这正是“一次编译，到处运行”的哲理所在）。

## BPF 重定位的推断

> BPF CO-RE 重定位经常用到两种 BTF 类型。一种是 BPF 程序本地所期望的类型定义（比如，vmlinux.h 中的类型、或者手工使用 `preserve_access_index` 属性定义的类型）。这种 **本地 BTF 类型** 使得 libbpf 可以 **知道在内核 BTF 中搜索什么**。因此，它仅仅是必要字段、枚举器集合的类型、字段、枚举的最小定义。

> 然后，libbpf 就能够使用 **本地 BTF 类型定义** 去 **找到完全匹配的内核 BTF 类型**。这就允许为 CO-RE 重定位相关的（本地/内核）两种类型找到对应的 BTF 类型 ID。对于在运行时去区分不同的内核/本地类型、对于调试和打日志的目的、甚至对于未来能接收 BTF 类型 ID 作为输入参数的 BPF API 而言，都是非常有用的。尽管这些 API 还不存在，但它们将会出现。

至此，我（原作者）将指出 libbpf 进行重定位的关键所在。这有助于理解 **BTF 生成器** 是如何使用 libbpf 已有能力去实现的。

当加载 eBPF 对象时，直至 CO-RE 重定位的执行路径是：

`bpf_prog_load_xattr()` or `bpf_object__load_skeleton()`
 - `bpf_object__load_xattr()`
        - `bpf_object__relocate()`
                - **`bpf_object__relocate_core()`**
                        - `bpf_core_apply_relo()`

`bpf_object__relocate_core()` 就是那关键函数，它遍历 `.BTF.ext` ELF 片段，并且应用在 eBPF 对象 BTF 数据中被 LLVM 编译器标记的重定位。

`bpf_core_relo` 使用的基本重定位信息是：

- `insn_off`：BPF 程序中指令 imm 属性所指的指令的（字节数量）偏移。
- `type_id`：可重定位的类型或字段所在的根实体类型的 BTF 类型 ID。
- `access_str_off`：（ELF 文件中）`.BTF` 字符串片段中的偏移（字符串的解释依赖于具体的重定位类别：基于属性的、基于类型的、基于枚举值的）。

**这些信息对 BTF 生成器很关键，所以简要讲解一下。**

`bpf_object__relocate_core()` 将一个一个地尝试执行 `.BTF.ext` 里的每一个重定位。它甚至使用循环去处理每一个 `.BTF.ext` CO-RE  重定位结构到给定指令的映射。

这就带给我们理解 **BTF  生成器** 的第二个最重要的函数（或一对函数）：`bpf_core_apply_relo()` 及其姊妹函数 `bpf_core_apply_relo_insn()`。两者都有参数 `bpf_core_relo` 结构体。

> 记住：本地 = 将加载进目标内核的 eBPF 对象 = 尝试装载 eBPF 的内核

第一个函数，`bpf_core_apply_relo()`，将获取 **本地类型 ID** 并为 **目标类型初始化用于给定重定位的缓存**，名为 `cand_cache`（candidates cache 候选缓存）。

第二个函数，`bpf_core_apply_relo_insn()`，将执行以下步骤：

1. 将 `bpf_core_relo` 转为低层次和高层次的推断，并命名为 `local_spec`。
2. 从 `cand_cache` 中检查每一个候选重定位，检查他们是否满足重定位的需要；如果满足，则生成一个低层次和高层次的目的推断，名为 `targ_spec`。

![local_specs and cand_specs](./eBPF%20CO-RE%20%E7%BB%88%E6%9E%81%E4%B8%80%E6%AD%A5.assets/image07-01.png)

1. 调用 `bpf_core_calc_relo()` 函数去计算给定 `local_spec` （本地推断）和 `targ_spec` （目标推断）的重定位。取决于需要处理的重定位类型，它将调用合适的处理函数：
        - `bpf_core_calc_field_relo()`
        - `bpf_core_calc_type_relo()`
        - `bpf_core_calc_enumval_relo()`
        并且将结果保存到名为 `targ_res` 的结构体中，该结构体中包含了 _本地和目标推断_ 的目标推断结果。2 个 `bpf_core_relo` 构成的 `bpf_core_relo_res` 中包含了以下值：
        i. 指令中的原始值（期望值）
        ii. 用于打补丁的新值
        iii. 警告有（由于某些错误）投毒的重定位的布尔值
        iv. 类型 ID 的原始大小
        v. 原始类型
        vi. 新类型 ID 的新大小
        vii. 新类型 ID


![local_spec + targ_spec = targ_res](./eBPF%20CO-RE%20%E7%BB%88%E6%9E%81%E4%B8%80%E6%AD%A5.assets/image07-02.png)

1. 给指定 `local_spec` 和 `targ_res` 信息的重定位打上指令补丁（`bpf_core_patch_insn()`）。

## BTF 生成器及其工作原理

至此希望读者有个清晰的认知：

- eBPF CO-RE 和 BTF 信息是如何生成并使用的
- BTF 信息模型：类型之间是怎么互相链接的
- eBPF 重定位类别：基于属性的、基于类型的、和基于枚举值的
- 基于本地的（eBPF 对象的）和目标（内核）的 BTF 信息的 eBPF 重定位推断

现在，我们暂停一下，重新思考一下 **BTFHUB** 及其最大的问题：大小！

主要想法是：

1. 你有一个 eBPF 程序，它能够跑在多个近期的内核中（自带 BTF 和重定位信息）。
2. 你有 **BTFHUB**，它带有非常多老旧内核的 BTF 文件（目标 BTF）。

为什么不“过滤出”那些 eBPF 对象使用的类型，且仅包含 **目标 BTF** 所需要的类型呢？这样依赖，为了完全支持老旧内核，你并不需要每个老旧内核的 1.5MB 文件。

当 **Mauricio (Kinvolk/Microsoft)** 告诉我们他的 POC（Proof Of Concept）代码时，他已经解决了我们一开始就遇到的困难。我们的主要意愿是得到一个 **目标文件**，并使其很小且仅包含我们 eBPF 对象所使用的类型。

不幸的是，想要简单地从本地的（eBPF 对象自带的）BTF 获取已有的 BTF 类型信息，并尝试用来生成目标 BTF，并不可行。需要先计算出重定位，并使用 **那些重定位结果** 去生成目标 BTF。（当运行在没带有 BTF 的老旧内核时，）需要确保所需要的所有类型信息都是存在的。

总而言之，接收一堆不同内核不同 BTF 文件、和一堆不同 eBPF 应用不同 eBPF 对象文件作为参数，**BTF  生成器负责**：

1. 计算所有的 eBPF 对象文件到每个内核 BTF 文件的重定位信息
2. 为每个给定的内核生成仅包含一个 eBPF 对象所使用类型信息的局部（partial）BTF 文件（以此便能将一堆 BTF 文件和一个应用进行分发，使得它能支持多个发行商的所有老旧内核）。

**BTF 生成器是如何做到的呢？**

- [libbpf 补丁](https://github.com/kinvolk/libbpf/tree/btfgen) （基于重定位）生成特定 BTF
- [bpftool 补丁](https://github.com/kinvolk/btfgen) 生成 BTF 文件

### 生成 BTF 文件时需要什么

Libbpf 已提供便捷处理 BTF 信息的函数。基于前面的图片，如果你想知道 BTF 类型是如何组织的，你就会看到生成一个 BTF 文件就是创建一个空白 BTF（`btf__new_empty()`）信息结构体并往其中添加 BTF 类型信息的问题。

存在多种方式往 BTF 结构体添加 BTF 类型。不同类型使用不同函数：

- `btf__add_int()`
- `btf__add_float()`
- `btf__add_ref_kind()` (PTR, TYPEDEF, CONST/VOLATILE/RESTRICT)
- `btf__add_ptr()`
- `btf__add_array()`
- `btf__add_composite()` (STRUCT/UNION by providing existing fields)
- `btf__add_struct()` (STRUCT with no fields)
- `btf__add_union()` (UNION with no fields)
- `btf__add_enum()` (ENUM with no enum values)
- `btf__add_fwd()` (FWD declaration to STRUCT, UNION or ENUM)
- `btf__add_typedef()`
- `btf__add_volatile()`
- `btf__add_const()`
- `btf__add_restrict()`
- `btf__add_func()`
- `btf__add_func_proto()` (FUNCTION prototype with no arguments)
- `btf__add_var()`
- `btf__add_datasec()`

并辅以以下填充函数：

- `btf__add_field()` (STRUCT/UNION new field)
- `btf__add_enum_value()` (ENUM new value)
- `btf__add_func_param()` (FUNCTION arguments)
- `btf__add_datasec_var_info()`

于此之外，还有一种通用（generic）的方式：

- `btf__add_type()`

### 如何使用 libbpf  CO-RE 重定位去生成 BTF 文件

诚如前面所说，我们需要构建一个仅包含从 eBPF 重定位分析得到的类型的 BTF 文件。而在 libbpf 将重定位应用到指令之前，我们已能够拿到所有我们所需要的东西：

- `bpf_core_relo`：表示一个从原始 eBPF 对象得到的重定位，即 `local_spec`
- `bpf_core_relo`：表示一个匹配重定位类型的候选类型，即 `targ_spec`
- `bpf_core_relo_res`：表示这两个 `bpf_core_relo` 的推断

我们所需要做的是按照类型、成员属性去遍历重定位（walk the relocation），遍历时将发现的类型、成员属性添加到内存中的 BTF 文件。通过这种方式，最终我们得到的 BTF 文件将是原先巨大无比的 BTF 文件的最小子集。这正是我们解决 BTFHUB 大小问题的方法。

因此，就让我们遍历所有的重定位。对于每个重定位，遍历重定位中的所有 BTF 类型。接着就让我们去深究 **BTF 生成器** 的实现原理吧。

通过带有调试信息的 **BTF 生成器**，当使用外部的 （Ubuntu Bionic）5.4.0-87 内核
BTF 文件给一个复杂的 eBPF 对象
（[Tracee](https://github.com/aquasecurity/tracee/blob/main/tracee-ebpf/tracee/tracee.bpf.c)）
修剪 BTF 文件的时候，我们就可以看到该 eBPF 对象中用于运行在 5.4.0-87 内核的所依赖的重定位信息：

```txt
RELOCATION: [26219] struct bpf_raw_tracepoint_args.args[1] (0:0:1 @ offset 8)
RELOCATION: [28233] struct task_struct.real_parent (0:68 @ offset 2256)
RELOCATION: [28233] struct task_struct.pid (0:65 @ offset 2240)
RELOCATION: [28233] struct task_struct.nsproxy (0:105 @ offset 2760)
RELOCATION: [28421] struct nsproxy.pid_ns_for_children (0:4 @ offset 32)
RELOCATION: [28409] struct pid_namespace.level (0:6 @ offset 72)
RELOCATION: [28233] struct task_struct.thread_pid (0:75 @ offset 2344)
RELOCATION: [28411] struct pid.numbers (0:5 @ offset 80)
RELOCATION: [28408] struct upid.nr (0:0 @ offset 0)
RELOCATION: [28421] struct nsproxy.pid_ns_for_children (0:4 @ offset 32)
...
RELOCATION: [198] struct pt_regs.di (0:14 @ offset 112)
RELOCATION: [198] struct pt_regs.si (0:13 @ offset 104)
RELOCATION: [3484] struct socket.sk (0:4 @ offset 24)
RELOCATION: [3012] struct sock.__sk_common.skc_family (0:0:3 @ offset 16)
RELOCATION: [47942] struct inet_sock.sk.__sk_common.skc_rcv_saddr (0:0:0:0:1:1 @ offset 4)
RELOCATION: [47942] struct inet_sock.sk.__sk_common.skc_num (0:0:0:2:1:1 @ offset 14)
RELOCATION: [47942] struct inet_sock.sk.__sk_common.skc_daddr (0:0:0:0:1:0 @ offset 0)
RELOCATION: [47942] struct inet_sock.sk.__sk_common.skc_dport (0:0:0:2:1:0 @ offset 12)
RELOCATION: [47875] struct sockaddr_in.sin_family (0:0 @ offset 0)
RELOCATION: [47875] struct sockaddr_in.sin_port (0:1 @ offset 2)
RELOCATION: [47875] struct sockaddr_in.sin_addr.s_addr (0:2:0 @ offset 4)
RELOCATION: [49718] struct unix_sock.addr (0:1 @ offset 760)
RELOCATION: [49716] struct unix_address.len (0:1 @ offset 4)
RELOCATION: [49716] struct unix_address.name (0:3 @ offset 12)
RELOCATION: [3012] struct sock.__sk_common.skc_state (0:0:4 @ offset 18)
RELOCATION: [47942] struct inet_sock.pinet6 (0:1 @ offset 760)
...
```

选择其中一个重定位为例：

```txt
RELOCATION: [47942] struct inet_sock.sk.__sk_common.skc_num (0:0:0:2:1:1 @ offset 14)
```

有它出现是因为 `struct inet_sock`。这就是此重定位的 **根实体**。其余所有的成员或属性皆是从此重定位而来。此 `struct inet_sock` 是一个 ID 为 47942 的 `BTF_TYPE`。

执行 bpftool 我们可以看到：

```Bash
$ bpftool btf dump file ./btfs/5.4.0-87-generic.btf format raw
...
[47942] STRUCT 'inet_sock' size=968 vlen=30
        'sk' type_id=3012 bits_offset=0
        'pinet6' type_id=47944 bits_offset=6080
        'inet_saddr' type_id=2996 bits_offset=6144
        'uc_ttl' type_id=16 bits_offset=6176
        'cmsg_flags' type_id=18 bits_offset=6192
        'inet_sport' type_id=2995 bits_offset=6208
        'inet_id' type_id=18 bits_offset=6224
        'inet_opt' type_id=47938 bits_offset=6272
        'rx_dst_ifindex' type_id=21 bits_offset=6336
        'tos' type_id=13 bits_offset=6368
        'min_ttl' type_id=13 bits_offset=6376
        'mc_ttl' type_id=13 bits_offset=6384
        'pmtudisc' type_id=13 bits_offset=6392
        'recverr' type_id=13 bits_offset=6400 bitfield_size=1
        'is_icsk' type_id=13 bits_offset=6401 bitfield_size=1
        'freebind' type_id=13 bits_offset=6402 bitfield_size=1
        'hdrincl' type_id=13 bits_offset=6403 bitfield_size=1
        'mc_loop' type_id=13 bits_offset=6404 bitfield_size=1
        'transparent' type_id=13 bits_offset=6405 bitfield_size=1
        'mc_all' type_id=13 bits_offset=6406 bitfield_size=1
        'nodefrag' type_id=13 bits_offset=6407 bitfield_size=1
        'bind_address_no_port' type_id=13 bits_offset=6408 bitfield_size=1
        'defer_connect' type_id=13 bits_offset=6409 bitfield_size=1
        'rcv_tos' type_id=13 bits_offset=6416
        'convert_csum' type_id=13 bits_offset=6424
        'uc_index' type_id=21 bits_offset=6432
        'mc_index' type_id=21 bits_offset=6464
        'mc_addr' type_id=2996 bits_offset=6496
        'mc_list' type_id=47945 bits_offset=6528
        'cork' type_id=47941 bits_offset=6592
...
```

此时你该认识到外部的带有内核镜像所有类型的 BTF 文件 `5.4.0-87-generic.btf` 是多么的巨大。我们继续。在此重定位中 `sk` 是 `struct inet_sock` 的成员。在 BTF 输出中，我们可以找到成员 `sk` 及其指向的 BTF\_TYPE ID。

```txt
        'sk' type_id=3012 bits_offset=0
```

继续，现在我们必须找到 ID 为 3012 的 BTF 类型：

```txt
[3012] STRUCT 'sock' size=760 vlen=88
        '__sk_common' type_id=4507 bits_offset=0
        'sk_lock' type_id=4492 bits_offset=1088
        'sk_drops' type_id=79 bits_offset=1344
        'sk_rcvlowat' type_id=21 bits_offset=1376
        'sk_error_queue' type_id=3582 bits_offset=1408
        'sk_rx_skb_cache' type_id=3247 bits_offset=1600
        'sk_receive_queue' type_id=3582 bits_offset=1664
        'sk_backlog' type_id=4510 bits_offset=1856
        'sk_forward_alloc' type_id=21 bits_offset=2048
        'sk_ll_usec' type_id=9 bits_offset=2080
        'sk_napi_id' type_id=9 bits_offset=2112
        'sk_rcvbuf' type_id=21 bits_offset=2144
        'sk_filter' type_id=4514 bits_offset=2176
        '(anon)' type_id=4511 bits_offset=2240
        'sk_policy' type_id=4515 bits_offset=2304
        'sk_rx_dst' type_id=3382 bits_offset=2432
        'sk_dst_cache' type_id=3382 bits_offset=2496
        'sk_omem_alloc' type_id=79 bits_offset=2560
        'sk_sndbuf' type_id=21 bits_offset=2592
        'sk_wmem_queued' type_id=21 bits_offset=2624
        'sk_wmem_alloc' type_id=790 bits_offset=2656
...
```

它是一个 `struct sock`。所以，如今我们找到了一个重定位 `struct inet_sock`-\>`struct sock`-\>XXX。现在，此重定位的第 3 个成员是 `__sk_common`，且它刚好出现在刚才 `struct sock` 的 BTF 信息中：

```txt
        '__sk_common' type_id=4507 bits_offset=0
```

ID 为 4507 的 BTF 类型是：

```txt
[4507] STRUCT 'sock_common' size=136 vlen=25
        '(anon)' type_id=4496 bits_offset=0
        '(anon)' type_id=4497 bits_offset=64
        '(anon)' type_id=4500 bits_offset=96
        'skc_family' type_id=19 bits_offset=128
        'skc_state' type_id=2991 bits_offset=144
        'skc_reuse' type_id=14 bits_offset=152 bitfield_size=4
        'skc_reuseport' type_id=14 bits_offset=156 bitfield_size=1
        'skc_ipv6only' type_id=14 bits_offset=157 bitfield_size=1
        'skc_net_refcnt' type_id=14 bits_offset=158 bitfield_size=1
        'skc_bound_dev_if' type_id=21 bits_offset=160
        '(anon)' type_id=4501 bits_offset=192
        'skc_prot' type_id=4509 bits_offset=320
        'skc_net' type_id=3613 bits_offset=384
        'skc_v6_daddr' type_id=3289 bits_offset=448
        'skc_v6_rcv_saddr' type_id=3289 bits_offset=576
        'skc_cookie' type_id=81 bits_offset=704
        '(anon)' type_id=4502 bits_offset=768
        'skc_dontcopy_begin' type_id=106 bits_offset=832
        '(anon)' type_id=4504 bits_offset=832
        'skc_tx_queue_mapping' type_id=19 bits_offset=960
        'skc_rx_queue_mapping' type_id=19 bits_offset=976
        '(anon)' type_id=4505 bits_offset=992
        'skc_refcnt' type_id=790 bits_offset=1024
        'skc_dontcopy_end' type_id=106 bits_offset=1056
        '(anon)' type_id=4506 bits_offset=1056
```

至此我们的重定位 `inet_sock.sk.__sk_common.skc_num (0:0:0:2:1:1 @ offset 14) ` 都只有结构体成员（structs as members）。下一个字段 `skc_num` 则是个意外，它不是 4507 类型中的成员属性。这是因为它的类型是隐匿的（anonymous type）。是时候关注一下那些字段，而不仅仅是成员类型了。在此重定位中，

- `inet_sock.sk.__sk_common.skc_num (0:0:0:**2:1:1** @ offset 14)`

在此例子中，其中结尾的数字告诉我们，如果它们没有名字的时候该使用什么字段。第 4 个成员有字段 #2：

```txt
        '(anon)' type_id=4500 bits_offset=96
```

查一下 BTF 类型 ID 所指：

```txt
[4500] UNION '(anon)' size=4 vlen=2
        'skc_portpair' type_id=4493 bits_offset=0
        '(anon)' type_id=4499 bits_offset=0
```

我们还需要知道第 5 个成员字段 #1 所依赖的重定位：

```txt
        '(anon)' type_id=4499 bits_offset=0
```

继续查看 ID 为 4499 的 BTF 类型，同时也是第 6 个成员字段 #1:

```txt
[4499] STRUCT '(anon)' size=4 vlen=2
        'skc_dport' type_id=2995 bits_offset=0
        'skc_num' type_id=18 bits_offset=16
```

我们终于找到名为 `skb_num` 的成员。

因此，如上所示，对于每个指定的重定位，我们必须使用其中所包含的类型、根实体、字段等信息去用目标重定位合成另一个 BTF 文件。这另一个 BTF 文件则是所指定的 5.4.0-87 内核 BTF 文件的子集，但它仅带有我们所需的 BTF 类型。通过这种方式，我们可以将此生成的 BTF 当作外部 BTF 文件，提供给 libbpf 去将我们的 eBPF 程序加载进 5.4.0-87 内核中；这是一个非常小的 BTF 文件，去替代那个大的。当然，此 BTF 文件仅为此 eBPF 对象而生，不适用于其它 eBPF 对象。想法如下：为你的 eBPF 应用生成一堆所需要支持的内核所对应的 BTF 文件，并内置于应用程序中；这样依赖，你的代码就能到处运行了（这就是 CO-**RE** 中的 **到处运行**）。

### 如何组织我们所需的数据

**BTF 生成器** 使用三个结构体去组织信息：`btf_reloc_info`、`btf_reloc_type` 和 `btf_reloc_member`。也使用已存在的 `btf_type` 和 `btf_member` 结构体。

看下是如何组织数据的：

![BTF reloc result](./eBPF%20CO-RE%20%E7%BB%88%E6%9E%81%E4%B8%80%E6%AD%A5.assets/image08.png)

生成了一个 `BTF_RELOC_INFO` 结构体：

- `SRC_BTF`：指向源（eBPF 对象）BTF 文件的指针
- `TYPES`：包含所有 `BTF_RELOC_TYPE` 结构体的 hashmap
- `IDS_MAP`：包含 **新类型 ID** 值到已有 **旧类型 ID** 值的 hashmap

专注 `BPF_RELO_TYPE`，它包含：

- `BTF_TYPE`：指向重定位根实体的 BTF 类型的指针（如前面例子中的 `inet_sock`）。它可以是任意 BTF 类型类别（结构体/联合体、整数、浮点数、指针等等）。
- `ID`：重定位根实体的 BTF 类型 ID（如前面例子中的 47942）。
- `MEMBERS`：包含所有 `BTF_RELOC_MEMBERS` 的 hashmap（一一保存 BTF 类型成员）

所以，如果 `BTF_RELOC_TYPE` 是一个联合体或者是一个结构体，我们将用 `BTF_RELOC_MEMBER` 保存重定位中的已有成员。`BTF_RELOC_MEMBER` 包括：

- 指向该重定位中表示该成员的 `BTF_MEMBER` 结构体指针。

最终我们得到：所有重定位都解决了，因此该重定位中每个字段、成员对应的类型组成了最终 BTF 文件中的一个新类型。每个类型都拥有它自己的 `BTF_RELOC_TYPE` 结构体；该结构体中包含、或不包含 `BTF_RELOC_MEMBER`。

**根实体类型 =\> 成员** 的映射关系非常重要，因为它提供给我们最终的 BTF 图。如果这些类型足够简单，就没有成员、其它结构体添加到根实体 BTF 类型后面。如果类型很复杂，我们就必须从头开始添加每个类型，接着为每一个复杂的类型（比如结构体和联合体）一个一个地添加字段/成员。

庆幸的是， libbpf 允许我们（通过 `btf__add_type()`）增加一个有诸多成员的复杂 BTF 类型。这就是函数 `bpf_reloc_info__get_btf()` 在 **第一阶段** 所干的事情：遍历 `BTF_RELOC_INFO` 结构体中的所有类型。

不幸的是，无论何时新增一个 BTF 类型到新 BTF 文件时它都会生成一个新的 BTF 类型 ID。这意味着（根实体）BTF 类型和已有字段、成员之间的联系断开了。请查看：

```Bash
$ bpftool btf dump file ./generated/5.4.0-87-generic.btf format raw

[1] PTR '(anon)' type_id=28278
[2] TYPEDEF 'u32' type_id=23
[3] TYPEDEF '__be16' type_id=18
[4] PTR '(anon)' type_id=47943
[5] TYPEDEF '__u8' type_id=14
[6] PTR '(anon)' type_id=49716
[7] STRUCT 'mnt_namespace' size=120 vlen=1
        'ns' type_id=1949 bits_offset=64
[8] TYPEDEF '__kernel_gid32_t' type_id=9
[9] STRUCT 'iovec' size=16 vlen=2
        'iov_base' type_id=103 bits_offset=0
        'iov_len' type_id=48 bits_offset=64
[10] PTR '(anon)' type_id=28881
[11] STRUCT '(anon)' size=8 vlen=2
        'skc_daddr' type_id=2996 bits_offset=0
        'skc_rcv_saddr' type_id=2996 bits_offset=32
[12] TYPEDEF '__u64' type_id=27
[13] PTR '(anon)' type_id=36422
[14] TYPEDEF 'pid_t' type_id=45
[15] PTR '(anon)' type_id=28284
[16] PTR '(anon)' type_id=8
...
```

所以，尽管找到了我们 eBPF 对象所使用的所有 BTF 类型，指向不存在的类型的复杂类型，比如：

```
        'skc_daddr' type_id=2996 bits_offset=0
```

在生成的 BTF 文件中并不存在 ID 为 2996 的 BTF 类型。这就是原因，为什么在数据组织图中你能找到新旧类型 ID 的 hashmap。所有被引用的 BTF 类型 ID 都使用这个新旧类型 ID 的 hashmap 就行修复；而这就由函数 `bpf_reloc_info__get_btf()` 的 **第二阶段** 完成。

让我们看看在这 **第二阶段** 后所生成文件的样子：

```
[1] PTR '(anon)' type_id=97
[2] TYPEDEF 'u32' type_id=35
[3] TYPEDEF '__be16' type_id=22
[4] PTR '(anon)' type_id=51
[5] TYPEDEF '__u8' type_id=82
[6] PTR '(anon)' type_id=29
[7] STRUCT 'mnt_namespace' size=120 vlen=1
        'ns' type_id=71 bits_offset=64
[8] TYPEDEF '__kernel_gid32_t' type_id=74
[9] STRUCT 'iovec' size=16 vlen=2
        'iov_base' type_id=16 bits_offset=0
        'iov_len' type_id=84 bits_offset=64
[10] PTR '(anon)' type_id=57
[11] STRUCT '(anon)' size=8 vlen=2
        'skc_daddr' type_id=80 bits_offset=0
        'skc_rcv_saddr' type_id=80 bits_offset=32
[12] TYPEDEF '__u64' type_id=87
[13] PTR '(anon)' type_id=7
[14] TYPEDEF 'pid_t' type_id=104
[15] PTR '(anon)' type_id=62
[16] PTR '(anon)' type_id=0
...
```

你可以辨别出同样的例子已经被修复而后指向正确的 BTF 类型 ID：

```
        'skc_daddr' type_id=80 bits_offset=0
```

ID 为 80 的 BTF 类型是：

```
[80] TYPEDEF '__be32' type_id=35
```

而它指向 ID 为 35 的 BTF 类型：

```
[35] TYPEDEF '__u32' type_id=74
```

指向 ID 为 74 的 BTF 类型：

```
[74] INT 'unsigned int' size=4 bits_offset=0 nr_bits=32 encoding=(none)
```

这是一个简单类型，且无需指向其它地方。

## 是时候检验 BTFGEN 和 BTFHUB 了

读者可以选择不同的方式去使用 BTFHUB：

1. 下载特定内核版本的外部 BTF 文件，并在使用 libbpf 加载你的应用时当作外部 BTF 文件。比如：
        - [tracee-ebpf](https://github.com/aquasecurity/tracee/blob/main/tracee-ebpf/main.go#L155)
        - [inspektor-gadget](https://github.com/kinvolk/inspektor-gadget/commit/f7a807e86ea90d4df1bfea013139381c07aab6d2)
        对于每个不同的内核，你都需要从 BTFHUB 中下载对应的 BTF 文件。为不支持 BTF 的内核去做 CO-RE，这就是老旧的、问题多多的方式了。
2. 将 **BTFHUB** 仓库克隆下来，并使用 **btfgen** 去生成 *你的应用所需要支持的* 内核对应的 **BTF 文件**。比如：
        ```
        [user@host:~/.../aquasec-btfhub/tools][main]$ ./btfgen.sh ~/aquasec-tracee/tracee-ebpf/dist/tracee.bpf.core.o
        ```

如果你打开 [BTFHUB](https://github.com/aquasecurity/btfhub) 里的 [tools](https://github.com/aquasecurity/btfhub/tree/main/tools) 目录，你将看到如何使用已经编译好的 **btfgen**（即 **BTF 生成器**）的指令。

[btfgen.sh](https://github.com/aquasecurity/btfhub/blob/main/tools/btfgen.sh) 脚本将在目录 `tools/output/{centos,fedora,ubuntu}/` 下生成近包含指定 eBPF 对象（本例中的 `tracee.bpf.core.o`）中所使用类型的 **多个 BTF 文件和符号链接**，其中每个特定内核版本的重定位皆已重新计算过了。

然后你就可以通过加载所运行的内核所对应的部分生成的 BTF 文件去执行你的应用。

举例：

```Bash
$ uname -a
Linux bionic 5.4.0-87-generic #98~18.04.1-Ubuntu SMP Wed Sep 22 10:45:04 UTC 2021 x86_64 x86_64 x86_64 GNU/Linux

$ lsb_release -a
No LSB modules are available.
Distributor ID: Ubuntu
Description:    Ubuntu 18.04.6 LTS
Release:        18.04
Codename:        bionic

$ bpftool btf dump file ./generated/5.4.0-87-generic.btf format raw | grep "^\[" | wc -l
122

[user@host:~/.../aquasec-tracee/tracee-ebpf]$ sudo TRACEE_BTF_FILE=~/aquasec-btfhub/tools/output/ubuntu/18.04/x86_64/5.4.0-87-generic.btf ./dist/tracee-ebpf --debug --trace event=execve,execveat,uname

OSInfo: VERSION: "18.04.6 LTS (Bionic Beaver)"
OSInfo: ID: ubuntu
OSInfo: ID_LIKE: debian
OSInfo: PRETTY_NAME: "Ubuntu 18.04.6 LTS"
OSInfo: VERSION_ID: "18.04"
OSInfo: VERSION_CODENAME: bionic
OSInfo: KERNEL_RELEASE: 5.4.0-87-generic
BTF: bpfenv = false, btfenv = true, vmlinux = false
BPF: using embedded BPF object
BTF: using BTF file from environment: ~/aquasec-btfhub/tools/output/ubuntu/18.04/x86_64/5.4.0-87-generic.btf
unpacked CO:RE bpf object file into memory

TIME             UID    COMM             PID     TID     RET              EVENT                ARGS
05:08:45:175699  1000   bash             5176    5176    0                execve               pathname: /bin/ls, argv: [ls --color=auto]
05:08:45:188780  1000   bash             5180    5180    0                execve               pathname: /usr/bin/git, argv: [git branch]
05:08:45:189986  1000   bash             5181    5181    0                execve               pathname: /bin/sed, argv: [sed -e /^[^*]/d -e s/* \(.*\)/\1/]
05:08:45:971635  1000   bash             5183    5183    0                execve               pathname: /bin/ps, argv: [ps]
05:08:46:015186  1000   bash             5186    5186    0                execve               pathname: /usr/bin/git, argv: [git branch]
05:08:46:015415  1000   bash             5187    5187    0                execve               pathname: /bin/sed, argv: [sed -e /^[^*]/d -e s/* \(.*\)/\1/]

End of events stream
Stats: {EventCount:6 ErrorCount:0 LostEvCount:0 LostWrCount:0 LostNtCount:0}
```

## 更多文档

这里有你所需要的更多关于 eBPF CO-RE 的信息源：

1. [https://ebpf.io/what-is-ebpf](https://ebpf.io/what-is-ebpf)
2. [https://github.com/libbpf/libbpf](https://github.com/libbpf/libbpf)
3. [https://nakryiko.com/posts/bpf-portability-and-co-re/](https://nakryiko.com/posts/bpf-portability-and-co-re/)
4. [https://nakryiko.com/posts/bpf-portability-and-co-re/#btf](https://nakryiko.com/posts/bpf-portability-and-co-re/#btf)

值得一读的其它链接：

1. [Introduction to eBPF](https://ebpf.io/what-is-ebpf#introduction-to-ebpf) (from: ebpf.io/what-is-ebpf)
2. [Development Toolchains](https://ebpf.io/what-is-ebpf#development-toolchains) (from: ebpf.io/what-is-ebpf)
3. [BCC to libbpf conversion guide](https://nakryiko.com/posts/bcc-to-libbpf-howto-guide/) (if you're coming from BCC)
4. [Building BPF applications with libbpf-bootstrap](https://nakryiko.com/posts/libbpf-bootstrap/)
5. [BPF Design FAQ](https://01.org/linuxgraphics/gfx-btfgen-internals/drm/bpf/bpf_design_QA.html)
6. [eBPF features per kernel version](https://github.com/iovisor/bcc/blob/master/docs/kernel-versions.md#program-types)
7. [BTFHUB code example](https://github.com/aquasecurity/btfhub/tree/main/example)
8. [BCC's libbpf-tools directory](https://github.com/iovisor/bcc/tree/master/libbpf-tools)

现在，一些关于格式、eBPF 和 BTF 内部实现的链接：

1. [ELF - Executable and Linkable Format](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format)
2. [BPF Type Format](https://www.kernel.org/doc/html/latest/bpf/btf.html)
3. [BTF deduplication and Linux Kernel BTF](https://nakryiko.com/posts/btf-dedup/)
4. [BPF LLVM Relocations](https://www.kernel.org/doc/html/latest/bpf/llvm_reloc.html)

---

关键单词的翻译：

- relocation：重定位
- section：在 ELF 上下文中翻译为 ELF 片段
- field：属性，字段
- kind：类别
- instruction：指令
- speculation：推断
- load：加载、装载
- poisoned：投毒的，本文中特指为未知重定位而专门设置某个特定值的行为
