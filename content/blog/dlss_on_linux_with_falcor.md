+++
title = "Trying to Enable DLSS on Linux in Falcor...... This is What I Tried"
date = "2024-12-17 18:13:42"
authors = ["茨月"]
+++

一个研究生尝试在 Linux 的 Falcor 上强行启用 DLSS，这是他的系统产生的变化……

<!-- more -->

## 起因

我在使用 Falcor 提供的一个标准的 Path Tracer render graph，定义如下：

```python
def render_graph_PathTracer():
    g = RenderGraph("PathTracer")
    PathTracer = createPass("PathTracer", {'samplesPerPixel': 1})
    g.addPass(PathTracer, "PathTracer")
    VBufferRT = createPass("VBufferRT", {'samplePattern': 'Stratified', 'sampleCount': 16, 'useAlphaTest': True})
    g.addPass(VBufferRT, "VBufferRT")
    AccumulatePass = createPass("AccumulatePass", {'enabled': True, 'precisionMode': 'Single'})
    g.addPass(AccumulatePass, "AccumulatePass")
    ToneMapper = createPass("ToneMapper", {'autoExposure': False, 'exposureCompensation': 0.0})
    g.addPass(ToneMapper, "ToneMapper")
    g.addEdge("VBufferRT.vbuffer", "PathTracer.vbuffer")
    g.addEdge("VBufferRT.viewW", "PathTracer.viewW")
    g.addEdge("VBufferRT.mvec", "PathTracer.mvec")
    g.addEdge("PathTracer.color", "AccumulatePass.input")
    g.addEdge("AccumulatePass.output", "ToneMapper.src")
    g.markOutput("ToneMapper.dst")
    return g
```

首先做一次 RT 得到轻量级的 V-Buffer，然后将 V-Buffer 的输出交给 PathTracer 去渲染，经由 Accumulate 积累之后最后用 ToneMapper 输出 LDR 颜色。作为一个完整的 real time path tracer，它的实现完全没有问题，但一个很重要问题是：它太慢了。每帧的时间要大约 60ms，意味着输出中只有 15-20 fps，而查看 profiler 可以得知，几乎所有的时间都花在了 PathTracer 上。

于是，我们很轻易地得到了任何游戏都会遇到的问题（和自然而然的解决方案）：光追太贵了（，于是让我们打开 DLSS 吧！）

因为 Falcor 用 Python 作为脚本的设计，改写 render graph 非常简单，只需要添加 DLSSPass 的构建

```python
DLSSPass = createPass("DLSSPass", {'motionVectorScale': 'Relative'})
g.addPass(DLSSPass, "DLSSPass")
```

并且将对应的 buffer 交给 DLSSPass 即可

```python
g.addEdge("VBufferRT.mvec", "DLSSPass.mvec")
g.addEdge("VBufferRT.depth", "DLSSPass.depth")
g.addEdge("PathTracer.color", "DLSSPass.color")
g.addEdge("DLSSPass.output", "AccumulatePass.input")
```

事情本应如此——但我很快发现，在编译好的 bin/Release 里，并没有 `DLSSPass.so`. 这是怎么回事？

## 排查

首先检查 CMake 的变量，`FALCOR_HAS_DLSS` 确实是启用的。再查看 `DLSSPass/CMakeLists.txt` 很快就能发现问题：

```
if(NOT (FALCOR_HAS_DLSS AND FALCOR_HAS_D3D12))
    return()
endif()
```

只有 D3D12 也启用的情况下，才能启用 DLSS，我在 Linux 环境上，自然没有这个东西——但我在 Linux 上打游戏的经验告诉我，Linux 上的 Vulkan somehow 是能启用 DLSS 的，于是看看能不能删掉这个条件让他强行编译——

答案是不能，在 `DLSSPass/NGXWrapper.cpp::initializeNGX` 里出现了 `char *` 和 `wchar_t *` 不能隐式转换的问题——NVIDIA 的朋友们在写代码的时候可能是只考虑了 Windows，至少这个在 linux-gcc 上是过不了编译的——先手动转换 `wstring` 即可解决问题。

紧接着是 `DLSSPass/NGXWrapper.cpp::queryOptimalSettings` 里出现了 `NGX_DLSS_GET_OPTIMAL_SETTINGS` 找不到的问题——翻找 DLSS 3.5 的 API Toolkit 可以发现，这个函数在 `nvsdk_ngx_helpers.h` 里，而 `DLSSPass/NGXWrapper.cpp` 只有在有 D3D12 的时候才会 include 这个头文件——手动将它 include 进来也可以解决问题。

在最后修好一切之后，我们得到了

![](/images/dlss_on_linux_with_falcor/error.webp)

好吧。

## 尾声

最后查阅 NVIDIA 的公告，他们确实从未有过 DLSS for Vulkan on Linux 的原生支持，唯一一次[与 Linux 有关的公告](https://www.nvidia.com/en-us/geforce/news/june-2021-rtx-dlss-game-update/)还是在说通过 Proton 支持了 Linux.

或许是时候装 Windows 了，welcome to modern graphics :(