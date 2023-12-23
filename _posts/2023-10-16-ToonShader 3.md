---
layout: post
title: "在Unity中实现NPR渲染是否搞错了什么？-特殊卡通角色着色"
subtitle: "——Unity URP实现"
author: "HciDsi"
header-img: "img/h000.png"
header-mask: 0.3
mathjax: true
tags:
  - Unity
  - Shader
  - NPR
---
>[项目地址](https://github.com/HciDsi/GenshinRender_Like)

>Unity 2023.1
---
layout: post
title: "在Unity中实现NPR渲染是否搞错了什么？-特殊卡通角色着色"
subtitle: "——Unity URP实现"
author: "HciDsi"
header-img: "img/h000.png"
header-mask: 0.3
mathjax: true
tags:
  - Unity
  - Shader
  - NPR
---
>[项目地址](https://github.com/HciDsi/GenshinRender_Like)

>Unity 2023.1
# 前言

上一篇我们完成了角色卡通渲染的基础着色，但是我们还有一些问题没有解决，这一次我们来解决上次没有解决的问题，我们来解决等屏幕边缘光和角色面部阴影的问题

# 着色效果分析

首先我来分析卡通渲染的面部阴影，卡通渲染的面部阴影应该满足赛璐璐风格的色彩特征（大色块，边缘锐利的阴影），使用上一篇的shader我们得到的效果与原神的角色渲染对比如下

![基础阴影与原神阴影对比](https://hcidsi-blog-1317560990.cos.ap-shanghai.myqcloud.com/img/sh3001.jpg)

同时我们可以看出来原神的渲染是存在边缘光的

![原神边缘光](https://hcidsi-blog-1317560990.cos.ap-shanghai.myqcloud.com/img/Snipaste_2023-10-30_21-15-43.png)

这一次我们来实现这些效果

# 从一开始的卡通着色器之旅

（1） 延续上一节Unity urp项目

（2） 继续使用上一节的着色器和材质

（3）继续使用继续使用我们上一次的顶点着色器
    

（4） 接下来我们来实现面部阴影的着色

 将我们需要在面部阴影计算的变量在Properties中声明


    _FaceDirection("Face Direction", Vector) = (0,0,1,0) //面朝方向向量
    _FaceShadowOffset("Face Shadow Offset", Float) = 0 //面部阴影偏移距离
    _FaceBlushColor("Face Blush Color", Color) = (1,1,1,1) //
    _FaceBlushStrength("Face Blush Strength", Float) = 1
    _FaceLightMap("Face Light Map", 2D) = "white" {} //面部光照贴图
    _FaceShadow("Face Shadow", 2D) = "white" {} //面部阴影范围

接下来我们使用SDF贴图获取面部球形阴影



    half3 faceDir = half3(_FaceDirection.x, 0.0, _FaceDirection.z); //获取面部向量
    half3 lightDir = half3(L.x, 0.0, L.z); //获取光照方向向量
    half FdotL = dot(faceDir, lightDir); //获取光照方向与面部向量的点乘
    half FcrossL = cross(faceDir, lightDir).y; //获取光照方向与面部向量的朝向
    
    half2 faceUV = i.uv;
    faceUV.x = lerp(faceUV.x, 1 - faceUV.x, step(0, FcrossL)); //获取面部阴影方向用于面部阴影UV
    half faceshadow = SAMPLE_TEXTURE2D(_FaceLightMap, sampler_FaceLightMap, faceUV); //采样阴影光照贴图
    shadow = step(-0.5 * FdotL + 0.5, faceshadow); //以兰伯特法对面部阴影采样
    
    half faceMask = SAMPLE_TEXTURE2D(_FaceShadow, sampler_FaceShadow, i.uv).a; //获取面部阴影范围 
    shadow =lerp(shadow, 1.0, faceMask); //对面部阴影范围进行限制

![SDF贴图和面部阴影范围](https://hcidsi-blog-1317560990.cos.ap-shanghai.myqcloud.com/img/sh3003.png)

这样我们可以获得一个不错的面部阴影范围使用ramp贴图我们可以获得效果不错的面部漫反射效果

![SDF贴图采样渲染结果](https://hcidsi-blog-1317560990.cos.ap-shanghai.myqcloud.com/img/SH3003.png)

（5）接下来我们来实现等屏幕距离边缘光

将我们需要在边缘光使用的变量在Properties中声明

    _RimOffset("Rim Offset", Range(0, 1)) = 0.6 //边缘光宽度
    _RimThreshold("Rim Threshold", Range(0, 1)) = 1 //边缘光范围
    _RimIntensity("Rim Intensity", Range(0, 1)) = 1 //边缘光强度

使用深度图偏移获取我们需要的边缘光

    half2 screenUV = i.positionNDC.xy / i.positionNDC.w; //通过屏幕顶点坐标获取屏幕UV 
    half sceneDepth = SampleSceneDepth(screenUV); //获取场景深度
    half depth = LinearEyeDepth(sceneDepth, _ZBufferParams); //获取观察深度
    half2 offsetUV = half2(normalVS.x * _RimOffset * 10.0 / _ScreenParams.x, 0.0); //获取边缘光宽度（偏移量）
    half offsetDepth = SampleSceneDepth(screenUV + offsetUV);
    half offset = LinearEyeDepth(offsetDepth, _ZBufferParams);
    
    rim = smoothstep(0.0, _RimThreshold, offset - depth) * _RimIntensity; //通过偏移量与观察深度的差获取等屏幕距离边缘光
    rim = rim * pow(saturate(1 - dot(N, V)), 5.0); //软化边缘光

使用深度图时我们需要新建一个Pass,使用URP中的方法

    Pass
    {
        Name "DepthNormals"
        Tags{"LightMode" = "DepthNormals"}
    
        ZWrite On
        Cull[_Cull]
    
        HLSLPROGRAM
    
        #pragma vertex DepthNormalsVertex
        #pragma fragment DepthNormalsFragment
    
        #include "Packages/com.unity.render-pipelines.universal/Shaders/LitInput.hlsl"
        #include "Packages/com.unity.render-pipelines.universal/Shaders/LitDepthNormalsPass.hlsl"
    
        ENDHLSL
    }

![等屏幕距离边缘光效果](https://hcidsi-blog-1317560990.cos.ap-shanghai.myqcloud.com/img/Snipaste_2023-10-31_11-54-04.png)

（6）最后我们将发光材质（如神之眼等）加入

    emission = step(0.5, baseMap.a) * _EmissionIntensity * 10; //通过材质透明度判断材质是否为发光件

![](https://hcidsi-blog-1317560990.cos.ap-shanghai.myqcloud.com/img/2023-10-30 224117.png)

以上是发光材质的效果

# 结语

![渲染最终结果](https://hcidsi-blog-1317560990.cos.ap-shanghai.myqcloud.com/img/Snipaste_2023-10-31_16-46-45.png)

我们将面部SDF阴影，等距离边缘光和发光材质结合上一节的渲染效果得到以上结果，我们得到了一个基本完成着色的模型，完整代码已上传[Github](https://github.com/HciDsi/GenshinRender_Like)代码在Shades目录下的Hutao.shader，下一节我们将实现角色基础卡通渲染的最后环节描边。


# 参考
>https://zhuanlan.zhihu.com/p/109101851

>https://zhuanlan.zhihu.com/p/508319122

>https://zhuanlan.zhihu.com/p/31194204

>https://zhuanlan.zhihu.com/p/110025903

>https://zhuanlan.zhihu.com/p/330599077