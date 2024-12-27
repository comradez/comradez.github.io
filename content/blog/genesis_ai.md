+++
title = "短评 Genesis AI 视频——为什么渲染仍然无可替代"
date = "2024-12-18 22:46:22"
authors = ["茨月"]
+++

刚刚朋友给我发了一个叫 [Genesis](https://genesis-embodied-ai.github.io/) 的集成了 AI 的物理引擎让我点评。借着这个机会在 AI 视频和渲染的话题上稍微发挥两句。

<!-- more -->

从他们的 project page 上看，Genesis 是一个非常巨大的框架，包含几个部分：

- 一个通用的物理引擎，实现了很多不同的求解器
- 一个机器人模拟平台，推测是基于这个物理引擎实现的一套用户友好的抽象
- 一个照片级渲染系统，用了 LuisaRender，作为项目的参与者我与有荣焉
- 一个生成式数据引擎，通过用户提供的 prompt 生成不同模态的数据

是一个工程量很大的 rocket science 工作。这样说来可能抽象，但整个工作流配合起来能够实现「用户输入描述，系统输出视频」的效果。给定 Prompt 为「A miniature Wukong holding a stick in his hand sprints across a table surface for 3 seconds, then jumps into the air, and swings his right arm downward during landing. The camera begins with a close-up of his face, then steadily follows the character while gradually zooming out. When the monkey leaps into the air, at the highest point of the jump, the motion pauses for a few seconds. The camera circles around the character for 360 degrees, and slowly ascends, before the action resumes」，这是 [Genesis 输出的视频](https://bucket.zcy.moe/wukong_genesis.mov)：

<p style="text-align: center;">
<video controls="" muted="" loop="" playsinline="" width="75%">
    <source src="https://bucket.zcy.moe/wukong_genesis.mov" type="video/mp4">
</video>
</p>

没错，这个功能和 OpenAI 此前大火、最近正式发布的 Sora 是一致的。Sora 的一大特点是想象力丰富，但对现实世界的规则理解不足。就生成一段体操运动视频这个任务而言，这是[Sora 生成的结果](https://bucket.zcy.moe/sora_gymnastics.mp4)：

<p style="text-align: center;">
<video controls="" muted="" loop="" playsinline="" width="75%">
    <source src="https://bucket.zcy.moe/sora_gymnastics.mp4" type="video/mp4">
</video>
</p>

充满了胳膊和腿的突变，难以形成一致的动作。这也是生成式大模型，尤其是文生图和文生视频大模型的一个问题：Hallucination，它并没有充分理解现实世界中的规则（如人体的肌肉和组织如何工作，能够做到什么、不能够做到什么），进而生成出带有幻觉的内容。而 Genesis 则输出了清晰锐利的视频，里面含有非常合理的动作以及合适的运镜。

诚然，把一个精挑细选出的好结果和一个最难的任务下表现出的坏结果作对比是不公平的，但我还是想说，Genesis 的视频在一致性、合理性的表现上远超 Sora——而且这是有原因的。Genesis 用生成式 AI 生成不同模态的数据，尽管具体是什么我无从得知，但我推测应该是场景（模型、贴图）以及动作，将它交由物理求解器求解出骨骼动画或其他的动作序列，然后交由渲染器去生成最终的视频。在这个工作流中，求解器和渲染器都是基于物理规律运作的，因此可以保证输出可靠的动作/图像/视频，而且可以轻松地在时间和分辨率两个维度上 scale。

前段时间与一位 CV 领域的学者交流时，对方对 Monte Carlo rendering 提出了这样的评价:「I would not be able to work on monte-carlo ray tracing for image generation. This topic has some disadvantages that I would like to avoid.」因为缺乏进一步的交流，我无从得知这个缺陷是什么，但这个评价反映了很长一段时间内人们对于渲染的意见：它太老了/已经被解决了/会被AI替代，或者说，**渲染死了**。但借由 Genesis 和 Sora 的质量对比，我可以更自信地讲出我的这个判断：

<div class="collapsible">
    <div class="inner">
        生成式 AI 很强，但缺乏对规则的显式理解。希望将渲染（或者说图像/视频生成）这整个问题交给一个生成式 AI 模型，寄希望于毕其功于一役，是做不到的。物理模拟与渲染带来的精确控制和真实感是它们不被生成式 AI 替代的理由。
    </div>
</div>


站在 2024 年末这个时间点上，我认为由生成式 AI 来生成数据（站在 AI 人的视角上，可以视为某种 latent code），交由基于物理规律的求解器和渲染器（视为某种 decoder）进行处理，最终得到视频，是文本生成视频这个任务的一个非常聪明也非常正确的途径。希望在 GPU 算力增长逐渐陷入瓶颈的 2025 年，图形学能有更多新鲜血液，学术界和工业界也能逐渐褪去对 AI 能力的盲信，静下来认真审视它的**能**与**不能**，用聪明的方式创造出**适合行业工作者使用、真正提高生产力**的 AI 工具。