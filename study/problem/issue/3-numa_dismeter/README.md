---

title: Intel CPU 上使用 pmu-tools 进行 TopDown 分析
date: 2021-01-24 18:40
author: gatieme
tags:
        - debug
        - linux
        - todown
categories:
        - debug

thumbnail:
blogexcerpt: 这篇文章旨在帮助希望更好地分析其应用程序中性能瓶颈的人们. 有许多现有的方法可以进行性能分析, 但其中没有很多方法既健壮又正式. 而 TOPDOWN 则为大家进行软硬协同分析提供了无限可能. 本文通过 pmu-tools 入手帮助大家进行 TOPDOWN 分析.


---

| CSDN | GitHub | OSKernelLAB |
|:----:|:------:|:-----------:|
| [紫夜阑珊-青伶巷草](https://blog.csdn.net/gatieme/article/details/113269052) | [`LDD-LinuxDeviceDrivers`](https://github.com/gatieme/LDD-LinuxDeviceDrivers/tree/master/study/debug/tools/topdown/pmu-tools) | [Intel CPU 上使用 pmu-tools 进行 TopDown 分析](https://oskernellab.com/2021/01/24/2021/0127-0001-Topdown_analysis_as_performed_on_Intel_CPU_using_pmu-tools/) |

<br>

<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/"><img alt="知识共享许可协议" style="border-width:0" src="https://i.creativecommons.org/l/by-nc-sa/4.0/88x31.png" /></a>

本作品采用<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/">知识共享署名-非商业性使用-相同方式共享 4.0 国际许可协议</a>进行许可, 转载请注明出处, 谢谢合作.

因本人技术水平和知识面有限, 内容如有纰漏或者需要修正的地方, 欢迎大家指正, 鄙人在此谢谢啦

<br>




时间线

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:--:|:----:|:---------:|:----:|
| 2020/03/24 | Valentin Schneider | [sched/topology: Fix overlapping sched_group build](https://lore.kernel.org/patchwork/patch/1214752) | 修复 | v1 | [PatchWork](https://lore.kernel.org/patchwork/cover/1214752) |
| 2020/8/14 | Valentin Schneider | [sched/topology: NUMA topology limitations](https://lkml.org/lkml/2020/8/14/214) | 修复 | v1 | [LKML](https://lkml.org/lkml/2020/8/14/214) |
| 2020/11/10 | Valentin Schneider | [sched/topology: Warn when NUMA diameter > 2](https://lore.kernel.org/patchwork/patch/1336369) | WARN | v1 | [PatchWork](https://lore.kernel.org/patchwork/cover/1336369) |
| 2021/01/22 | Valentin Schneider | [sched/topology: NUMA distance deduplication](https://lore.kernel.org/patchwork/cover/1369363) | 修复 | v1 | [PatchWork](https://lore.kernel.org/patchwork/cover/1369363), [LKML](https://lkml.org/lkml/2021/1/22/460) |

| 时间  | 提问者 | 问题描述 |
|:----:|:-----:|:-------:|
| 2021/1/21 | Meelis Roos | [Shortest NUMA path spans too many nodes](https://lkml.org/lkml/2021/1/21/726) |


| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:--:|:----:|:---------:|:----:|
| 2021/01/15 | Song Bao Hua (Barry Song) | [sched/fair: first try to fix the scheduling impact of NUMA diameter > 2 1366256 diffmboxseries](https://lore.kernel.org/patchwork/patch/1366256) | 尝试修复 3 跳问题导致的性能蜕化 | RFC | [PatchWork](https://lore.kernel.org/patchwork/cover/1366256) |
| 2021/01/27 | Song Bao Hua (Barry Song) | [sched/topology: fix the issue groups don't span domain->span for NUMA diameter > 2](https://lore.kernel.org/patchwork/patch/1371875)| 修复 | v2 | [PatchWork](https://lore.kernel.org/patchwork/cover/1371875), [LKML]() |


# 1 背景描述
-------


## 1.1 NUMA OVERLAP 的 3 跳问题
-------


社区最早上报这个问题是 ARM 的 Valentin Schneider, 他在调试 HUAWEI KunPeng 920 的机器时, 发现在构建调度域的时候, 基本上每个 CPU 在 cpu_attach_domain 时都会报一个错误信息 `ERROR: groups don't span domain->span`.

```cpp
[344276.794534] CPU0 attaching sched-domain(s):
[344276.794536]  domain-0: span=0-23 level=MC
[344276.794539]   groups: 0:{ span=0 }, 1:{ span=1 }, 2:{ span=2 }, 3:{ span=3 }, 4:{ span=4 }, 5:{ span=5 }, 6:{ span=6 }, 7:{ span=7 }, 8:{ span=8 }, 9:{ span=9 }, 10:{ span=10 }, 11:{ span=11 }, 12:{ span=12 }, 13:{ span=13 }, 14:{ span=14 }, 15:{ span=15 }, 16:{ span=16 }, 17:{ span=17 }, 18:{ span=18 }, 19:{ span=19 }, 20:{ span=20 }, 21:{ span=21 }, 22:{ span=22 }, 23:{ span=23 }
[344276.794554]   domain-1: span=0-47 level=NUMA
[344276.794555]    groups: 0:{ span=0-23 cap=24576 }, 24:{ span=24-47 cap=24576 }
[344276.794558]    domain-2: span=0-71 level=NUMA
[344276.794560]     groups: 0:{ span=0-47 cap=49152 }, 48:{ span=48-95 cap=49152 }
[344276.794563] ERROR: groups don't span domain->span
[344276.799346]     domain-3: span=0-95 level=NUMA
[344276.799353]      groups: 0:{ span=0-71 mask=0-23 cap=73728 }, 72:{ span=48-95 mask=72-95 cap=49152 }
```

发现是在 domain-2 的 NUMA 层次, 该层次调度域的 span(cpumask) 是 0-71, 但是其子 sched_group 的 span 域的并集跟 sched_domain span 域不相同, 也就是说当前调度域不能包含所有的 sched_group 域.


## 1.2 出问题的实际拓扑信息
-------

KUNPENG 920 的 CPU 包含了 2 个 scoket, 每个 socket 上有 2 个 DIE(ARM64 下每个 DIE 作为一个 NUMA NODE), 每个 DIE 上有 24 个 CPU CORE

他们的 numa distance 如下所示:

| node | 0 | 1 | 2 | 3 |
|:----:|:-:|:-:|:-:|:-:|
|  0  | 10 | 12 | 20 | 22 |
|  1  | 12 | 10 | 22 | 24 |
|  2  | 20 | 22 | 10 | 12 |
|  3  | 22 | 24 | 12 | 10 |

1 和 3 之间的距离是最远的, 因此这是正式硬件拓扑的两个最远端节点. 因此 CPU 的硬件拓扑结构就大致是如下的形式:

```cpp
      2       10      2
  1 <---> 0 <---> 2 <---> 3
```

> 【注意】
> 本地到本地的距离为 10
> 因此 0-1 的距离为 0 <--> 0 <--> 1, 就是 12
> 其他节点距离计算类似


## 1.3 模拟复现拓扑信息
-------


每个 DIE 上 24 个 core 不利于我们分析, 我们将此模型简化为每个 DIE 上 2 个 CPU core 的模型

那么我们可以很容易的使用 QEMU 复现这个问题. QEMU 模拟如下的 NUMA 拓扑:

```cpp
-smp cores=8                                                        \
-numa node,cpus=0-1,nodeid=0 -numa node,cpus=2-3,nodeid=1,          \
-numa node,cpus=4-5,nodeid=2, -numa node,cpus=6-7,nodeid=3,         \
-numa dist,src=0,dst=1,val=12, -numa dist,src=0,dst=2,val=20,       \
-numa dist,src=0,dst=3,val=22, -numa dist,src=1,dst=2,val=22,       \
-numa dist,src=1,dst=3,val=24, -numa dist,src=2,dst=3,val=12
```

启动参数中添加 `sched_debug=1`, 内核启动阶段的调度域调试信息如下所示:

```cpp
[    1.346911] CPU0 attaching sched-domain(s):
[    1.347160]  domain-0: span=0-1 level=MC
[    1.347690]   groups: 0:{ span=0 }, 1:{ span=1 }
[    1.348101]   domain-1: span=0-3 level=NUMA
[    1.348312]    groups: 0:{ span=0-1 cap=2048 }, 2:{ span=2-3 cap=2048 }
[    1.348639]    domain-2: span=0-5 level=NUMA
[    1.348772]     groups: 0:{ span=0-3 cap=4096 }, 4:{ span=4-7 cap=4096 }
[    1.349152] ERROR: groups don't span domain->span
[    1.349343]     domain-3: span=0-7 level=NUMA
[    1.349558]      groups: 0:{ span=0-5 mask=0-1 cap=6144 }, 6:{ span=4-7 mask=6-7 cap=4096 }
[    1.350413] CPU1 attaching sched-domain(s):
[    1.350563]  domain-0: span=0-1 level=MC
[    1.351274]   groups: 1:{ span=1 }, 0:{ span=0 }
[    1.351558]   domain-1: span=0-3 level=NUMA
[    1.351714]    groups: 0:{ span=0-1 cap=2048 }, 2:{ span=2-3 cap=2048 }
[    1.351862]    domain-2: span=0-5 level=NUMA
[    1.352225]     groups: 0:{ span=0-3 cap=4096 }, 4:{ span=4-7 cap=4096 }
[    1.352462] ERROR: groups don't span domain->span
[    1.352847]     domain-3: span=0-7 level=NUMA
[    1.353180]      groups: 0:{ span=0-5 mask=0-1 cap=6144 }, 6:{ span=4-7 mask=6-7 cap=4096 }
[    1.353759] CPU2 attaching sched-domain(s):
[    1.354211]  domain-0: span=2-3 level=MC
[    1.354853]   groups: 2:{ span=2 }, 3:{ span=3 }
[    1.355373]   domain-1: span=0-3 level=NUMA
[    1.356004]    groups: 2:{ span=2-3 cap=2048 }, 0:{ span=0-1 cap=2048 }
[    1.357009]    domain-2: span=0-5 level=NUMA
[    1.357359]     groups: 2:{ span=0-3 mask=2-3 cap=4096 }, 4:{ span=0-1,4-7 mask=4-5 cap=6144 }
[    1.357766] ERROR: groups don't span domain->span
[    1.358233]     domain-3: span=0-7 level=NUMA
[    1.358517]      groups: 2:{ span=0-5 mask=2-3 cap=6144 }, 6:{ span=0-1,4-7 mask=6-7 cap=6144 }
[    1.358871] CPU3 attaching sched-domain(s):
[    1.359259]  domain-0: span=2-3 level=MC
[    1.359578]   groups: 3:{ span=3 }, 2:{ span=2 }
[    1.359847]   domain-1: span=0-3 level=NUMA
[    1.360158]    groups: 2:{ span=2-3 cap=2048 }, 0:{ span=0-1 cap=2048 }
[    1.360846]    domain-2: span=0-5 level=NUMA
[    1.361127]     groups: 2:{ span=0-3 mask=2-3 cap=4096 }, 4:{ span=0-1,4-7 mask=4-5 cap=6144 }
[    1.361668] ERROR: groups don't span domain->span
[    1.361844]     domain-3: span=0-7 level=NUMA
[    1.362141]      groups: 2:{ span=0-5 mask=2-3 cap=6144 }, 6:{ span=0-1,4-7 mask=6-7 cap=6144 }
[    1.362869] CPU4 attaching sched-domain(s):
[    1.363123]  domain-0: span=4-5 level=MC
[    1.363369]   groups: 4:{ span=4 }, 5:{ span=5 }
[    1.363720]   domain-1: span=4-7 level=NUMA
[    1.363848]    groups: 4:{ span=4-5 cap=2048 }, 6:{ span=6-7 cap=2048 }
[    1.364293]    domain-2: span=0-1,4-7 level=NUMA
[    1.364557]     groups: 4:{ span=4-7 cap=4096 }, 0:{ span=0-3 cap=4096 }
[    1.364830] ERROR: groups don't span domain->span
[    1.365135]     domain-3: span=0-7 level=NUMA
[    1.365426]      groups: 4:{ span=0-1,4-7 mask=4-5 cap=6144 }, 2:{ span=0-3 mask=2-3 cap=4096 }
[    1.365908] CPU5 attaching sched-domain(s):
[    1.366104]  domain-0: span=4-5 level=MC
[    1.366844]   groups: 5:{ span=5 }, 4:{ span=4 }
[    1.367236]   domain-1: span=4-7 level=NUMA
[    1.367545]    groups: 4:{ span=4-5 cap=2048 }, 6:{ span=6-7 cap=2048 }
[    1.367847]    domain-2: span=0-1,4-7 level=NUMA
[    1.368209]     groups: 4:{ span=4-7 cap=4096 }, 0:{ span=0-3 cap=4096 }
[    1.368841] ERROR: groups don't span domain->span
[    1.369170]     domain-3: span=0-7 level=NUMA
[    1.369467]      groups: 4:{ span=0-1,4-7 mask=4-5 cap=6144 }, 2:{ span=0-3 mask=2-3 cap=4096 }
[    1.369890] CPU6 attaching sched-domain(s):
[    1.370123]  domain-0: span=6-7 level=MC
[    1.370843]   groups: 6:{ span=6 }, 7:{ span=7 }
[    1.371237]   domain-1: span=4-7 level=NUMA
[    1.371543]    groups: 6:{ span=6-7 cap=2048 }, 4:{ span=4-5 cap=2048 }
[    1.371845]    domain-2: span=0-1,4-7 level=NUMA
[    1.372183]     groups: 6:{ span=4-7 mask=6-7 cap=4096 }, 0:{ span=0-5 mask=0-1 cap=6144 }
[    1.372717] ERROR: groups don't span domain->span
[    1.372853]     domain-3: span=0-7 level=NUMA
[    1.373133]      groups: 6:{ span=0-1,4-7 mask=6-7 cap=6144 }, 2:{ span=0-5 mask=2-3 cap=6144 }
[    1.373858] CPU7 attaching sched-domain(s):
[    1.374171]  domain-0: span=6-7 level=MC
[    1.374431]   groups: 7:{ span=7 }, 6:{ span=6 }
[    1.374844]   domain-1: span=4-7 level=NUMA
[    1.375209]    groups: 6:{ span=6-7 cap=2048 }, 4:{ span=4-5 cap=2048 }
[    1.375683]    domain-2: span=0-1,4-7 level=NUMA
[    1.376050]     groups: 6:{ span=4-7 mask=6-7 cap=4096 }, 0:{ span=0-5 mask=0-1 cap=6144 }
[    1.376605] ERROR: groups don't span domain->span
[    1.376856]     domain-3: span=0-7 level=NUMA
[    1.377163]      groups: 6:{ span=0-1,4-7 mask=6-7 cap=6144 }, 2:{ span=0-5 mask=2-3 cap=6144 }
[    1.377971] root domain span: 0-7 (max cpu_capacity = 1024)
```

由此所构建的 sched_domain 和 sched_group 信息如下所示.

| SCHED_DOAMIN | NODE  0 | NODE  1 | NODE  2 | NODE  3 |
|:------------:|:-------:|:-------:|:-------:|:-------:|
|   DOMAIN  0  |     0-1     |       2-3       |       4-5       |       6-7       |
|   GROUP   0  |   {0},{1}   |     {2},{3}     |     {4},{5}     |     {6},{7}     |
|--------------|-------------|-----------------|-----------------|-----------------|
|   DOMAIN  1  |     0-3     |       0-3       |       4-7       |       4-7       |
|   GROUP   1  | {0-1},{2-3} |   {2-3},{0-1}   |   {4-5},{6-7}   |   {6-7},{4-5}   |
|--------------|-------------|-----------------|-----------------|-----------------|
|   DOMAIN  2  |     0-5     |       0-5       |     0-1,4-7     |     0-1,4-7     |
|   GROUP   2  | {0-3},{4-7} | {0-3},{0-1,4-7} |   {4-7},{0-3}   |   {4-7},{0-5}   |
|   BALANCE 2  |     XXX     | {2-3},{4-----5} |       XXX       |   {6-7},{0-1}   |
|--------------|-------------|-----------------|-----------------|-----------------|
|   DOMAIN  3  |     0-7     |       0-7       |       0-7       |       0-7       |
|   GROUP   3  | {0-5},{4-7} | {0-5},{0-1,4-7} | {0-1,4-7},{0-3} | {0-1,4-7},{0-5} |
|   BALANCE 3  | {0-1},{6-7} | {2-3},{6-----7} | {4-----5},{2-3} | {6-----7},{2-3} |

> 表中 BALANCE 空缺或者填 XXX 均表示, sched_balance_mask 与 sched_group_span 相同.

很明显, 跟 KUNGPENG 920 环境上存在同样的问题, 在 DOMAIN 2 层级, 调度域的 span 没法包含所有的 sched_group span.

# 2 问题分析 
-------


## 2.1 NUMA 层级构建 sched_init_numa
-------


### 2.1.1 numa 层级建立
-------

在内核启动阶段会通过 [`sched_init_numa`](https://elixir.bootlin.com/linux/v5.10/source/kernel/sched/topology.c#L1552) 来构建 NUMA 层级.

我们之前总是以为, 每个 CPU 是以自己为中心想外辐射, 逐渐构建自己所在的调度域和调度组的.但是这样的一个算法复杂度是 O(n^2) 的.

因此内核在 sched_init_numa 中实现的时候进行了简化和优化. 内核做了这样一个假定:

> 以 NODE_DISTANCE(0, J) 一定包含 NODE_DISTANCE(I, J) [0 <= I <= J]

这样每个 CPU 可以借助 CPU0 为原点去构建距离跳变的阶梯, 从而获取到 NUMA 的层级. 每个 CPU 以 NODE 0 为原点构建的解体距离范围, 以自我为中心向周边进行辐射.

```cpp
void sched_init_numa(void)
{
    int next_distance, curr_distance = node_distance(0, 0);
    struct sched_domain_topology_level *tl;
    int level = 0;
    int i, j, k;

    sched_domains_numa_distance = kzalloc(sizeof(int) * (nr_node_ids + 1), GFP_KERNEL);
    if (!sched_domains_numa_distance)
        return;

    /* Includes NUMA identity node at level 0. */
    sched_domains_numa_distance[level++] = curr_distance;
    sched_domains_numa_levels = level;

    /*
     * O(nr_nodes^2) deduplicating selection sort -- in order to find the
     * unique distances in the node_distance() table.
     *
     * Assumes node_distance(0,j) includes all distances in
     * node_distance(i,j) in order to avoid cubic time.
     */
    next_distance = curr_distance;
    for (i = 0; i < nr_node_ids; i++) {
        for (j = 0; j < nr_node_ids; j++) {
            for (k = 0; k < nr_node_ids; k++) {
                int distance = node_distance(i, k);

                if (distance > curr_distance &&
                    (distance < next_distance ||
                     next_distance == curr_distance))
                    next_distance = distance;

                /*
                 * While not a strong assumption it would be nice to know
                 * about cases where if node A is connected to B, B is not
                 * equally connected to A.
                 */
                if (sched_debug() && node_distance(k, i) != distance)
                    sched_numa_warn("Node-distance not symmetric");

                if (sched_debug() && i && !find_numa_distance(distance))
                    sched_numa_warn("Node-0 not representative");
            }
            if (next_distance != curr_distance) {
                sched_domains_numa_distance[level++] = next_distance;
                sched_domains_numa_levels = level;
                curr_distance = next_distance;
            } else break;
        }

        /*
         * In case of sched_debug() we verify the above assumption.
         */
        if (!sched_debug())
            break;
    }

    ......
```

这样完成之后, 我们就以 NODE0 为原点, 构建好了一个当前系统的 NUMA 距离阶梯.

如果我们的拓扑结构是以 NODE 0 为边缘(起点)的, 那么我们的 NUMA 层级就应该是

```cpp
拓扑结构
      2       10      2
  0 <---> 1 <---> 2 <---> 3

NUMA 距离层级(阶梯)
10 -=> 12 -=> 22 -=> 24
```

但是很抱歉, 我们的环境中 NUMA0 并不是边缘节点, 而是如下的形式:

```cpp
拓扑结构
      2       10      2
  1 <---> 0 <---> 2 <---> 3
```

于是我们的构建的阶梯就是如下层级:

```
NUMA 距离层级(阶梯)
10  -=> 12  -=> 20 -=> 22 -=> 24
```

OK, 请记下这里, 这里是调度域构建过程中的第一个分歧, 他势必会对调度域造成一些影响.
这个我们后面会分析.

### 2.1.2 按照 numa level 构建每个层级的 MASK
-------



下面将按照这些 NUMA 层级来构建拓扑结构. 内核将每个阶梯之间的 CPU 构建成一组, 组成一个当前层级的调度域.
也就是说距离 [0, 10] 的一组, (10, 12] 的一组, (12, 20] 的一组, (20, 22] 的一组, (22, 24 的一组).

问题也就出现在这里, 由于我们 NODE 0 并不是边缘节点, 因此 (12, 20] 这个层级很明显, 并是所有的 CPU 都能覆盖.
也就是说:

*   有一部分 NODE 节点(边缘节点 NODE1 和 NODE 3) (10, 12] 和 (12, 20] 这两个层级是一模一样的;

*   有一部分 NODE 节点(非边缘节点 NODE 0 和 NODE 2) 将没有 (22, 24] 这一层级.

那么根据 NUMA distance 构建出的 NUMA 层级如下:

每个 NUMA 层次所对应的 NUMA 节点信息

| 距离 | NODE 0 | NODE 1 | NODE 2 | NODE 3 |
|:---:|:------:|:------:|:---------:|:-------:|
| 10 |   {0}   |  {1}   |    {2}    |   {3}   |
| 12 |  {0-1}  |  {0-1} |   {2-3}   |  {2-3}  |
| 20 |  {0-2}  |  {0-1} |  {0,2-3}  |  {2-3}  |
| 22 |  {0-3}  |  {0-2} |   {0-3}   | {0,2-3} |
| 24 |         |  {0-3} |           |  {0-3}  |

每个 NUMA 层次所对应的 CPU 节点信息


| 距离 | CPU0/1 | CPU2/3 |  CPU4/5   |  CPU6/7   |
|:---:|:------:|:------:|:---------:|:---------:|
| 10 |  {0-1}  |  {2-3} |   {4-5}   |   {6-7}   |
| 12 |  {0-3}  |  {0-3} |   {4-7}   |   {4-7}   |
| 20 |  {0-5}  |  {0-3} | {0-1,4-7} |   {4-7}   |
| 22 |  {0-7}  |  {0-5} |   {0-7}   | {0-1,4-7} |
| 24 |         |  {0-7} |           |   {0-7}   |

这部分代码比较容易理解, 如下所示:

```cpp
// https://elixir.bootlin.com/linux/v5.10/source/kernel/sched/topology.c#L1610
    /*
     * 'level' contains the number of unique distances
     *
     * The sched_domains_numa_distance[] array includes the actual distance
     * numbers.
     */

    /*
     * Here, we should temporarily reset sched_domains_numa_levels to 0.
     * If it fails to allocate memory for array sched_domains_numa_masks[][],
     * the array will contain less then 'level' members. This could be
     * dangerous when we use it to iterate array sched_domains_numa_masks[][]
     * in other functions.
     *
     * We reset it to 'level' at the end of this function.
     */
    sched_domains_numa_levels = 0;

    sched_domains_numa_masks = kzalloc(sizeof(void *) * level, GFP_KERNEL);
    if (!sched_domains_numa_masks)
        return;

    /*
     * Now for each level, construct a mask per node which contains all
     * CPUs of nodes that are that many hops away from us.
     */
    for (i = 0; i < level; i++) {
        sched_domains_numa_masks[i] =
            kzalloc(nr_node_ids * sizeof(void *), GFP_KERNEL);
        if (!sched_domains_numa_masks[i])
            return;

        for (j = 0; j < nr_node_ids; j++) {
            struct cpumask *mask = kzalloc(cpumask_size(), GFP_KERNEL);
            if (!mask)
                return;

            sched_domains_numa_masks[i][j] = mask;

            for_each_node(k) {
                if (node_distance(j, k) > sched_domains_numa_distance[i])
                    continue;

                cpumask_or(mask, mask, cpumask_of_node(k));
            }
        }
    }

    ......
```

### 2.1.2 按照 numa level 初始化每个层级的 sched_domain
-------


接着, 先初步初始化了 `sched_domain` 的信息, 目前为止, 内核还不知道当前系统的层次结构具体是什么样子的.
因此我们创建了一个可能最全最冗余的拓扑结构.


```cpp
    /* Compute default topology size */
    for (i = 0; sched_domain_topology[i].mask; i++);

    tl = kzalloc((i + level + 1) *
            sizeof(struct sched_domain_topology_level), GFP_KERNEL);
    if (!tl)
        return;

    /*
     * Copy the default topology bits..
     */
    for (i = 0; sched_domain_topology[i].mask; i++)
        tl[i] = sched_domain_topology[i];

    /*
     * Add the NUMA identity distance, aka single NODE.
     */
    tl[i++] = (struct sched_domain_topology_level){
        .mask = sd_numa_mask,
        .numa_level = 0,
        SD_INIT_NAME(NODE)
    };

    /*
     * .. and append 'j' levels of NUMA goodness.
     */
    for (j = 1; j < level; i++, j++) {
        tl[i] = (struct sched_domain_topology_level){
            .mask = sd_numa_mask,
            .sd_flags = cpu_numa_flags,
            .flags = SDTL_OVERLAP,
            .numa_level = j,
            SD_INIT_NAME(NUMA)
        };
    }

    sched_domain_topology = tl;

    sched_domains_numa_levels = level;
    sched_max_numa_distance = sched_domains_numa_distance[level - 1];

    init_numa_topology_type();
}
```

他首先拷贝了初始化默认的 [default_topology](https://elixir.bootlin.com/linux/v5.10/source/kernel/sched/topology.c#L1443) 信息, 这个里面一般包括了 SMT, MC, DIE 等级别(具体跟你 CONFIG 有关系).

```cpp
// https://elixir.bootlin.com/linux/v5.10/source/kernel/sched/topology.c#L1443
/*
 * Topology list, bottom-up.
 */
static struct sched_domain_topology_level default_topology[] = {
#ifdef CONFIG_SCHED_SMT
    { cpu_smt_mask, cpu_smt_flags, SD_INIT_NAME(SMT) },
#endif
#ifdef CONFIG_SCHED_MC
    { cpu_coregroup_mask, cpu_core_flags, SD_INIT_NAME(MC) },
#endif
    { cpu_cpu_mask, SD_INIT_NAME(DIE) },
    { NULL, },
};
```

接着默认创建了一个名为 NODE 的节点, 然后后续层级都将设置为 NUMA 层级.

在我们的示例中, 将生成如下层次的序列:

```cpp
MC -=> DIE -=> NODE -=> NUMA -=> NUMA
```

然后 numa level 为 2 层.

### 2.1.3 sched_init_numa 总结
-------

[sched_init_numa](https://elixir.bootlin.com/linux/v5.10/source/kernel/sched/topology.c#L1552) 主体完成了如下几个功能


1.  先根据 distance 确定了当前系统的 NUMA 层级, 

2.  根据 NUMA 层级, 初始化了每个 NUMA 节点在每个层级的 cpumask 信息

3.  根据 NUMA 层级, 初始化 sched_domain 结构.

至此 `NUMA` 的基本信息都已经初始化好了, 下面要做的就是真正把这个调度域构建起来 [build_sched_domains](https://elixir.bootlin.com/linux/v5.10/source/kernel/sched/topology.c#L1977), 他的主要工作就是将每个 CPU 与对应的 NUMA 层级关联起来 [cpu_attach_domain](https://elixir.bootlin.com/linux/v5.10/source/kernel/sched/topology.c#L668), 同时构建各个 NUMA 层级的 sched_group [build_sched_group](https://elixir.bootlin.com/linux/v5.10/source/kernel/sched/topology.c#L1106).


## 2.2 DOMAIN & GROUP 构建
-------

### 2.2.1 build_sched_domain
-------

对每个 CPU 每层 topology 通过 build_sched_domain 构建其基础的 sched_domain 信息.
这个是按照前面初始化好的最冗余的 topology 来进行的, 直到当前 CPU 的某一层调度域包含了全量的 cpu_map 则停止.

```cpp
// https://elixir.bootlin.com/linux/v5.10/source/kernel/sched/topology.c#L1977

static int
build_sched_domains(const struct cpumask *cpu_map, struct sched_domain_attr *attr)
{
    // ......

    /* Set up domains for CPUs specified by the cpu_map: */
    for_each_cpu(i, cpu_map) {
        struct sched_domain_topology_level *tl;
        int dflags = 0;

        sd = NULL;
        for_each_sd_topology(tl) {
            if (tl == tl_asym) {
                dflags |= SD_ASYM_CPUCAPACITY;
                has_asym = true;
            }

            if (WARN_ON(!topology_span_sane(tl, cpu_map, i)))
                goto error;

            sd = build_sched_domain(tl, cpu_map, attr, sd, dflags, i);

            if (tl == sched_domain_topology)
                *per_cpu_ptr(d.sd, i) = sd;
            if (tl->flags & SDTL_OVERLAP)
                sd->flags |= SD_OVERLAP;
            if (cpumask_equal(cpu_map, sched_domain_span(sd)))
                break;
        }
    }
```


```cpp
// https://elixir.bootlin.com/linux/v5.10/source/kernel/sched/topology.c#L1852

static struct sched_domain *build_sched_domain(struct sched_domain_topology_level *tl,
        const struct cpumask *cpu_map, struct sched_domain_attr *attr,
        struct sched_domain *child, int dflags, int cpu)
{
    struct sched_domain *sd = sd_init(tl, cpu_map, child, dflags, cpu);

    if (child) {
        sd->level = child->level + 1;
        sched_domain_level_max = max(sched_domain_level_max, sd->level);
        child->parent = sd;

        if (!cpumask_subset(sched_domain_span(child),
                    sched_domain_span(sd))) {
            pr_err("BUG: arch topology borken\n");
#ifdef CONFIG_SCHED_DEBUG
            pr_err("     the %s domain not a subset of the %s domain\n",
                    child->name, sd->name);
#endif
            /* Fixup, ensure @sd has at least @child CPUs. */
            cpumask_or(sched_domain_span(sd),
                   sched_domain_span(sd),
                   sched_domain_span(child));
        }

    }
    set_domain_attribute(sd, attr);

    return sd;
}
```


对所有 CPU 完成 build_sched_domain 之后, 调度域的 span 信息如下.
由于构建到 sched_domain 的 span 与 cpu_map 相同时, 就停止构建.
因此 CPU 0/1/6/7 比 CPU 2/3/4/5 要少一层.

| 距离  | CPU0/1  | CPU2/3 |  CPU4/5   |  CPU6/7   |
|:----:|:-------:|:------:|:---------:|:---------:|
|  MC  |  {0-1}  |  {2-3} |   {4-5}   |   {6-7}   |
| DIE  |  {0-1}  |  {2-3} |   {4-5}   |   {6-7}   |
| NODE |  {0-1}  |  {2-3} |   {4-5}   |   {6-7}   |
| NUMA |  {0-3}  |  {0-3} |   {4-7}   |   {4-7}   |
| NUMA |  {0-5}  |  {0-3} | {0-1,4-7} |   {4-7}   |
| NUMA |  {0-7}  |  {0-5} |   {0-7}   | {0-1,4-7} |
| NUMA |   NA    |  {0-7} |    NA     |   {0-7}   |


### 2.2.2 build_sched_group
-------


接着通过构建每个 `sched_domain` 的 `sched_group` 域.

*   对于 NUMA 域一般配置了 SD_OVERLAP, 因此通过 `build_overlap_sched_groups` 来构建

*   对于没有配置 SD_OVERLAP 的域, 则通过 `build_sched_groups` 来构建.

```cpp
// https://elixir.bootlin.com/linux/v5.10/source/kernel/sched/topology.c#L2027
static int
build_sched_domains(const struct cpumask *cpu_map, struct sched_domain_attr *attr)
{
    // ......

    /* Build the groups for the domains */
    for_each_cpu(i, cpu_map) {
        for (sd = *per_cpu_ptr(d.sd, i); sd; sd = sd->parent) {
            sd->span_weight = cpumask_weight(sched_domain_span(sd));
            if (sd->flags & SD_OVERLAP) {
                if (build_overlap_sched_groups(sd, i))
                    goto error;
            } else {
                if (build_sched_groups(sd, i))
                    goto error;
            }
        }
    }

    // ......
```


```cpp
// https://elixir.bootlin.com/linux/v5.10/source/kernel/sched/topology.c#L938
// build_overlap_sched_groups

// https://elixir.bootlin.com/linux/v5.10/source/kernel/sched/topology.c#L1106
// build_sched_groups
```

由于 sched_group_span 必须要覆盖到 sched_domain_span 的所有 CPU.

每次对于某个 CPU, 处理其对应层级的时候, 会依次遍历 sched_domain_span 中的所有 CPU 保证把其对应的 mask 全部加进来.
最终形成的每个 sched_domain 的 GROUP 信息如下所示:

| SCHED_DOAMIN |     CPU  0    |     CPU 1     |   CPU 2   |    CPU 3   |   CPU 4   |   CPU 5   |    CPU 6    |    CPU 7   |
|:------------:|:-------------:|:-------------:|:---------:|:----------:|:---------:|:---------:|:-----------:|:----------:|
|DOMAIN  0(MC) |      0-1      |      0-1      |    2-3    |    2-3     |    4-5    |    4-5    |     6-7     |    6-7     |
|   GROUP   0  |   {0},{[1]}   |   {1},{[0]}   | {2},{[3]} |  {3},{[2]} |  {4},{5}  | {5},{[4]} |  {6},{[7]}  |  {7},{[6]} |
|--------------|---------------|---------------|-----------|------------|-----------|-----------|-------------|------------|
|DOMAIN  1(DIE)|      0-1      |      0-1      |    2-3    |    2-3     |    4-5    |    4-5    |     6-7     |    6-7     |
|   GROUP   1  |     {0-1}     |     {0-1}     |   {2-3}   |   {2-3}    |   {4-5}   |   {4-5}   |    {6-7}    |   {6-7}    |
|--------------|---------------|---------------|-----------|------------|-----------|-----------|-------------|------------|
|DOMAIN 2(NODE)|      0-1      |      0-1      |    2-3    |    2-3     |    4-5    |    4-5    |     6-7     |    6-7     |
|   GROUP   2  |     {0-1}     |     {0-1}     |   {2-3}   |   {2-3}    |   {4-5}   |   {4-5}   |    {6-7}    |   {6-7}    |
|--------------|---------------|---------------|-----------|------------|-----------|-----------|-------------|------------|
|DOMAIN 3(NUMA)|      0-3      |      0-3      |    0-3    |    0-3     |    4-7    |    4-7    |     4-7     |    4-7     |
|   GROUP   3  | {0-1},{[2]-3} | {0-1},{[2]-3} | {2-3},{[0]-1} | {2-3},{[0]-1} | {4-5},{[6]-7} | {4-5},{[6]-7} | {6-7},{[4]-5} | {6-7},{[4]-5} |
|--------------|---------------|---------------|-----------|------------|-----------|-----------|-------------|------------|
|DOMAIN 4(NUMA)|      0-5      |      0-5      |    0-3    |    0-3     |  0-1,4-7  |  0-1,4-7  |     4-7     |    4-7     |
|   GROUP   4  | {0-3},{[4]-7} | {0-3},{[4]-7} |   {0-3}   |   {0-3}    | {[0]-3},{4-7} | {[0]-3},{4-7} |    {4-7}    |   {4-7}    |
|--------------|---------------|---------------|-----------|------------|-----------|-----------|-------------|------------|
|DOMAIN 5(NUMA)|     {0-7}     |     {0-7}     |   {0-5}   |   {0-5}    |   {0-7}   |   {0-7}   |  {0-1,4-7}  |  {0-1,4-7} |
|   GROUP   5  | {0-5},{4-[6]-7} | {0-5},{4-[6]-7} | {0-3},{0-1,[4]-7} | {0-3},{0-1,[4]-7} | {[0]-3},{0-1,[4]-7} | {[0]-3},{0-1,4-7} | {[0]-5},{4-7}  |   {[0]-5},{4-7}  |
|--------------|---------------|---------------|-----------|------------|-----------|-----------|-------------|------------|
|DOMAIN 6(NUMA)|     NA     |     NA     |   {0-7}   |   {0-7}    |  NA  |  NA  |  {0-7}  |  {0-7} |
|   GROUP   6  | NA | NA | {0-5},{0-1,4-[6]-7} | {0-5},{0-1,4-[6]-7} | NA | NA  | {0-[2]-5},{0-1,4-7} | {0-[2]-5},{0-1,4-7}  |


举例来说:
CPU 0 的 DOMAIN-4 层级 sched_domains_span {0-5}
会依次遍历 sched_domains_span 的所有 CPU
自己所在的 CPU 上一个层级的 sched_domains_span 是 0-3, 先加进来, 已经包含了 0-3
接着继续遍历到 CPU 4, 发现  CPU 4 还没有覆盖, 继续添加, 将其 DOMAIN-3 层级的 sched_domain_span {4-7} 加进来.
此时已经包含了 {0-5} 所有的 CPUMASK 因此形成的 GROUP 有两个: {0-3},{4-7}.

> 注意
>
> 带 [] 是我们为了清晰显示添加过程的一种展示方式, 表示某个 group 由于该 CPU 加进来的.
>
> 为了清楚的表示添加的过程, 我们使用 [CPU] 标记将该 GROUP 加入 的 CPU
> 默认如果是本 CPU 加进来的, 我们会省略其对应的 []
> 举例来说: CPU 0 的 DOMAIN-4 sched_group 为  {0-3},{4-7}
> 表示为: {[0]-3},{[4]-7} -=> {0-3}, {[4]-7}
> 表示 CPU 0 自己将 {0-3} 加入
> 另外一个 GROUP {4-7} 是 CPU 4 加进来的, 记录为 {[4]-7}

我们的 3 跳问题其实就出在这里, CPU0 DOMAIN-4 只期望包含 0-5, 但是在覆盖 CPU 4 添加 sched_group 的时候, 将 {4-7} 都加了进来.
这样所有 sched_group_span 的并集超出了 sched_domain_span 的范围.
因此在后面进行 [sched_domain_debug_one](https://elixir.bootlin.com/linux/v5.10/source/kernel/sched/topology.c#L122) 中就会触发 ERROR

```cpp
static int sched_domain_debug_one(struct sched_domain *sd, int cpu, int level,
                  struct cpumask *groupmask)
{
    // ......
    if (!cpumask_equal(sched_domain_span(sd), groupmask))
        printk(KERN_ERR "ERROR: groups don't span domain->span\n");
    // ......
}
```

这个调试信息在 cpu_attach_domain 中通过调试信息打出来的, 需要手动在启动参数中配置 sched_debug=1, 我们继续接着来看.

### 2.2.3 cpu_attach_domain
-------

// https://elixir.bootlin.com/linux/v5.10/source/kernel/sched/topology.c#L668


我们之前所构建的调度域, 是一个可能存在冗余层级的, 因此可能存在一些层级是重复的, 在 cpu_attach_domain 的时候, 如果发现当前层级和他父层级是相同的, 那么就会销毁一个层级.

比如

*   对于所有的 CPU, 发现 MC/DIE/NODE 发现 sched_domain_span 是一样的, 于是 DIE/NODE 两个层级就会被销毁掉, 只保留 MC.

*   对于 NODE1(CPU2 和 CPU3) 以及 NODE3(CPU6 和 CPU7) 等边缘节点, DOMAIN-3 和 DOMAIN-4 是一样的, 因此 DOMAIN-4 将被删除.

最终 

*   NODE0 和 NODE2 的 CPU, DOMAIN-0/3/4/5 将保留

*   NODE1 和 NODE3 的 CPU, DOMAIN-0/3/5/6 将保留

> 注意
>
> 为什么删除父层级? 或者 为什么父层级可以删除?
>
> 这个还记得上面 sched_group 构建过程么, 父层级的调度域会用上一个子层级域的 sched_domain_span 作为其一个子 sched_group.
> 因此如果父层级的 sched_domain_span 和子的是一样的, 那么从底往高构建的时候, 父层级把子层级 sched_domain_span 作为一个 sched_group 直接加进来就已经覆盖了自己的 sched_domain_span.
> 因此此时父层级将只包含一个 sched_group. 这种调度域对于负载均衡是没有任何帮助的. 
> 因此可以也有必要直接删除掉.

由此所构建的 sched_domain 和 sched_group 信息如下所示.


| SCHED_DOAMIN | NODE  0 | NODE  1 | NODE  2 | NODE  3 |
|:------------:|:-------:|:-------:|:-------:|:-------:|
|   DOMAIN  0  |     0-1     |       2-3       |       4-5       |       6-7       |
|   GROUP   0  |   {0},{1}   |     {2},{3}     |     {4},{5}     |     {6},{7}     |
|--------------|-------------|-----------------|-----------------|-----------------|
|   DOMAIN  1  |     0-3     |       0-3       |       4-7       |       4-7       |
|   GROUP   1  | {0-1},{2-3} |   {2-3},{0-1}   |   {4-5},{6-7}   |   {6-7},{4-5}   |
|--------------|-------------|-----------------|-----------------|-----------------|
|   DOMAIN  2  |     0-5     |       0-5       |     0-1,4-7     |     0-1,4-7     |
|   GROUP   2  | {0-3},{4-7} | {0-3},{0-1,4-7} |   {4-7},{0-3}   |   {4-7},{0-5}   |
|   BALANCE 2  |     XXX     | {2-3},{4-----5} |       XXX       |   {6-7},{0-1}   |
|--------------|-------------|-----------------|-----------------|-----------------|
|   DOMAIN  3  |     0-7     |       0-7       |       0-7       |       0-7       |
|   GROUP   3  | {0-5},{4-7} | {0-5},{0-1,4-7} | {0-1,4-7},{0-3} | {0-1,4-7},{0-5} |
|   BALANCE 3  | {0-1},{6-7} | {2-3},{6-----7} | {4-----5},{2-3} | {6-----7},{2-3} |


### 2.2.4 cpu_attach_domain 中 sched_debug 信息
-------


在这个过程中, 我们发现整个 DOMAIN-2 层级都有问题. 他的所有 sched_group 的 span 域的并集 groupmask 超出了 sched_domain span 的范围.
因此如果开启了 sched_debug 后, 会打印一些调度域 sched_domain 和 sched_group 的信息. 调试信息通过如下路径打印.

```cpp
cpu_attach_domain
    -=> sched_domain_debug
        -=> sched_domain_debug_one
```

也正是在这里面发现了, 如果这两个 span 信息有差异, 那么会提示用户

```cpp
static int sched_domain_debug_one(struct sched_domain *sd, int cpu, int level,
                  struct cpumask *groupmask)
{
    // ......
    if (!cpumask_equal(sched_domain_span(sd), groupmask))
        printk(KERN_ERR "ERROR: groups don't span domain->span\n");
    // ......
}
```

### 2.2.5 3 跳问题触发的深层次原因 
-------


但是有几个问题

1.  这个信息需要用户手动开启 sched_debug 才能发现, 有没有更直观的办法呢?

2.  这个现象更外在的出发原因是什么?

我们接着看 :

如果两个 NUMA 节点之间的最短距离 numa_distance 需要两个或者两个以上的节点来完成.

就比如 numa_distance(i, j) = numa_distance(i, k) + numa_distance(k, j)

```cpp
      A       B
  i <---> k <---> j
```

| node |   i   |   k  |    j   |
|:----:|:-----:|:----:|:------:|
|  i  | 10     | 10+A | 10+A+B |
|  k  | 10+A   | 10   | 10+B   |
|  j  | 10+A+B | 10+B | 10     |


那么 sched_init_numa 阶段根据 NUMA DISTANCE 构建的阶梯可能有这几个节点:

```cpp
10, 10+A, 10+B, 10+A+B
```

最多 4 个 NUMA LEVEL, 最少 3 个 NUMA LEVEL.

因此两跳不会出现问题.

如果两个 NUMA 节点之间的最短距离 numa_distance 需要三个或者三个以上的节点来完成.

就比如 numa_distance(i, j) = numa_distance(i, m) + numa_distance(m, n) + numa_distance(n, j)

```cpp
      A       B       C
  i <---> m <---> n <---> j
```

| node |     i     |    m   |   n    |    j     |
|:----:|:---------:|:------:|:------:|:--------:|
|   i  |  10       | 10+A   | 10+A+B | 10+A+B+C |
|   m  |  10+A     | 10     | 10+B   | 10+B+C   |
|   n  |  10+A+B   | 10+B   | 10     | 10+C     |
|   j  |  10+A+B+C | 10+B+C | 10+C   | 10       |

那么 sched_init_numa 阶段根据 NUMA DISTANCE 构建的阶梯可能有这几个节点:

```cpp
10, 10+A, 10+B, 10+C, 10+A+B, 10+B+C, 10+A+B+C
```

最多可能有 7 个 NUMA LEVEL, 最少 4 个 NUMA LEVEL.

这样在 

由于 10 < 10+A < 10+A+B < 10+A+B+C, 因此 NODE i 所在的 NUMA 层级如下:

*   NUMA DOMAIN 0 的时候, sched_domain 只包含 1 个 NODE, 即 span {i}

*   NUMA DOMAIN 1 的时候, sched_domain 可能包含 2 个 NODE, 即 span {i, m}, groups {i}, {m}

*   NUMA DOMAIN 2 的时候, sched_domain (span {i, m}) 会尝试再加一个 NODE n 加进来, 此时加进来的节点可能只有 1 个, 但是根据上面的算法, 内核总是把自己上一个层级的 domain (span {i, m})作为一个 group(包含了两个 NODE {i,m}), 新加入的 CPU 的上一级 DOMAIN (span {n,j}) 作为一个新的 group 加进来, 此时 sched_domain 中只加入一个 NODE n 的 CPU, 但是加入的 group span 却包含了 2 个 NODE 的(n 和 j).

综上: 3 个 NUMA 节点不会出现问题, 4 个 NUMA 节点的 NUMA LEVEL 2 才会出现问题, 此时



# 3 问题的影响
-------

# 4 问题修复
-------

`Valentin Schneider` 在发现这个问题之后给出的解决方案 [`sched/topology: Fix overlapping sched_group build`](https://lore.kernel.org/patchwork/patch/1214752), 就是

还记得前面根据 NUMA 距离阶梯, 算出来的各个 `NUMA NODE` 各个层级上的 `cpumask` 信息么

| 距离 | CPU0/1 | CPU2/3 |  CPU4/5   |  CPU6/7   |
|:---:|:------:|:------:|:---------:|:---------:|
| 10 |  {0-1}  |  {2-3} |   {4-5}   |   {6-7}   |
| 12 |  {0-3}  |  {0-3} |   {4-7}   |   {4-7}   |
| 20 |  {0-5}  |  {0-3} | {0-1,4-7} |   {4-7}   |
| 22 |  {0-7}  |  {0-5} |   {0-7}   | {0-1,4-7} |
| 24 |         |  {0-7} |           |   {0-7}   |


最终将依照这个来将各个 `CPU` (或者说各个 `NUMA NODE`) 跟调度域 `sched_domain` 绑定(`attach`).


| SCHED_DOAMIN |   NODE  0   |     NODE  1     |     NODE  2     |     NODE  3     |
|:------------:|:-----------:|:---------------:|:---------------:|:---------------:|
|   DOMAIN 0   |     0-1     |       2-3       |       4-5       |       6-7       |
|   GROUP  0   |   {0},{1}   |     {2},{3}     |     {4},{5}     |     {6},{7}     |
|--------------|-------------|-----------------|-----------------|-----------------|
|   DOMAIN 1   |     0-3     |       0-3       |       4-7       |       4-7       |
|   GROUP  1   | {0-1},{2-3} |   {2-3},{0-1}   |   {4-5},{6-7}   |   {6-7},{4-5}   |
|--------------|-------------|-----------------|-----------------|-----------------|
|   DOMAIN 2   |     0-5     |       0-5       |     0-1,4-7     |     0-1,4-7     |
|   GROUP  2   | {0-3},{4-5} | {0-3},{0-1,4-5} |   {4-7},{0-1}   | {4-7},{0-1,4-5} |
|   MASK   2   |     XXX     | {0-1},{0-----1} |   {4-5},{0-3}   | {XXX},{0-----1} |
|--------------|-------------|-----------------|-----------------|-----------------|
|   DOMAIN 3   |     0-7     |       0-7       |       0-7       |       0-7       |
|   GROUP  3   | {0-5},{4-7} | {0-5},{0-1,4-7} | {0-1,4-7},{0-3} | {0-1,4-7},{0-5} |
|   MASK   3   | {0-1},{XXX} | {2-3},{6-----7} | {4-----7},{0-1} | {6-----7},{2-3} |



<br>

*	本作品/博文 ( [AderStep-紫夜阑珊-青伶巷草 Copyright ©2013-2017](http://blog.csdn.net/gatieme) ), 由 [成坚(gatieme)](http://blog.csdn.net/gatieme) 创作.

*	采用<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/"><img alt="知识共享许可协议" style="border-width:0" src="https://i.creativecommons.org/l/by-nc-sa/4.0/88x31.png" /></a><a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/">知识共享署名-非商业性使用-相同方式共享 4.0 国际许可协议</a>进行许可. 欢迎转载、使用、重新发布, 但务必保留文章署名[成坚gatieme](http://blog.csdn.net/gatieme) ( 包含链接: http://blog.csdn.net/gatieme ), 不得用于商业目的.

*	基于本文修改后的作品务必以相同的许可发布. 如有任何疑问, 请与我联系.
