+++
title = "A Very Short Story on Coordinate Systems"
date = "2024-10-21 22:10:12"
authors = ["茨月"]
+++

一个关于左手/右手坐标系的小故事……以及其他。

<!-- more -->

> Come, let us go down and confuse their language so they will not understand each other - Genesis 11:7

## 事情的起因

这个学期我在做 Introduction to Computer Graphics 的 TA，今天在负责出 assignment 3 的代码框架，是基础的「光栅化你的第一个三角形」。因为一些原因 [^1]，我们不采用任何 graphics API 而是自己实现软光栅。

实现软光栅的过程中包含了 Model, View, Projection 变换，以及其对应的矩阵。在 Slides 里我们给出了正交投影变换的矩阵，包含一个 translate 和一个 scale:

<p>
$$
M_{\text{ortho}} = \begin{pmatrix}
\frac{2}{r-l} & 0 & 0 & 0 \\
0 & \frac{2}{t-b} & 0 & 0 \\
0 & 0 & \frac{2}{n-f} & 0 \\
0 & 0 & 0 & 1
\end{pmatrix} \begin{pmatrix}
1 & 0 & 0 & -\frac{r+l}{2} \\
0 & 1 & 0 & -\frac{t+b}{2} \\
0 & 0 & 1 & -\frac{n+f}{2} \\
0 & 0 & 0 & 1
\end{pmatrix} = \begin{pmatrix}
\frac{2}{r-l} & 0 & 0 & -\frac{r+l}{r-l} \\
0 & \frac{2}{t-b} & 0 & -\frac{t+b}{t-b} \\
0 & 0 & \frac{2}{n-f} & -\frac{n+f}{n-f} \\
0 & 0 & 0 & 1
\end{pmatrix}
$$
</p>

其中 \\(n, f\\) 是 near plane 和 far plane 的距离, \\(l, r, b, t\\) 分别表示 left, right, bottom, top, 标明这个长方体的其他维度。值得注意的是，因为通用的相机模型中相机的朝向是 \\(-Z\\)，因此这里 \\(n > f\\) 且均为负值——这一事实并不符合我们对深度「越小离得越近」的直觉，也就有了接下来的一系列麻烦。

我们采用右手坐标系，如下图所示

<div class="side-by-side-container">
<img alt="Right Hand Coordinate System" style="width: 35s%" src="/images/coordinates/rhs.svg"/>
</div>

## 问题出现

Prof 在 Z-Buffer 一节中提到了「简明起见，我们认为 \\(z\\) 值均为正的，而且越小表示距离相机越近」。在 [OpenGL](https://registry.khronos.org/OpenGL-Refpages/gl4/html/glDepthRange.xhtml), [DirectX](https://microsoft.github.io/DirectX-Specs/d3d/DepthBoundsTest.html) 和 [Vulkan](https://docs.vulkan.org/guide/latest/depth.html#primitive-clipping) 等 API 中，介于 \\([0,1]\\) 之间的 depth 也是越小距离相机越近——那么问题来了：

- 首先，NDC 上的 \\(z\\) 值应当落在 \\([-1,1]\\) 上而不是 \\([0,1]\\) 上；
- 其次，在指定 \\(n, f\\) 的时候还是值越大离相机越近，为何此处变成了越小离相机越近？这里发生了什么？

我和[另一位TA](https://zheng95z.github.io/)——同时也是我的好前辈——大战了三十分钟，讨论在我们的框架中到底是 \\(z\\) 小为近还是 \\(z\\) 大为近，以及如何把 \\(z\\) 变换到全正。我的观点是，\\(z\\) 越大越靠近相机是正确的，因为我这样进行 Z-Buffering 得到了三角形的正确遮挡结果；而曾峥的观点是，\\(z\\) 越小越靠近相机是正确的，因为各大 Graphics API 均是如此，我可能有什么地方写错了。

## 问题求解

首先，我们研究了 OpenGL 的坐标系——作为我们最亲切最熟悉的坐标系统，OpenGL 一直使用右手系，和我们的框架相一致……但，是这样吗？

<div class="collapsible">
  <details>
    <summary>答案</summary>
    <div class="inner"><p>
        LearnOpenGL 中的<a href="https://learnopengl.com/Getting-started/Coordinate-Systems">这一节</a> Right-handed system 的卡片里提到，OpenGL 在世界坐标下采用右手系，而<strong><i>在 NDC 空间下采用左手系</i></strong>.
    </p></div>
  </details>
</div>

其次，我们参考了 [Real Time Rendering 4th Edition](https://www.realtimerendering.com/)，发现其中的 Projection Matrix 和我们的定义略有不同……

{{ collapsible(
    summary="不同在哪？"
    content="$$
M_{\text{ortho}} = \begin{pmatrix}
\frac{2}{r-l} & 0 & 0 & -\frac{r+l}{r-l} \\
0 & \frac{2}{t-b} & 0 & -\frac{t+b}{t-b} \\
0 & 0 & \frac{2}{f-n} & -\frac{f+n}{f-n} \\
0 & 0 & 0 & 1
\end{pmatrix}
$$
其中 \(z\) 分量上的远近平面颠倒，相当于关于 \(xOy\) 平面做了一次镜像！")}}

最后，在再一次画了草图之后，我们得到了以下结论：

- 在右手坐标系（也就是我们的框架）中，NDC 中的 \\(z\\) 值确实在 \\([-1,1]\\) 上，而且越大越靠近相机是正确的。
- OpenGL 之所以 \\(z\\) 值越小越靠近相机，是因为它在 projection matrix 中进行了一次无声的 flip，而且将 \\([-1,1]\\) 先加 1 再除以 2，线性变换到了 \\([0,1]\\) 上。
- DirectX 从世界空间到 NDC 一直都是左手坐标系，因此不受该问题影响。
- Vulkan 从世界空间到 NDC 一直都是右手坐标系，但为了保证 \\(z\\) 值的直觉牺牲了 framebuffer 的直觉——Vulkan 以左上角为原点，y 轴向下。

## 最终的解决方案

为了保证与 Lecture 相一致，不给学生创造更多的麻烦，我们取

```C++
float final_z = 1.f - interpolated_z;
```

来将 \\(z\\) 值变换到 \\([0,2]\\) 上并保证近小远大。

## 感悟

绘制之神对建设图形巴别塔的人类降下神罚，使得他们在不同的 API 中选择不同的坐标空间，让每个人都焦头烂额，并在面对他人代码时战战兢兢。

即使是极度富有经验的图形学老手也会对坐标系统感到棘手，遑论学生。祝我 section 答疑好运。

[^1]: 具体来说是创造统一且 distract-free 的编程框架，来让学生耗费最少的精力即可开始实现算法。