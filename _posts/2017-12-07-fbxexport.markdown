---
layout:     post
title:      "fbx导出轴变换及骨骼动画加载Tip"
subtitle:   " \"踩坑记录\""
date:       2017-12-07 21:00:00
author:     "A-SHIN"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - 技术
---

> “Yeah It's on. ”


## 前言
之前在使用fbx sdk导入模型渲染的时候，发现在模型的朝向跟3DMAX里的不一致。即使添加如下代码还是不起作用：
```
// Convert Axis System to what is used in this example, if needed
FbxAxisSystem SceneAxisSystem = m_pScene->GetGlobalSettings().GetAxisSystem();
FbxAxisSystem OurAxisSystem(FbxAxisSystem::eYAxis, FbxAxisSystem::eParityOdd, FbxAxisSystem::eRightHanded);
if (SceneAxisSystem != OurAxisSystem)
{
	OurAxisSystem.ConvertScene(m_pScene);
}
```
当时重心在架构引擎，所以简单的在游戏里使用欧拉旋转强行把模型调成想要的朝向。。。
## 正文
最近随着渲染模型的增多，因此采用了配置文件的方法加载场景里的模型。后续还会写个编辑器方便配置(暂时考虑使用QT)。  
之前挖的坑就暴露出来了。朝向的问题使得配置场景文件变得很麻烦。没办法重新review了fbx模型加载模块，终于找到了问题。  
当从fbx文件读取顶点对应的controlpoint坐标保存后，需要乘这个mesh的globaltransform矩阵：
```
matrixMeshGlobalPositionIn3DMax = pFbxMesh->GetNode()->EvaluateGlobalTransform();
FbxVector4 curPoint = pCtrlPoint[nCtrlPointIndex];
curPoint = matrixMeshGlobalPositionIn3DMax.MultT(curPoint);
```
EvaluateGlobalTransform方法返回的矩阵包含了导出时坐标系轴变换以及模型在3DMAX中的位移、旋转、缩放等
## 后记
另外关于骨骼动画的加载可参考sdk中的demo。不同的是我们需要在加载fbx时预计算关键帧的顶点调色矩阵并保存，然后渲染时插值使用便可，如此FbxScene便可清除并去加载后续fbx。而不像demo中那样每帧渲染的时候需要使用FbxScene下的接口去计算顶点调色矩阵。