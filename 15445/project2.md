# PROJECT #2 - B+Tree Index

本实验为CMU-15445(fa20)系列第3个实验，实验spec[在此](https://15445.courses.cs.cmu.edu/fall2020/project2/)。

## Overview

本次实验分为两个部分，checkpoint1 和 checkpoint2。

- checkpoint1：完成B+树的插入，分裂。
- checkpoint2：完成B+树的删除，合并，重新分布，以及并发。

下面从我的实现角度来谈谈这个实验。在做实验之前，强烈建议先看看 [Database System Concepts 7th](https://www.db-book.com/db7/index.html) 14.1 到 14.4 部分。

## checkpoint1

> 本部分完成B+树的插入，分裂。在那之前，先介绍下实验中可能碰到麻烦的变量及方法。首先需要明确一点：叶节点的大小和内部节点的大小完全独立，互不影响。

### 名词解释

`size`：在官方 spec 中定义为结点中键值对的个数。

 - 对于 Internal Page 来说，是包含开头 Invalid Key 及其值在内的所有键值对的数量。
 - 对于 Leaf Page 来说，就是所有键值对的个数。

`max_size`：物理上键值对数量的最大值。如果大于这个数量，数据会溢出，修改到其他页的数据。因此，**在任何情况下都不能大于这个数量**。

### Q & A

Q: 关于分裂时机的选择？
A: 对于Leaf Page，如果在插入后到达 MaxSize，那就直接分裂。对于 Internal Page，在插入之前就到 MaxSize，那就先分裂，再插入。最终，如果按照 `1 2 3 4 5` 的顺序插入到B+树中，B+树应该如下图所示：

<div style="text-align:center"><img src="./assets/Insertion.svg" alt="Insertion"></div>

## checkpoint2

> 本部分需要实现数据的删除、树的迭代器以及树的并发访问。

### 名词解释

`min_size`：结点中键值对数量的逻辑最小值，小于这个值后就需要考虑是否需要 `Redistribute` 或是 `Coalesce` 了。但是根结点是个例外，根结点需要用 `AdjustRoot` 来调整，如果根节点是：

- Internal Page：经历 `Remove` 后根结点的 Size 为 1 时，需要调整。
- Leaf Page：经历 `Remove` 后根节点的 Size 为 0 时，需要调整。

### Q & A

Q: 为什么 `Coalesce` 的参数为二级指针？
A: 因为 `Coalesce` 内部可能需要交换一级指针所指向的东西。

Q: `Coalesce` 和 `Redistribute` 的参数 `index` 是什么？
A：根据注释描述，不难发现 `index==0` 暗含的意思是 `node` 在 `sibling` 之前，所以大胆推断，`index` 指的就是 `node` 在父结点中的位置。

Q: 单线程时，何时 `Unpin`？
A: 我的选择是，在哪个函数里 `Pin` 下的页，就在哪个函数里 `Unpin`。可以用 RAII 实现，但是没必要，因为多线程下需要用 `Transaction` 来管理页。

Q: 布尔返回值有何意义？
A: 可以指导你选择是否调用删除页的函数。

Q: 何时把需要删除的页加入`Transaction`？
A: 我的选择是在 `Coalesce`。当然还有在 `CoalesceOrRedistribute` 中，如果结点为根节点，且 `AjustRoot` 返回值为 `true` 时，也需要把结点加到 `transaction` 中。

Q: 为何单线程中 `BufferPoolManager::DeletePage()` 的返回值一定是 `true`，而多线程时却不一定了？
A: 单线程要删页自然是 `pin_count_` 为 0 了才能删（如果你的实现在单线程时返回 `false` 了，说明有问题）。多线程时删页时，可能之前有线程释放了锁（但是还没 `Unpin`），然后锁又被当前线程拿上，调用 `Remove` 后需要删除，但是之前那个线程还没有机会 `Unpin`，导致返回值为 `false`，所以多线程时不一定。

Q: 那如果多线程删页时返回 `false` 我该做什么？
A: 啥都不用做，由于树中已经不存在该页了，该页仅存在于 `BufferPoolManager` 中，随着页被换进和驱逐，这个页会自己出去的。但 `BufferPoolManager::DeletePage()` 的实现要严格参考注释。即使在返回 `false` 的情况下，也要调用 `DiskManager::DeallocatePage()`。

> 另外，还需注意加入到 `Transaction` 中，需要删除的页是否正确。需要注意 `Coalesce` 的参数是二级指针，所以在 `CoalesceOrRedistribute` 中可以看到被调换了。而 `CoalesceOrRedistribute` 的参数是一级指针，所以调用 `CoalesceOrRedistribute` 的函数并不知道 `node` 是否被修改。

删除示意图：

![Deletion](./assets/Deletion.svg)

## 结语

本次实验还是相当有难度的，边摸鱼边做实验花了不少时间。不过收获也不少，在做本次实验之前只是听说过 B+ 树，更遑论支持并发的 B+ 树了。截至本文写完，在 Leaderboard 上排名为 12.
