---
layout:     post
title:      "GameplayFrameWork for CococsCreator"
subtitle:   " \"开发笔记\""
date:       2019-01-01 10:00:00
author:     "A-SHIN"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - 杂记
---

> “Yeah It's on. ”

## 前言
最近使用cocoscreator开发小游戏时，为了提高效率、增加各项目间代码复用度、减少耦合便简单封装了个基于MVC的[游戏逻辑开发框架](https://github.com/huangx916/GameplayFramework)。
## 正文  

##### 项目工程目录细分
游戏引擎一般都会提供一个开发过程中存放各种资源的文件夹(如creator、u3d的`assets`，ue4的`content`等)。我们首先需要了解特定文件夹的作用，如`resources`。然后在`assets`下细分各种资源建立相应的文件夹，并告知美术人员。好处就是清晰、一目了然，模块化的texture创建`AutoAtlas`后更能减少drawcall以及防止未使用功能的资源提前下载载入内存。各个游戏资源划分大同小异
<img class="shadow" src="/img/in-post/gpfw/1.png" width="300">

##### 框架使用简介

##### `GameMain`
游戏入口，挂`Canvas`下。  

所有管理类使用单例模式，使用`getInstance()`获取实例。

###### `AudioManager`
音效及背景音乐管理类，负责游戏音乐音效开关，状态可在`GameData`中序列化保存。

###### `ConfigManager`
配置文件管理类，配置文件使用json格式存储，通过继承`BaseConfigContainer`，反序列化json文件数据。然后交给`ConfigManager`统一管理。

###### `GameController`
游戏控制类，负责游戏初始化、场景跳转等。

###### `GameDataManager`
数据管理类，所有游戏数据存放在`GameData`中，视图层可通过`GameDataManager`进行数据读写操作。

###### `ListenerManager`
监听管理类，观察者模式，解耦调用视图层。  
UI中注册：```ListenerManager.getInstance().add(ListenerType.GameStart, this, this.onGameStart);```  
管理类中触发：```ListenerManager.getInstance().trigger(ListenerType.GameStart);```

###### `TimeManager`
时间管理类，可获取系统当前时间(单位毫秒)

###### `UIManager`
UI管理类，可通过类名打开、关闭、显示、隐藏、获取对应UI。 UI类需继承`BaseUI`  
打开：```UIManager.getInstance().openUI(LoadingUI);```  
关闭：```UIManager.getInstance().closeUI(LoadingUI);```  
获取:```let tipUI = UIManager.getInstance().getUI(TipUI) as TipUI```

###### Shader
感谢小伙伴的[分享](https://forum.cocos.com/t/creator-2-0-shader/64755)，就不重造轮子了。  
`ShaderComponent`挂到`Sprite`下，选择自定义的Shader  
`ShaderLab`中添加自定义Shader  
`ShaderManager`  
`ShaderMaterial`
<img class="shadow" src="/img/in-post/gpfw/2.png" width="300">

###### Utils
`MathExtension`数学扩展库  
`StringExtension`字符串格式化  
`UIHelp`Tip提示  
`LogWrap`Log封装  
参考[此文章](https://blog.csdn.net/u013158916/article/details/53537922)增加log调用堆栈、时间、类别、颜色等。通过`OPENLOGFLAG`开关。
```
onLoad()
{
    LogWrap.log("test log");
    LogWrap.info("test info");
    LogWrap.warn("test warn");
    LogWrap.err("test err");
}
```
<img class="shadow" src="/img/in-post/gpfw/3.png" width="350">


###### `gulpfile`
下载部署好nodejs、npm、gulp、gulp-tinypng-nokey、gulp-javascript-obfuscator  
cd到游戏工程根目录运行gulp  

* 自动化图集压缩：  
将构建出的ZIP包解压，修改下面的`Dragon`为对应的工程名，cd到当前目录下运行gulp命令，自动压缩完再打成ZIP包便可发布  

```
var gulp = require('gulp');
var tinypng_nokey = require('gulp-tinypng-nokey');

gulp.task('default', function (cb) {
    gulp.src(["./build/fb-instant-games/Dragon/res/raw-assets/**/*.{png,jpg,jpeg}"])
        .pipe(tinypng_nokey())
        .pipe(gulp.dest("./build/fb-instant-games/Dragon/res/raw-assets/"))
        .on("end", cb);
});
```  

* 代码混淆：  
将构建出的ZIP包解压，修改下面的`Dragon`为对应的工程名，cd到当前目录下运行gulp命令，自动混淆完再打成ZIP包便可发布  

```
var gulp = require('gulp');
var javascriptObfuscator = require("gulp-javascript-obfuscator");

gulp.task("default", function (cb) {
    gulp.src(["./build/fb-instant-games/Dragon/src/project.js"])
        .pipe(javascriptObfuscator({
            // compact: true,//类型：boolean默认：true
            mangle: true,//短标识符的名称，如a，b，c
            stringArray: true,
            target: "browser",
        }))
        .pipe(gulp.dest("./build/fb-instant-games/Dragon/src/")
            .on("end", cb));
});
```

###### Others
自定义`*.d.ts`编码提示等

## 后记  
使用ts和cocoscreator不多久，所以封装的比较简陋，后续慢慢完善吧。欢迎小伙伴补充扩展。  
笔力有限、框架详细代码可前往[https://github.com/huangx916/GameplayFramework](https://github.com/huangx916/GameplayFramework)下载。
