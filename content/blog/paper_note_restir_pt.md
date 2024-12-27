+++
title = "Generalized Resampled Importance Sampling 阅读笔记 - ReSTIR PT"
date = "2024-10-17 15:49:21"
authors = ["茨月"]
+++

[书接上回](https://blog.zcy.moe/blog/paper-note-gris/)，ReSTIR PT 即使用 ReSTIR 复用 path。由于不同像素的 path space 并不相同，我们必须找到合适的 path reuse 方式——也即 GRIS 理论中的 shift mapping。

<!-- more -->

## ReSTIR PT 中 shift mapping 的形式定义

对于像素 \\(k\\) 和像素 \\(j\\)， \\(T: \Omega_k \to \Omega_j\\) 是 k 的路径空间到 j 的路径空间上的映射：

<p>
$$
T\left(\left[ 
    \textbf{x}_0, \textbf{x}_1, \textbf{x}_2, \textbf{x}_3, ... \right]\right)
 = \left[ \textbf{y}_0, \textbf{y}_1, \textbf{y}_2, \textbf{y}_3, ... \right]
$$
</p>

注意到 \\(\mathbf y_0\\) 是相机，\\(\mathbf y_1\\) 是 primary RT 交出来的，我们要 map 的就是 \\(\mathbf{y}_n, n \geq 2\\)

- Shift Mapping 是 \\(\Omega_k\\) 子集到 \\(\Omega_j\\) 子集的双射——如果 \\(T_{k\to j}(\overline {\mathbf x}) = \overline{\mathbf y}\\) 那么 \\(T_{j\to k}(\overline {\mathbf y}) = \overline{\mathbf x}\\)

- 同时，我们在设计时希望让 map 后的路径尽量与 map 前的路径相似，即 \\(f_k(T_k(\mathbf{\overline x})) \approx f_j(\overline {\mathbf x})\\) 且 \\(\left| \frac{\partial T_k}{\partial \overline{\mathbf x}} \right| \approx 1\\)

## 可用的 shift mapping 策略

- 顶点复用 Vertex copy / Reconnection

   直接复用原 path 的顶点，\\(y_{i+1} = x_{i+1}\\)（从 i+1 开始后面的整条路径全部复用）

   - 仅适用于 \\(x_i, x_{i+1}, y_{i+1}\\) 都落在粗糙表面上的情况

- 半程向量复用 Half-vector copy

   把 base path 的半程向量转换到局部切空间，复用到 offset path 再转回去 trace \\(y_{i+1}\\)

   - 可以处理 reflection

- 方向复用 Direction copy

   把 base path 上的出射角度复用到 offset path

   - 一般用于采样 envmap

- 随机数复用 Random Replay

   复用 base path 在当前 bounce 的随机数来采样 \\(y_{i+1}\\)

   - 比较贵

   - 时常和半程向量 / 方向复用的表现类似

- 流形探索 manifold exploration

   见 [Lehtinen et al. 2013](https://mediatech.aalto.fi/publications/graphics/GMLT/), [Jakob and Marschner 2012](https://www.cs.cornell.edu/projects/manifolds-sg12/)



+ ReSTIR GI 的策略

   ~~虽然当时的人类并没有 GRIS 理论但是事后诸葛亮地说~~直接在 probe 上记录 outgoing radiance 来进行 single bounce GI 的采样，实质上就是无条件的 \\(y_2 = x_2\\) reconnection shift

+ ReSTIR PT 的策略

   混合 random replay 和 reconnection [\[Hua et al. 2019\]](https://profs.etsmtl.ca/agruson/publication/2019_gradientstar/)，在适合 reconnect 的时候复用顶点，否则复用随机数

   本文后半部分的核心贡献就是提出了一个有效的确定复用条件的 heuristic

## 什么时候复用顶点？

作者提出了两个新的判断条件，用于决定顶点何时不能复用

### 距离条件

观察到在渲染方程

<p>
$$
L_o(\mathbf{x}, \omega_o) = L_e(\mathbf{x}, \omega_o) + \int_{\Omega} f_r(\mathbf{x}, \omega_i, \omega_o) L_i(\mathbf{x}, \omega_i) G(\mathbf{x}, \mathbf{x'}) V(\mathbf{x}, \mathbf{x'}) \text{d}\omega_i
$$
</p>

 中，几何项

<p>
$$
G(\mathbf{x}, \mathbf{x'}) = \frac{ \cos \theta_x \cos \theta_{x'} }{ |\mathbf{x} - \mathbf{x'}|^2 }
$$
</p>

包含一个平方反比的顶点距离——在两个顶点距离特别近的时候会导致一个很大的权重，进而带来方差。因此，距离条件定义为当

<p>
$$
\|\mathbf{x}_i - \mathbf{x}_{i+1} \| \geq d_{\max}
$$
</p>

满足时才可以复用，也即「若 \\(\mathbf{x}_{i+1}\\) 与 \\(\mathbf{x}_i\\) 的距离小于阈值，则不能复用顶点」

### Lobe 条件

在检查表面 roughness 的时候，我们会遇到多个 lobe 的 BSDF（如一个 diffuse 上面加一层清漆）。在检查时一次只选择一个 lobe，当 \\(x_i, x_{i+1}, y_{i+1}\\) 的 roughness 都足够大的时候才能 reconnect。因为要每次选一个 lobe，因此要在路径上额外记录每个 path vertex 的 lobe ID.

> 最终使用的 hybrid mapping 就是在满足以上两个条件时进行 reconnection shift 而在不满足时进行 random replay。

## Shift Mapping 的 Jacobian 矩阵

定义 \\(\mathbf x_i \to \mathbf x_{i+1}\\) 的单位向量为 \\(\omega_i^\mathcal X\\)，这一步光传输的随机数是 \\(\overline{\mathbf u}_i^\mathcal X\\)，那么

### 立体角视角

- 顶点复用的 Jacobian

<p>
$$
\left|\frac{\partial \omega_i^y}{\partial \omega_i^x}\right| = \left|\frac{\cos \theta_2^y}{\cos \theta_2^x}\right| \cdot \frac{\|\mathbf x_{i+1} - \mathbf x_i\|^2}{\|\mathbf x_{i+1} - \mathbf y_i\|^2}
$$
</p>

其中 \\(\theta_2^\circ\\) 是 \\(\omega_i^\circ\\) 和它对应的 surface normal 的夹角.

- 随机数复用的 Jacobian

<p>
$$
\left|\frac{\partial \omega_i^y}{\partial \omega_i^x}\right| = \left|\frac{\partial \omega_i^y}{\partial \overline{\mathbf u}_i^y}\right| \left|\frac{\partial \overline{\mathbf u}_i^y}{\partial \overline{\mathbf u}_i^x}\right|\left|\frac{\partial \overline{\mathbf u}_i^x}{\partial \omega_i^x}\right| = \frac{p_{\omega_i^x}(\mathbf x_{i+1})}{p_{\omega_i^y}(\mathbf y_{i+1})}
$$
</p>

即对应立体角采样的概率密度之比.

### Primary Sample Space 视角

- 随机数复用 Jacobian 永远是 1.

- 顶点复用的 Jacobian

<p>
$$
\left|\frac{\partial \overline{\mathbf u}_i^y}{\partial \overline{\mathbf u}_i^x}\right| = 
\left|\frac{\partial \overline{\mathbf u}_i^y}{\partial \omega_i^y}\right|
\left|\frac{\partial \omega_i^y}{\partial \omega_i^x}\right|
\left|\frac{\partial \omega_i^x}{\partial \overline{\mathbf u}_i^x}\right| =
\frac{p_{\omega_i^y}(\mathbf y_{i+1})}{p_{\omega_i^x}(\mathbf x_{i+1})} \cdot
\left|\frac{\partial \omega_i^y}{\partial \omega_i^x}\right|
$$
</p>

注意到因为每一步复用的决策是局部的，全局 Jacobian 就是所有局部 Jacobian 的乘积.


