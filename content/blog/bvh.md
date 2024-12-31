+++
title = "渲染拾遗：BVH"
date = "2022-11-21 22:52:49"
authors = ["茨月"]
+++

本文讨论了光线追踪中常用的加速结构 BVH 的设计、构造与优化，内容较为基础，没有包含比较新的前沿算法。

<!-- more -->

最近打算补一补渲染中一些缺漏的知识，于是重新翻开了 pbrt，然后意识到我的知识漏洞比我想象的还要大一点……这次是 BVH 的笔记，因为我个人写渲染器的时候写的都是 KD-Tree，还从来没写过 BVH，更没写过优化，所以以此为契机重新学习一次这个实际应用中更常见的加速结构。

## 什么是 BVH

Bounding Volume Hierarchy，中文翻译「层次包围盒」。是 KD-Tree 之外的另一种加速光线求交的树状数据结构。与 KD-Tree 划分空间不同，BVH 划分的是物体，这是这两种加速结构的最大区别。

## BVH 的构造

BVH 的构造思路非常简单：首先，我们对场景中的每个物体都计算出其轴向包围盒（Axis-Aligned Bounding Box, AABB），然后我们自底向上，每次选择两个已有的轴向包围盒合并为一个新的、能够包住选中的两个包围盒的最小轴向包围盒并放到树上作为一个节点，直到得到包住所有物体的全局包围盒为止。

<div class="side-by-side-container"><img id="should-invert" src="https://www.pbr-book.org/3ed-2018/Primitives_and_Intersection_Acceleration/Primitives%20and%20hierarchy.svg" style="width: 60%"/></div>

实现中，我们首先自顶向下，从场景中的全体物体开始向下递归，每次都选择一个维度进行2-划分，再自底向上确定当前的包围盒。

这里就出现了一个一个问题：如何进行划分？

### 比较基础的：中点划分和等量划分

最简单的两种划分方式是[中点划分]((https://github.com/mmp/pbrt-v4/blob/6aa009b7b1b650407867c8f84c1f1bbc7601ea93/src/pbrt/cpu/aggregates.cpp#L238-L251))和[等量划分](https://github.com/mmp/pbrt-v4/blob/6aa009b7b1b650407867c8f84c1f1bbc7601ea93/src/pbrt/cpu/aggregates.cpp#L252-L262)。

首先我们找出现存的所有包围盒的中点构成的包围盒，找出 span 最大的维度作为分割的维度。中点划分就选择这个区间的中点，左侧和右侧分别放在一起，等量划分则在这个维度上分割且保证两部分物体的数目完全相等，都是比较基本的划分方式。

### 进阶一些的：表面积启发

以上两种虽然简单，但是效果不太好。实际使用中更常见的是[表面积启发式算法](https://www.pbr-book.org/3ed-2018/Primitives_and_Intersection_Acceleration/Bounding_Volume_Hierarchies#TheSurfaceAreaHeuristic)（SAH, Surface Area Heuristic），这一算法基于以下思路：

在构建 BVH 的每个状态下，我们都可以选择直接建立叶子节点，或者以某种划分向下分割。如果我们设第 \\(i\\) 个物体的求交开销是 \\(t_{\text{sect}}(i)\\)，那么直接建立叶节点的情况下，该节点的求交开销是
$$
\sum_{i=1}^Nt_{\text{sect}}(i)
$$
其中 $N$ 是物体数；而将物体分为 A 和 B 两部分的情况下求交开销是
$$
c(A, B) = t_{\text{trav}} + p_A\sum_{i=1}^{N_A}t_{\text{isect}}(a_i) + p_B\sum_{i=1}^{N_B}t_{\text{isect}}(b_i)
$$
其中 \\(t_{\text{trav}}\\) 是遍历子节点并决定光线进入哪个子节点的开销，\\(p_A, p_B\\) 分别为光线进入 A 和 B 的概率，\\(N_A, N_B\\) 是两部分的物体数。

我们要做的就是找到一个合理的划分，使得 \\(c(A, B)\\) 最小——这依赖对 \\(t_{\text{trav}}, t_{\text{sect}}(i)\\) 和 $p$ 的良好估计。在 PBRT 中，作者进行了以下假设：

- \\(t_{\text{sect}}(i)\\) 为常值，且为 \\(t_{\text{trav}}\\) 的 8 倍
- \\(p\\) 的大小与包围盒的表面积成正比

在 [PBRT 的实现](https://github.com/mmp/pbrt-v4/blob/6aa009b7b1b650407867c8f84c1f1bbc7601ea93/src/pbrt/cpu/aggregates.cpp#L263-L353)中，作者将分割维度的 span 分成 12 段（这样就有 11 种可能的划分），计算每一种的 \\(c(A,B)\\) 然后取期望开销最小的一种。

## 更加 GPU-friendly 的做法：HLBVH

上面的做法基本都基于二分，对于并行度很高的 GPU 而言并不友好。问题的瓶颈在于确定划分的开销很高，因此每次只能二分；而如果我们可以对所有的物体给一个确定的排序，而且满足相邻的节点位置相近，那么我们直接排序再切分成任意多段就可以了。

### 将高维坐标映射至一维：Morton Code

对于高维整点坐标（以三维 \\((x, y, z)\\) 为例），我们首先将每个点表示为二进制形式 
$$
(\overline{x_nx_{n-1}\cdots x_1x_0}, \overline{y_ny_{n-1}\cdots y_1y_0}, \overline{z_nz_{n-1}\cdots z_1z_0})
$$
然后交错排列各位，得到
$$
\cdots z_2y_2x_2z_1y_1x_1z_0y_0x_0
$$
再转换成无符号整数，就得到了这一坐标全序的 morton code 表示。morton code 有很好的局部性，以下图的二维 morton code 为例：

<div class="side-by-side-container"><img id="should-invert" src="https://www.pbr-book.org/3ed-2018/Primitives_and_Intersection_Acceleration/Morton%20Split%20Horiz.svg" style="width: 24%" /><img id="should-invert" src="https://www.pbr-book.org/3ed-2018/Primitives_and_Intersection_Acceleration/Morton%20Split%20Vert.svg" style="width: 24%" /></div>
<div class="side-by-side-container"><img id="should-invert" src="https://www.pbr-book.org/3ed-2018/Primitives_and_Intersection_Acceleration/Morton%20Shaded%20Region.svg" style="width: 24%" /><img id="should-invert" src="https://www.pbr-book.org/3ed-2018/Primitives_and_Intersection_Acceleration/Morton%20Split%20Horiz%203.svg" style="width: 24%" /></div>

- (a) 最高位是 1/0 对应水平中线划分
- (b) 次高位是 1/0 对应竖直中线划分
- (c) 最高位是 1，次高位是 0 可以选出第二象限的四个点
- (d) 第三高位则对应水平方向的四等分。

实际计算 morton code 时，我们将包含所有物体的全局包围盒在各个象限上先归一化再 1024 等分，将每个物体的包围盒中心坐标转换到 \\([0, 2^{10})\\) 上去，然后就可以得到 30 位的 morton code，正好可以放进一个 `uint32_t` 里。

得到所有 morton code 之后，我们对这些点进行一次基数排序，然后按照固定步长将所有物体划分为若干个组，就得到了彼此不交而组内点的距离也比较接近的很多个不同组，组内建 BVH 的操作在组间可以并行。

### 两层建树：HLBVH

先对各个组内建树，得到很多的 [LBVH Treelet](https://github.com/mmp/pbrt-v4/blob/6aa009b7b1b650407867c8f84c1f1bbc7601ea93/src/pbrt/cpu/aggregates.cpp#L450-L502)，然后对这些 Treelet 以 SAH 方式建树，就是 [HLBVH](https://github.com/mmp/pbrt-v4/blob/6aa009b7b1b650407867c8f84c1f1bbc7601ea93/src/pbrt/cpu/aggregates.cpp#L388-L427)。

## BVH 的压缩

树形结构的 BVH 逻辑清晰但内存不连续，访存性能不佳，因此在建立 BVH 之后还可以将它压缩为内存中的连续数组，通过记录下标的方式代替指针。在这种表示下，每个节点的后继节点就是它的左孩子，只需要额外记录右孩子的下标（如果是叶子节点则记录它表示的物体）。

<div class="side-by-side-container"><img id="should-invert" src="https://www.pbr-book.org/3ed-2018/Primitives_and_Intersection_Acceleration/BVH%20linearization.svg" style="width: 75%" /></div>

压缩过程就是[简单的 DFS](https://github.com/mmp/pbrt-v4/blob/6aa009b7b1b650407867c8f84c1f1bbc7601ea93/src/pbrt/cpu/aggregates.cpp#L504-L521)。

## BVH 的查询

当给定一条光线去查询交点时，我们只需要手动维护一个栈。每次访问一个节点时，检查光线是否与该节点的包围盒相交，如果不相交则退出。如果相交，则把它的右孩子加入到栈中，然后访问它的左孩子。

[这个算法](https://github.com/mmp/pbrt-v4/blob/6aa009b7b1b650407867c8f84c1f1bbc7601ea93/src/pbrt/cpu/aggregates.cpp#L528-L578)可以只用循环实现，不需要昂贵的递归。

