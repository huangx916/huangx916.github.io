---
layout:     post
title:      "GLSL 切线空间光照计算 抖动BUG解决"
subtitle:   " \"游戏引擎\""
date:       2018-07-21 16:38:00
author:     "A-SHIN"
header-img: "img/bg/020.jpg"
catalog: true
tags:
    - 技术
---

> “Yeah It's on. ”

## 前言
970显卡上正常编写完支持法线及细节纹理的shader后，在核显的MBP上运行时却遇到了几个问题：  
1.shader linking failed not enough space for defined varyings  
2.光照像素抖动
## 正文  
第一个问题可能由于不同显卡支持的变量数不同引起的，因此把shader支持的灯光数减少以降低uniform变量数便解决。  
第二个问题找了好久的原因，最后用排除法发现每帧顶点计算的切线空间下的视向量会变。检查公式没发现不对的地方。最后经反复试验后才找到解决方法。
```
vec3 v;
v.x = dot (eyeObjectDir, t);
v.y = dot (eyeObjectDir, b);
v.z = dot (eyeObjectDir, n);
//eyeTangentDir = normalize (v);
vec3 v1 = normalize (v);
eyeTangentDir = v1;
```
eyeTangentDir = normalize (v);  直接赋值每帧值不同。  改用vec3 v1 = normalize (v);  eyeTangentDir = v1;  意外的解决了问题 - -!
## 后记
后续的lightTangentDir直接赋值却没问题，好奇怪
```
vec3 v;
v.x = dot (lightObjectDir, t);
v.y = dot (lightObjectDir, b);
v.z = dot (lightObjectDir, n);
lightTangentDir[i] = normalize (v);
```