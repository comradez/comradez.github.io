+++
title = "为什么我不再使用 Anaconda？"
date = "2022-06-16 21:03:37"
authors = ["茨月"]
+++

本文讨论了 Anaconda 的好与坏，以及为什么我不再使用它。

<!-- more -->

今天终于下定决心扬掉了占用十余个 GiB 的 Anaconda. Anaconda 在我学习和使用 `Python` 的路径中起到了很大的作用，我也很感谢它的存在。

但是，我现在放弃了 Anaconda，并且希望学习计算机专业的读者也不要使用它。原因有三个：

## 独立于系统的编译器/动态库

Linux 系统中自带的 Python 都是 CPython，由 C/C++ 编译器编译而来。因此，如果使用系统自带的编译器编译 [pybind11](https://pybind11.readthedocs.io/en/stable/) 工程，那么得到的模块可以直接被系统 Python 识别。

Anaconda 的 Python 则不然：为了保证行为的一致性和使用方便，Anaconda 会采用预构建的二进制文件和对应版本的动态库，独立于系统的编译器和动态库之外。因此，Anaconda 的虚拟环境中的 Python 和系统编译器 & 动态库的互操作性很差。这在我参与的一个用到了 pybind11（我们需要将 C++ 接口导出到 Python）的工程中带来了不小的问题。

不可否认的是，这种 「另立中央」 的方案使得 Anaconda 可以开箱即用，但是其代价也非常巨大。

## 污染 PATH

如上文所述，Anaconda 自己携带了二进制文件和动态库，而且它携带的二进制文件远不止于 Python 自己，包括 OpenSSL, mpic{c|xx} 等一系列工具它都有包含。这些工具为 Anaconda 维护属于自己的运行环境提供了很大的方便，但是当 conda 将它的 bin 添加到 PATH 后，它将会污染系统的 PATH，盖住系统 openssl 等很多关键的文件，导致**极端难以调试**的神秘 bug.

## 空间占用极大

其实原因也在上文里了。Anaconda 为了维护自己的环境，需要安装各种神奇的包和工具。在我删掉 Anaconda 之前，它占用了我 11.4GiB 空间。

## 那用什么呢？

我现在正在使用 [poetry](https://python-poetry.org/)。如果需要手动多版本依赖，可以考虑 [pyenv](https://github.com/pyenv/pyenv) 配合 [pyenv-virtualenv](https://github.com/pyenv/pyenv-virtualenv) 插件。
