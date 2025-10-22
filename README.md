> [【从UnityURP开始探索游戏渲染】](https://github.com)**专栏-直达**

# **Unity URP 视差贴图介绍与分类**

视差贴图（Parallax Mapping）是一种通过动态偏移纹理坐标来模拟表面凹凸效果的渲染技术，主要用于增强低多边形模型的细节表现。在Unity URP（Universal Render Pipeline）中，它通过高度图（Height Map）和视角方向计算UV偏移，实现更真实的深度感，而无需增加模型顶点数量‌。

# **‌视差贴图的核心分类‌**

## ‌标准视差贴图（Parallax Mapping）

* ‌**原理**‌：基于单次高度采样计算UV偏移，性能消耗低但效果有限，适合移动端或性能敏感场景‌。
* ‌**实现**‌：通过高度图的灰度值（黑色为低点，白色为高点）和视角方向，在切线空间内偏移UV坐标，模拟遮挡效果‌。
* ‌**示例**‌：URP中通过`ParallaxOffset1Step`函数实现单步偏移计算‌。
  + ‌**关键参数**‌

    - `_HeightMap`：存储高度信息的纹理（R通道）
    - `_Parallax`：控制凹凸强度的缩放系数（建议0.02-0.05）
  + ‌**切线空间转换**‌

    - 使用URP内置函数`GetVertexNormalInputs`构建TBN矩阵，将视角方向转换到切线空间
  + ‌**性能优化**‌

    - 单步偏移计算（`ParallaxOffset`）相比多步光线步进（如POM）性能更高，适合移动端
  + ParallaxLit.shader

    ```
    |  |  |
    | --- | --- |
    |  | Shader "Universal Render Pipeline/ParallaxLit" |
    |  | { |
    |  | Properties |
    |  | { |
    |  | _MainTex("Albedo", 2D) = "white" {} |
    |  | _NormalMap("Normal Map", 2D) = "bump" {} |
    |  | _HeightMap("Height Map", 2D) = "white" {} |
    |  | _Parallax("Height Scale", Range(0, 0.1)) = 0.02 |
    |  | } |
    |  |  |
    |  | SubShader |
    |  | { |
    |  | Tags { "RenderPipeline"="UniversalPipeline" } |
    |  |  |
    |  | HLSLINCLUDE |
    |  | #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl" |
    |  |  |
    |  | TEXTURE2D(_MainTex);    SAMPLER(sampler_MainTex); |
    |  | TEXTURE2D(_NormalMap);  SAMPLER(sampler_NormalMap); |
    |  | TEXTURE2D(_HeightMap);  SAMPLER(sampler_HeightMap); |
    |  | float _Parallax; |
    |  |  |
    |  | float2 ParallaxOffset(float3 viewDirTS, float height) |
    |  | { |
    |  | height = height * _Parallax - _Parallax * 0.5; |
    |  | return height * (viewDirTS.xy / (viewDirTS.z + 0.42)); |
    |  | } |
    |  | ENDHLSL |
    |  |  |
    |  | Pass |
    |  | { |
    |  | HLSLPROGRAM |
    |  | #pragma vertex vert |
    |  | #pragma fragment frag |
    |  |  |
    |  | struct Attributes |
    |  | { |
    |  | float4 positionOS : POSITION; |
    |  | float2 uv : TEXCOORD0; |
    |  | float3 normalOS : NORMAL; |
    |  | float4 tangentOS : TANGENT; |
    |  | }; |
    |  |  |
    |  | struct Varyings |
    |  | { |
    |  | float4 positionCS : SV_POSITION; |
    |  | float2 uv : TEXCOORD0; |
    |  | float3 viewDirTS : TEXCOORD1; |
    |  | }; |
    |  |  |
    |  | Varyings vert(Attributes IN) |
    |  | { |
    |  | Varyings OUT; |
    |  | VertexPositionInputs posInput = GetVertexPositionInputs(IN.positionOS.xyz); |
    |  | OUT.positionCS = posInput.positionCS; |
    |  |  |
    |  | // 计算切线空间视角方向 |
    |  | VertexNormalInputs normInput = GetVertexNormalInputs(IN.normalOS, IN.tangentOS); |
    |  | float3 viewDirWS = GetWorldSpaceViewDir(posInput.positionWS); |
    |  | OUT.viewDirTS = TransformWorldToTangent(viewDirWS, normInput.tangentWS, normInput.bitangentWS, normInput.normalWS); |
    |  |  |
    |  | OUT.uv = IN.uv; |
    |  | return OUT; |
    |  | } |
    |  |  |
    |  | half4 frag(Varyings IN) : SV_Target |
    |  | { |
    |  | // 采样高度图并计算UV偏移 |
    |  | float height = SAMPLE_TEXTURE2D(_HeightMap, sampler_HeightMap, IN.uv).r; |
    |  | float2 offset = ParallaxOffset(normalize(IN.viewDirTS), height); |
    |  |  |
    |  | // 应用偏移后采样纹理 |
    |  | float2 parallaxUV = IN.uv + offset; |
    |  | half4 albedo = SAMPLE_TEXTURE2D(_MainTex, sampler_MainTex, parallaxUV); |
    |  | half3 normalTS = UnpackNormal(SAMPLE_TEXTURE2D(_NormalMap, sampler_NormalMap, parallaxUV)); |
    |  |  |
    |  | return half4(albedo.rgb, 1); |
    |  | } |
    |  | ENDHLSL |
    |  | } |
    |  | } |
    |  | } |
    ```

## ‌陡峭视差贴图（Steep Parallax Mapping）

* ‌**特点**‌：针对陡峭地形优化，通过多次采样（如二分法）减少失真，适合高落差表面（如岩石、冰缝）‌。
* ‌**实现**‌：在URP中结合光线步进（Raymarching）算法，逐层检测高度图以确定最终UV偏移‌。

## ‌视差遮挡贴图（Parallax Occlusion Mapping, POM）

* ‌**优势**‌：在标准视差基础上增加遮挡计算，通过多次采样模拟更精确的深度感，但性能开销较高‌。
* ‌**应用**‌：常用于风格化材质（如风化岩石、冰面裂缝），需配合高度图和法线贴图使用‌。

# **‌技术实现细节‌**

## **核心原理**

* ‌**高度图采样**‌
  + 使用灰度图（通常存储在法线贴图的Alpha通道）表示表面深度，黑色（0）为最低点，白色（1）为最高点。
* ‌**切线空间转换**‌
  + 将视角方向从世界空间转换到切线空间（通过TBN矩阵），确保偏移方向与模型表面法线对齐。
* ‌**UV偏移计算**
  + 根据高度值和视角方向动态调整UV坐标，核心公式为‌ $offset=\frac{height \cdot viewDir\_{xy}}{viewDir\_z}$
  + 其中`viewDir`需归一化，且需避免除零（通常添加小偏移量如`viewDir.z + 0.42`）

## ‌**URP中的Shader实现**‌

* 核心函数：`ParallaxOffset1Step`（单步偏移）或自定义光线步进算法‌。
* 示例代码（HLSL）：

  ```
  |  |  |
  | --- | --- |
  |  | hlsl |
  |  | float2 ParallaxOffset(half3 viewDirTS, half height, half scale) { |
  |  | return height * (viewDirTS.xy / viewDirTS.z) * scale; |
  |  | } |
  ```

## ‌**性能优化**‌

* 使用`_Parallax`参数控制强度，避免过度偏移导致穿帮‌。
* 移动端建议采用标准视差贴图，PC端可尝试POM‌。

# **‌与其他贴图技术的对比‌**

| 技术 | 原理差异 | 性能开销 | 适用场景 | 优势 | 局限性 |
| --- | --- | --- | --- | --- | --- |
| 法线贴图 | 仅改变光照计算 | 低 | 通用细节增强 | 性能最优 | 平视视角易穿帮 |
| 视差贴图 | 动态UV偏移模拟深度 | 中 | 风格化材质、地形 | 真实遮挡效果 | 陡峭边缘可能失真 |
| 置换贴图 | 实际修改顶点位置 | 高 | 高精度模型（如地形） | 几何级精度 | 需要曲面细分支持 |

# **‌总结‌**

视差贴图在URP中通过动态UV偏移和高度图采样，有效平衡了性能与视觉效果。根据需求选择标准、陡峭或POM变种，可适配不同场景的细节要求‌

---

> [【从UnityURP开始探索游戏渲染】](https://github.com):[FlowerCloud机场订阅官网](https://hanlianfangzhi.com)**专栏-直达**
> （欢迎*点赞留言*探讨，更多人加入进来能更加完善这个探索的过程，🙏）
