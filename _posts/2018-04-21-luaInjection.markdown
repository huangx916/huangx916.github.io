---
layout:     post
title:      "U3D绑定注入Lua数据编辑器扩展"
subtitle:   " \"开发笔记\""
date:       2018-04-21 22:22:22
author:     "A-SHIN"
header-img: "img/bg/025.jpg"
catalog: true
tags:
    - 技术
---

> “Yeah It's on. ”

## 前言
为了贯彻数据驱动，序列化绑定，使lua编程和C#一样便捷，在此扩展了注入lua的各种属性类型，方便编辑器中编辑及在lua中直接使用。  
编辑器界面:
<img class="shadow" src="/img/in-post/luaInjection/1.png" width="490">
Lua中便捷使用:
<img class="shadow" src="/img/in-post/luaInjection/2.png" width="790">
<img class="shadow" src="/img/in-post/luaInjection/3.png" width="710">

## 正文  
#### 实现
`Injection.cs`中定义了各种属性类型，每条属性包含`Name`、`Type`、`Value`，容器类型属性可嵌套包含其他属性。  
目前支持类型如下:
<img class="shadow" src="/img/in-post/luaInjection/4.png" width="245">

`LuaInjectionData`类继承自`MonoBehaviour`:  将编辑器上的属性组织到LuaTable中。  
```
public LuaTable Data
{
    get
    {
        LuaTable dataTable = LuaMain.LuaEnv.NewTable();
        foreach (Injection4 injection in m_InjectionList)
        {
            if (!string.IsNullOrEmpty(injection.Name))
            {
                object value = injection.Value;
                if (injection.Type == InjectionType.List || injection.Type == InjectionType.Dict)
                {
                    value = Injection.ToValueTable(injection.Type, injection.Value);
                }
                dataTable.Set(injection.Name, value);
            }
        }
        return dataTable;
    }
}
```
`LuaInjectionDataInspector`类继承自`Editor`：`LuaInjectionData`的编辑器扩展，提供编辑器中添加编辑属性。  
`LuaBehaviour`类继承自`MonoBehaviour`：Lua脚本组件，包含Lua脚本初始化，属性注入，特定函数绑定(`Awake`、`Start`等)，监听器添加。  
```
public virtual void InitLua(params object[] args)
{
    string luaPath = GetLuaPath();
    if (!string.IsNullOrEmpty(luaPath))
    {
        LuaTable luaClassTable = LuaMain.Instance.Require(luaPath);
        if (luaClassTable != null)
        {
            m_LuaTable = LuaMain.Instance.FuncInvoke(luaClassTable, args) as LuaTable;
            if (m_LuaTable != null)
            {
                OnTableRequired();

                if (m_LuaAwake != null)
                {
                    m_LuaAwake(m_LuaTable);
                }
            }
            else
            {
                Debugger.LogError("LuaObject has not instanced: " + luaPath);
                Dispose();
            }
        }
        else
        {
            Debugger.LogError("LuaTable not returned: " + luaPath);
            Dispose();
        }
    }
}

protected virtual void OnTableRequired()
{
    m_LuaTable.Set("m_CSBehaviour", this);
    InjectData();

    m_LuaAwake = m_LuaTable.Get<Action<LuaTable>>("Awake");
    m_LuaStart = m_LuaTable.Get<Action<LuaTable>>("Start");
    m_LuaOnDestroy = m_LuaTable.Get<Action<LuaTable>>("OnDestroy");

    AddUpdateListener();
    AddFixedUpdateListener();
    AddLateUpdateListener();
    AddApplicationListener();
    AddEnabledListener();
    AddPhysicsListener();
    AddGUIListener();
}

[ContextMenu("Inject Data")]
private void InjectData()
{
    LuaInjectionData injections = GetComponent<LuaInjectionData>();
    if (injections)
    {
        injections.Data.ForEach<object, object>((key, value) => m_LuaTable.Set(key, value));
    }
}
```
`LuaBehaviourInspector`类继承自`Editor`：`LuaBehaviour`编辑器扩展，用于在编辑中输入目标Lua脚本。  
<img class="shadow" src="/img/in-post/luaInjection/5.png" width="1024">

## 后记
热更方案见上篇，[U3D热更新方案](https://huangx916.github.io/2018/04/12/hotupdate/)
