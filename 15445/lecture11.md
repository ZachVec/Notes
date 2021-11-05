---
title : Lecture 11: Joins Algorithms
author: vector
source: https://15445.courses.cs.cmu.edu/fall2019/notes/11-joins.pdf
---

# Lecture 11: Joins Algorithms

## Joins

一个好的数据库设计的目标是尽量减少信息重复的数量，这就是我们基于 **规范化理论 (Normalization Theory)** 建表的原因。 因此需要 `join` 来重建原始表。

### Operator Output

对于 r ∈ R 和 s ∈ S 来说，如果它们 `join` 的属性一样，那么 `join` 就会把 r 和 s 连接起来，生成一个新的 tuple，输出内容共有两种可能：

- **Data**: 把来自 outer 和 inner tuple 的数据直接写到新的输出 tuple 中，称为 *early materialization*
  - 好处：最后不用再去磁盘拿数据了
  - 坏处：使用的内存更多了
- **Record Ids**: DBMS 只会写匹配的 *键* 和 *record id*，称为 *late materialization*
  - 这样对于 column stores 数据库非常理想，因为 DBMS 不会复制 query 用不到的数据。

> `Join` 的输出内容取决于 *Processing Model*, *Storage Model* 以及 *data requirements in query*.

### Cost Analysis

> ``` SQL
> SELECT R.id, S.cdate
> FROM R JOIN S
> ON R.id = S.id
> WHERE S.value > 100
> ```
>
> 假设表 R 中有 M 页， m 个 tuple；类似地，S 中有 N 页， n 个 tuple.

开销分析 (Cost Analysis) 中的开销衡量标准为： 计算 `join` 所需 IO 的次数（读写一页都消耗一次 IO）

## Nested Loop Join

分为 Nested Loop Join (Simple/stupid, Block, Index)，Sort-Merge Join 以及 Hash Join. 

总体而言，这些 `join` 算法都有两层嵌套 `for` 循环来在两个表中迭代，如果有匹配就输出。外层循环表称为 *外层表 outer table*，内层循环表称为 *内层表 inner table*，且 DBMS 总会选择更小的表作为 *外层表*，即页数更少的表。除此之外，DBMS 也更倾向于在缓存中放入更多的 *外层表*。

在以下不同的算法中，我们假设 M = 1000, m=100,000; N = 500, n = 40,000; 0.1 ms/IO；借此来衡量不同算法的开销。

### Simple Nested Loop Join

```pseudocode
foreach tuple r in R: 		// <==== This is outer table, M pages, m tuples
	foreach tuple s in S:	// <==== This is inner table, N pages, n tuples
		emit, if r and s match
```

最糟情况为 *外层表* 遍历每一个 r 时，需要重新遍历一遍 *内层表*，即每一次都用不到之前缓存的结果。

- Cost = *M + (m × N)*；
- 对于假设，Cost = 501,000 IOs，大约需要 1.1 小时 (S 作为 outer table, R 作为 inner table，因为 S 更小)

### Block Nested Loop Join

```pseudocode
foreach block BR in R:
	foreach block BS in S:
		foreach tuple r in BR:
			foreach tuple s in BS:
				emit, if r and s match
```

与前一个类似，但是用上缓存了，减少了硬盘读写。对于 R 中的每一块，都会遍历一遍 S。注意这里说的是块，不是页。

- 如果缓冲池只有 3 页：*Cost = M + （M · N)* ，一页放 R 的页，一页用来遍历 S 的页，一页用来输出。此时 BR = 1 
  - 对于假设 Cost = 501,000 IOs假设 0.1 ms/IO，则大约需要 50 秒。
- 如果缓冲池有 B > 3 页：*Cost = M + (*⌈*M / (B - 2)*⌉ *· N)*，一页用来放 S，一页用来输出，剩下的放 R。此时 BR = B - 2
- 如果缓冲池有 B > M + 2 页： Cost = M + N，把 R 直接全放进缓冲池。此时 BR = M

> 为什么嵌套循环 `join` 效果这么差？因为对于外表中的每一个 tuple 而言，我们都得遍历一遍 inner table。我们用索引来避免遍历：可以直接用索引，也可以临时建一个。

### Index Nested Loop Join

```pseudocode
foreach tuple r in R:						// <==== Outer table doesn't have index
	foreach tuple s in Index(ri = sj):		// <==== Inner table does
		emit, if r and s match
```

如果索引是已经存在的，则 cost = *M + (m × C)*，其中 C 为在内层表中索引外层 tuple r 的时间，假定为常数。

## Sort-Merge Join

> 当至少有一个表有序，或是要求输出有序时，用这种方法。

**阶段一** Sort: 基于 `join` 的键给两个表排序，可以使用上一讲的外部排序算法

**阶段二** Merge: 遍历两个有序表，输出匹配到的 tuple；可能需要回溯，这取决于 `join` 的类型

```pseudocode
sort R, S on join keys
cursor_R = R_sorted, cursor_S = S_sorted
while cursor_R and cursor_S:
	if cursor_R > cursor_S:
		cursor_S = cursor_S + 1
	if cursor_R < cursor_S:
		cursor_R = cursor_R + 1
	elif cursor_R and cursor_S match:
		emit
		cursor_S = cursor_S + 1
```

- Cost = Sort + Merge
  - Sort Cost (R)：2M · (1 + ⌈ ㏒<sub>B-1</sub>⌈ M / B ⌉⌉)
  - Sort Cost (S)：2N · (1 + ⌈ ㏒<sub>B-1</sub>⌈ N / B ⌉⌉)
  - Merge Cost：( M+N )
- M = 1000, m = 100,000; N = 500, n = 40,000
  - Sort Cost (R) = 4000 IOs
  - Sort Cost (S) = 2000 IOs
  - Merge Cost = 1500 IOs
  - Total Cost = 7500 IOs. At 0.1 ms/IO, Total time ≈ 0.75 s.

## Hash Join

### Basic Hash Join Algorithm

**第一阶段**：Build。 用 h<sub>1</sub> 作用在 R 的 join key上，建立哈希表。

**第二阶段**：Probe。遍历内表，用 h<sub>1</sub> 作用于 S 的 join key，看看哈希表里有没有匹配，有就输出。

```pseudocode
build hash table HashTableR for R
foreach tuple s in S:
	emit, if hash(s) in HashTableR
```

哈希表的内容：键值为 join key 的哈希值，值取决于实现，可能为

- Full tuple: 可以避免之后再读硬盘，但是需要更多内存。
- Tuple Identifier: 对 columns stores 很理想。

> 第二阶段 Probe 还可以使用 *Bloom Filter* 优化。Bloom Filter 是一个 Probabilistic data structure (bitmap)，可以用来查询元素是否存在。虽然它说元素存在的时候，元素可能不存在，但是它说元素不存在的时候，元素一定不存在。
>
> 具体应用为：在第一阶段的时候建立一个 Bloom Filter，然后在第二阶段，线程会在 probing 哈希表之前先问问 filter. 这应该很快，因为 filter 非常小，能放进 CPU cache 中。
>
> 有时叫做 *sideways information passing*.

### Grace Hash Join / Hybrid Hash Join

但是缓存池中如果放不下哈希表呢？我们可以用 Grace Hash Join.

**第一阶段 Build**: 对两张表都做哈希，分到对应的桶中。

**第二阶段 Probe**: 在对应的桶中 Nested Loop Join.

```pseudocode
buckets_R = hash1(R); buckets_S = hash1(S)
foreach bucket_R, bucket_S in zip(buckets_R, buckets_S):
	foreach tuple r in bucket_R:
		foreach tuple s in bucket_S:
			emit, if match(r, s)
```

如果一个桶还是放不到缓存池中，那么可以递归地哈希（换一个哈希函数）直到能放下。

- Cost = 3 (M + N)
  - M = 1000, m = 100,000
  - N = 500, n = 40,000
  - Cost = 3 (1000 + 500), assuming that 0.1ms / IO, then total time = 0.45 seconds
