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

## 加权蓄水池采样 Weighted Reservoir Sampling, WRS

给定 \\(M\\) 个值组成的数据流，每个值都有权重 \\(w_i\\)。WRS 的任务是只经过一次扫描，依据权值选出 N 个样本（\\(M\\) 在算法开始时是未知的，只能随着输入逐次更新自己的样本选择）。

在这里我们将 N 简化为 1，只需要选择一个。

WRS 的伪代码如下，其中定义了一个蓄水池结构，实质上就是动态维护的「当前选中的样本」，每次从流中取得一个新值的时候随机决定要不要将它换掉：

<img src="/images/restir/wrs.png" id="should-invert"  alt="Pseudocode for WRS" style="zoom: 25%;" />

## 流式 RIS（结合 WRS 和 RIS）

将 RIS 的 M 个采样使用 WRS 进行流式优化，就可以在增大采样数的同时降低内存占用。

<img src="/images/restir/stream-ris.png" id="should-invert"  alt="Pseudocode for stream RIS" style="zoom: 25%;" />

仅仅使用流式 RIS 进行采样，同算力同时间下其降噪前的结果已经比 LightBVH 要明显更好，而同质量下用时会更短。

注意这里为蓄水池引入了一个新的字段 \\(W\\)，它就是此前 RIS 中估计值
$$
\langle L\rangle_{\text{ris}}^{1,M} = \frac{f(y)}{\hat p(y)}\cdot\left(\frac{1}{M}\sum_{j=1}^M\text{w}(x_j)\right)
$$
中除 \\(f(y)\\) 以外的部分。

## 时间和空间重用

### 空间重用

在不考虑遮挡项 \\(V\\) 的情况下，我们往往认为临近像素的 \\(\hat p \propto \rho\cdot L_e\cdot G\\) 是相似的，因此我们希望可以在当前像素重用周边像素的采样——一般情况下这意味着额外的存储，但在 WRS 下我们可以在不额外存储的情况下，直接合并两个或多个蓄水池：

<img src="/images/restir/combining_reservoirs.png" id="should-invert"  alt="Pseudocode for reservoir combination" style="zoom: 25%;" />

其采样结果与直接对合并的序列采样等价。

如果 RIS 本身采样 \\(M\\) 次并且重用 \\(k\\) 个邻居的结果，那么计算复杂度是 \\(O(k+M)\\) 但像素看到的采样数是 \\(k\cdot M\\)；如果重复 \\(n\\) 次空间重用，则可以以 \\(O(nk+M)\\) 的复杂度采样 \\(k^n\cdot M\\) 次。

### 时间重用

在动画中，使用上一帧的蓄水池来初始化当前帧的蓄水池。

这样我们得到了一个利用了时间和空间重用的、结合了 WRS 和 RIS 的采样方法。

<img src="/images/restir/restir_biased.png" id="should-invert"  alt="Pseudocode for ReSTIR (but biased)" style="zoom: 25%;" />

## 偏差分析

空间重用会导致 ReSTIR 的采样结果有偏。对 RIS 的形式进行一个改写：
$$
\langle L\rangle_{\text{ris}}^{1,M} = \frac{f(y)}{\hat p(y)}\cdot\left(\frac{1}{M}\sum_{j=1}^M\text{w}(x_j)\right) = f(y) \cdot W(\mathbf{x},z)
$$
其中
$$
W(\mathbf{x},z) = \frac{1}{\hat p(x_z)}\left(\frac{1}{M}\sum_{j=1}^M\text{w}(x_j)\right)
$$
如果要与 IS 的形式相对应，即 \\(\frac{f(y)}{p(y)}\\)，则应当保证 \\(W(\mathbf{x},z)\\) 的期望等于 \\(\frac{1}{p(y)}\\) 才能保证结果无偏。接下来的分析将说明，对于给定的 \\(y\\)，遍历所有的使得 \\(x_z = y\\) 的组合 \\((\mathbf{x},z)\\)，其 \\(W(\mathbf{x},z)\\) 的平均值
$$
E_{x_z=y}[W(\mathbf{x},z)] \leq \frac{1}{p(y)}
$$
只有在采样 \\(\mathbf{x}\\) 的所有 pdf（注：原本我们假定 \\(\mathbf{x} \sim p\\)，但在空间重用后不再能保证来自相邻像素的样本来自同一分布，所以可能会有很多分布） 在 \\(y\\) 处均非零，等号才成立。

> ### 具体的推导过程
> 首先推广 RIS，假设 \\(\mathbf x\\) 中的各个样本从不同的分布中选出，第 \\(i\\) 个样本 \\(x_i \sim p_i\\)，那么
> $$
> p(\mathbf x) = \prod_{i=1}^M p_i(x_i)
> $$
> 对应修改重采样的权重为（即用 \\(p_i\\) 代替原本的 \\(p\\)）
> $$
> p(z | \mathbf x) = \frac{w_z(x_z)}{\sum_iw_i(x_i)}, \text{where }w_i(x) = \frac{\hat p(x_i)}{p_i(x_i)}
> $$
> 我们可以计算 \\(\mathbf x, z\\) 联合分布的 pdf 是
> $$
> p(\mathbf x, z) = p(z|\mathbf x)\cdot p(\mathbf x) = \left[\prod_{i=1}^Mp_i(x_i)\right]\cdot \frac{w_z(x_z)}{\sum_iw_i(x_i)}
> $$
> 对于给定的 \\(y\\)，有很多种可能的 \\(\mathbf x, z\\) 组合（例如 \\(x_1 = y, z = 1\\) 或者 \\(x_M = y, z=M\\)）等。为了表达方便，我们首先定义集合 \\(Z(y)\\)：
> 
> $$
> Z(y) = \\{i | 1\leq i\leq M \land p_i(y)\gt 0\\}
> $$
> 即「在这个位置上可能出现 \\(y\\) 值的下标 i 构成的集合」。
> 固定下标 \\(i\\) 后，剩下的 \\(M-1\\) 个位置的值可以自由选择，都能保证 \\(\mathbf x, z\\) 是采样得到 \\(y\\) 的组合。因此 \\(p(y)\\) 实际上是所有 \\(M-1\\) 重积分的和：
> $$
> p(y) = \sum_{i\in Z(y)}\int\cdots\int p(\mathbf{x}^{i\to y}, i)\text{d}x_1\cdots\text{d}x_M
> $$
> 而对于 \\(W(\mathbf x, z)\\) 而言，固定采样结果为 \\(y\\) 时它的均值为
> $$
> \begin{aligned}
E_{x_z=y} &= \frac{\int\cdots\int W(\mathbf{x}^{i\to y}, i)p(\mathbf{x}^{i\to y}, i)\text{d}x_1\cdots\text{d}x_M}{\int\cdots\int p(\mathbf{x}^{i\to y}, i)\text{d}x_1\cdots\text{d}x_M} \\
> \end{aligned}
> $$
> 简单手推一下（真的很简单 XD，基本都约掉了）可以得到
> $$
> E_{x_z=y} = \frac{1}{p(y)} \frac{|Z(y)|}{M} \leq \frac{1}{p(y)}
> $$
> 因此只有在 \\(|Z(y)| = M\\)，即所有 PDF 在 \\(y\\) 处都非零时才能保证无偏，否则结果会偏小。
>
> 具体来说，上文的 ReSTIR 算法会把被遮挡的采样丢弃（相当于把 PDF 直接设置为 0），但在相邻像素中被遮挡的样本可能在当前像素中没有被遮挡，对它进行重用的结果就是期望出现偏差、图像变暗。
## 偏差消除

把 \\(W(\mathbf x, z)\\) 里的权重 \\(1/M\\) 改为可变的 \\(m(x_z)\\)，则上文的期望变为
$$
E_{x_z=y} = \frac{1}{p(y)}\sum_{i\in Z(y)}m(x_i)
$$
那么无偏性等价于 \\(\sum_{i\in Z(y)}m(x_i) = 1\\)。

- 这可以通过简单地将 \\(m(x_z)\\) 设置为 \\(\frac{1}{|Z(x_z)|}\\) 实现，但效果很差，会非常噪。 

- 更好的做法是使用类似 MIS 的 \\(m\\) 估算，例如
  $$
  m(x_z) = \frac{p_z(x_z)}{\sum_{i=1}^Mp_i(x_z)}
  $$

实际计算中，作者使用了 \\(\hat p_{q_i}(x_i)\\) 来模拟真实 PDF：因为只要真实 PDF 非零，它也一定非零。以下是一个平均权值（即上面两者中方法1）的无偏蓄水池合并算法的伪代码。

<img src="/images/restir/combining_reservoirs_unbiased.png" id="should-invert"  alt="Pseudocode for unbiased reservoir combination" style="zoom: 25%;" />

## 实现细节

作者在 Falcor 上实现了 ReSTIR 采样器，具体参数为

- RIS 第一次采样的\\(M = 32\\)，分布 \\(p \propto L_e\\)
    - 如果有环境贴图则 25% 的样本从环境贴图上重要性采样
- 目标 PDF 是 \\(\hat p \propto\\ \rho \cdot L_e \cdot G\\)
    - 所有的 BRDF 均视为底层 Lambertian 上层 GGX 的两层介质
- 选择固定位置的邻居容易导致 artifact，从 30px 的半径中随机采样 5 个（无偏是 3 个）像素作为邻居
    - 由于在有偏算法中，几何差异很大的邻居会带来很大的偏差，因此采样时会计算相机夹角和深度，如果差异太大直接拒绝
- 要得到 interactive 的帧率，无偏法 N = 1，有偏法 N = 4，离线渲染则可以更大
