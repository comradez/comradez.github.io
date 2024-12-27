+++
title = "A Very Short Story on Coordinate Systems"
date = "2024-10-21 22:10:12"
authors = ["茨月"]
+++

A short story about left-handed/right-handed coordinate systems... and more.

<!-- more -->

> Come, let us go down and confuse their language so they will not understand each other - Genesis 11:7

## The Cause

This semester, I am a TA for Introduction to Computer Graphics. Today, I was working on the code framework for assignment 3, "Rasterizing Your First Triangle". For some reasons [^1], we do not use any graphics API but instead implement software rasterization ourselves.

The process of implementing software rasterization includes Model, View, and Projection transformations, along with their corresponding matrices. In the slides, we provided the orthographic projection transformation matrix, which includes a translate and a scale:

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

where \\(n\\) and \\(f\\) are the distances to the near and far planes, and \\(l\\), \\(r\\), \\(b\\), and \\(t\\) represent the left, right, bottom, and top dimensions of the cuboid. It is important to note that, because the common camera model faces the \\(-Z\\) direction, \\(n > f\\) and both are negative values—this fact does not align with our intuition that "the smaller the depth, the closer it is," leading to a series of complications.

We use a right-handed coordinate system, shown below:

<div class="side-by-side-container">
<img alt="Right Hand Coordinate System" style="width: 35s%" src="/images/coordinates/rhs.svg"/>
</div>

## The Problem

Prof mentioned in the Z-Buffer section that "for simplicity, we assume that \\(z\\) values are all positive, and the smaller the value, the closer it is to the camera." In [OpenGL](https://registry.khronos.org/OpenGL-Refpages/gl4/html/glDepthRange.xhtml), [DirectX](https://microsoft.github.io/DirectX-Specs/d3d/DepthBoundsTest.html), and [Vulkan](https://docs.vulkan.org/guide/latest/depth.html#primitive-clipping) APIs, the depth values between \\([0,1]\\) also indicate that the smaller the value, the closer it is to the camera—so the question arises:

- Firstly, the \\(z\\) values in NDC should fall within \\([-1,1]\\) instead of \\([0,1]\\);
- Secondly, when specifying \\(n, f\\), the larger the value, the closer it is to the camera. Why does it become the smaller the value, the closer it is here? What is happening?

I and [another TA](https://zheng95z.github.io/)—who is also my good senior—debated for thirty minutes, discussing whether in our framework \\(z\\) being smaller means closer or \\(z\\) being larger means closer, and how to transform \\(z\\) to be all positive. My view is that \\(z\\) being larger means closer to the camera is correct because I obtained the correct triangle occlusion results by performing Z-Buffering this way; while Zheng's view is that \\(z\\) being smaller means closer to the camera is correct because all major Graphics APIs are like this, and I might have made a mistake somewhere.

## The Investigation

First, we studied the OpenGL coordinate system—being the most familiar and closest to us, OpenGL has always used a right-handed system, consistent with our framework... but is that really the case?

<div class="collapsible">
  <details>
    <summary>The Answer</summary>
    <div class="inner"><p>
        The `Right-handed system` card in <a href="https://learnopengl.com/Getting-started/Coordinate-Systems">this section</a> of LearnOpenGL mentions that OpenGL uses a right-handed system in world coordinates, but <strong><i>uses a left-handed system in NDC space</i></strong>.
    </p></div>
  </details>
</div>

Secondly, we referred to [Real Time Rendering 4th Edition](https://www.realtimerendering.com/) and found that the Projection Matrix defined there is slightly different from ours...

{{ collapsible(
    summary="What's the difference?"
    content="$$
M_{\text{ortho}} = \begin{pmatrix}
\frac{2}{r-l} & 0 & 0 & -\frac{r+l}{r-l} \\
0 & \frac{2}{t-b} & 0 & -\frac{t+b}{t-b} \\
0 & 0 & \frac{2}{f-n} & -\frac{f+n}{f-n} \\
0 & 0 & 0 & 1
\end{pmatrix}
$$
where the far and near planes on the \(z\) component are inverted, equivalent to a mirroring about the \(xOy\) plane!

Finally, after sketching once more, we reached the following conclusions:

- In the right-handed coordinate system (which is our framework), the \\(z\\) values in NDC are indeed within \\([-1,1]\\), and it is correct that the larger the value, the closer it is to the camera.
- The reason why in OpenGL the smaller the \\(z\\) value, the closer it is to the camera, is because it performs a silent flip in the projection matrix and then linearly transforms \\([-1,1]\\) to \\([0,1]\\) by adding 1 and dividing by 2.
- DirectX uses a left-handed coordinate system from world space to NDC, so it is not affected by this issue.
- Vulkan uses a right-handed coordinate system from world space to NDC, but to maintain the intuition of \\(z\\) values, it sacrifices the intuition of the framebuffer—Vulkan uses the top-left corner as the origin, with the y-axis pointing downwards.

## The Solution

To ensure consistency with the lecture and avoid creating additional trouble for the students, we use

```C++
float final_z = 1.f - interpolated_z;
```

To transform the \\(z\\) values to the range \\([0,2]\\) and ensure that nearer values are smaller and farther values are larger.

## The Experience

The god of rendering punished the humans building the graphical Tower of Babel by making them choose different coordinate spaces in different APIs, leaving everyone frustrated and apprehensive when dealing with others' code.

Even highly experienced graphics veterans find coordinate systems tricky, let alone students. Wish me luck with the section Q&A.

## Some interesting sidenotes

Although not directly related to the content and discussion of this article, reverse Z is an interesting example of using the properties of IEEE 754 to optimize algorithms:

[https://tomhultonharrop.com/mathematics/graphics/2023/08/06/reverse-z.html](https://tomhultonharrop.com/mathematics/graphics/2023/08/06/reverse-z.html
)

[^1]: Specifically, it is about creating a unified and distraction-free programming framework that allows students to start implementing algorithms with minimal effort.