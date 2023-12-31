---
layout: post
title: "在Unity中实现NPR渲染是否搞错了什么？-卡通角色描边实现"
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

上一篇我们完成了角色卡通渲染的着色，除了着色以外我们还需要实现描边效果，使我们的渲染更贴近赛璐璐风格动画

# 着色效果分析

赛璐璐风格动画的三个特点：清晰的线条描边，大色块，边缘锐利的阴影，之前的文章中已经实现了大色块和边缘锐利的阴影，我们希望我们的渲染更加卡通加入线条描边是一个不错的方法。

描边的算法在《Real-Time Rendering 4rd》中作者总结了五种方法来实现

- 基于视点方向的描边
- 基于过程几何方法的描边
- 基于图像处理的描边
- 基于轮廓边缘检测的描边
- 混和轮廓描边

本次我们使用基于过程几何方法的Back facing描边法，其原理是通过两次绘制一次绘制对象一次以法线偏移的方式绘制描边

# 从二开始的卡通着色器之旅

（1） 延续上一节Unity urp项目

（2） 继续使用上一节的着色器和材质

（3）新建一个Pass

    Pass
    {
        Name "Outline"
        Tags {"LightMode" = "SRPDefaultUnlit"}
    
        Cull Front
    
        HLSLPROGRAM
    
        #pragma vertex vert
        #pragma fragment frag
    
        ENDHLSL
    }


（4） 接下来我们来实现描边

 将我们需要描边计算的变量在Properties中声明

    _OutlineWidth ("Outline Width", Range(0, 1.0)) = 0.2 //描边宽度
    _OutlineColor ("outline Color", Color) = (0, 0, 0, 0) //描边颜色

让我们的顶点向法线方向偏移

    v2f vert(appdata v) 
    {
        VertexPositionInputs vertexInput = GetVertexPositionInputs(v.vertex.xyz + v.normal * _OutlineWidth );//获取沿法线方向偏移的顶点信息
        
        v2f o;
        o.pos = vertexInput.positionCS;
    
        return o;
    }
    
    half4 frag(v2f i) : SV_TARGET
    {
        return _OutlineColor; //输出描边颜色
    }

![](https://hcidsi-blog-1317560990.cos.ap-shanghai.myqcloud.com/img/Snipaste_2023-10-31_14-04-36.png)

得到以上结果（为了便于观察去除对象着色，将描边颜色调为红色），但我们的描边还存在一些问题比如

![](https://hcidsi-blog-1317560990.cos.ap-shanghai.myqcloud.com/img/Snipaste_2023-10-31_14-10-38.png)

如果对象有硬边缘会导致描边断裂，这会使我们最后的效果大打折扣

这个问题困扰了我好久，试了很多方法去解决这个问题比如尝试后处理描边（但后处理得到的效果并不好），之后看了一些其他的解决方法后得到了几种不错的方法解决这个问题，平滑法线，修改模型信息等，这一次我们使用平滑法线来得到不错的描边效果

![产生问题的原因](https://hcidsi-blog-1317560990.cos.ap-shanghai.myqcloud.com/img/Snipaste_2023-10-31_14-33-40.png)

我们的描边会出现问题主要是因为法线垂直于顶点所在平面，所以在尖锐边缘处法线方向会剧烈变化，导致我们沿法线方向偏移的顶点之间产生空隙，为了解决这个问题，我们可以先对模型法线进行一次平均，再使用平滑后的法线进行描边计算，有了这个方法，那我们就写一个平滑模型法线的脚本挂载在需要平滑法线的模型上

    using System.Collections;
    using System.Collections.Generic;
    using UnityEngine;
    
    public class SmoothNormalsWeighted : MonoBehaviour
    {
        private void Awake()
        {
            Mesh mesh = GetComponent<SkinnedMeshRenderer>().sharedMesh;
            Vector3[] vertices = mesh.vertices;
            Vector3[] normals = mesh.normals;
            Vector4[] tangents = mesh.tangents;
    
            // 创建一个字典以存储每个顶点的加权平均法线
            Dictionary<Vector3, Vector3> smoothNormals = new Dictionary<Vector3, Vector3>();
    
            for (int i = 0; i < vertices.Length; i++)
            {
                Vector3 vertex = vertices[i];
                Vector3 normal = normals[i];
    
                if (smoothNormals.ContainsKey(vertex))
                {
                    // 如果顶点已经在字典中，则累加法线
                    smoothNormals[vertex] += normal;
                }
                else
                {
                    // 否则，在字典中创建一个新条目
                    smoothNormals[vertex] = normal;
                }
            }
    
            // 更新切线数据以存储平滑法线
            for (int i = 0; i < vertices.Length; i++)
            {
                Vector3 vertex = vertices[i];
                Vector3 smoothNormal = smoothNormals[vertex].normalized;
    
                // 更新切线数据
                tangents[i] = new Vector4(smoothNormal.x, smoothNormal.y, smoothNormal.z, 1f);
            }
    
            // 将更新后的切线数据应用到网格
            mesh.tangents = tangents;
        }
    }
![描边对比右为平滑法线后的效果](https://hcidsi-blog-1317560990.cos.ap-shanghai.myqcloud.com/img/sh4001.png)

我们使用这个脚本对模型每个顶点都法线做了一次加权平均，并将平滑的法线信息储存在切线信息中，为什么是切线信息中对于有骨骼动画的模型只有法线和切线会随骨骼动画而变换，这样可以不用对平滑后的法线再做变换，节省计算资源，因为现在我们的平滑法线信息储存在切线中，所以我们的着色器也需要做一点变化

     //获取沿平滑法线（储存在切线中）方向偏移的顶点信息
     VertexPositionInputs vertexInput = GetVertexPositionInputs(v.vertex.xyz + v.tangent * _OutlineWidth * 0.1);//获取沿平滑法线（储存在切线中）方向偏移的顶点信息

我们将着色器应用在我们的角色上 

![描边对比：右为平滑法线后的效果](https://hcidsi-blog-1317560990.cos.ap-shanghai.myqcloud.com/img/sh4002.png)

右边的断线情况明显更少

有时候LightMap中会储存描边的颜色信息，通过LightMap和Ramp结合提取最后的描边颜色，稍微修改一下我们的着色器得到

    v2f vert(appdata v) 
    {
        //获取沿平滑法线（储存在切线中）方向偏移的顶点信息
        VertexPositionInputs vertexInput = GetVertexPositionInputs(v.vertex.xyz + v.tangent * _OutlineWidth * 0.002);//获取沿平滑法线（储存在切线中）方向偏移的顶点信息
        
        v2f o;
        o.pos = vertexInput.positionCS;
        o.uv = TRANSFORM_TEX(v.uv, _LightMap);
    
        return o;
    }
    
    half4 frag(v2f i) : SV_TARGET
    {
        half4 lightMap = SAMPLE_TEXTURE2D(_LightMap, sampler_LightMap, i.uv);
        int index = 4;
    
        index = lerp(index, 1, step(0.2, lightMap.a));
        index = lerp(index, 3, step(0.8, lightMap.a));


        half4 FColor = SAMPLE_TEXTURE2D(_ShadowRamp, sampler_ShadowRamp, half2(0.75, index / 10.0 + 0.005 +0.5));
    
        return FColor; //输出描边颜色
    }

![彩色描边](https://hcidsi-blog-1317560990.cos.ap-shanghai.myqcloud.com/img/Snipaste_2023-10-31_16-26-10.png)

# 结语

![渲染最终结果](https://hcidsi-blog-1317560990.cos.ap-shanghai.myqcloud.com/img/Snipaste_2023-10-31_16-45-27.png)

我们将描边效果结合上一节的渲染效果得到以上结果，我们得到了一个完成卡通渲染的效果，完整代码已上传[Github](https://github.com/HciDsi/GenshinRender_Like)代码在Shades目录下的Hutao.shader，现在我们的渲染已经实现了卡通渲染效果但和原神的效果还有差距，比如原神的阴影，原神的后处理，之后我们会逐一实现


# 参考
>https://zhuanlan.zhihu.com/p/109101851

>https://zhuanlan.zhihu.com/p/508319122

>https://zhuanlan.zhihu.com/p/31194204

>https://zhuanlan.zhihu.com/p/110025903

>https://zhuanlan.zhihu.com/p/330599077