+++
title = "A short comment of AI-generated Videos, and why I think rendering is still important"
date = "2024-12-18 22:46:22"
authors = ["茨月"]
+++

A friend just sent me a link to [Genesis](https://genesis-embodied-ai.github.io/), an AI-integrated physical engine, and asked for my comments. Taking this opportunity, I would like to share some thoughts on AI-generated videos and rendering.

<!-- more -->

From their project page, Genesis appears to be a very large framework that includes several components:

- A universal physics engine re-built from the ground up, capable of simulating a wide range of materials and physical phenomena.
- A lightweight, ultra-fast, pythonic, and user-friendly robotics simulation platform.
- A powerful and fast photo-realistic rendering system. I feel proud as one of the authors of the LuisaRender framework. 
- A generative data engine that transforms user-prompted natural language description into various modalities of data.

It is a highly complex, rocket science-ish task. This may sound abstract, but the entire workflow can achieve the effect of "user inputs description, system outputs video."

Given the prompt "A miniature Wukong holding a stick in his hand sprints across a table surface for 3 seconds, then jumps into the air, and swings his right arm downward during landing. The camera begins with a close-up of his face, then steadily follows the character while gradually zooming out. When the monkey leaps into the air, at the highest point of the jump, the motion pauses for a few seconds. The camera circles around the character for 360 degrees, and slowly ascends, before the action resumes," this is the [video output by Genesis](https://bucket.zcy.moe/wukong_genesis.mov):

<p style="text-align: center;">
<video controls="" muted="" loop="" playsinline="" width="75%">
    <source src="https://bucket.zcy.moe/wukong_genesis.mov" type="video/mp4">
</video>
</p>

This feature is exactly consistant with OpenAI's popular and recently officially released Sora. One major characteristic of Sora is its rich imagination but insufficient understanding of real-world rules. For the task of generating a gymnastics video, this is the [result generated by Sora](https://bucket.zcy.moe/sora_gymnastics.mp4):

<p style="text-align: center;">
<video controls="" muted="" loop="" playsinline="" width="75%">
    <source src="https://bucket.zcy.moe/sora_gymnastics.mp4" type="video/mp4">
</video>
</p>

This video is filled with mutated arms and legs, making it difficult to form coherent movements. This highlights a common issue with generative large models, especially text-to-image and text-to-video models: hallucination. These models do not fully understand the rules of the real world (such as how human muscles and tissues work, what they can and cannot do), leading to the generation of content with unrealistic elements. In contrast, Genesis produces clear and sharp videos with very reasonable movements and appropriate camera work.

Admittedly, comparing a carefully selected good result with a poor result from the most challenging task is unfair. However, I still want to say that Genesis far surpasses Sora in terms of consistency and realism in its performance — and there’s a reason for this. Genesis uses generative AI to produce data in different modalities. Although I don’t know the exact details, I speculate that it generates elements like scenes (models, textures) and motions, which are then passed to a physics solver to compute skeletal animations or other motion sequences. Afterward, the renderer generates the final video. In this workflow, both the solver and the renderer operate based on physical principles, ensuring reliable outputs in terms of motion, imagery, and video. Additionally, this approach can easily scale across both the temporal and resolution dimensions.

Some time ago, during a conversation with a scholar in the field of computer vision, they made the following comment about Monte Carlo rendering: 'I would not be able to work on Monte Carlo ray tracing for image generation. This topic has some disadvantages that I would like to avoid.' Due to the lack of further discussion, I couldn’t determine what these disadvantages were, but this remark reflects the opinions people have held about rendering for quite some time: it’s too old/has already been solved/will be replaced by AI — in other words, ***rendering is dead***. However, based on the quality comparison between Genesis and Sora, I can now state my conclusion with greater confidence:

<div class="collapsible">
    <div class="inner">
        Generative AI is powerful, but it lacks an explicit understanding of rules. Hoping to entrust the entire problem of rendering (or image/video generation) to a single generative AI model and expecting it to solve everything in one go is simply not feasible. The precise control and realism brought by physics simulation and rendering are the reasons why they cannot be replaced by generative AI.
    </div>
</div>

Standing at the end of 2024, I believe that using generative AI to produce data (which, from the perspective of AI researchers, can be seen as a form of latent code), and then processing it through physics-based solvers and renderers (viewed as a type of decoder) to produce the final video, represents a very clever and correct approach to the task of text-to-video generation. As we approach 2025, with GPU performance growth gradually hitting a bottleneck, I hope to see more fresh blood injected into the field of computer graphics. I also hope that both academia and industry can shed their blind faith in AI’s capabilities, take a step back, and seriously examine what AI ***can*** and ***cannot*** do. By doing so, we can create AI tools that are ***truly useful for industry professionals*** and ***genuinely enhance productivity*** in a smart and thoughtful way.