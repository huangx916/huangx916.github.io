---
layout:     post
title:      "TextureStreaming Optimize"
subtitle:   " \"Unity引擎魔改\""
date:       2020-05-23 12:00:00
author:     "A-SHIN"
header-img: "img/bg/016.jpg"
mathjax: true
catalog: true
tags:
    - 技术
---

> “Yeah It's on. ”

## 编辑器设置：

Quality设置  
Texture Streaming Budget：显存的预算  
Texture Streaming Mipmaps Renderers Per Frame：每帧(每个Batch)处理的Renderers的个数  
Texture Streaming Mipmaps Max Level Reduction：至少保留的Mip个数-1(minimum mip level requirements = MipmapCount  - 1 -  最大可流出的MipLevel)  
Texture Streaming Mipmaps Max File IO Requests：每帧流入流出的最大Texture数  

Texture设置  
isStreamable：该Texture是否开启Streamable  
Streaming Priority：流入流出优先级，优先级越低表示越远(比距离相机距离作用更大)，越优先流出  

## 核心代码：

#### TextureStreamingManager  
    ThreadUpdate  
    FlushTextureOperations  
    FlushRendererOperations  
    StreamingTextures       m_StreamingTextures;  
    StreamingRenderers      m_Renderers;  
    DesiredMipLevels        m_DesiredMipLevels;  
    FinalMipLevels          m_FinalMipLevels;  

#### TextureStreamingJob  
    TextureStreamingInitDesiredMipLevels：将desiredMipLevels初始化为smallestMip  
    TextureStreamingUpdateCamera：texture根据最近的render距离相机距离和纹素密度、缩放、纹素等计算MipmapLevel保存到desiredMipLevels中  
    TextureStreamingCombineDesiredMipLevels：将desiredMipLevels各个batch的mipLevel和finalMipLevels的mipLevel取最小值合并到finalMipLevels中，然后根据Mipcount和mipLevelReduction Clamp到smallestMipLevel保存到finalMipLevels中  
    TextureStreamingAdjustWithBudget：防止反复流入流出  
    TextureStreamingReduceToBudget：当预测内存大于预算时，从远到近将Miplevel从期望开始再下降，直到预测小于等于预算  
    TextureStreamingRetainExistingMips：当预测内存小于预算时，将需要流出的Texture的Miplevel从期望恢复到当前显存Miplevel，直到预测接近预算  
    TextureStreamingCalculateLoadOrder：将需要流入流出的Texture的index push到loadOrder(主线程m_LoadOrder的引用，主线程中遍历进行流入流出接口调用)中  


 在TextureStreamingAdjustWithBudget之前计算所有texture的desiredMipLevel，然后在TextureStreamingAdjustWithBudget根据预测和预算之间的关系，调整targetMipLevel，确保预测小于等于预算  

// It's possible that the used memory always exceeds the budget due to the minimum mip level requirements  
// So we must continue to load the increased mip levels even if we will blow the budget  
// Hence the !decreaseResolutionMipLevelIterator check here  

当当前显存占有量小于Budget时，将当前Miplevel高于期望Miplevel的Texture流入到期望的Miplevel，直到到达Budget  

当没有可流出的Texture时，即使超出预算，也需要将需要流入的Texture流入到显存中  

当Texture流入和流出时都需要重新不同MipCount的CreateTexture2D进行数据拷贝及上传(流入时)  

## 优化方案：
**通过对StreamingTextures增加引用计数，来决定是否需要流入流出，使得场景中初始就隐藏的物体所用的texture能流出到smallest miplevel来大幅降低显存**

## 资源规范：
目前只针对Texture2D，不涉及CubeMap和RenderTexture等  

地表：Basemap和SpecularMetallicMap为RenderTexture不进行流式管理  
* Control贴图m_IsStreamable为false，不进行流式管理  
* SplatTexture和SplatNormalMap进行流式管理  
* 树：原型所用的贴图进行流式管理  
* 草：生成的atlas m_IsStreamable为false，不进行流式管理  

美术资源制作时，将地形相关贴图都设置成streamable为false，将HLOD所用贴图设置成streamable为false，粒子特效所用贴图设置成streamable为false，Road期望Mip计算偏差所以也设置为false  

某些在budget足够的情况下也比较糊的贴图也需要设置成streamable为false，这个是因为纹素密度计算的问题，跟资源制作和Mip算法选择都有关  

还有如果不是特别需要，priority不用特意设置，不然会影响距离判断  

某些资源的纹素密度预计算有问题，需要重新导入下  