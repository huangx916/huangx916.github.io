---
layout:     post
title:      "U3D热更新方案"
subtitle:   " \"开发笔记\""
date:       2018-04-12 22:22:00
author:     "A-SHIN"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - 技术
---

> “Yeah It's on. ”

## 前言
实现自动打包资源，使用MD5码校验一键化CDN更新资源筛选，游戏启动热更新解决方案。脚本使用xlua。在此简单介绍使用方法，最后给出方案源码工程链接。
## 正文  
首先打开Window->Asset Bundle Builder界面，如下图
<img class="shadow" src="/img/in-post/hotupdate/1.png" width="720">
`AssetBundleBuilder.cs`编辑器扩展，显示`Resources`下文件夹。勾选需要打包的模块的文件夹，点击`One Click`按钮(或顺序点击`Set Name`、`Build Bundle`、`Create File List`、`Create Version File`)。会将打包后的ab资源和版本信息文件生成到`StreamingAssets`对应平台目录下(首次配置热更直接拷贝到CDN相应目录下)。  
然后打开Window->HotUpdateFile Package Auto界面，如下图
<img class="shadow" src="/img/in-post/hotupdate/2.png" width="725">  
`AutoHotUpdatePackage.cs`编辑器扩展，选择上一步`StreamingAssets`中生成资源的目录，顺序点击`Init`、`Clear NativeCdn & Package Directory`、`Get Cur FileList & Version From Cdn`、`ExportChage`按钮会将CDN上版本信息文件`FileList.json`、`Version.json`下载到`_CdnVersion`目录下。然后和本地版本信息文件对比后将需更新资源拷贝到`_HotUpdatePackage`目录下。最后将这些资源更新到CDN上。  
<img class="shadow" src="/img/in-post/hotupdate/4.png" width="238">  
CDN目录：
<img class="shadow" src="/img/in-post/hotupdate/3.png" width="673">  
游戏打包时将`Resources`下进行热更的资源删掉。启动游戏时`HotUpdateManager.cs`负责热更。
## 后记
方案具体源码工程链接:[https://github.com/huangx916/HotUpdateSolution](https://github.com/huangx916/HotUpdateSolution)