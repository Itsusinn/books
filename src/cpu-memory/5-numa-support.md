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

To support NUMA machines, the OS has to take the distributed nature of the memory into account.  For instance, if a process is run on a given processor, the physical RAM assigned to the process's address space should come from local memory.  Otherwise each instruction has to access remote memory for code and data.  There are special cases to be taken into account which are only present in NUMA machines.  The text segment of DSOs is normally present exactly once in a machine's physical RAM.  But if the DSO is used by processes and threads on all CPUs (for instance, the basic runtime libraries like `libc`) this means that all but a few processors have to have remote accesses.  The OS ideally would “mirror” such DSOs into each processor's physical RAM and use local copies.  This is an optimization, not a requirement, and generally hard to implement.  It might not be supported or only in a limited fashion.

To avoid making the situation worse, the OS should not migrate a process or thread from one node to another.  The OS should already try to avoid migrating processes on normal multi-processor machines because migrating from one processor to another means the cache content is lost.  If load distribution requires migrating a process or thread off of a processor, the OS can usually pick an arbitrary new processor which has sufficient capacity left.  In NUMA environments the selection of the new processor is a bit more limited.  The newly selected processor should not have higher access costs to the memory the process is using than the old processor; this restricts the list of targets.  If there is no free processor matching that criteria available, the OS has no choice but to migrate to a processor where memory access is more expensive.

In this situation there are two possible ways forward.  First, one can hope the situation is temporary and the process can be migrated back to a better-suited processor.  Alternatively, the OS can also migrate the process's memory to physical pages which are closer to the newly-used processor. This is quite an expensive operation.  Possibly huge amounts of memory have to be copied, albeit not necessarily in one step.  While this is happening the process, at least briefly, has to be stopped so that modifications to the old pages are correctly migrated.  There are a whole list of other requirements for page migration to be efficient and fast.  In short, the OS should avoid it unless it is really necessary.

Generally, it cannot be assumed that all processes on a NUMA machine use the same amount of memory such that, with the distribution of processes across the processors, memory usage is also equally distributed.  In fact, unless the applications running on the machines are very specific (common in the HPC world, but not outside) the memory use will be very unequal.  Some applications will use vast amounts of memory, others hardly any.  This will, sooner or later, lead to problems if memory is always allocated local to the processor where the request is originated.  The system will eventually run out of memory local to nodes running large processes.

In response to these severe problems, memory is, by default, not allocated exclusively on the local node.  To utilize all the system's memory the default strategy is to stripe the memory.  This guarantees equal use of all the memory of the system.  As a side effect, it becomes possible to freely migrate processes between processors since, on average, the access cost to all the memory used does not change.  For small NUMA factors, striping is acceptable but still not optimal (see data in Section 5.4).

This is a pessimization which helps the system avoid severe problems and  makes it more predictable under normal operation.  But it does decrease overall system performance, in some situations significantly. This is why Linux allows the memory allocation rules to be selected by each process.  A process can select a different strategy for itself and its children.  We will introduce the interfaces which can be used for this in Section 6.



### 5.3 Published Information

The kernel publishes, through the `sys` pseudo file system (sysfs), information about the processor caches below

```
/sys/devices/system/cpu/cpu*/cache
```

In Section 6.2.1 we will see interfaces which can be used to query the size of the various caches.  What is important here is the topology of the caches.  The directories above contain subdirectories (named `index*`) which list information about the various caches the CPU possesses.  The files `type`, `level`, and `shared_cpu_map` are the important files in these directories as far as the topology is concerned.  For an Intel Core 2 QX6700 the information looks as in Table 5.1.



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

What this data means is as follows:

-  Each core {The knowledge that `cpu0` to `cpu3`  are cores comes from another place that will be explained shortly.}  has three caches: L1i, L1d, L2.

-  The L1d and L1i caches are not shared with anybody—each core  has its own set of caches.  This is indicated by the bitmap  in shared_cpu_maphaving only one set bit.

-  The L2 cache on `cpu0` and `cpu1` is shared, as is the  L2 on `cpu2` and `cpu3`.

If the CPU had more cache levels, there would be more `index*` directories.

For a four-socket, dual-core Opteron machine the cache information looks like Table 5.2:



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

  As can be seen these processors also have three caches: L1i, L1d, L2.  None of the cores shares any level of cache.  The interesting part for this system is the processor topology.  Without this additional information one cannot make sense of the cache data.  The `sys` file system exposes this information in the files below



```
    /sys/devices/system/cpu/cpu*/topology
```

Table 5.3 shows the interesting files in this hierarchy for the SMP Opteron machine.



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

Taking Table 5.2 and Table 5.3 together we can see that no CPU has hyper-threads (the `thread_siblings` bitmaps have one bit set), that the system in fact has four processors (`physical_package_id` 0 to 3), that each processor has two cores, and that none of the cores share any cache.  This is exactly what corresponds  to earlier Opterons.

What is completely missing in the data provided so far is information about the nature of NUMA on this machine.  Any SMP Opteron machine is a NUMA machine.  For this data we have to look at yet another part of the `sys` file system which exists on NUMA machines, namely in the hierarchy below



```
    /sys/devices/system/node
```

This directory contains a subdirectory for every NUMA node on the system.  In the node-specific directories there are a number of files.  The important files and their content for the Opteron machine described in the previous two tables are shown in Table 5.4.



> |         | `cpumap` | `distance`  |
> | ------- | -------- | ----------- |
> | `node0` | 00000003 | 10 20 20 20 |
> | `node1` | 0000000c | 20 10 20 20 |
> | `node2` | 00000030 | 20 20 10 20 |
> | `node3` | 000000c0 | 20 20 20 10 |
>
> **Table 5.4: `sysfs` Information for Opteron Nodes**

This information ties all the rest together; now we have a complete picture of the architecture of the machine.  We already know that the machine has four processors.  Each processor constitutes its own node as can be seen by the bits set in the value in `cpumap` file in the `node*` directories.  The `distance` files in those directories contains a set of values, one for each node, which represent a cost of memory accesses at the respective nodes.  In this example all local memory accesses have the cost 10, all remote access to any other node has the cost 20.  {*This is, by the way, incorrect.  The ACPI information is apparently wrong since, although the processors used have three coherent HyperTransport links, at least one processor must be connected to a Southbridge.  At least one pair of nodes must therefore have a larger distance.*}  This means that, even though the processors are organized as a two-dimensional hypercube (see Figure 5.1), accesses between processors which are not directly connected is not more expensive.  The relative values of the costs should be usable as an estimate of the actual difference of the access times.  The accuracy of all this information is another question.





### 5.4 Remote Access Costs

The distance is relevant, though.  In [amdccnuma] AMD documents the NUMA cost of a four socket machine.  For write operations the numbers are shown in Figure 5.3.  

> ![img](assets/cpumemory.49.png)
>
> **Figure 5.3: Read/Write Performance with Multiple Nodes**

Writes are slower than reads, this is no surprise.  The interesting parts are the costs of the 1- and 2-hop cases.  The two 1-hop cases actually have slightly different costs.  See [amdccnuma] for the details.  The fact we need to remember from this chart is that 2-hop reads and writes are 30% and 49% (respectively) slower than 0-hop reads.  2-hop writes are 32% slower than 0-hop writes, and 17% slower than 1-hop writes. The relative position of processor and memory nodes can make a big difference.  The next generation of processors from AMD will feature four coherent HyperTransport links per processor.  In that case a four socket machine would have diameter of one.  With eight sockets the same problem returns, with a vengeance, since the diameter of a hypercube with eight nodes is three.



All this information is available but it is cumbersome to use.  In Section 6.5 we will see an interface which helps accessing and using this information easier.

The last piece of information the system provides is in the status of a process itself.  It is possible to determine how the memory-mapped files, Copy-On-Write (COW) pages and anonymous memory are distributed over the nodes in the  system. {*Copy-On-Write is a method often used in OS implementations when a memory page has one user at first and then has to be copied to allow independent users.  In many situations the copying is unnecessary, at all or at first, in which case it makes sense to only copy when either user modifies the memory.  The operating system intercepts the write operation, duplicates the memory page, and then allows the write instruction to proceed.*}  Each process has a file `/proc/**PID**/numa_maps`, where `**PID**` is the ID of the process, as shown in Figure 5.2.  



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

The important information in the file is the values for `N0` to `N3`, which indicate the number of pages allocated for the memory area on nodes 0 to 3.  It is a good guess that the program was executed on a core on node 3.  The program itself and the dirtied pages are allocated on that node.  Read-only mappings, such as the first mapping for `ld-2.4.so` and `libc-2.4.so` as well as the shared file `locale-archive` are allocated on other nodes.

As we have seen in Figure 5.3 the read performance across nodes falls by 9% and 30% respectively for 1- and 2-hop reads. For execution, such reads are needed and, if the L2 cache is missed, each cache line incurs these additional costs.  All the costs measured for large workloads beyond the size of the cache would have to be increased by 9%/30% if the memory is remote to the processor.

> ![img](assets/cpumemory.66.png)
>
> **Figure 5.4: Operating on Remote Memory**

To see the effects in the real world we can measure the bandwidth as in Section 3.5.1 but this time with the memory being on a remote node, one hop away.  The result of this test when compared with the data for using local memory can be seen in Figure 5.4.  The numbers have a few big spikes in both directions which are the result of a problem of measuring multi-threaded code and can be ignored.  The important information in this graph is that read operations are always 20% slower.  This is significantly slower than the 9% in Figure 5.3, which is, most likely, not a number for uninterrupted read/write operations and might refer to older processor revisions.  Only AMD knows.

For working set sizes which fit into the caches, the performance of write and copy operations is also 20% slower.  For working sets exceeding the size of the caches, the write performance is not measurably slower than the operation on the local node.  The speed of the interconnect is fast enough to keep up with the memory.  The dominating factor is the time spent waiting on the main memory.
