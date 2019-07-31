---
layout:     post
title:      "CocosCreator transform atlas uv coord into frame uv coord in shader"
subtitle:   " \"开发笔记\""
date:       2019-07-30 12:00:00
author:     "A-SHIN"
header-img: "img/bg/036.jpg"
mathjax: true
catalog: true
tags:
    - 技术
---

> “Yeah It's on. ”

## 前言  
shader编写时很多效果会用到uv坐标，creator中uv左上角为(0,0),右下角为(1,1)。然而如果我们使用的不是单独的texture，而是TexturePacker这些三方软件打出来的图集或者使用了creator的自动图集功能。那么shader中获取的内置uv0变量表示的是该像素在图集中的坐标，并不是在该spriteframe中的坐标。此时左上,右下uv并不是(0,0),(1,1).从而造成错误的效果。本篇介绍如何在任何情况下(使用图集(包含旋转)或非图集)在shader中得到标准uv坐标(左上(0,0),右下(1,1))。

## 正文  
#### 思路  
* application阶段，通过`spriteFrame`中的rect信息获取该frame在图集中的偏移位置及大小信息。然后根据frame是否旋转计算出该frame在图集中的归一化的上下左右偏移率，将是否旋转和上下左右偏移率传给shader进行下一步处理。  
* shader阶段，获取上阶段传入的变量。根据偏移率可将图集中的uv0坐标转换到SpriteFrame下的标准uv坐标，如果图集中的该frame进行过旋转，则还需进行逆旋转uv转化。最后便得到了我们想要的frame uv坐标。  

##### 偏移率计算
```
setUVoffset(frame: cc.SpriteFrame)
{
    let rect = frame.getRect();
    let texture = frame.getTexture();
    let texw = texture.width;
    let texh = texture.height;
    let l = 0, r = 0, b = 1, t = 1;
    l = texw && rect.x / texw;
    t = texh && rect.y / texh;
    if(frame.isRotated())
    {
        r = texw && (rect.x+rect.height)/texw;
        b = texh && (rect.y+rect.width)/texh;
    }
    else
    {
        r = texw && (rect.x+rect.width)/texw;
        b = texh && (rect.y+rect.height)/texh;
    }
    this._UVoffset.x = l;
    this._UVoffset.y = t;
    this._UVoffset.z = r;
    this._UVoffset.w = b;
    this._rotated = frame.isRotated() ? 1.0 : 0.0;
    
    this._effect.setProperty('UVoffset', this._UVoffset);
    this._effect.setProperty('rotated', this._rotated);
}
```

##### atlas uv转换frame uv
```
vec2 UVnormalize;
UVnormalize.x = (uv0.x-UVoffset.x)/(UVoffset.z-UVoffset.x);
UVnormalize.y = (uv0.y-UVoffset.y)/(UVoffset.w-UVoffset.y);
```

##### 旋转uv转换
```
if(rotated > 0.5)
{
    float temp = UVnormalize.x;
    UVnormalize.x = UVnormalize.y;
    UVnormalize.y = 1.0 - temp;
}
```

##### 示例展示
<img class="shadow" src="/img/in-post/uvNormalize/1.png" width="300">  
<img class="shadow" src="/img/in-post/uvNormalize/2.png" width="200">  

## 后记  
示例源码[点此下载](https://github.com/huangx916/GameplayFramework/tree/master/assets/scripts/shader)