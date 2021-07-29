# PROJECT #1 - Buffer Pool

本实验为 CMU - 15445 (fa20) 系列的第二个实验，实验Spec[在此](https://15445.courses.cs.cmu.edu/fall2020/project1/)。

## Overview

实验上实现两个东西，一个是`LRUReplacer`，一个是`BufferPoolManager`。实验相对简单，但如果想要提高并行度就要多想想了。

`LRUReplacer`是用来管理处于`unpin`状态的`frame_id`的，在必要的时候将最近少用(lease recently used)的`frame_id`驱逐回收。

`BufferPoolManager`是用来对外提供一个抽象，只通过`page_id`来管理页，使得`frame_id`对于外部透明。其内部维护着页表，用来记录`page_id`与`frame_id`之间的关系。页表(`page_table`)与页目录(`page_directory`)之间的关系如下：

![page table and page directory](./assets/BufferPoolAndDisk.svg)

## 注意事项

整个实验按照spec、代码里的注释以及如下状态机完成，应该没有啥问题。

![Frame FSM](./assets/FrameFsm.svg)

然后我写一个打包脚本，如下

```bash
#!/usr/bin/env bash

print_help() {
    echo -e "Usage:\n"
    echo -e "  $ ./auto-archive proj#\n"
    echo -e "Example:\n"
    echo -e "  $ ./auto-archive 0\n"
    echo -e "This will archive project 0. Choices:\n"
    echo -e "  0 - C++ Primer"
    echo -e "  1 - Buffer Pool Manager"
    echo -e "  2 - B+ Tree Index"
    echo -e "  3 - Query Execution"
    echo -e "  4 - Concurrency Control"
}

out="proj$1-submission"
if [[ -f ./$out.zip ]]; then
    rm $out.zip
fi

case $1 in
  4)
    zip $out -urq \
    src/concurrency/lock_manager.cpp \
    src/include/concurrency/lock_manager.h
  ;&
  3)
    zip $out -urq \
    src/include/catalog/catalog.h \
    src/include/execution/execution_engine.h \
    src/include/execution/executor_factory.h \
    src/include/execution/executors/seq_scan_executor.h \
    src/include/execution/executors/index_scan_executor.h \
    src/include/execution/executors/insert_executor.h \
    src/include/execution/executors/update_executor.h \
    src/include/execution/executors/delete_executor.h \
    src/include/execution/executors/nested_loop_join_executor.h \
    src/include/execution/executors/nested_index_join_executor.h \
    src/include/execution/executors/limit_executor.h \
    src/include/execution/executors/aggregation_executor.h \
    src/include/storage/index/b_plus_tree_index.h \
    src/include/storage/index/index.h \
    src/execution/executor_factory.cpp \
    src/execution/seq_scan_executor.cpp \
    src/execution/index_scan_executor.cpp \
    src/execution/insert_executor.cpp \
    src/execution/update_executor.cpp \
    src/execution/delete_executor.cpp \
    src/execution/nested_loop_join_executor.cpp \
    src/execution/nested_index_join_executor.cpp \
    src/execution/limit_executor.cpp \
    src/execution/aggregation_executor.cpp \
    src/storage/index/b_plus_tree_index.cpp
  ;&
  2)
    zip $out -urq \
    src/include/storage/page/b_plus_tree_page.h \
    src/storage/page/b_plus_tree_page.cpp \
    src/include/storage/page/b_plus_tree_internal_page.h \
    src/storage/page/b_plus_tree_internal_page.cpp \
    src/include/storage/page/b_plus_tree_leaf_page.h \
    src/storage/page/b_plus_tree_leaf_page.cpp \
    src/include/storage/index/b_plus_tree.h \
    src/storage/index/b_plus_tree.cpp \
    src/include/storage/index/index_iterator.h \
    src/storage/index/index_iterator.cpp
  ;&
  1)
    zip $out -urq \
    src/include/buffer/lru_replacer.h \
    src/buffer/lru_replacer.cpp \
    src/include/buffer/buffer_pool_manager.h \
    src/buffer/buffer_pool_manager.cpp
  ;;
  0)
    zip $out -urq src/include/primer/p0_starter.h
  ;;
  *)
    echo "Invalid input: $1"
    print_help
    exit 1
  ;;
esac
echo "Files has been writen into $out.zip..."
unzip -l $out.zip
```

## 实验结果

本次实验难度适中，最后`Gradescope`上的结果如下：

```shell
STUDENT
ZachVec
AUTOGRADER SCORE
160.0 / 160.0
PASSED TESTS
test_BufferPoolManager_ConcurrencyTest (__main__.TestProject1) (10.0/10.0)
test_BufferPoolManager_DeletePage (__main__.TestProject1) (5.0/5.0)
test_BufferPoolManager_FetchPage (__main__.TestProject1) (5.0/5.0)
test_BufferPoolManager_HardTestA (__main__.TestProject1) (15.0/15.0)
test_BufferPoolManager_HardTestB (__main__.TestProject1) (15.0/15.0)
test_BufferPoolManager_HardTestC (__main__.TestProject1) (15.0/15.0)
test_BufferPoolManager_HardTestD (__main__.TestProject1) (15.0/15.0)
test_BufferPoolManager_IntegratedTest (__main__.TestProject1) (10.0/10.0)
test_BufferPoolManager_IsDirty (__main__.TestProject1) (5.0/5.0)
test_BufferPoolManager_NewPage (__main__.TestProject1) (5.0/5.0)
test_BufferPoolManager_SampleTest (__main__.TestProject1) (5.0/5.0)
test_BufferPoolManager_UnpinPage (__main__.TestProject1) (5.0/5.0)
test_ClockReplacer_ConcurrencyTest (__main__.TestProject1) (10.0/10.0)
test_ClockReplacer_IntegratedTest (__main__.TestProject1) (10.0/10.0)
test_ClockReplacer_Pin (__main__.TestProject1) (5.0/5.0)
test_ClockReplacer_SampleTest (__main__.TestProject1) (5.0/5.0)
test_ClockReplacer_Size (__main__.TestProject1) (5.0/5.0)
test_ClockReplacer_Victim (__main__.TestProject1) (5.0/5.0)
test_build (__main__.TestProject1) (0.0/0.0)
test_memory_safety (__main__.TestProject1) (10.0/10.0)
```

最后打算以后有空了看看能不能给`BufferPoolManager`的锁粒度整细一点。
