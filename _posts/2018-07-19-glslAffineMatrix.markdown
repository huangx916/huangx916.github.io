---
layout:     post
title:      "线性变化、仿射变换、正交矩阵概念及矩阵求逆"
subtitle:   " \"游戏引擎\""
date:       2018-07-29 21:38:00
author:     "A-SHIN"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - 技术
---

> “Yeah It's on. ”

## 前言
基础概念：  
3D向量(x,y,z)变换的处理方式如下：  
旋转变换：乘法运算  
缩放变换：乘法运算  
平移变换：加法运算  
加入齐次坐标(x,y,z,w)概念后，使得平移变换也能使用矩阵乘法。点w为1，向量w为0。  
其中旋转缩放属于线性变化，平移为非线性。  
仿射变换：先进行线性变换，再进行平移变换的变换称之为仿射变换。  
如果矩阵第四行为[0,0,0,1]则为仿射矩阵。  
3D游戏开发中常用MVP，其中M、V、MV都属于仿射矩阵。P不属于仿射矩阵。常用的矩阵求逆操作(如：求视图空间法线)多数为仿射矩阵求逆。  
正交矩阵：矩阵中的任意两个行向量的乘积为0，矩阵中每一个行向量都是单位向量。  
没有经过缩放的旋转矩阵就是一个正交矩阵。 其转置矩阵即为逆矩阵。 
## 正文  
仿射矩阵求逆:  
3x3一般矩阵：  
If cannot find inverse (det=0), set identity matrix  
M^-1 = adj(M) / det(M)  
       | m4m8-m5m7  m5m6-m3m8  m3m7-m4m6 |  
     = | m7m2-m8m1  m0m8-m2m6  m6m1-m7m0 | / det(M)  
       | m1m5-m2m4  m2m3-m0m5  m0m4-m1m3 |  
```
Matrix3& Matrix3::invert()
{
    float determinant, invDeterminant;
    float tmp[9];

    tmp[0] = m[4] * m[8] - m[5] * m[7];
    tmp[1] = m[7] * m[2] - m[8] * m[1];
    tmp[2] = m[1] * m[5] - m[2] * m[4];
    tmp[3] = m[5] * m[6] - m[3] * m[8];
    tmp[4] = m[0] * m[8] - m[2] * m[6];
    tmp[5] = m[2] * m[3] - m[0] * m[5];
    tmp[6] = m[3] * m[7] - m[4] * m[6];
    tmp[7] = m[6] * m[1] - m[7] * m[0];
    tmp[8] = m[0] * m[4] - m[1] * m[3];

    // check determinant if it is 0
    determinant = m[0] * tmp[0] + m[1] * tmp[3] + m[2] * tmp[6];
    if(fabs(determinant) <= EPSILON)
    {
        return identity(); // cannot inverse, make it idenety matrix
    }

    // divide by the determinant
    invDeterminant = 1.0f / determinant;
    m[0] = invDeterminant * tmp[0];
    m[1] = invDeterminant * tmp[1];
    m[2] = invDeterminant * tmp[2];
    m[3] = invDeterminant * tmp[3];
    m[4] = invDeterminant * tmp[4];
    m[5] = invDeterminant * tmp[5];
    m[6] = invDeterminant * tmp[6];
    m[7] = invDeterminant * tmp[7];
    m[8] = invDeterminant * tmp[8];

    return *this;
}
```
4x4仿射矩阵:  
M = [ R | T ]  
    [ --+-- ]    (R 表示 3x3 rotation/scale/shear matrix)  
    [ 0 | 1 ]    (T 表示 1x3 translation matrix)  

y = M*x  ->  y = R*x + T  ->  x = R^-1*(y - T)  ->  x = R^-1*y - R^-1*T  

  [ R | T ]-1   [ R^-1 | -R^-1 * T ]  
  [ --+-- ]   = [ -----+---------- ]  
  [ 0 | 1 ]     [  0   +     1     ]  
  
```
Matrix4& Matrix4::invertAffine()
{
    // R^-1
    Matrix3 r(m[0],m[1],m[2], m[4],m[5],m[6], m[8],m[9],m[10]);
    r.invert();
    m[0] = r[0];  m[1] = r[1];  m[2] = r[2];
    m[4] = r[3];  m[5] = r[4];  m[6] = r[5];
    m[8] = r[6];  m[9] = r[7];  m[10]= r[8];

    // -R^-1 * T
    float x = m[12];
    float y = m[13];
    float z = m[14];
    m[12] = -(r[0] * x + r[3] * y + r[6] * z);
    m[13] = -(r[1] * x + r[4] * y + r[7] * z);
    m[14] = -(r[2] * x + r[5] * y + r[8] * z);

    // last row should be unchanged (0,0,0,1)
    //m[3] = m[7] = m[11] = 0.0f;
    //m[15] = 1.0f;

    return * this;
}
```

## 后记
容易混淆概念：线性变化、仿射变换、正交矩阵
