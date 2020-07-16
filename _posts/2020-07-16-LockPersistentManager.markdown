---
layout:     post
title:      "Loading.LockPersistentManager问题解析"
subtitle:   " \"Unity引擎魔改\""
date:       2020-07-16 12:00:00
author:     "A-SHIN"
header-img: "img/bg/006.jpg"
mathjax: true
catalog: true
tags:
    - 技术
---

> “Yeah It's on. ”

##### 资源加载机制：

主线程ReadObject时，如果资源未加载，则会等待m_Mutex直到可用后进行资源加载，如果此时子线程正在加载资源，占用了m_Mutex，则主线程等待m_Mutex时造成了阻塞。  

<img class="shadow" src="/img/in-post/LockPersistentManager/1.png" width="666">  

##### 产生问题的原因：

引用关系在standalone下使用PPtr仅保存InstanceID进行关联，引用的object析构时，PPtr的InstanceID仍旧存在，造成找不到object后每次都需要走加载流程。  

```
namespace UI
{
    class Canvas : public Behaviour
    {
        PPtr<Camera>        m_Camera;
        ...
    }
}
```

```
template<class T> inline
PPtr<T>::operator T*() const
{
    if (GetInstanceID() == InstanceID_None)
        return NULL;
 
    Object* temp = Object::IDToPointer(GetInstanceID());
 
    #if ALLOW_AUTOLOAD_PPTR_DEREFERNCE
    if (temp == NULL)
        temp = ReadObjectFromPersistentManager(GetInstanceID());
    #endif
  
    ...
}
```
如果Canvas引用的Camera销毁了，那么m_Camera转Camera*时会走资源加载流程(ReadObjectFromPersistentManager)，如果此时子线程正在加载资源，占用了m_Mutex，则主线程等待m_Mutex时造成了阻塞。  

##### 解决：  

避免object销毁后，仍旧使用  
<img class="shadow" src="/img/in-post/LockPersistentManager/2.png" width="666">  