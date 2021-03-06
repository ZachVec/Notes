---
title : Lecture 05: Buffer Pools
author: vector
source: https://15445.courses.cs.cmu.edu/fall2020/notes/05-bufferpool.pdf
---

> 本系列主要不再是搬运，而是谈谈个人理解以及一些要点。

# Lecture #05: Buffer Pools

## 1 Introduction

在数据库中，由于数据可能远大于内存大小，DBMS需要负责内存管理，即在硬盘与内存之间来回地移动数据，并把由数据移动而引入的开销尽可能地降低。理想情况下，对于查询过程而言，应该有所有的数据看起来好像都放在了内存中。

## 2 Locks vs. Latches

我们需要搞清楚 **Locks** 和 **Latches** 的区别：

**Locks**

- 用来保护数据库内容。每次业务（transaction）在持续时间内都要持有 Locks
- 更高层、逻辑上的原子性
- 数据库系统可以把正在运行的 query 持有的 Locks 暴露给用户。
- Locks 需要能够回滚（rollback）。

**Latches**

- 用来保护 DBMS 内部数据结构，防止操作进入临界区。每个操作在持续时间内都需要持有 Latches
- 更底层的原子性。
- 不需要能够回滚。

> 这里的 Lock 与操作系统中的 Lock 不一样，相比较而言，Lock 更像是文件系统中的 Log。
>
> - 它们都是更高层的逻辑锁
> - Lock 需要能够回滚
> - 文件系统中虽然不需要回滚，但在Crash Recovery里需要完成没有完成的IO。
>
> 而这里的 Latch，在我看来与操作系统中的 Lock一致（事实上，我觉得它们就是一个东西）
>
> - 都是底层的，依赖硬件
> - 用来保护并发访问的数据
> - 不需要回滚。

## 3 Buffer Pool

这里我把 Buffer Poll 叫做 **缓存池**。基本上就是由 DBMS 进程分配的一大块内存区域，用来存放来自硬盘的页。

> 这里的页与OS中的页不是一个东西，基本等价于硬盘中的块。页的大小取决于数据库的实现。典型值为1~16KB。PostgreSQL 中为8KB。

缓存池所在内存区域被组织成一组形式、固定数量的页，组内单个*条目*（entry）叫做*帧*（frame）。当DBMS索要页时，DBMS就在缓存池中寻找页，如果没有找到，系统就会从硬盘中取出该页，放到缓存池里。

### 缓存池元数据（meta-data）

为了高效使用缓存池，必须要维护元数据，即页表。与OS的页表类似，DBMS页表的功能如下：

- 记录着 `page id` 与 `frame` 的映射。
- 维护着每一页额外的元数据，如：dirty-flag，pin/reference counter等。 reference counter大于0时，不允许驱逐。

> 页表(page table) vs. 页目录(page directory)
>
> - 页表记录的是内存中页在缓存池中的位置，页目录记录的是页在硬盘中的位置。
> - 页表不需要具有持续性（persistency），而页目录需要具有持续性。

### 内存分配策略

数据库中的内存分配策略分为两种：

- 全局策略：考虑所有正在进行的业务，找到分配内存的最佳策略。使整个workload受益。
- 局部策略：让单个查询或业务运行得更快，不考虑整个workload。

大部分系统都使用混合的策略。

## 4 Buffer Pool Optimizations

### Multiple Buffer Pools

多建几个缓存池，可有效减少锁竞争并提升局部性。下面说两种将页映射到缓存池中的方法：

- Object IDs：把数据的ID拓展，使之包括元数据。然后让每个东西（object）都去对应的缓存池。
- Hashing： Hash 一下`page id`，然后用该结果找到对应的缓存池。

### Pre-fetching

DBMS可以针对 query plan 预先缓存页。比如：在前一个的页正在被处理的时候，后一个的页就可以被预先取出放到缓存池里。这个方案在对大量的页连续读写时被广泛采用。

### Scan Sharing

Query Cursor可以在多个Query之间共享，比如下面两个query：

```sql
SELECT SUM(num) FROM A;
SELECT AVG(num) FROM A;
```

假设表A共有5页，第一条query已经执行到了第3页，第二条才刚开始。

如果第二条也从头开始的话，在缓存池不够大的情况下，可能会驱逐即将要读的页（由第一条语句缓存的）。这样的话，基本少第二条query的所有页都需要从硬盘读，所以让第一条和第二条共用一个cursor。一起读完后，第二条需要再从头开始把没有读上的读了。

### Buffer Pool Bypass

对表的连续读写可以不放在缓存池里，借此来避免开销（因为可能只有这一个query用，但它却把缓存池几乎刷新了一遍）。相应地，把这些数据放在本地（线程里）。这对于处理连续分布在硬盘上的一大串页很有用，对于临时数据也很有用（如sorting，joins）

## 5 OS Page Cache

由于大多数硬盘操作都要走OS API。除非特别说明，否则OS就会在文件系统中进行缓存。

大多数 DBMS 都会用 *direct I/O* 来说明不希望OS的文件系统进行缓存，避免额外的开销。

Postgres 是一个为数不多的、使用OS文件系统缓存的数据库系统。

## 6 Buffer Replacement Policies

当缓存池不够用了，DBMS 就必须决定驱逐哪一页。替换策略的实现目标是更高的正确性、更快及更小的元数据额外开销。

### Least Recently Used

给每一页都维护一个最近访问的时间戳。驱逐页时，DBMS会驱逐一个最久不用的页。

### CLOCK

这就是实验要用的。它与LRU类似，每一页都有个访问位，如果这一页被访问（读或写），则就把这个参考位置1.

在驱逐的时候，从**当前位置**（不是从头）开始，如果：

- 参考位为1：把参考位置0，去看下一页
- 参考位为0：直接驱逐（还有可能的写回）

如果转了一圈发现所有的参考位都为1，由于已经把开始的1置为0了，所以可以驱逐开始位置的页。指向页的指针就像时针一样转动，可以看原视频获得更佳理解。

### Alternatives

由于LRU和CLOCK在 *sequential flooding* 的时候表现不佳：有的数据可能读一下就不要了，有的数据要反复读，但是新读经来的、读一下的数据的时间戳比要老的、需要反复读的数据更大，故而优先驱逐需要的数据，而不是不需要的数据。换言之，最近使用过的数据可能不是我们想要的数据。针对这个问题，有三种方法：

- LRU-K：记录每一页的最近K次访问时间，并计算它们之间的时间差。
- localization： DBMS基于每一个query来驱逐页，不驱逐别的query的页。
- priority hints：给每一页打上优先级。

### Dirty Pages

在驱逐时，如果页被写过，在写回时需要写硬盘，这样就会很慢。如果有两个页可以驱逐，那么优先选没有被写过的页驱逐。

但是，早晚都要驱逐的，这个写操作是否无法避免呢？我们可以通过 *background writing* 来避免这个问题：DBMS在后台会周期性的遍历页表，并把脏页写回，写完之后把脏位置0.
