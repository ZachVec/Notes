# PROJECT #0 - C++ PRIMER

本实验为 CMU - 15445 (fa20) 系列的第一个实验，实验指导[在此](https://15445.courses.cs.cmu.edu/fall2020/project0/)。主要是配好环境，以及写少量代码完成矩阵的加法乘法和简化的通用矩阵乘法，最终熟悉一下提交代码的流程。实验前要先熟悉C++相关知识，如：`virtual`, `unique_ptr`, `move`, `template`以及C++ OOP。本次实验所有代码均在`src/include/primer/p0_starter.h`下完成。最终测试时，需要把`test/primer/starter_test.cpp`中每个测试名称头部的`DISABLED`去掉来进行测试。

## 注意事项

整个实验没有什么难的地方，最难的地方在于配环境和提交作业了。配环境时，输入如下指令之后：

```shell
$ mkdir build
$ cd build
$ cmake ..
```

会自动从 GitHub 上面用`https`协议`git clone` Google Test 套件。但是由于GFW，从 GitHub 上面用`https`协议`git clone`多少有些不现实。经过一天的折磨后，最后我发现可以用如下指令把`https`协议替换成`git`协议来下载：

```shell
$ git config --global url."git@github.com:".insteadOf https://github.com/
```

当然，如果你不想设置成全局的，你也可以把`--global`去掉。至此，终于可以愉快的码代码了。PS：其实还有一个问题，就是vscode+ssh连到阿里云ECS上面的时候，server端的vscode的C++插件一直下载不下来。解决方案：下载对应版本的离线安装包（`remote`用`wget`或`host`直接用浏览器下都可以，但是`host`下载显然更快，然后搜索下vscode如何安装`vsix`，照着安就行）。

在提交前，要先看看对不对：

```shell
~/15445/build$ make starter_test
~/15445/build$ ./test/starter_test
~/15445/build$ make format
~/15445/build$ make check-lint
~/15445/build$ make check-clang-tidy
```

最后都没啥问题了就可以提交了。最后提交的时候需要`zip`打个压缩包，没有zip的直接用`apt`安就好了。需要注意的是打包文件的父级目录都要打进去：

```shell
~/15445$ zip project0-submission.zip src/ src/include/ src/include/primer/ src/include/primer/p0_starter.h
```

最后把压缩包搞到`host`然后提交到`Gradescope`上。另外，就算本地全过了，`Gradescope`也不一定能过。注意看`Gradescope`上面的报错。PS：吐槽一下，为啥不能学学CS61B直接push到GitHub然后测啊。

## 实验结果

本次实验比较简单（除了配环境），最后`Gradescope`上的结果如下：

```shell
STUDENT
ZachVec
AUTOGRADER SCORE
100.0 / 0.0
PASSED TESTS
test_Starter_Add (__main__.TestProject1) (20.0/20.0)
test_Starter_Gemm (__main__.TestProject1) (20.0/20.0)
test_Starter_Multiply (__main__.TestProject1) (20.0/20.0)
test_Starter_SampleTest (__main__.TestProject1) (20.0/20.0)
test_build (__main__.TestProject1) (0.0/0.0)
test_memory_safety (__main__.TestProject1) (20.0/20.0)
```

算是完成了第一个实验把。PS：最后还用了CSAPP里的矩阵乘法优化（`cache lab`里讲的那个），应该可以显著提高`cache hit`的概率，进而提高吞吐量。


