---
title: Spatiotemporal reservoir resampling for real-time ray tracing with dynamic direct lighting 笔记
date: 2023-08-06 23:19:44
---

大名鼎鼎的 ReSTIR 采样器，用极为简单的数据结构实现了非常高效的采样。

渲染方程本质上是积分，例如从点 \\(y\\) 向方向 \\(\vec{\omega}\\) 出射的 Radiance 是
$$
L(y,\omega) = \int_A \rho(y, \vec{yx}\leftrightarrow \vec{\omega}) \cdot L_e(x\to y)\cdot G(x\leftrightarrow y)V(x\leftrightarrow y)\text{d}A_x
$$
其中 \\(\rho\\) 是 BRDF，\\(L_e\\) 是上一个点 \\(x\\) 的出射 radiance，\\(V\\) 是可见性，\\(G\\) 是几何项（即与法线夹角的余弦）。简化后它可以写成
$$
L= \int_Af(x)\text{d}x, f(x) =\rho(x)L_e(x)G(x)V(x)
$$
渲染的过程就是对这个积分进行离散采样并估计的过程。

## 重要性采样 Importance Sampling

从一个分布接近 \\(f\\) 的 PDF \\(p\\) 出发，以 \\(p\\) 为权重采 \\(N\\) 个样本 \\(x_1, x_2, \cdots, x_n\\)
$$
\langle L \rangle^N_{\text{is}} = \frac{1}{N}\sum_{i=1}^N\frac{f(x_i)}{p(x_i)}
$$
只要 \\(f\\) 非零且 \\(p\\) 恒正那 IS 就是无偏的；\\(p\\) 越接近 \\(f\\) 方差越小。

IS 的问题在于，虽然 \\(f\\) 的一些组成部分（\\(L_e, \rho\\)）的 PDF 是可以知道的， 但我们很难找到一个与 \\(f\\) 自己相近的 pdf。

## 多重重要性采样 Multiple Importance Sampling, MIS

MIS 选择使用不同的策略并进行加权混合来解决 IS 的被积函数很难拟合的问题，分别使用容易拟合的 \\(L_e\\) 和 \\(\rho\\) 等不同的权重，然后对采样结果进行加权的混合。
$$
\langle L\rangle_{\text{mis}}^{M,N} = \sum_{s=1}^M\frac{1}{N_s}\sum_{i=1}^{N_s}w_s(x_i)\frac{f(x_i)}{p_s(x_i)}
$$
其中 \\(M\\) 是采样的策略数（即 PDF 的个数）而 \\(N_s\\) 是各个策略下的采样数。只要保证不同策略的 \\(w_s\\) 总和是 1，那么 MIS 仍然是无偏的。

实际使用中一般采用采样数×概率密度为权值：
$$
w_s(x) = \frac{N_sp_s(x)}{\sum_jN_jp_j(x)}
$$

 ## 重采样重要性采样 Resampled Importance Sampling, RIS

与 MIS 不同，RIS 不使用线性加权混合来进行拟合被积函数。

假设理想的分布是 \\(\hat p\\)（例如 \\(\hat p\propto\rho\cdot L_e\cdot G\\)），RIS 从一个可以很坏的分布 \\(p\\) 出发（比如 \\(p \propto L_e\\)）采 M 个样本 \\(x_1, x_2, \cdots x_M\\)。随后我们从这 M 个样本中以权重 \\(\text{w}\\) 再次采得样本 \\(x_z\\)
$$
\text{w}(x) = \frac{\hat{p}(x)}{p(x)}
$$
我们把重采样得到的样本命名为 \\(y\\)，那么估计值就是
$$
\langle L\rangle_{\text{ris}}^{1,M} = \frac{f(y)}{\hat p(y)}\cdot\left(\frac{1}{M}\sum_{j=1}^M\text{w}(x_j)\right)
$$
将这一过程重复 \\(N\\) 次并求平均，即可得到 RIS 的估计结果
$$
\langle L\rangle_{\text{ris}}^{N,M} = \frac{1}{N}\sum_{i=1}^{N}\left(\frac{f(y_i)}{\hat p(y_i)}\cdot\left(\frac{1}{M}\sum_{j=1}^M\text{w}(x_{ij})\right)\right)
$$
这一形式类似 IS，但是因为 \\(y_i\\) 实际上不是直接通过 \\(\hat p\\) 抽样得到的，而是通过 \\(p\\) 进行重采样得到的，所以要给每个 \\(f(y_i)\\) 添加一个含 \\(\text{w}\\) 的补偿项，来保证无偏。

在 \\(M \to \infty\\) 是，\\(y\\) 的分布将逼近 \\(\hat p\\)；然而对于有限大小的 \\(M\\)，\\(p\\) 的质量对于采样效果也很关键。

单次 RIS 的伪代码为：

<img src="/images/restir/ris.png" id="should-invert"  alt="Pseudocode for RIS" style="zoom: 25%;" />

## 加权水库采样 Weighted Reservoir Sampling, WRS

给定 \\(M\\) 个值组成的数据流，每个值都有权重 \\(w_i\\)。WRS 的任务是只经过一次扫描，依据权值选出 N 个样本（\\(M\\) 在算法开始时是未知的，只能随着输入逐次更新自己的样本选择）。

在这里我们将 N 简化为 1，只需要选择一个。

WRS 的伪代码如下，其中定义了一个水库结构，实质上就是动态维护的「当前选中的样本」，每次从流中取得一个新值的时候随机决定要不要将它换掉：

<img src="/images/restir/wrs.png" id="should-invert"  alt="Pseudocode for WRS" style="zoom: 25%;" />

## 流式 RIS（结合 WRS 和 RIS）

将 RIS 的 M 个采样使用 WRS 进行流式优化，就可以在增大采样数的同时降低内存占用。

<img src="/images/restir/stream-ris.png" id="should-invert"  alt="Pseudocode for stream RIS" style="zoom: 25%;" />

仅仅使用流式 RIS 进行采样，同算力同时间下其降噪前的结果已经比 LightBVH 要明显更好，而同质量下用时会更短。

注意这里为水库引入了一个新的字段 \\(W\\)，它就是此前 RIS 中估计值
$$
\langle L\rangle_{\text{ris}}^{1,M} = \frac{f(y)}{\hat p(y)}\cdot\left(\frac{1}{M}\sum_{j=1}^M\text{w}(x_j)\right)
$$
中除 \\(f(y)\\) 以外的部分。

## 时间和空间重用

### 空间重用

在不考虑遮挡项 \\(V\\) 的情况下，我们往往认为临近像素的 \\(\hat p \propto \rho\cdot L_e\cdot G\\) 是相似的，因此我们希望可以在当前像素重用周边像素的采样——一般情况下这意味着额外的存储，但在 WRS 下我们可以在不额外存储的情况下，直接合并两个或多个水库：

<img src="/images/restir/combining_reservoirs.png" id="should-invert"  alt="Pseudocode for reservoir combination" style="zoom: 25%;" />

其采样结果与直接对合并的序列采样等价。

如果 RIS 本身采样 \\(M\\) 次并且重用 \\(k\\) 个邻居的结果，那么计算复杂度是 \\(O(k+M)\\) 但像素看到的采样数是 \\(k\cdot M\\)；如果重复 \\(n\\) 次空间重用，则可以以 \\(O(nk+M)\\) 的复杂度采样 \\(k^n\cdot M\\) 次。

### 时间重用

在动画中，使用上一帧的水库来初始化当前帧的水库。



这样我们得到了一个利用了时间和空间重用的、结合了 WRS 和 RIS 的采样方法。

<img src="/images/restir/restir_biased.png" id="should-invert"  alt="Pseudocode for ReSTIR (but biased)" style="zoom: 25%;" />

## 偏差分析

时间重用会导致 ReSTIR 的采样结果有偏，具体分析和消除 WIP