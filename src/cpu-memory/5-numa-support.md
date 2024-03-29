## 5 NUMA 支持

在第二部分中，我们看到在某些机器上，访问物理内存特定区域的成本取决于访问的来源。这种类型的硬件需要操作系统和应用程序特别注意。我们将从 NUMA 硬件的一些细节开始，然后再介绍 Linux 内核为 NUMA 提供的一些支持。

### 5.1 NUMA Hardware

非均匀内存体系结构越来越普遍。在最简单的 NUMA 形式中，处理器可以具有本地内存（参见图2.3），其访问成本比其他处理器本地内存要便宜。这种类型的 NUMA 系统的成本差异并不高，即 NUMA 因子很低。

NUMA 还特别用于大型机器。我们已经描述了许多处理器访问同一内存所带来的问题。对于商用硬件，所有处理器都将共享同一个北桥（暂不考虑 AMD Opteron NUMA 节点，它们有自己的问题）。这使得北桥成为一个严重的瓶颈，因为所有内存流量都通过它进行路由。当然，大型机器可以使用自定义硬件代替北桥，但是，除非使用的内存芯片具有多个端口，即它们可以从多个总线中使用，否则仍然存在瓶颈。多端口 RAM 很复杂，成本很高，因此几乎不会使用。

复杂度的下一个增加是 AMD 使用的模型，其中一种互连机制（在 AMD 的情况下是 Hypertransport，这是他们从 Digital 获得许可的技术）为未直接连接到 RAM 的处理器提供访问。这种方式可以形成的结构的大小受到限制，除非想任意增加直径（即任意两个节点之间的最大距离）。

> ![img](assets/cpumemory.20.png)
>
> **Figure 5.1: Hypercubes**

高效的节点拓扑结构是超立方体，它将节点数限制为2C，其中C是每个节点的互连接口数。对于所有带有2n个CPU的系统，超立方体具有最小的直径。图5.1显示了前三个超立方体。每个超立方体的直径为C，这是绝对最小的。AMD的第一代Opteron处理器每个处理器有三个HyperTransport连接。至少有一个处理器必须连接一个Southbridge，这意味着目前可以直接且高效地实现C=2的超立方体。下一代处理器宣布将有四个连接，到那时C=3的超立方体将成为可能。

但这并不意味着不支持更大的处理器积累。一些公司已经开发了交叉开关，允许使用更大的处理器集合（例如Newisys的Horus）。但是这些交叉开关会增加NUMA因素，并且在一定数量的处理器上不再有效。

下一步是连接CPU组并为它们实现共享内存。所有这样的系统都需要专用硬件，绝不是商品系统。这样的设计存在多个复杂度级别。一个仍然非常接近商品机的系统是IBM x445和类似的机器。它们可以作为普通的4U，8路机器购买，带有x86和x86-64处理器。然后可以将两个（在某些时候最多四个）这样的机器连接起来，以共享内存的方式作为一个单一的机器。所使用的互连引入了一个显著的NUMA因素，这是操作系统以及应用程序必须考虑的。

在另一端的系统中，像SGI的Altix就是专门设计用于互连。SGI的NUMAlink互连结构非常快且延迟低，这些都是高性能计算（HPC）的要求，特别是当使用消息传递接口（MPI）时。缺点当然是这种复杂性和专业化非常昂贵。它们可以实现相对较低的NUMA因素，但由于这些机器可以拥有数千个CPU并且互连的容量有限，NUMA因素实际上是动态的，取决于工作负载而可能达到不可接受的水平。

更常见的解决方案是使用高速网络连接商品机集群。但这些不是NUMA机器；它们不实现共享地址空间，因此不属于本文讨论的任何类别。

### 5.2 OS Support for NUMA

为了支持NUMA机器，操作系统必须考虑内存的分布性质。例如，如果在给定的处理器上运行一个进程，则分配给该进程地址空间的物理RAM应来自本地内存。否则，每个指令都必须访问远程内存以获取代码和数据。在NUMA机器中存在需要考虑的特殊情况。DSO的文本段通常仅存在于一台机器的物理RAM中。但是，如果DSO由所有CPU上的进程和线程使用（例如基本运行库如libc），这意味着除了少数处理器外，其他处理器都必须具有远程访问。理想情况下，操作系统应该将这些DSO“镜像”到每个处理器的物理RAM中，并使用本地副本。这是一种优化而不是要求，并且通常难以实现。它可能不被支持或仅以有限的方式支持。

为避免情况恶化，操作系统不应将进程或线程从一个节点迁移至另一个节点。在普通的多处理器机器上，操作系统已经尝试避免迁移进程，因为从一个处理器迁移到另一个处理器意味着缓存内容会丢失。如果负载分配需要将进程或线程从处理器迁移，操作系统通常可以选择具有足够剩余容量的任意新处理器。在NUMA环境中，选择新处理器的范围略有限制。新选择的处理器不应该比旧处理器更高地访问进程正在使用的内存，这限制了可选目标列表。如果没有可用满足该标准的空闲处理器，则操作系统别无选择，只能迁移到访问内存更昂贵的处理器。

在这种情况下，有两种可能的方法。首先，可以希望情况是暂时的，并且进程可以迁回到更合适的处理器。或者，操作系统也可以将进程的内存迁移到更靠近新使用的处理器的物理页面。这是一项非常昂贵的操作。可能需要复制大量的内存，尽管不一定在一步完成。在此过程中，进程至少要被暂停，以便正确迁移旧页面的修改。还有一系列其他要求，以使页面迁移高效快速。简而言之，除非确实需要，否则操作系统应该避免进行此操作。

通常情况下，Linux内核不会假设所有在NUMA机器上运行的进程都使用相同数量的内存，因此在进程分布在不同处理器上的情况下，内存使用情况也是不平衡的。实际上，除非在HPC领域等特定应用中，运行在机器上的应用程序所使用的内存将会非常不平衡。一些应用程序会使用大量的内存，而其他应用程序则几乎不使用内存。如果总是将内存分配到发出请求的处理器上，这将最终导致问题。当运行大型进程的节点内存不足时，系统将出现问题。

为了应对这些严重的问题，默认情况下不会仅在本地节点上分配内存。为了利用系统的所有内存， 默认策略是将内存分割成多个条带。这可以保证所有系统内存的使用是均衡的。副作用是可以在处理器之间自由迁移进程，因为平均而言，所有使用的内存的访问成本都不会改变。对于小的NUMA因子，分条是可以接受但仍不是最优的（请参见第5.4节中的数据）。

这是一种负优化，有助于系统避免严重问题并使其在正常操作下更可预测。但是，它确实降低了整个系统的性能，在某些情况下甚至会显著降低。这就是为什么Linux允许每个进程选择内存分配规则。进程可以为自身及其子进程选择不同的策略。我们将在第6节中介绍可以用于此的接口。

### 5.3 Published Information

内核通过 sys 伪文件系统（sysfs）发布有关处理器缓存的信息，其路径如下：

```
/sys/devices/system/cpu/cpu*/cache
```

在第6.2.1节中，我们将了解可用于查询各种缓存大小的接口。这里重要的是缓存的拓扑结构。上面的目录包含子目录（命名为 index *），列出 CPU 拥有的各种缓存的信息。对于拓扑结构而言，文件 type、level 和 shared_cpu_map 是这些目录中的重要文件。对于 Intel Core 2 QX6700，其信息如表5.1所示。



> |        | type   | level       | shared_cpu_map |          |
> | ------ | ------ | ----------- | -------------- | -------- |
> | `cpu0` | index0 | Data        | 1              | 00000001 |
> |        | index1 | Instruction | 1              | 00000001 |
> |        | index2 | Unified     | 2              | 00000003 |
> | `cpu1` | index0 | Data        | 1              | 00000002 |
> |        | index1 | Instruction | 1              | 00000003 |
> |        | index2 | Unified     | 2              | 00000003 |
> | `cpu2` | index0 | Data        | 1              | 00000004 |
> |        | index1 | Instruction | 1              | 00000004 |
> |        | index2 | Unified     | 2              | 0000000c |
> | `cpu3` | index0 | Data        | 1              | 00000008 |
> |        | index1 | Instruction | 1              | 00000008 |
> |        | index2 | Unified     | 2              | 0000000c |
>
> **Table 5.1: `sysfs` Information for Core 2 CPU Caches**

这些数据的含义如下：

-  每个核心 {cpu0到cpu3是核心的信息来自另一个即将介绍的地方。} 有三个缓存：L1i，L1d，L2。

-  L1d和L1i缓存不与任何人共享，每个核心都有自己的缓存集。这通过在shared_cpu_map位图中只有一个置位来表示。

-  cpu0和cpu1的L2缓存是共享的，cpu2和cpu3的L2也是共享的。

如果CPU有更多的缓存级别，将会有更多的index*目录。

对于一个双核Opteron机器，它有四个插槽，缓存信息如表5.2所示：

> |        | type        | level | shared_cpu_map |          |
> | ------ | ----------- | ----- | -------------- | -------- |
> | `cpu0` | index0      | Data  | 1              | 00000001 |
> | | index1 | Instruction | 1     | 00000001       |
> | | index2 | Unified     | 2     | 00000001       |
> | `cpu1` | index0      | Data  | 1              | 00000002 |
> | | index1 | Instruction | 1     | 00000002       |
> | | index2 | Unified     | 2     | 00000002       |
> | `cpu2` | index0      | Data  | 1              | 00000004 |
> | | index1 | Instruction | 1     | 00000004       |
> | | index2 | Unified     | 2     | 00000004       |
> | `cpu3` | index0      | Data  | 1              | 00000008 |
> | | index1 | Instruction | 1     | 00000008       |
> | | index2 | Unified     | 2     | 00000008       |
> | `cpu4` | index0      | Data  | 1              | 00000010 |
> | | index1 | Instruction | 1     | 00000010       |
> | | index2 | Unified     | 2     | 00000010       |
> | `cpu5` | index0      | Data  | 1              | 00000020 |
> | |  index1 | Instruction | 1     | 00000020       |
> | |  index2 | Unified     | 2     | 00000020       |
> | `cpu6` | index0      | Data  | 1              | 00000040 |
> | | index1 | Instruction | 1     | 00000040       |
> | | index2 | Unified     | 2     | 00000040       |
> | `cpu7` | index0      | Data  | 1              | 00000080 |
> | | index1 | Instruction | 1     | 00000080       |
> | | index2 | Unified     | 2     | 00000080       |
>
> **Table 5.2: `sysfs` Information for Opteron CPU Caches**

正如可以看到的那样，这些处理器也有三个缓存：L1i、L1d和L2。没有任何一个核心共享任何一级缓存。对于这个系统来说，有趣的部分是处理器拓扑结构。如果没有这些额外的信息，就无法理解缓存数据。sys文件系统通过以下文件公开了这些信息。

```
    /sys/devices/system/cpu/cpu*/topology
```

表格5.3展示了SMP Opteron机器在这个层次结构中的有趣文件。



> |        | `physical_package_id` | `core_id` | `core_siblings` | `thread_siblings` |
> | ------ | --------------------- | --------- | --------------- | ----------------- |
> | `cpu0` | 0                     | 0         | 00000003        | 00000001          |
> | `cpu1` | 1                     | 00000003  | 00000002        |                   |
> | `cpu2` | 1                     | 0         | 0000000c        | 00000004          |
> | `cpu3` | 1                     | 0000000c  | 00000008        |                   |
> | `cpu4` | 2                     | 0         | 00000030        | 00000010          |
> | `cpu5` | 1                     | 00000030  | 00000020        |                   |
> | `cpu6` | 3                     | 0         | 000000c0        | 00000040          |
> | `cpu7` | 1                     | 000000c0  | 00000080        |                   |
>
> **Table 5.3: `sysfs` Information for Opteron CPU Topology**

将表5.2和表5.3结合起来，我们可以看到没有任何CPU具有超线程（“thread_siblings”位图只设置了一个位），实际上系统有四个处理器（physical_package_id从0到3），每个处理器有两个核心，而且没有一个核心共享任何缓存。这正是早期Opteron所对应的情况。

迄今为止提供的数据完全缺乏有关此机器NUMA性质的信息。任何SMP Opteron机器都是NUMA机器。对于这些数据，我们必须查看“sys”文件系统的另一部分，该部分存在于NUMA机器下面的层次结构中。

```
    /sys/devices/system/node
```
该目录包含系统上每个NUMA节点的子目录。在节点特定的目录中，有许多文件。对于前两个表中描述的Opteron机器，其重要文件及其内容如表5.4所示。


> |         | `cpumap` | `distance`  |
> | ------- | -------- | ----------- |
> | `node0` | 00000003 | 10 20 20 20 |
> | `node1` | 0000000c | 20 10 20 20 |
> | `node2` | 00000030 | 20 20 10 20 |
> | `node3` | 000000c0 | 20 20 20 10 |
>
> **Table 5.4: `sysfs` Information for Opteron Nodes**

这些信息将所有其他信息联系在一起；现在我们已经有了机器架构的完整图像。我们已经知道机器有四个处理器。每个处理器构成自己的节点，如在`node*`目录中的`cpumap`文件中的值所示。这些目录中的`distance`文件包含一组值，每个节点一个，表示在各个节点上的内存访问成本。在这个例子中，所有本地内存访问的成本都是10，所有对任何其他节点的远程访问的成本都是20。 {*顺便说一句，这是错误的。ACPI信息显然是错误的，因为虽然使用的处理器具有三个一致的HyperTransport链接，但至少一个处理器必须连接到Southbridge。因此，至少有一对节点必须有更大的距离。*} 这意味着，即使处理器被组织成二维超立方体（见图5.1），不直接连接的处理器之间的访问也不会更加昂贵。这些成本的相对值可用作访问时间实际差异的估计。所有这些信息的准确性是另一个问题。

### 5.4 Remote Access Costs

然而，距离是相关的。在[amdccnuma]中，AMD记录了四个插槽机器的NUMA成本。对于写操作，这些数字显示在图5.3中。

> ![img](assets/cpumemory.49.png)
>
> **Figure 5.3: Read/Write Performance with Multiple Nodes**

写比读慢，这并不令人意外。有趣的部分是1和2跳的成本。两个1跳的情况实际上具有略微不同的成本。有关详细信息，请参见[amdccnuma]。我们需要从这张图表中记住的事实是，2跳读和写分别比0跳读慢30％和49％。2跳写比0跳写慢32％，比1跳写慢17％。处理器和内存节点的相对位置可能会产生很大的差异。来自AMD的下一代处理器将每个处理器配备四个一致的HyperTransport链接。在这种情况下，四个插槽机器的直径为1。具有八个插槽的情况会再次出现，因为具有八个节点的超立方体的直径为三。

所有这些信息都是可用的，但使用起来很麻烦。在第6.5节中，我们将看到一个界面，它可以帮助更容易地访问和使用这些信息。

系统提供的最后一条信息是进程本身的状态。可以确定内存映射文件、写时复制(COW)页面和匿名内存在系统中的节点上如何分布。{写时复制是操作系统实现中经常使用的一种方法，当一个内存页一开始只有一个用户时，然后必须复制以允许独立的用户。在许多情况下，复制是不必要的，或者起初是不必要的，在这种情况下，只有当任一用户修改内存时才有意义。操作系统拦截写操作，复制内存页，然后允许写指令继续进行。}每个进程都有一个文件/proc/**PID**/numa_maps，其中**PID**是进程的ID，如图5.2所示。

> ```
> 00400000 default file=/bin/cat mapped=3 N3=3
> 00504000 default file=/bin/cat anon=1 dirty=1 mapped=2 N3=2
> 00506000 default heap anon=3 dirty=3 active=0 N3=3
> 38a9000000 default file=/lib64/ld-2.4.so mapped=22 mapmax=47 N1=22
> 38a9119000 default file=/lib64/ld-2.4.so anon=1 dirty=1 N3=1
> 38a911a000 default file=/lib64/ld-2.4.so anon=1 dirty=1 N3=1
> 38a9200000 default file=/lib64/libc-2.4.so mapped=53 mapmax=52 N1=51 N2=2
> 38a933f000 default file=/lib64/libc-2.4.so
> 38a943f000 default file=/lib64/libc-2.4.so anon=1 dirty=1 mapped=3 mapmax=32 N1=2 N3=1
> 38a9443000 default file=/lib64/libc-2.4.so anon=1 dirty=1 N3=1
> 38a9444000 default anon=4 dirty=4 active=0 N3=4
> 2b2bbcdce000 default anon=1 dirty=1 N3=1
> 2b2bbcde4000 default anon=2 dirty=2 N3=2
> 2b2bbcde6000 default file=/usr/lib/locale/locale-archive mapped=11 mapmax=8 N0=11
> 7fffedcc7000 default stack anon=2 dirty=2 N3=2
> ```
>
> **Figure 5.2: Content of `/proc/\*PID\*/numa_maps`**

文件中重要的信息是`N0`到`N3`的值，它们表示在节点0到3上分配的内存区域的页面数量。很有可能该程序在节点3上的核心上执行，程序本身和脏页面都分配在该节点上。只读映射，如`ld-2.4.so`和`libc-2.4.so`的第一个映射以及共享文件`locale-archive`，则分配在其他节点上。

如我们在图5.3中所见，对于1和2跳读取，跨节点的读取性能分别下降了9%和30%。对于执行，这些读取是必需的，如果L2缓存未命中，则每个缓存行会产生这些额外的成本。如果内存远离处理器，则所有大型工作负载的成本都将增加9％/30％。

> ![img](assets/cpumemory.66.png)
>
> **Figure 5.4: Operating on Remote Memory**
为了看到现实世界中的影响，我们可以像3.5.1节那样测量带宽，但这次是在内存位于远程节点，距离一个跳跃的情况下进行的。将此测试的结果与使用本地内存的数据进行比较，可以在图5.4中看到。这些数字在两个方向上都有一些很大的峰值，这是测量多线程代码的问题，可以忽略不计。这个图表中的重要信息是读操作总是比本地内存慢20%。这比图5.3中的9%要慢得多，这很可能不是连续读写操作的数字，可能指的是旧的处理器版本。只有AMD知道。

对于适合缓存的工作集大小，写入和复制操作的性能也会慢20%。对于超过缓存大小的工作集，写入性能与本地节点上的操作相比没有明显变慢。互联速度足够快，可以跟上内存速度。主导因素是等待主内存的时间。
