> [【从UnityURP开始探索游戏渲染】](https://github.com)**专栏-直达**

# **陡峭视差贴图(Steep Parallax Mapping)实现原理**

陡峭视差贴图通过‌**分层深度比较**‌和‌**动态UV偏移**‌技术增强岩石表面立体感.

## ‌**视角自适应分层采样**‌

* 根据视线与表面法线的夹角动态分配采样层数（平视视角增加至12层，俯视视角减少至5层），解决标准视差贴图在平视角度下的失真问题

## ‌**深度图梯度修正**‌

* 引入`_LayerBias`参数（推荐值0.2-0.4）调整UV偏移量计算公式，避免陡峭区域出现采样断裂：

$\Delta UV=\frac{ParallaxScale \cdot ViewDir\_{xy}}{(ViewDir\_z+LayerBias) \cdot LayerCount}$

## ‌**风格化深度增强**‌

* 在最终插值阶段使用`pow(weight,2)`强化轮廓对比度，配合ramp贴图实现卡通化光影过渡效果

---

# **URP HLSL完整实现代码**

## **关键特性说明**

* ‌**动态层数优化**‌：通过`lerp(_MaxLayers, _MinLayers, saturate(dot(float3(0,0,1), viewDirTS)))`实现平视视角自动增加采样精度
* ‌**抗失真处理**‌：`_LayerBias`参数修正陡峭表面的UV偏移计算，避免采样断裂
* ‌**风格化增强**‌：ramp贴图控制光影过渡，边缘光强化轮廓立体感
* StylizedRockParallax.shader

  ```
  |  |  |
  | --- | --- |
  |  | Shader "Universal Render Pipeline/StylizedRockParallax" |
  |  | { |
  |  | Properties |
  |  | { |
  |  | [Header(Base Textures)] |
  |  | _MainTex("Albedo (RGB)", 2D) = "white" {} |
  |  | _NormalMap("Normal Map", 2D) = "bump" {} |
  |  | _HeightMap("Height Map", 2D) = "white" {} |
  |  | _RampTex("Stylized Ramp", 2D) = "white" {} |
  |  |  |
  |  | [Header(Parallax Settings)] |
  |  | _ParallaxScale("Depth Scale", Range(0, 0.15)) = 0.08 |
  |  | _LayerBias("Layer Bias", Range(0.1, 0.5)) = 0.3 |
  |  | _MinLayers("Min Layers", Int) = 5 |
  |  | _MaxLayers("Max Layers", Int) = 12 |
  |  |  |
  |  | [Header(Stylized Lighting)] |
  |  | _RimPower("Rim Power", Range(1, 10)) = 3 |
  |  | _ShadowTint("Shadow Tint", Color) = (0.3,0.3,0.4,1) |
  |  | } |
  |  |  |
  |  | SubShader |
  |  | { |
  |  | Tags { "RenderType"="Opaque" "RenderPipeline"="UniversalPipeline" } |
  |  |  |
  |  | HLSLINCLUDE |
  |  | #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl" |
  |  | #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Lighting.hlsl" |
  |  |  |
  |  | TEXTURE2D(_MainTex);    SAMPLER(sampler_MainTex); |
  |  | TEXTURE2D(_NormalMap);  SAMPLER(sampler_NormalMap); |
  |  | TEXTURE2D(_HeightMap);  SAMPLER(sampler_HeightMap); |
  |  | TEXTURE2D(_RampTex);    SAMPLER(sampler_RampTex); |
  |  |  |
  |  | float _ParallaxScale; |
  |  | float _LayerBias; |
  |  | int _MinLayers, _MaxLayers; |
  |  | float _RimPower; |
  |  | float4 _ShadowTint; |
  |  |  |
  |  | // 陡峭视差映射核心算法 |
  |  | float2 SteepParallaxMapping(float3 viewDirTS, float2 uv) |
  |  | { |
  |  | // 动态层数计算（平视视角增加层数） |
  |  | int numLayers = (int)lerp(_MaxLayers, _MinLayers, saturate(dot(float3(0,0,1), viewDirTS))); |
  |  | float layerHeight = 1.0 / numLayers; |
  |  | float2 deltaUV = _ParallaxScale * viewDirTS.xy / (viewDirTS.z + _LayerBias) / numLayers; |
  |  |  |
  |  | // 光线步进初始化 |
  |  | float currentLayerHeight = 0; |
  |  | float2 currentUV = uv; |
  |  | float currentDepth = 1 - SAMPLE_TEXTURE2D(_HeightMap, sampler_HeightMap, currentUV).r; |
  |  |  |
  |  | // 分层深度检测 |
  |  | [loop] |
  |  | for (int i = 0; i < _MaxLayers; ++i) { |
  |  | if (currentLayerHeight >= currentDepth) break; |
  |  | currentUV -= deltaUV; |
  |  | currentDepth = 1 - SAMPLE_TEXTURE2D(_HeightMap, sampler_HeightMap, currentUV).r; |
  |  | currentLayerHeight += layerHeight; |
  |  | } |
  |  |  |
  |  | // 风格化插值修正 |
  |  | float2 prevUV = currentUV + deltaUV; |
  |  | float prevDepth = currentDepth - layerHeight; |
  |  | float weight = pow((currentLayerHeight - currentDepth) / (prevDepth - currentDepth + 0.001), 2); |
  |  | return lerp(currentUV, prevUV, saturate(weight * 1.5)); |
  |  | } |
  |  |  |
  |  | // 风格化光照计算 |
  |  | half3 StylizedShading(float3 normalWS, float3 viewDirWS, float NdotL) |
  |  | { |
  |  | float rim = pow(1 - saturate(dot(normalWS, viewDirWS)), _RimPower); |
  |  | float2 rampUV = float2(NdotL * 0.5 + 0.5, 0.5); |
  |  | half3 rampColor = SAMPLE_TEXTURE2D(_RampTex, sampler_RampTex, rampUV).rgb; |
  |  | return lerp(rampColor * _ShadowTint.rgb, rampColor, saturate(NdotL + rim)); |
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
  |  | float3 normalWS : TEXCOORD2; |
  |  | float3 viewDirWS : TEXCOORD3; |
  |  | float4 shadowCoord : TEXCOORD4; |
  |  | }; |
  |  |  |
  |  | Varyings vert(Attributes IN) |
  |  | { |
  |  | Varyings OUT; |
  |  | VertexPositionInputs posInput = GetVertexPositionInputs(IN.positionOS.xyz); |
  |  | OUT.positionCS = posInput.positionCS; |
  |  |  |
  |  | VertexNormalInputs normInput = GetVertexNormalInputs(IN.normalOS, IN.tangentOS); |
  |  | float3 viewDirWS = GetWorldSpaceViewDir(posInput.positionWS); |
  |  | OUT.viewDirTS = TransformWorldToTangent(viewDirWS, |
  |  | normInput.tangentWS, normInput.bitangentWS, normInput.normalWS); |
  |  | OUT.normalWS = normInput.normalWS; |
  |  | OUT.viewDirWS = viewDirWS; |
  |  | OUT.shadowCoord = GetShadowCoord(posInput); |
  |  | OUT.uv = IN.uv; |
  |  | return OUT; |
  |  | } |
  |  |  |
  |  | half4 frag(Varyings IN) : SV_Target |
  |  | { |
  |  | // 计算陡峭视差UV |
  |  | float2 parallaxUV = SteepParallaxMapping(normalize(IN.viewDirTS), IN.uv); |
  |  |  |
  |  | // 采样纹理 |
  |  | half4 albedo = SAMPLE_TEXTURE2D(_MainTex, sampler_MainTex, parallaxUV); |
  |  | half3 normalTS = UnpackNormal(SAMPLE_TEXTURE2D(_NormalMap, sampler_NormalMap, parallaxUV)); |
  |  |  |
  |  | // 转换法线到世界空间 |
  |  | float3x3 TBN = float3x3( |
  |  | normalize(cross(IN.normalWS, IN.viewDirWS)), |
  |  | normalize(IN.normalWS), |
  |  | normalize(IN.viewDirWS) |
  |  | ); |
  |  | float3 normalWS = mul(TBN, normalTS); |
  |  |  |
  |  | // 光照计算 |
  |  | Light mainLight = GetMainLight(IN.shadowCoord); |
  |  | float NdotL = saturate(dot(normalWS, mainLight.direction)); |
  |  | half3 lighting = StylizedShading(normalWS, normalize(IN.viewDirWS), NdotL); |
  |  |  |
  |  | return half4(albedo.rgb * lighting * mainLight.color, 1); |
  |  | } |
  |  | ENDHLSL |
  |  | } |
  |  | } |
  |  | } |
  ```

# **材质配置**

| 参数组合 | 风格化效果 |
| --- | --- |
| `_ParallaxScale=0.05` + `_RimPower=5` | 轻度凹凸+柔和边缘光 |
| `_ParallaxScale=0.1` + `_LayerBias=0.4` | 强烈凹凸+抗失真处理 |
| `_ShadowTint=(0.4,0.2,0.6)` | 紫色调阴影增强风格化表现 |

---

> [【从UnityURP开始探索游戏渲染】](https://github.com):[ssrdog](https://mianhuabang.com)**专栏-直达**
> （欢迎*点赞留言*探讨，更多人加入进来能更加完善这个探索的过程，🙏）
