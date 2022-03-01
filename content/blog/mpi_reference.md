---
title: MPI 入门笔记
date: 2022-03-01 01:04:44
---

周二的高性能计算导论课上提到了 MPI，小作业里也用到了，但是代码没太看懂。于是寻找了一份 [MPI 的教程](https://mpitutorial.com/tutorials/mpi-introduction/zh_cn/)，中文翻译质量不错但必要时还是得看英文。

## Basic Facts

- MPI 是 Message Passing Interface 的简称

- MPI 是一套标准，而实际的实现有许多
    - 包括但不限于 OpenMPI（conv 集群上使用的）和 MPICH（教程中使用的）

- MPI 为包括 Fortran77, C, Fortran90 和 C++ 在内的一系列编程语言提供了并行函数库

- MPI 是进程 (process) 级并行，process 间内存不共享，与线程 (thread) 级并行不同

## 简单使用指南

### 基本要求

- MPI 程序入口处应当调用 `MPI_Init(int* argc, char*** argv)` 函数，实际上两个参数可以都填 `NULL`。

- MPI 程序结束前应当调用 `MPI_Finalize()` 函数

### 进程间点对点通信

- 使用 `MPI_Comm_size(MPI_COMM_WORLD, &world_size)` 来获取总进程数，保存在 `world_size` 中

- 使用 `MPI_Comm_rank(MPI_COMM_WORLD, &world_rank)` 来获取自己的进程编号，保存在 `world_rank` 中

- 使用 `MPI_Send(const void *buf, int count, MPI_Datatype datatype, int dest, int tag, MPI_Comm comm)` 来从一个 process 将数据发送到另一个 process
    - buf 是待发送缓冲区
    - count 是发送数据数目
    - datatype 是数据类型，MPI 内部自己的枚举类，与 C 原生类型存在对应
    - dest 是目标 process 编号
    - tag 是编号，只有 tag 对应的消息才会被接收
    - 最后一个参数干啥的啊，我现在只知道无脑填 `MPI_COMM_WORLD`

- 使用 `MPI_Recv(void *buf, int count, MPI_Datatype datatype, int source, int tag, MPI_Comm comm, MPI_Status *status)` 来接收另一个 process 发来的数据
    - buf 是接收用缓冲区
    - count 是最大接收数据数目
        - 可以接收不超过这个数目的数据，如果超过会报错
    - datatype 是数据类型
    - source 是源 process 编号
        - 可以用 `MPI_ANY_SOURCE` 来接收所有源的消息
    - tag 是编号，只有 tag 对应的消息才会被接收
        - 可以用 `MPI_ANY_TAG` 来接收所有任意 tag 的消息
    - 我还是只知道无脑填 `MPI_COMM_WORLD`
    - status 是一个指向 `MPI_Status` 结构体的指针，这次接收的更多信息会保存在这个结构体里
        - 可以用 `MPI_STATUS_IGNORE` 来丢弃这个东西

- `MPI_Status` 结构体中记录了这些东西
    - 发送端的进程编号
    - 消息的 tag
    - 消息的长度（在指定类型后，元素个数也就可以知道了）

- 使用 `MPI_Get_count(MPI_Status* status, MPI_Datatype datatype, int* count)` 来得到消息中的元素个数。

- 使用 `MPI_Probe(int source, int tag, MPI_Comm comm, MPI_Status* status)` 来在不接收的情况下得到 `MPI_Status`。

### 多进程同步

WIP