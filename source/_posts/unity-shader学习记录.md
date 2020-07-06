---
title: unity-shader学习记录
date: 2020-06-16 23:05:44
top_img: /img/cover/shader.jpg
cover: /img/cover/shader.jpg
tags: 笔记
---

## 《Unity Shader 入门摘要》阅读笔记
最近想要系统的学习一下unity shader相关的东西，根据别人的推荐看到了这本书。于是决定边看边把一些比较重要或者原本不知道的知识点记录一下。

原书讲解十分详细，这里只记录本人相关，欲知详情，请移步至[作者github](https://github.com/candycat1992/Unity_Shaders_Book)

### 第3章 Unity Shader基础
**ShaderLab**：Unity为shader开发者专门设计的语言，详情见[官网](https://docs.unity3d.com/Manual/SL-Shader.html)

我们手动开发的Shader都是用ShaderLab实现的，当然中间需要内置```CG/HLSL```代码，完成实际的过程，ShaderLab的作用更像是一个框架。

shader的完整结构如下：
```shader
Shader "ShaderName" {
    Properties {
        // 属性
    }
    SubShader {
        // 显卡A使用的着色器

        [Tags] // 标签：可选的

        [RenderSetup] // 状态：可选的

        Pass {
            [Name]
            [Tags]
            [RenderSetup]
            // other code
        }
        // other Passes
    }
    SubShader {
        // 显卡B使用的着色器
    }
    Fallback "default shader name" // Fall Off
}
```
+ ```Properties```支持如下类型：
    <center>
        <img style="border-radius: 0.3125em;
        box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
        src="/img/shader/properties.png">
        <br>
        <div style="color:orange; border-bottom: 1px solid #d9d9d9;
        display: inline-block;
        color: #999;
        padding: 2px;">
            Unity支持的Propertiies类型
        </div>
    </center>
+ ```Subshader```的```RenderSetup```支持如下类型，其对内部```Pass```来说是全局存在的
    <center>
        <img style="border-radius: 0.3125em;
        box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
        src="/img/shader/subshader_setup.png">
        <br>
        <div style="color:orange; border-bottom: 1px solid #d9d9d9;
        display: inline-block;
        color: #999;
        padding: 2px;">
            常见的渲染状态设置选项
        </div>
    </center>
+ ```Subshader```的```Tags```支持如下类型：
    <center>
        <img style="border-radius: 0.3125em;
        box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
        src="/img/shader/subshader_tags.png">
        <br>
        <div style="color:orange; border-bottom: 1px solid #d9d9d9;
        display: inline-block;
        color: #999;
        padding: 2px;">
            SubShader的标签类型
        </div>
    </center>
+ ```Pass```的```Tags```支持如下类型：
    <center>
        <img style="border-radius: 0.3125em;
        box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
        src="/img/shader/pass_tags.png">
        <br>
        <div style="color:orange; border-bottom: 1px solid #d9d9d9;
        display: inline-block;
        color: #999;
        padding: 2px;">
            Pass的标签类型
        </div>
    </center>
+ ```UsePass```复用其他shader里的代码，```GrabPass```抓取屏幕存储在一张纹理中供后续Pass使用

### 第4章
#### 坐标空间与变换
+ 坐标空间
    + 模型空间：又称“对象空间”，顾名思义是与模型本身有关的空间
    + 世界空间：广义上“最大的空间”
    + 观察空间：摄像机所在的空间，并且摄像机的位置是原点
    + 裁剪空间：摄像机实际上真正能看到的空间，由视锥体决定
    + 屏幕空间：最终显现的二维屏幕（空间）
+ 坐标空间变换
    + 模型变换：模型空间 -> 世界空间
    + 观察变换：世界空间 -> 观察空间
    + 投影变换：观察空间 -> 裁剪空间
    + 屏幕映射：裁剪空间 -> 屏幕空间

转换关系图如下：
<center>
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
    src="/img/shader/space.png">
    <br>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">
        坐标空间及对应变换关系
    </div>
</center>

#### Unity内置的变换矩阵
Unity内置的变换矩阵见下图：（对应上述的空间变换好好研究下）
<center>
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
    src="/img/shader/transform_matrix.png">
    <br>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">
        Unity内置的变换矩阵
    </div>
</center>

#### Unity内置的摄像机和屏幕参数
<center>
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
    src="/img/shader/camera_param.png">
    <br>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">
        Unity内置的摄像机和屏幕参数
    </div>
</center>

### 第5章
+ ShaderLab属性类型与CG变量类型的匹配关系
    | ShaderLab属性类型 | CG变量类型 |
    |---|---|
    | Color, Vector | float4, half4, fixed4 |
    | Range, Float | float, half, fixed |
    | 2D | sampler2D |
    | Cube | SamplerCube |
    | 3D | sampler3D |
+ UnityCG.cginc中一些常用的结构体
    | 名称 | 描述 | 包含的变量 |
    | --- | --- | --- | 
    | appdata_base | 可用于顶点着色器的输入 | 顶点位置、顶点法线、第一组纹理坐标 |
    | appdata_tan | 可用于顶点着色器的输入 | 顶点位置、顶点切线、顶点法线、第一组纹理坐标 |
    | appdata_full | 可用于顶点着色器的输入 | 顶点位置、顶点切线、顶点法线、四组(或更多)纹理坐标 |
    | appdata_img | 可用于顶点着色器的输入 | 顶点位置、第一组纹理坐标 |
    | v2f_img | 可用于顶点着色器的输出 | 裁剪空间中的位置、纹理坐标 |
+ UnityCG.cginc中一些常用的帮助函数
    <center>
        <img style="border-radius: 0.3125em;
        box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
        src="/img/shader/cg_cginc.png">
        <br>
        <div style="color:orange; border-bottom: 1px solid #d9d9d9;
        display: inline-block;
        color: #999;
        padding: 2px;">
            UnityCG.cginc中一些常用的帮助函数
        </div>
    </center>

+ 可用于顶点着色器的输入的语义
    | 语义 | 描述 |
    | --- | --- |
    | POSITION | 模型空间中的顶点位置，通常是float4类型 |
    | NORMAL | 顶点法线，通常是float3类型 |
    | TANGENT | 顶点切线，通常是float4类型 |
    | TEXCOORDn | 该顶点的第n组纹理坐标，通常是float2或float4类型 |
    | COLOR | 顶点颜色，通常是float4或fixed4类型 |

    + 系统数值语义：System-Value sematics，以```SV```开头
        + SV_POSITION: 顶点着色器的输出pos
        + SV_TARGET: 片元着色器的输出（输出到颜色缓冲）

### 第6章
#### 光照模型的计算
+ 逐像素光照
    + 主要计算过程在**片元着色器**中；
    + 以每个像素为基础，得到其法线（顶点发现插值或法线纹理采样），再进行光照模型的计算。这种在面片之间对顶点法线进行插值的技术被称为**Phong着色(Phong Shading)**
+ 逐顶点光照
    + 主要计算过程在**顶点着色器**中；
    + **Gouraud shading（高洛德着色）**，在每个顶点上计算光照，然后在渲染图元内部进行线性插值，最后输出像素颜色。

由于顶点数往往小于像素数量，因此逐顶点光照的计算量更小；但是如果模型中有非线性插值的计算，逐顶点光照的结果会有问题（棱角现象）。

### 第7章
**纹理映射**：将纹理图附在模型表面，逐纹素的控制模型的颜色
**凹凸映射**：使用一张纹理图（法线纹理/高度纹理）来改变模型表面的法线，为模型提供更多细节
#### 凹凸映射
+ 两种实现方式：
    + **高度映射**：使用一张**高度纹理**模拟表面位移，以得到一个修改后的法线值
    + **法线映射**：使用一张**法线纹理**直接存储表面法线
        + 模型空间的法线纹理：修改后的模型空间的表面法线存储在一张纹理中
        + 切线空间的法线纹理：对每个顶点，求出顶点在其切线空间内的法线，将其存储到法线纹理中。***这种方式由于存储的是相对法线，效果更好，更受欢迎。***

### 第8章
#### 透明度测试
设置一个阈值，当片元的透明度不满足条件时，该片元会被舍弃（不再做任何处理），否则就视为不透明物体处理。因此它并不能实现真正的半透明效果。

在片元着色器中调用`clip(float/float2/float3/float4 x)`函数进行透明度测试，当传入的参数有任何一个分量是负数，就会舍弃当前像素的输出颜色

#### 透明度混合
可以实现真正的半透明效果。他会使用当前片元的透明度作为混合因子，与已经存储在颜色缓冲中的颜色值进行混合，得到新的颜色。
    
需要关闭**深度写入**，但不关闭**深度检测**，即计算半透明物体时，深度缓冲是**只读**的。这会导致渲染的顺序十分重要，因为这时深度缓冲无法完全正常工作。

**解决思路**：先渲染所有的不透明物体，这里深度关系会一切正常；再将半透明物体加入渲染（半透明物体之间的遮挡关系仍有问题，先不考虑）。

混合命令：**Blend**，开启混合后，对alpha的修改才有意义。
<center>
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
    src="/img/shader/blend.png">
    <br>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">
        ShaderLab的Blend命令
    </div>
</center>
<center>
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
    src="/img/shader/blend_param.png">
    <br>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">
        ShaderLab的混合因子
    </div>
</center>
<center>
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
    src="/img/shader/blend_op.png">
    <br>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">
        ShaderLab的混合操作
    </div>
</center>

#### 渲染队列
Unity提供了**渲染队列**以解决渲染顺序的问题。使用`Subshader`里的**Queue**标签决定模型归于哪个队列。

Unity内部使用整数索引来表示每个渲染队列，索引号越小表示越早渲染。

<center>
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
    src="/img/shader/queue.png">
    <br>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">
        Unity提前定义的5个渲染队列
    </div>
</center>

#### Cull命令
| 命令 | 含义 |
| --- | --- |
| Cull Off | 关闭剔除 |
| Cull Back | 剔除背面，只渲染正面，default |
| Cull Front | 提出正面，只渲染背面 |

### 第12章 屏幕后处理效果
**屏幕后处理**：通常指的是在渲染完整个场景得到屏幕图像之后，再对这个图像进行一系列操作，实现各种屏幕特效（景深、运动模糊等）。

####  函数接口：
+ OnRenderImage函数：
    ```c#
    MonoBehaviour.OnRenderImage(RenderTexture src, RenderTexture dest)
    ```
    `src`是Unity当前渲染得到的源渲染纹理；`dest`是经过`OnRenderImage`函数处理之后的目标渲染纹理，之后会被渲染到屏幕上
+ Graphics.Blit函数
    ```c#
    public static void Blit(Texture src, RenderTexture dest);
    public static void Blit(Texture src, RenderTexture dest, Material mat, int pass=-1);
    public static void Blit(Texture src, Material mat, int pass=-1);
    ```
    `src`对应了源纹理，通常就是当前屏幕的渲染纹理或是上一步处理后得到的渲染纹理；
    `dest`对应了目标渲染纹理，若为null则直接渲染到屏幕上；
    `mat`是我们使用的材质，它使用的shader会进行一系列屏幕后处理操作，而`src`纹理将会被传递给shader中`_MainTex`的纹理属性；
    `pass`的默认值为-1，表示会一次调用shader内的所有Pass；否则只调用给定索引的Pass；
