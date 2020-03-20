---
layout:     post
title:      "游戏数据设计及读写中间件"
subtitle:   " \"开发笔记\""
date:       2019-03-16 13:00:00
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
XXGame
  |--MainWorld
    |--Mainland1
      |--Subland1
        |--SublandOtherInfo
           |--sublandCoins
```
`XXGame_MainWorld_Mainland1_Subland1_SublandOtherInfo_sublandCoins`  

#### 数据结构拆分  
这一步将传入的数据结构进行逐条拆分，用来后续将子数据和上一步生成的subkey进行对应存取操作(这里只进行一层拆分，支持数据传入时两层嵌套，暂时够用，后续考虑进行递归拆分)。  
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
由于游戏里有很多数据更新非常频繁，所以需要对这些数据进行缓存，每隔一段时间才将它们写入本地存储中。  
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

### 公有接口说明  
`register`：定时器开关注册，用于切到后台后关闭延迟存储定时器  
`getAllLocalData`：游戏开始时获取所有本地存储的数据  
`setLocalItemDefer`：延迟存储接口  
`setLocalItemImmediately`：立即存储接口  
`getLocalItem`：获取本地存储数据  
`getGameDataItem`：获取游戏内存数据  

### 使用说明  
##### 数据读取  
游戏开始时调用`getAllLocalData`传入需要从本地存储读取数据的Key(如没有此数据则返回输入的默认值)  
```
getDataKeys() 
{
    var keys = {};
    keys[this.worldInfo.settingInfo._storageKey] = this.worldInfo.settingInfo;
    keys[this.worldInfo.worldOtherInfo._storageKey] = this.worldInfo.worldOtherInfo;
    for(let i = 0; i < this.worldInfo.mainlandInfoList.length; ++i)
    {
        let mainlandInfo = this.worldInfo.mainlandInfoList[i];
        keys[mainlandInfo.mainlandOtherInfo._storageKey] = mainlandInfo.mainlandOtherInfo;
        for(let j = 0; j < mainlandInfo.sublandInfoList.length; ++j)
        {
            let sublandInfo = mainlandInfo.sublandInfoList[j];
            keys[sublandInfo.sublandOtherInfo._storageKey] = sublandInfo.sublandOtherInfo;
        }
    }
    return keys;
}

StorageUtil.getAllLocalData(this.getDataKeys(), (first)=>{
    if(first)
    {
        this.initSettingInfo(null);
        this.initWorldOtherInfo(null);
        for(let i = 0; i < this.worldInfo.mainlandInfoList.length; ++i)
        {
            let mainlandInfo = this.worldInfo.mainlandInfoList[i];
            this.initMainlandOtherInfo(null, mainlandInfo.mainlandOtherInfo);
            for(let j = 0; j < mainlandInfo.sublandInfoList.length; ++j)
            {
                let sublandInfo = mainlandInfo.sublandInfoList[j];
                this.initSublandOtherInfo(null, sublandInfo.sublandOtherInfo);
            }
        }
    }
    else
    {
        this.initSettingInfo(StorageUtil.getGameDataItem(this.worldInfo.settingInfo._storageKey));
        this.initWorldOtherInfo(StorageUtil.getGameDataItem(this.worldInfo.worldOtherInfo._storageKey));
        for(let i = 0; i < this.worldInfo.mainlandInfoList.length; ++i)
        {
            let mainlandInfo = this.worldInfo.mainlandInfoList[i];
            this.initMainlandOtherInfo(StorageUtil.getGameDataItem(mainlandInfo.mainlandOtherInfo._storageKey), mainlandInfo.mainlandOtherInfo);
            for(let j = 0; j < mainlandInfo.sublandInfoList.length; ++j)
            {
                let sublandInfo = mainlandInfo.sublandInfoList[j];
                this.initSublandOtherInfo(StorageUtil.getGameDataItem(sublandInfo.sublandOtherInfo._storageKey), sublandInfo.sublandOtherInfo);
            }
        }
    }
});

initSettingInfo(settingInfo: SettingInfo)
{
    if(settingInfo && Object.getOwnPropertyNames(settingInfo).length > 0 )
    {
        this.worldInfo.settingInfo = settingInfo;
        this.worldInfo.settingInfo["__proto__"] = SettingInfo.prototype;
    }
    else
    {
        this.updateSettingInfo();
    }
}
updateSettingInfo()
{
    StorageUtil.setLocalItemDefer(this.worldInfo.settingInfo._storageKey, this.worldInfo.settingInfo);
}
```
##### 数据写入  
在需要本地存储的数据的set方法里调用`setLocalItemDefer`，每隔一段时间(默认500ms)进行存储  
```
export default class SublandOtherInfo
{
    public _storageKey = "SublandOtherInfo";

    constructor(prefix: string)
    {
        this._storageKey = prefix + "_" + this._storageKey;
    }

    private _sublandCoins: string = '100';
    public get sublandCoins(): string
    {
        if(this._sublandCoins === undefined)
        {
            this._sublandCoins = "0";
        }
        return this._sublandCoins;
    }
    public set sublandCoins(value: string)
    {
        this._sublandCoins = value;
        GameDataManager.getInstance().getGameData().updateSublandOtherInfo(this);
        ListenerManager.getInstance().trigger(ListenerType.SublandCoinsChanged);
    }
}
```
##### 本地存储结构截图  
业务逻辑中的层级嵌套数据结构  
```
WorldInfo
  |--WorldOtherInfo
    |--_worldCoins
  |--SettingInfo
    |--_closeAudio
  |--MainlandInfo
    |--MainlandOtherInfo
      |--_mainlandCoins
    |--SublandInfo
      |--SublandOtherInfo
        |--_sublandCoins
```
被中间件拆分后的扁平化存储  
<img class="shadow" src="/img/in-post/dataMiddleware/1.png" width="738">  

## 后记  
完整代码和示例已集成到框架中[https://github.com/huangx916/GameplayFramework](https://github.com/huangx916/GameplayFramework)