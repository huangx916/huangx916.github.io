---
layout:     post
title:      "游戏引擎场景管理及动态批处理"
subtitle:   " \"游戏引擎\""
date:       2018-07-22 10:00:00
author:     "A-SHIN"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - 技术
---

> “Yeah It's on. ”

## 前言
周末对游戏引擎[HXEngine](https://github.com/huangx916/HXEngine)进行了简单的场景管理及动态批处理。后续考虑进行八叉树管理及静态drawcall合并。
## 正文  
#总体思路：  
首选在编辑器或者配置文件中为所有场景中的gameobject配置一个render queue值，值越小就越先渲染。这里大致分为几类：
```
enum ERenderQueue
{
	RQ_BACKGROUND = 1000,
	RQ_GEOMETRY = 2000,
	RQ_ALPHATEST = 2450,
	RQ_TRANSPARENT = 3000,
	RQ_OVERLAY = 4000
};
```
然后根据是否需要alpha blend(render queue == 3000 为分界线)，把gameobject分为两大类opaque和transparent。分别将gameobject的submesh对应的可渲染单位放入相应的列表中。  
1. opaque以render queue为key，把相同render queue下相同材质的renderable放入到一个队列中，每帧渲染时这个队列就作为一个batch，只需设置一次材质状态。  
2. transparent以render queue为key，把相同render queue的renderable放入到一个队列中，每帧渲染时需要对这个队列按同相机距离从远到近排序。把相邻的相同材质的renderable作为一个batch，顺序渲染。
#具体实现：  
场景管理器中的数据结构如下：
```
// gameobject tree root
HXGameObject* gameObjectTreeRoot;

typedef std::vector<HXRenderable*> vectorRenderable;
typedef std::map<std::string, vectorRenderable> mapStringVector;
// opaque
std::map<int, mapStringVector> opaqueMap;
// transparent
std::map<int, vectorRenderable> transparentMap;
```
gameObjectTreeRoot为gameobject树状结构根节点，用于具体管理场景中的gameobject(查询、创建、删除等)。  
按上述思路根据gameObjectTree创建opaqueMap和transparentMap用于渲染。  
具体渲染过程：
```
void HXSceneManager::OnDisplay(bool shadow)
{
	if (!mainCamera)
	{
		return;
	}
	HXRenderSystem* pRenderSystem = HXRoot::GetInstance()->GetRenderSystem();
	if (NULL == pRenderSystem)
	{
		return;
	}

	if (shadow)
	{
		HXStatus::GetInstance()->ResetStatus();
		mainCamera->Update();
		gameObjectTreeRoot->Update();
	}
	// render opaque
	HXMaterial* curMaterial = NULL;
	for (std::map<int, mapStringVector>::iterator itr = opaqueMap.begin(); itr != opaqueMap.end(); ++itr)
	{
		for (mapStringVector::iterator itr1 = itr->second.begin(); itr1 != itr->second.end(); ++itr1)
		{
			for (vectorRenderable::iterator itr2 = itr1->second.begin(); itr2 != itr1->second.end(); ++itr2)
			{
				HXRenderable* renderable = *itr2;
				if (shadow && !renderable->m_pSubMesh->IsCastShadow)
				{
					continue;
				}
				if (curMaterial != renderable->m_pMaterial)
				{
					curMaterial = renderable->m_pMaterial;
					if (shadow)
					{
						curMaterial->SetShadowMapMaterialRenderStateAllRenderable();
					}
					else
					{
						curMaterial->SetMaterialRenderStateAllRenderable();
					}
				}
				HXStatus::GetInstance()->nVertexCount += renderable->m_pSubMesh->vertexList.size();
				HXStatus::GetInstance()->nTriangleCount += renderable->m_pSubMesh->triangleCount;
				renderable->SetModelMatrix(renderable->m_pTransform->mCurModelMatrix);
				renderable->SetViewMatrix(mainCamera);
				renderable->SetProjectionMatrix(mainCamera);
				if (shadow)
				{
					curMaterial->SetShadowMapMaterialRenderStateEachRenderable(renderable);
				}
				else
				{
					curMaterial->SetMaterialRenderStateEachRenderable(renderable);
				}
				pRenderSystem->RenderSingle(renderable, shadow);
			}
		}
	}
	curMaterial = NULL;
	// render transparent
	// Z排序
	if (shadow)
	{
		for (std::map<int, vectorRenderable>::iterator itr = transparentMap.begin(); itr != transparentMap.end(); ++itr)
		{
			for (vectorRenderable::iterator itr1 = itr->second.begin(); itr1 != itr->second.end(); ++itr1)
			{
				HXRenderable* renderable = *itr1;
				renderable->SetModelMatrix(renderable->m_pTransform->mCurModelMatrix);
				renderable->SetViewMatrix(mainCamera);
				renderable->SetProjectionMatrix(mainCamera);
			}
		}
		for (std::map<int, vectorRenderable>::iterator itr = transparentMap.begin(); itr != transparentMap.end(); ++itr)
		{
			std::sort(itr->second.begin(), itr->second.end(), Zcompare);
		}
	}
	curMaterial = NULL;
	for (std::map<int, vectorRenderable>::iterator itr = transparentMap.begin(); itr != transparentMap.end(); ++itr)
	{
		for (vectorRenderable::iterator itr1 = itr->second.begin(); itr1 != itr->second.end(); ++itr1)
		{
			HXRenderable* renderable = *itr1;
			if (shadow && !renderable->m_pSubMesh->IsCastShadow)
			{
				continue;
			}
			if (curMaterial != renderable->m_pMaterial)
			{
				curMaterial = renderable->m_pMaterial;
				if (shadow)
				{
					curMaterial->SetShadowMapMaterialRenderStateAllRenderable();
				}
				else
				{
					curMaterial->SetMaterialRenderStateAllRenderable();
				}
			}
			HXStatus::GetInstance()->nVertexCount += renderable->m_pSubMesh->vertexList.size();
			HXStatus::GetInstance()->nTriangleCount += renderable->m_pSubMesh->triangleCount;
			if (shadow)
			{
				curMaterial->SetShadowMapMaterialRenderStateEachRenderable(renderable);
			}
			else
			{
				curMaterial->SetMaterialRenderStateEachRenderable(renderable);
			}
			pRenderSystem->RenderSingle(renderable, shadow);
		}
	}
	curMaterial = NULL;

	if (!shadow)
	{
		HXStatus::GetInstance()->ShowStatusInfo();
	}
}
```
## 后记
注意：每帧渲染时shader uniform的texture变量需要重新处理（赋值的只是纹理绑定点，纹理绑定点绑定的纹理可能被改掉了）
```
GLint tex_uniform_loc = glGetUniformLocation(render_scene_prog, (itr->name).c_str());
if (tex_uniform_loc == -1)
{
	// 未参被实际调用的变量编译后会被自动删除
	continue;
}
HXGLTexture* tex = (HXGLTexture*)HXResourceManager::GetInstance()->GetTexture("GL_" + itr->value);
glUniform1i(tex_uniform_loc, nTexIndex);
glActiveTexture(GL_TEXTURE0 + nTexIndex);
glBindTexture(GL_TEXTURE_2D, tex->texId);
```
后续考虑：  
进行静态批处理，把静态物体提前生成到同一个vbo中以合并drawcall。  
GameObjectTree进行八叉树管理等
