---
layout:     post
title:      "游戏数据设计及读写中间件"
subtitle:   " \"开发笔记\""
date:       2019-06-23 13:00:00
author:     "A-SHIN"
header-img: "img/bg/035.jpg"
mathjax: true
catalog: true
tags:
    - 技术
---

> “Yeah It's on. ”

## 前言  
程序=数据+算法  
游戏开发过程中数据处理必不可少，不管是面向对象还是面向数据的编程方式都会将同一模块的数据封装在一起，需要进行本地存储时，用一个key将该数据对象完整的序列化保存。
```
export class PlayerInfo
{
    public static className = "PlayerInfo";

    private _gold: number = 2000;
    public get gold(): number
    {
        return this._gold;
    }
    public set gold(value: number)
    {
        this._gold = value;
        GameDataManager.getInstance().getGameData().updatePlayerInfo();
        ListenerManager.getInstance().trigger(ListenerType.GoldChanged);
    }

    private _level: number = 1;
    public get level(): number
    {
        return this._level;
    }
    public set level(value: number)
    {
        this._level = value;
        GameDataManager.getInstance().getGameData().updatePlayerInfo();
    }
}
```
```
updatePlayerInfo()
{
    // serializeData
    cc.sys.localStorage.setItem(PlayerInfo.className, this.playerInfo);
}
```
这些看似简单明了，条理清晰，但是频繁的大块数据读写会对性能造成巨大的负担。
因此我们需要寻求一种既能保持上层数据结构关联性不破坏其封装，在数据读写时又可以无视其关联性能够有选择的单独读写其中某条数据的解决方案。  

## 正文  
按照软件工程的理念，没有什么问题是不能通过加一个间接层解决的。  
根据之前的问题分析，那么这个中间件的设计需求很明确：
* 存储数据时，接收一个Key和一个完整的数据结构，然后内部将该数据结构进行逐条拆分并按一定规则根据参数Key生成相对应的subkey进行存储  
* 读取数据时，接收一个Key和需要填充的数据结构，然后内部按上一步存储时相同的规则生成subkey进行数据读取并保存到需填充结构的相对应的字段中，最后用户便拿到了已填充完毕的数据结构  

#### Key的生成规则  
为防止不同模块下的子模块Key相同造成数据覆盖，需要生成模块唯一Key用来存储，首先能够想到的是根据模块嵌套结构进行拼接。
```
|-XXGame
	|-MainWorld
		|-Mainland1
			|-Skyland
				|-SublandOtherInfo
					|-sublandCoins
```
`XXGame_MainWorld_Mainland1_Skyland_SublandOtherInfo_sublandCoins`  

#### 数据结构拆分  
这一步将传入的数据结构进行逐条拆分，用来后续将子数据和上一步生成的subkey进行对应存取操作(这里只进行一层拆分，支持数据传入时两层嵌套)。  
```
for(let key in this.pureDataCache)
{
    let value = this.pureDataCache[key];
    if(value && typeof value === "object")
    {
        for(let childKey in value)
        {
            this.setLocalItemImmediately(key+childKey, value[childKey]);
        }
    }
    else
    {
        this.setLocalItemImmediately(key, value);
    }
}
```

#### 延迟存储  
由于游戏里有很多数据更新非常频繁，所以需要对这些数据进行缓存，每个一段时间才将它们写入本地存储中。  
```
private static _syncLocalDataInterval()
{
    if(!this.intervalId) 
    {
        this.intervalId = setTimeout(()=>{
            this.intervalId = null;
            this._syncLocalData();
        },this.syncLocalDataInterval);
    }
}

private static _syncLocalData()
{
    for(let uniKey in this.keyMap)
    {
        let keysObj = this.keyMap[uniKey];
        let key = keysObj["key"];
        let subKey = keysObj["subKey"];
        if(!subKey)
        {
            this._setData(uniKey, this.gameDataRef[key]);
        }
        else
        {
            this._setData(uniKey, this.gameDataRef[key][subKey]);
        }
    }
    this.keyMap = {};
}
```



未完待续。。。


## 后记  