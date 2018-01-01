---
layout:     post
title:      "可编程管线线性雾"
subtitle:   " \"游戏引擎\""
date:       2017-12-31 21:00:00
author:     "A-SHIN"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - 技术
---

> “Yeah It's on. ”


## 前言
用GLSL实现了简单的linear fog.  exp fog ect.待后续加入...

## 正文
#  首先vert shader 计算出每个点到视点距离，我是在视图坐标系下计算的:  
```
uniform int useFog;
out float vs_fs_distance;
```  

```
if(useFog == 1)
{
	vec4 mvPosition = view_matrix * model_matrix * position;
	vs_fs_distance = sqrt(mvPosition.x * mvPosition.x + mvPosition.y * mvPosition.y + mvPosition.z * mvPosition.z);
}
```
#  然后frag shader 根据距离计算出雾的权重对目标颜色和雾颜色进行线性插值:  
```
uniform int useFog;
uniform int fogType;
uniform vec3 fogColor;
uniform float fogStart;
uniform float fogEnd;
in float vs_fs_distance;
```  

```
if(useFog == 1)
{
	// linear fog
	if(fogType == 0)
	{
		vec4 _fogColor = vec4(fogColor, 1.0);
		float fog  = (vs_fs_distance - fogStart)/fogEnd;
		fog = clamp(fog, 0.0, 1.0);
		fColor = mix(fColor, _fogColor, fog);
	}
}
```  

对比效果如下:  
<img class="shadow" src="/img/in-post/linearfog/1.png" width="600">
<img class="shadow" src="/img/in-post/linearfog/2.png" width="600">  

具体代码[在此](https://github.com/huangx916/HXEngine/tree/master/HXEngine/HXGame/FBX/Terrain)
## 后记