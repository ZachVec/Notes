# PROJECT #2 - B+Tree Index

本实验为CMU-15445(fa20)系列第3个实验，实验spec[在此](https://15445.courses.cs.cmu.edu/fall2020/project2/)。

## Specification

本次实验分为两个部分，checkpoint1 和 checkpoint2。

- checkpoint1：完成B+树的插入，分裂。
- checkpoint2：完成B+树的删除，合并，重新分布，以及并发。

## checkpoint1

本部分完成B+树的插入，分裂。在那之前，先介绍下实验中可能碰到麻烦的变量及方法。首先需要明确一点：叶节点的大小和内部节点的大小完全独立，互不影响。

`size`：size与树的度、指针数、键的数目没有关系。在官方spec中定义为页（结点）中键值对的个数。除此之外，需要注意内部结点的size是包含开头的假结点的，而叶节点并没有假结点。

`min_size`：小于这个值就该合并了。

`MoveHalfTo`：这个方法是在split中调用的。如果是内部结点的话，还需要修改被移走的结点的父节点页号。

`Split`：分裂函数。需要进行`reinterpret_cast`来调用对应的`MoveHalfTo`。此外，分裂的时机要把握对：

- 对于叶节点：如果在插入后到达MaxSize，那就直接分裂。
- 对于内部节点：在插入之前就到MaxSize，那就先分裂，再插入。

> 其实内部结点好像没有测试用例测，但是看discord以及piazza上面，总结了三种实现可能：
> 
> 1. 插入之前到MaxSize，分裂，插入。
> 2. 插入之后到MaxSize，分裂。
> 3. 插入之后到MaxSize+1，分裂。
> 
> 其实 1 3 是一样的，只是分裂早晚的问题。但是当 max_size 为宏定义的最大值时，再插入会导致数据溢出（超过page size），所以第3种排除。至于第2种，这样分裂在max_size = 3的情况下会产生一堆只有开头假结点的内部结点。所以最终我还是觉得第一种实现比较靠谱。

最终，如果按照 `1 2 3 4 5` 的顺序插入到B+树中，B+树应该像如下图所示：

<div style="text-align:center"><img src="./assets/Insertion.svg" alt="Insertion"></div>

checkpoint 1 差不多就这么多。

## checkpoint1
