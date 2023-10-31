---
layout: post
title: "在Unity中实现NPR渲染是否搞错了什么？-基础卡通角色着色"
subtitle: "——Unity URP实现"
author: "HciDsi"
header-img: "img/IMG_2874.JPG"
header-mask: 0.3
mathjax: true
tags:
  - Unity
  - Shader
  - NPR
---
>[项目地址](https://github.com/HciDsi/GenshinRender_Like)

>Unity 2023.1
### 前言

从本篇开始我们开始要实现卡通渲染相关技术，这里我们使用Unity 来实现相关效果，首先我们以原神中的胡桃为例实现角色的卡通渲染

### 着色效果分析

我们需要实现2D风格的渲染，首先我们对赛璐璐风格进行分析，赛璐璐风格动画有以下的特点

![赛璐璐动画](https://hcidsi-blog-1317560990.cos.ap-shanghai.myqcloud.com/img/sh2001.png)

1. 清晰的线条描边
2. 大色块
3. 边缘锐利的阴影

我们的着色器也需要实现这些着色效果以达到仿卡通的着色效果

### 从零开始的卡通着色器之旅

（1） 新建一个Unity urp项目

（2） 新建一个Unity shader，直接新建一个Unlit shader就可以，然后将我们的shader改成urp 形式，以这个shader 新建一个Material

（3）首先我们来实现顶点着色器

    
    v2f vert (appdata v)
    {
        VertexPositionInputs vertexInput = GetVertexPositionInputs(v.vertex.xyz); 
        VertexNormalInputs normalInput = GetVertexNormalInputs(v.normal, v.tangent);

        v2f o;
        o.pos = vertexInput.positionCS; //获取裁剪空间顶点坐标
        o.positionWS = vertexInput.positionWS; //获取世界空间顶点坐标
        o.positionNDC = vertexInput.positionNDC; //获取标准化设备坐标的顶点坐标
        o.normalWS = normalInput.normalWS; //获取世界空间法线向量
        o.tangentWS = normalInput.tangentWS; //获取世界空间切线向量
        o.bitangentWS = normalInput.bitangentWS; //获取世界空间副切线向量
        o.viewDir = GetWorldSpaceNormalizeViewDir(o.positionWS); //获取世界空间视线向量
        o.uv = TRANSFORM_TEX(v.uv, _BaseMap); //获取对象UV信息

        return o;
    }
    

（4） 接下来我们来实现基础的着色

 将我们需要的变量在Properties中声明


    [MainTexture] _BaseMap ("Base Map", 2D) = "while" {} //基础纹理
    [MainColor] _BaseColor ("Base Color", Color) = (1, 1, 1, 1) //基础色
    _ToonFac ("Toon Fac", Range(0, 1)) = 0.5 //卡通纹理混合度
    _ToonMap ("Toon Map", 2D) = "while" {} //卡通纹理
    _LightMap ("Light Map", 2D) = "while" {} //光照贴图


      
 获取一些我们需要信息（如光线方向，法线方向，视线方向等），通过基础纹理和卡通纹理获取对象基础颜色
    
    
    #if _NORMAL_MAP //判断是否使用法线贴图
        half3 bump = UnpackNormal(SAMPLE_TEXTURE2D(_NormalMap, sampler_NormalMap, i.uv)); //获取切线空间的法线贴图信息
        half3x3 tangentToWorld = half3x3(i.tangentWS, i.bitangentWS, i.normalWS); //获取从切线空间到世界空间的矩阵
        i.normalWS = TransformTangentToWorld(bump, tangentToWorld, true); //得到世界空间的法线贴图信息
    #endif

    Light mainLight = GetMainLight(); //主光源
    half3 N = SafeNormalize(i.normalWS); //归一化的世界空间法线
    half3 V = SafeNormalize(i.viewDir); //归一化的世界空间视线方向
    half3 L = SafeNormalize(mainLight.direction); //归一化的世界空间主光源方向
    half3 H = SafeNormalize(V + L); //归一化的世界空间半程向量

    half2 matcapUV = SafeNormalize(mul((half3x3)UNITY_MATRIX_V, N)).xy * 0.5 + 0.5; //以视线空间法线获取一个matcapUV
    half4 lightMap = SAMPLE_TEXTURE2D(_LightMap, sampler_LightMap, i.uv); //获取光照贴图

    //BaseColor
    half4 baseMap = SAMPLE_TEXTURE2D(_BaseMap, sampler_BaseMap, i.uv); //对主纹理进行采样
    half3 toon = SAMPLE_TEXTURE2D(_ToonMap, sampler_ToonMap, matcapUV); //以matcapUV对卡通纹理进行采样
    half3 albedo = lerp(_BaseColor, _BaseColor.rgb * baseMap.rgb, 1); //将主纹理与基础色混合
    albedo = lerp(albedo, albedo * toon, _ToonFac); ////将基础色与卡通纹理混合
    
       
基础纹理的采样就先省略了，介绍一下卡通纹理实现的效果，首先我们去掉基础纹理（减少其他颜色对我们观察结果的影响），我们可以看下面的对比图，可以看出来使用了卡通纹理的渲染在没有光照参与的情况下也有立体效果

![左无卡通纹理：右使用卡通纹理](https://hcidsi-blog-1317560990.cos.ap-shanghai.myqcloud.com/img/sh2002.png)

（5） 接下来我们使用半兰伯特法配合光照贴图和ramp图实现我们的卡通渲染漫反射效果

获取漫反射需要的相关参数

    _ShadowColor ("Shadow Color", Color) = (1, 1, 1, 1) //阴影颜色
    _ShadowOffset ("Shadow Offset", Range(0, 1)) = 0.3 //阴影偏移量
    _ShadowSmoothness ("Shadow Smoothness", Range(0, 1)) = 0.2 //阴影软化范围
    _ShadowRamp("Shadow Ramp", 2D) = "white" {} //Ramp图
   
使用半兰伯特法求阴影范围
    
    half halfLambert = pow(dot(L, N) * 0.5 + 0.5, 2); //得到半兰伯特法的漫反射参数
    half shadow = shadow = saturate(halfLambert * lightMap.g * 2); //用光照贴图的g通道（ao贴图）获取对象阴影信息
    shadow = lerp(shadow, 1, step(0.9, lightMap.g)); //对阴影范围进行过滤
    

![lightMap](https://hcidsi-blog-1317560990.cos.ap-shanghai.myqcloud.com/img/sh2003.png)

光照贴图的r通道储存非金属高光信息，g通道存储阴影信息，b通道存储金属高光信息，a通道ramp通道信息。

![漫反射效果（使用lightMap为右）](https://hcidsi-blog-1317560990.cos.ap-shanghai.myqcloud.com/img/sh2004.png)

使用以上代码我们就可以得到这样的效果（我们结果都去掉基础纹理以减少对我们观察的影响）

        
    //获取Ramp的坐标信息
    int index = 4;
    index = lerp(index, 1, step(0.2, lightMap.a));
    index = lerp(index, 2, step(0.4, lightMap.a));
    index = lerp(index, 0, step(0.6, lightMap.a));
    index = lerp(index, 3, step(0.8, lightMap.a));
    //阴影范围
    half rampMax = _ShadowOffset / 2; 
    half rampMin = _ShadowOffset / 2 - _ShadowSmoothness / 2;
    //获取RampUV
    half rampU = smoothstep(rampMin, rampMax, shadow) ;
    half rampV = index / 10.0 + 0.05 + _IsDay * 0.5;
    half2 rampUV = half2(rampU, rampV);
    half3 diffuse = SAMPLE_TEXTURE2D(_ShadowRamp, sampler_ShadowRamp, rampUV);//用RampUV对Ramp采样
    diffuse = lerp(diffuse, 1, step(rampMax, shadow)); //将大于阴影范围的设置为无阴影
    

![Diffuse最终结果](https://hcidsi-blog-1317560990.cos.ap-shanghai.myqcloud.com/img/sh2005.png)

以上是我们最终的漫反射效果，可以看到箭头所指的地方有一个渐变阴影的效果，以ramp图和smoothstep()函数实现，我们可以看到漫反射效果得到了赛璐璐动画风格的大色块，边缘锐利的阴影效果

（6） 最后使用BlinnPhone配合光照贴图得到卡通高光效果
   
获取卡通高光需要的相关参数

    _SpecularSmoothness ("Specular Smoothness", Range(8, 256)) = 8 //高光范围
    _NonmetallicIntensity ("Nonmetallic Intensity", Range(0, 1)) = 0.5 //非金属度
    _MetallicIntensty ("Metallic Intensity", Range(1, 25)) = 5//金属度
    _MetalMap("Metal Map", 2D) = "white" {} //金属光泽贴图

使用BlinnPhone配合光照贴图得到卡通高光效果
   
    half blinnPhone = pow(saturate(dot(N, H)), _SpecularSmoothness);
    half3 metalMap = SAMPLE_TEXTURE2D(_MetalMap, sampler_MetalMap, matcapUV); //使用matcapUV对金属光泽纹理采样
    half3 metal = blinnPhone * metalMap * lightMap.b * _MetallicIntensty * 2; //获取金属高光范围
    half3 nonmetal = lightMap.r * step(1.1 - blinnPhone, lightMap.b) * _NonmetallicIntensity; //获取非金属高光范围
    specular = lerp(metal, nonmetal, step(0.9, lightMap.r)); //混合金属高光和非金属高光
    

![Specular最终结果](https://hcidsi-blog-1317560990.cos.ap-shanghai.myqcloud.com/img/sh2006.png)

以上我们得到头发高光等卡通高光的效果

### 结语

![渲染最终结果](https://hcidsi-blog-1317560990.cos.ap-shanghai.myqcloud.com/img/sh2007.png)

我们将基础颜色，漫反射和高光结合得到以上结果，我们可以得到一个不错的渲染效果，完整代码已上传[Github](https://github.com/HciDsi/GenshinRender_Like)，但是还是有一些其他问题比如没有描边，面部阴影问题等，接下来我会在之后的文章中逐渐解决。

    
### 参考
>https://zhuanlan.zhihu.com/p/109101851

>https://zhuanlan.zhihu.com/p/508319122

>https://zhuanlan.zhihu.com/p/31194204

>https://zhuanlan.zhihu.com/p/110025903

>https://zhuanlan.zhihu.com/p/330599077