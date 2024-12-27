+++
title = "Trying to Enable DLSS on Linux in Falcor...... This is What I Tried"
date = "2024-12-17 18:13:42"
authors = ["茨月"]
+++

A graduate student tried enabling DLSS on Falcor running on Linux, this is what happened to his system.

<!-- more -->

## What brought me here

I am using a standard Path Tracer render graph provided by Falcor, defined as follows:

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

First, perform an RT to obtain a lightweight V-Buffer, then pass the output of the V-Buffer to the PathTracer for rendering. After accumulating with Accumulate, finally use ToneMapper to output LDR color. As a complete real-time path tracer, its implementation is flawless, but a very important issue is: it is **TOO SLOW**. Each frame takes about 60ms, which means the output is only 15-20 fps. Checking the profiler reveals that almost all the time is spent on the PathTracer.

So, we easily encountered a common problem in any game (and the natural solution): ray tracing is too expensive(, so let's enable DLSS!)

Because Falcor uses Python for scripting, rewriting the render graph is very simple, just add the construction of DLSSPass:

```python
DLSSPass = createPass("DLSSPass", {'motionVectorScale': 'Relative'})
g.addPass(DLSSPass, "DLSSPass")
```

and pass the corresponding buffer to DLSSPass

```python
g.addEdge("VBufferRT.mvec", "DLSSPass.mvec")
g.addEdge("VBufferRT.depth", "DLSSPass.depth")
g.addEdge("PathTracer.color", "DLSSPass.color")
g.addEdge("DLSSPass.output", "AccumulatePass.input")
```

Things should have been like this—but I quickly found that in the compiled `bin/Release`, there was no `DLSSPass.so`. What happened?

## Troubleshooting

First, check the CMake variable, `FALCOR_HAS_DLSS` is indeed enabled. Then, check `DLSSPass/CMakeLists.txt` and you will quickly find the problem:

```
if(NOT (FALCOR_HAS_DLSS AND FALCOR_HAS_D3D12))
    return()
endif()
```

DLSS can only be enabled when D3D12 is also enabled. Since I am on a Linux environment, naturally, I don't have this. However, my experience playing games on Linux tells me that Vulkan on Linux can somehow enable DLSS, so I tried removing this condition to force it to compile.

The answer is no. In `DLSSPass/NGXWrapper.cpp::initializeNGX`, there was an issue with implicit conversion between `char *` and `wchar_t *`. It seems that NVIDIA's developers only considered Windows when writing the code, as this doesn't compile on linux-gcc. Manually converting to `wstring` can solve this problem.

Next, in `DLSSPass/NGXWrapper.cpp::queryOptimalSettings`, there was an issue with `NGX_DLSS_GET_OPTIMAL_SETTINGS` not being found. Searching through the DLSS 3.5 API Toolkit, I found that this function is in `nvsdk_ngx_helpers.h`, and `DLSSPass/NGXWrapper.cpp` only includes this header file when D3D12 is available. Manually including it can also solve this problem.

After fixing everything, we got

![](/images/dlss_on_linux_with_falcor/error.webp)

**FINE.**

## Epilogue

Finally, after checking NVIDIA's announcements, it turns out they have never officially supported DLSS for Vulkan on Linux. The only [announcement related to Linux](https://www.nvidia.com/en-us/geforce/news/june-2021-rtx-dlss-game-update/) mentioned support through Proton only.

It's probably time to install Windows back. Welcome to modern graphics :(