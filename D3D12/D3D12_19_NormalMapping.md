# 第 19 章 法线贴图（Normal Mapping）

---

第 9 章的纹理映射可以把图像中的精细细节映射到三角形上，但**法线向量**仍然只在顶点级别定义，整个三角形内只能靠插值。本章介绍如何用更高分辨率的方式指定表面法线——这会大幅提升光照的细节，但底层几何复杂度保持不变。

---

## 19.1 动机

回顾第 18 章 Cube Mapping demo 的圆锥柱（柱身贴砖纹理）。砖纹理本身凹凸不平，但**镜面高光看起来异常平滑**——这是因为：

- 光照计算依据的是**几何法线**（顶点法线插值得到的）；
- 纹理本身只影响 albedo，并不影响法线。

理想方案是把网格细分到能直接建模出砖块凹凸——但即使有硬件细分，也需要为新顶点提供高频法线。

把光照"烤"到纹理里？光源固定时可行，但光源一动就失效。

**法线贴图**的思路：把高频法线信息**存进一张纹理**，逐像素采样使用——这样动态光照就能反映出砖纹凹凸的精细细节。

---

## 19.2 法线贴图

法线贴图是一张纹理，但**每个纹素存的不是颜色而是法线向量**——红/绿/蓝分量分别存储 x/y/z 坐标。

**压缩原理**：单位向量的每个分量都在 `[-1, 1]`。把它平移缩放到 `[0, 1]` 再 ×255 截断为整数 `[0, 255]`：

```
f(x) = (0.5 * x + 0.5) * 255   →   [-1, 1] 映射到 [0, 255]
```

反向解压：

```
f⁻¹(x) = 2 * (x / 255) - 1     →   [0, 255] 映射回 [-1, 1]
```

着色器采样后，纹理硬件已自动把 `[0, 255]` 转成 `[0, 1]`，只需再做一步：

```hlsl
float3 normalT = gNormalMap.Sample(gsamLinear, pin.Tex).rgb;
// 解压：[0, 1] → [-1, 1]
normalT = 2.0f * normalT - 1.0f;
```

> **为什么法线贴图看起来偏蓝？** 法线大多基本指向 +z，z 分量值最大，而 z 存在蓝通道。

### 法线贴图工具

| 工具 | 说明 |
|------|------|
| NVIDIA Photoshop 插件 | 从图像生成法线贴图 |
| **CrazyBump** | 商业工具，效果好 |
| **ShaderMap** | 另一款法线贴图工具 |
| NVIDIA Melody | 从高模生成法线贴图 |

**压缩格式**：保存法线贴图推荐 **BC7（`DXGI_FORMAT_BC7_UNORM`）**，质量最佳，对法线贴图的失真最小。

---

## 19.3 纹理/切线空间

考虑一个 3D 三角形贴上纹理。假设映射没有扭曲——纹理像贴纸一样被搬上三角形：

```
        v2
       /|
      / |
     /  |
    /   |
   v0───v1
```

纹理空间的 **u 轴**和 **v 轴**就**贴在三角形所在平面**上：

- **T**（Tangent）：3D 空间中，纹理 u 轴方向；
- **B**（Bitangent/Binormal）：3D 空间中，纹理 v 轴方向；
- **N**（Normal）：三角形面法线，垂直于 TB 平面。

`(T, B, N)` 三个向量构成**切线空间（Tangent Space）**或**纹理空间**。每个三角形通常有不同的切线空间。

法线贴图中的法线**就是相对于切线空间存储的**。但光源在世界空间——所以需要把切线空间的法线变换到世界空间。

### 推导 T 和 B

设三角形顶点 `v0, v1, v2`，对应纹理坐标 `(u0, v0), (u1, v1), (u2, v2)`。边向量：

```
e0 = v1 - v0   = Δu0 · T + Δv0 · B
e1 = v2 - v0   = Δu1 · T + Δv1 · B
```

写成矩阵形式：

```
[e0.x  e0.y  e0.z]     [Δu0  Δv0]   [T.x  T.y  T.z]
[e1.x  e1.y  e1.z]  =  [Δu1  Δv1] × [B.x  B.y  B.z]
```

求解：

```
[T]   1            [ Δv1  -Δv0]   [e0]
[B] = ─── ×        [-Δu1   Δu0] × [e1]
      Δu0·Δv1 - Δu1·Δv0
```

> 一般 T、B 在物体空间中**不是单位长度**；如果纹理有扭曲也不一定正交。

---

## 19.4 顶点切线空间

如果每个三角形单独用一个切线空间，相邻三角形之间会出现可见的接缝。解决方法和顶点法线一样——**对共享顶点的所有三角形的 T 取平均**，得到顶点级 T。

然后通常要做 **Gram-Schmidt 正交化**，让 TBN 正交单位。

实践上不直接存储 B：

```cpp
struct Vertex {
    XMFLOAT3 Pos;
    XMFLOAT3 Normal;
    XMFLOAT2 Tex;
    XMFLOAT3 TangentU;  // 只存 T（u 方向），B 在着色器中算
};
```

着色器中 `B = N × T`。

> 任意三角形 mesh 的顶点切线可以通过 [Lengyel 算法](http://www.terathon.com/code/tangent.html) 计算，也可以由建模工具导出。
>
> 本书的 `GeometryGenerator` 程序生成的盒/网格/球/圆柱都直接知道 u 方向，可以解析地得到 T。

---

## 19.5 切线空间与物体空间的变换

把 TBN 作为行向量构成变换矩阵：

```
            [T.x  T.y  T.z]
M_T→O   =   [B.x  B.y  B.z]
            [N.x  N.y  N.z]
```

这是**切线空间 → 物体空间**的矩阵。由于正交，逆 = 转置：`M_O→T = M_T→Oᵀ`。

我们最终需要的是切线 → 世界。结合性允许我们直接构造：

```
M_T→W = M_T→O × M_O→W = TBN(in world space)
```

也就是说：**把 T、B、N 用世界坐标表示，三个向量作行向量构成的矩阵就是切线→世界**。

由于法线是向量（不是点），不需要平移分量，**3×3 矩阵就够了**。

---

## 19.6 法线贴图着色器代码

**整体流程**：

1. 工具生成法线贴图，运行时加载为 2D 纹理；
2. 每三角形/每顶点计算 T，运行时 `B = N × T`；
3. 顶点着色器把法线和切线变换到世界空间；
4. 像素着色器：用插值后的 N、T 重建 TBN 基底，把法线贴图采样到的法线变换到世界空间。

### 辅助函数（Common.hlsl）

```hlsl
// 把法线贴图采样到的法线变换到世界空间
float3 NormalSampleToWorldSpace(float3 normalMapSample,
                                 float3 unitNormalW,
                                 float3 tangentW)
{
    // 1. 解压：[0,1] → [-1,1]
    float3 normalT = 2.0f * normalMapSample - 1.0f;

    // 2. 构建正交单位 TBN（在世界空间）
    float3 N = unitNormalW;
    // 把 T 中沿 N 方向的分量减去（Gram-Schmidt 正交化）
    float3 T = normalize(tangentW - dot(tangentW, N) * N);
    float3 B = cross(N, T);

    float3x3 TBN = float3x3(T, B, N);

    // 3. 切线空间法线 → 世界空间
    float3 bumpedNormalW = mul(normalT, TBN);
    return bumpedNormalW;
}
```

**`tangentW - dot(tangentW, N) * N`**：把 T 中投影到 N 上的分量减去，保证 T 与 N 正交（见 Gram-Schmidt 正交化）。

### 完整着色器

```hlsl
struct VertexIn {
    float3 PosL    : POSITION;
    float3 NormalL : NORMAL;
    float2 TexC    : TEXCOORD;
    float3 TangentU: TANGENT;
};

struct VertexOut {
    float4 PosH     : SV_POSITION;
    float3 PosW     : POSITION;
    float3 NormalW  : NORMAL;
    float3 TangentW : TANGENT;
    float2 TexC     : TEXCOORD;
};

VertexOut VS(VertexIn vin)
{
    VertexOut vout = (VertexOut)0.0f;
    MaterialData matData = gMaterialData[gMaterialIndex];

    float4 posW = mul(float4(vin.PosL, 1.0f), gWorld);
    vout.PosW = posW.xyz;

    // 假设没有非均匀缩放
    vout.NormalW  = mul(vin.NormalL, (float3x3)gWorld);
    vout.TangentW = mul(vin.TangentU, (float3x3)gWorld);

    vout.PosH = mul(posW, gViewProj);

    float4 texC = mul(float4(vin.TexC, 0.0f, 1.0f), gTexTransform);
    vout.TexC = mul(texC, matData.MatTransform).xy;
    return vout;
}

float4 PS(VertexOut pin) : SV_Target
{
    MaterialData matData = gMaterialData[gMaterialIndex];
    float4 diffuseAlbedo = matData.DiffuseAlbedo;
    float3 fresnelR0     = matData.FresnelR0;
    float  roughness     = matData.Roughness;
    uint   diffuseMapIndex = matData.DiffuseMapIndex;
    uint   normalMapIndex  = matData.NormalMapIndex;

    // 插值后法线可能不再归一化
    pin.NormalW = normalize(pin.NormalW);

    // 采样法线贴图，变换到世界空间
    float4 normalMapSample = gTextureMaps[normalMapIndex].Sample(
        gsamAnisotropicWrap, pin.TexC);
    float3 bumpedNormalW = NormalSampleToWorldSpace(
        normalMapSample.rgb, pin.NormalW, pin.TangentW);

    // 注释掉以关闭法线贴图
    // bumpedNormalW = pin.NormalW;

    diffuseAlbedo *= gTextureMaps[diffuseMapIndex].Sample(
        gsamAnisotropicWrap, pin.TexC);

    float3 toEyeW = normalize(gEyePosW - pin.PosW);
    float4 ambient = gAmbientLight * diffuseAlbedo;

    // Alpha 通道存储 per-pixel 的 shininess mask
    const float shininess = (1.0f - roughness) * normalMapSample.a;

    Material mat = { diffuseAlbedo, fresnelR0, shininess };
    float3 shadowFactor = 1.0f;
    float4 directLight = ComputeLighting(gLights, mat,
        pin.PosW, bumpedNormalW, toEyeW, shadowFactor);

    float4 litColor = ambient + directLight;

    // 环境反射（用 bumpedNormalW 而非平面法线）
    float3 r = reflect(-toEyeW, bumpedNormalW);
    float4 reflectionColor = gCubeMap.Sample(gsamLinearWrap, r);
    float3 fresnelFactor   = SchlickFresnel(fresnelR0, bumpedNormalW, r);
    litColor.rgb += shininess * fresnelFactor * reflectionColor.rgb;

    litColor.a = diffuseAlbedo.a;
    return litColor;
}
```

注意两点：

1. **bumpedNormalW** 不仅用于光照，也用于反射方向计算——这样反射也会因为法线贴图的凹凸产生变化。
2. 法线贴图的 **alpha 通道**作为 per-pixel 的 shininess mask（白=高光，黑=暗哑）——为材质引入纹理级的镜面变化。

---

## 本章小结

| 主题 | 核心要点 |
|------|---------|
| **动机** | 顶点法线分辨率不够 → 用纹理存储高频法线 |
| **存储** | RGB 编码 XYZ：`f(x) = (x+1)/2 * 255`，反向 `f⁻¹` 解压 |
| **解压** | `normalT = 2.0 * normalMapSample - 1.0` |
| **格式** | 推荐 BC7（低失真）；BC6/BC7 用 BC6HBC7EncoderDecoder11 工具转换 |
| **切线空间** | T（u 方向）、B（v 方向）、N（法线）；逐三角形不同 |
| **顶点级 T** | 对共享三角形的 T 求平均，再 Gram-Schmidt 正交化 |
| **B 不存储** | 着色器中 `B = N × T` |
| **变换链** | 切线 → 物体 → 世界，可合并为单一矩阵 |
| **采样应用** | 法线 + 反射方向都用 bumpedNormal；alpha 可做 shininess mask |

---

## 课后练习思路

1. 试用 NVIDIA Photoshop 插件、CrazyBump 等工具制作不同的法线贴图。
2. 解释为什么纹理坐标旋转时切线空间也要旋转；并说明把光照搬到切线空间做能简化这个问题。
3. 把光照计算从世界空间搬到切线空间（**避免每像素 TBN 矩阵乘法**），比较性能。
4. **位移贴图（Displacement Mapping）**：用一张高度图配合硬件细分实现海浪——两张高度图以不同速度/方向滚动叠加。

---

> **下一步**：第 20 章将介绍**阴影贴图**——从光源视角渲染深度图，实现任意几何的实时阴影。
