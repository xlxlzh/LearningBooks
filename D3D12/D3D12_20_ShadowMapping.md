# 第 20 章 阴影贴图（Shadow Mapping）

---

阴影既能告诉观察者光源位置，也能传达场景中物体的相对位置。本章介绍**基础的阴影贴图算法**——游戏和 3D 应用中常用的动态阴影方法。更高级的技术（如级联阴影贴图 CSM）都建立在这个基础上。

---

## 20.1 渲染场景深度

阴影贴图算法依赖**从光源视角渲染场景深度**——本质上是 render-to-texture 的一种应用。"渲染场景深度"指的是从光源视角构建深度缓冲，结果就是**阴影贴图（Shadow Map）**。

### ShadowMap 工具类

```cpp
class ShadowMap {
public:
    ShadowMap(ID3D12Device* device, UINT width, UINT height);

    UINT Width() const;
    UINT Height() const;
    ID3D12Resource* Resource();
    CD3DX12_GPU_DESCRIPTOR_HANDLE Srv() const;
    CD3DX12_CPU_DESCRIPTOR_HANDLE Dsv() const;

    D3D12_VIEWPORT Viewport() const;
    D3D12_RECT     ScissorRect() const;

    void BuildDescriptors(
        CD3DX12_CPU_DESCRIPTOR_HANDLE hCpuSrv,
        CD3DX12_GPU_DESCRIPTOR_HANDLE hGpuSrv,
        CD3DX12_CPU_DESCRIPTOR_HANDLE hCpuDsv);

    void OnResize(UINT newWidth, UINT newHeight);

private:
    void BuildDescriptors();
    void BuildResource();

    ID3D12Device* md3dDevice = nullptr;
    D3D12_VIEWPORT mViewport;
    D3D12_RECT mScissorRect;

    UINT mWidth = 0, mHeight = 0;
    DXGI_FORMAT mFormat = DXGI_FORMAT_R24G8_TYPELESS;

    CD3DX12_CPU_DESCRIPTOR_HANDLE mhCpuSrv;
    CD3DX12_GPU_DESCRIPTOR_HANDLE mhGpuSrv;
    CD3DX12_CPU_DESCRIPTOR_HANDLE mhCpuDsv;

    Microsoft::WRL::ComPtr<ID3D12Resource> mShadowMap;
};
```

**关键点**：

- 阴影贴图分辨率越大、质量越好，但绘制和内存代价也越高；
- 使用 `R24G8_TYPELESS` 格式，可同时绑定为 DSV（深度写入）和 SRV（着色器采样）。

### 构建资源

```cpp
void ShadowMap::BuildResource()
{
    D3D12_RESOURCE_DESC texDesc = {};
    texDesc.Dimension = D3D12_RESOURCE_DIMENSION_TEXTURE2D;
    texDesc.Width = mWidth;
    texDesc.Height = mHeight;
    texDesc.DepthOrArraySize = 1;
    texDesc.MipLevels = 1;
    texDesc.Format = mFormat;
    texDesc.SampleDesc.Count = 1;
    texDesc.Layout = D3D12_TEXTURE_LAYOUT_UNKNOWN;
    texDesc.Flags = D3D12_RESOURCE_FLAG_ALLOW_DEPTH_STENCIL;

    D3D12_CLEAR_VALUE optClear;
    optClear.Format = DXGI_FORMAT_D24_UNORM_S8_UINT;
    optClear.DepthStencil.Depth = 1.0f;
    optClear.DepthStencil.Stencil = 0;

    md3dDevice->CreateCommittedResource(
        &CD3DX12_HEAP_PROPERTIES(D3D12_HEAP_TYPE_DEFAULT),
        D3D12_HEAP_FLAG_NONE, &texDesc,
        D3D12_RESOURCE_STATE_GENERIC_READ, &optClear,
        IID_PPV_ARGS(&mShadowMap));
}
```

阴影贴图算法需要**两个渲染 pass**：

1. **Pass 1**：从光源视角渲染场景深度到阴影贴图；
2. **Pass 2**：从玩家相机渲染场景，把阴影贴图作为 SRV 用来做阴影测试。

---

## 20.2 正交投影

之前我们都用透视投影（远小近大）。**正交投影（Orthographic Projection）**则使平行线在投影后仍保持平行——主要用于工程/科学应用，但也是**模拟平行光阴影**的关键。

正交投影的视体是一个**轴对齐的盒子**，由宽 `w`、高 `h`、近平面 `n`、远平面 `f` 定义：

```
            ┌─────┐
           /│     │
          / │     │
         /  │     │  ← 正交视体是一个盒子
        /   │     │
       /    └─────┘
      /      / 
     V (相机)
```

投影时**所有投影线都平行于视图空间 z 轴**，2D 投影坐标就是 `(x, y)`，z 用于深度。

### 推导

把视体从视图空间映射到 NDC 空间 `[-1, 1] × [-1, 1] × [0, 1]`：

- X: 区间 `[-w/2, w/2]` → `[-1, 1]`，缩放因子 `2/w`；
- Y: 区间 `[-h/2, h/2]` → `[-1, 1]`，缩放因子 `2/h`；
- Z: 区间 `[n, f]` → `[0, 1]`，线性映射 `(z - n)/(f - n)`。

矩阵形式：

```
              | 2/w     0           0          0 |
P_ortho   =   | 0       2/h         0          0 |
              | 0       0       1/(f-n)        0 |
              | 0       0      -n/(f-n)        1 |
```

> 与透视投影最大区别：**没有透视除法**——正交投影矩阵相乘就直接得到 NDC 坐标。

DirectXMath 提供 `XMMatrixOrthographicLH` 和 `XMMatrixOrthographicOffCenterLH`。

---

## 20.3 投影纹理坐标

**投影纹理（Projective Texturing）**像幻灯机一样把纹理"投影"到任意几何体上。它本身可用于实现幻灯片效果，但**也是阴影贴图算法的关键步骤**。

**核心思路**：为每个像素生成"看起来像被投影"的纹理坐标。

```
1. 把世界空间点 p 投影到光的投影窗口，并变换到 NDC 空间；
2. 把 NDC 坐标变换到纹理空间 [0, 1] × [0, 1]。
```

第二步的矩阵 **T**（NDC → 纹理空间）：

```
      | 0.5    0      0     0 |
T  =  | 0      -0.5   0     0 |    ← y 取负是因为 NDC +y 朝上，UV +v 朝下
      | 0      0      1     0 |
      | 0.5    0.5    0     1 |
```

可以合并为 `M = V × P × T`：把世界空间**直接**变换到纹理空间。透视除法可以在变换后做（结合律）。

### 实现代码

```hlsl
struct VertexOut {
    float4 PosH    : SV_POSITION;
    float3 PosW    : POSITION;
    float3 NormalW : NORMAL;
    float2 Tex     : TEXCOORD0;
    float4 ProjTex : TEXCOORD1;  // 投影纹理坐标
};

VertexOut VS(VertexIn vin)
{
    VertexOut vout;
    // ... 标准变换 ...

    // 变换到光的投影空间
    vout.ProjTex = mul(float4(vin.PosL, 1.0f), gLightWorldViewProjTexture);
    return vout;
}

float4 PS(VertexOut pin) : SV_Target
{
    // 完成透视除法
    pin.ProjTex.xyz /= pin.ProjTex.w;

    // NDC 空间深度
    float depth = pin.ProjTex.z;

    // 用投影 UV 采样纹理
    float4 c = gTextureMap.Sample(sampler, pin.ProjTex.xy);
    // ...
}
```

### 视锥外的点

视锥外的几何在投影时仍能得到 UV，但范围超出 `[0, 1]`——按采样器的寻址模式处理：

- **Border 模式 + 黑边**：常见做法；
- **聚光锥**：用聚光的方向锥控制可投影区域。

### 正交投影特例

正交投影下 `w = 1`，**不用透视除法**（可省略 `/= w`，性能更好）。如果保留则代码可以同时支持两种投影。

---

## 20.4 阴影贴图

### 20.4.1 算法描述

1. **第一遍**：从光源视角渲染场景深度到阴影贴图。完成后，阴影贴图中存的就是从光源看得到的、最近像素的深度。
2. **第二遍**：从玩家相机渲染场景。对每个像素 `p`：
   - 计算 `d(p)`：`p` 距光源的深度；
   - 用投影纹理采样阴影贴图得到 `s(p)`：从光源沿同一视线方向看到的最近像素深度；
   - **若 `d(p) > s(p)`**：`p` 被遮挡 → 在阴影中。

两个深度都在光的 NDC 空间中比较，因为阴影贴图就存的 NDC 深度。

**光的投影类型**：

- 透视投影 → 用于聚光（视锥内的光锥）；
- 正交投影 → 用于方向光（盒子内的平行光）。

### 20.4.2 偏移与混淆

阴影贴图分辨率有限，每个纹素对应场景中一**片区域**——这导致**阴影粉刺（Shadow Acne）**——表面像被涂了"楼梯"状的明暗条纹。

**根因**：眼睛看到的两个像素 `p1, p2` 在阴影贴图中可能对应同一个纹素 `s`，但它们的真实深度 `d(p1) ≠ d(p2)`，于是一个被错误判定在阴影中。

**解决方法**：给阴影贴图的深度加一个**偏移（Bias）**。

但是过大的偏移会导致 **Peter-Panning**——阴影从物体上"脱落"。

更巧妙的方法是用**斜率缩放偏移（Slope-Scaled Bias）**：相对于光源越倾斜的多边形需要越大的偏移。Direct3D 硬件原生支持：

```cpp
typedef struct D3D12_RASTERIZER_DESC {
    // ...
    INT   DepthBias;            // 固定偏移
    FLOAT DepthBiasClamp;       // 偏移上限
    FLOAT SlopeScaledDepthBias; // 斜率比例
    // ...
} D3D12_RASTERIZER_DESC;
```

公式（UNORM 深度格式）：

```
Bias = (float)DepthBias * r + SlopeScaledDepthBias * MaxDepthSlope
```

其中 `r = 1 / 2^24`（24 位深度缓冲）。

```cpp
// Demo 使用的值（场景相关，需要调试）
smapPsoDesc.RasterizerState.DepthBias            = 100000;  // 实际约 0.006
smapPsoDesc.RasterizerState.DepthBiasClamp       = 0.0f;
smapPsoDesc.RasterizerState.SlopeScaledDepthBias = 1.0f;
```

> 偏移在光栅化阶段（裁剪后）施加，不影响几何裁剪。

### 20.4.3 PCF 过滤

投影纹理坐标 `(u, v)` 通常落在 4 个纹素之间。对颜色用双线性插值；但**对深度不能直接插值**（会导致错误的阴影判定）。

**PCF（Percentage Closer Filtering）**：对每个邻近纹素**先做深度测试**，再对测试结果（0 或 1）插值：

```hlsl
static const float SMAP_SIZE = 2048.0f;
static const float SMAP_DX = 1.0f / SMAP_SIZE;

// 4 个采样
float s0 = gShadowMap.Sample(gShadowSam, projTexC.xy).r;
float s1 = gShadowMap.Sample(gShadowSam, projTexC.xy + float2(SMAP_DX, 0)).r;
float s2 = gShadowMap.Sample(gShadowSam, projTexC.xy + float2(0, SMAP_DX)).r;
float s3 = gShadowMap.Sample(gShadowSam, projTexC.xy + float2(SMAP_DX, SMAP_DX)).r;

// 各自深度测试
float result0 = depth <= s0;
float result1 = depth <= s1;
float result2 = depth <= s2;
float result3 = depth <= s3;

// 双线性插值
float2 texelPos = SMAP_SIZE * projTexC.xy;
float2 t = frac(texelPos);
return lerp(lerp(result0, result1, t.x),
            lerp(result2, result3, t.x), t.y);
```

#### 硬件加速：SampleCmpLevelZero

Direct3D 11+ 硬件原生支持 PCF——一次调用就完成 4-tap PCF：

```hlsl
Texture2D gShadowMap : register(t1);
SamplerComparisonState gsamShadow : register(s6);

shadowPosH.xyz /= shadowPosH.w;
float depth = shadowPosH.z;

// 4-tap PCF
float result = gShadowMap.SampleCmpLevelZero(gsamShadow, shadowPosH.xy, depth).r;
```

需要**比较采样器**（`SamplerComparisonState`），过滤器为 `D3D12_FILTER_COMPARISON_MIN_MAG_LINEAR_MIP_POINT`：

```cpp
const CD3DX12_STATIC_SAMPLER_DESC shadow(
    6,  // 寄存器
    D3D12_FILTER_COMPARISON_MIN_MAG_LINEAR_MIP_POINT,
    D3D12_TEXTURE_ADDRESS_MODE_BORDER,
    D3D12_TEXTURE_ADDRESS_MODE_BORDER,
    D3D12_TEXTURE_ADDRESS_MODE_BORDER,
    0.0f, 16,
    D3D12_COMPARISON_FUNC_LESS_EQUAL,
    D3D12_STATIC_BORDER_COLOR_OPAQUE_BLACK);
```

> 支持比较过滤的格式：`R32_FLOAT_X8X24_TYPELESS`、`R32_FLOAT`、`R24_UNORM_X8_TYPELESS`、`R16_UNORM`。

### 20.4.4 构建阴影贴图

```cpp
mShadowMap = std::make_unique<ShadowMap>(md3dDevice.Get(), 2048, 2048);
```

光的视图矩阵和投影矩阵从主光源派生，**投影矩阵覆盖整个场景的包围球**：

```cpp
void ShadowMapApp::UpdateShadowTransform(const GameTimer& gt)
{
    // 主光源（第一盏灯）才投阴影
    XMVECTOR lightDir = XMLoadFloat3(&mRotatedLightDirections[0]);
    XMVECTOR lightPos = -2.0f * mSceneBounds.Radius * lightDir;
    XMVECTOR targetPos = XMLoadFloat3(&mSceneBounds.Center);
    XMVECTOR lightUp   = XMVectorSet(0.0f, 1.0f, 0.0f, 0.0f);

    XMMATRIX lightView = XMMatrixLookAtLH(lightPos, targetPos, lightUp);

    // 把场景包围球变换到光空间
    XMFLOAT3 sphereCenterLS;
    XMStoreFloat3(&sphereCenterLS, XMVector3TransformCoord(targetPos, lightView));

    // 光空间的正交视体能包住整个场景
    float l = sphereCenterLS.x - mSceneBounds.Radius;
    float b = sphereCenterLS.y - mSceneBounds.Radius;
    float n = sphereCenterLS.z - mSceneBounds.Radius;
    float r = sphereCenterLS.x + mSceneBounds.Radius;
    float t = sphereCenterLS.y + mSceneBounds.Radius;
    float f = sphereCenterLS.z + mSceneBounds.Radius;

    XMMATRIX lightProj = XMMatrixOrthographicOffCenterLH(l, r, b, t, n, f);

    // NDC [-1,+1]^2 → 纹理空间 [0,1]^2
    XMMATRIX T(
        0.5f,  0.0f, 0.0f, 0.0f,
        0.0f, -0.5f, 0.0f, 0.0f,
        0.0f,  0.0f, 1.0f, 0.0f,
        0.5f,  0.5f, 0.0f, 1.0f);

    XMMATRIX S = lightView * lightProj * T;
    XMStoreFloat4x4(&mShadowTransform, S);
}
```

### 渲染场景到阴影贴图

```cpp
void ShadowMapApp::DrawSceneToShadowMap()
{
    mCommandList->RSSetViewports(1, &mShadowMap->Viewport());
    mCommandList->RSSetScissorRects(1, &mShadowMap->ScissorRect());

    // 转 DEPTH_WRITE
    mCommandList->ResourceBarrier(1, &CD3DX12_RESOURCE_BARRIER::Transition(
        mShadowMap->Resource(),
        D3D12_RESOURCE_STATE_GENERIC_READ,
        D3D12_RESOURCE_STATE_DEPTH_WRITE));

    mCommandList->ClearDepthStencilView(mShadowMap->Dsv(),
        D3D12_CLEAR_FLAG_DEPTH | D3D12_CLEAR_FLAG_STENCIL,
        1.0f, 0, 0, nullptr);

    // 关键：不绑定 RTV！我们只关心深度
    mCommandList->OMSetRenderTargets(0, nullptr, false, &mShadowMap->Dsv());

    // 用光源 pass CB
    auto passCB = mCurrFrameResource->PassCB->Resource();
    D3D12_GPU_VIRTUAL_ADDRESS passCBAddress =
        passCB->GetGPUVirtualAddress() + 1 * passCBByteSize;
    mCommandList->SetGraphicsRootConstantBufferView(1, passCBAddress);

    mCommandList->SetPipelineState(mPSOs["shadow_opaque"].Get());
    DrawRenderItems(mCommandList.Get(), mRitemLayer[(int)RenderLayer::Opaque]);

    // 转回 GENERIC_READ
    mCommandList->ResourceBarrier(1, &CD3DX12_RESOURCE_BARRIER::Transition(
        mShadowMap->Resource(),
        D3D12_RESOURCE_STATE_DEPTH_WRITE,
        D3D12_RESOURCE_STATE_GENERIC_READ));
}
```

**关键优化**：**不绑定渲染目标**——只输出深度。仅深度的 pass 显著比同时输出颜色快。

对应的 PSO：

```cpp
D3D12_GRAPHICS_PIPELINE_STATE_DESC smapPsoDesc = opaquePsoDesc;
smapPsoDesc.RasterizerState.DepthBias = 100000;
smapPsoDesc.RasterizerState.DepthBiasClamp = 0.0f;
smapPsoDesc.RasterizerState.SlopeScaledDepthBias = 1.0f;
smapPsoDesc.RTVFormats[0] = DXGI_FORMAT_UNKNOWN;  // 无 RTV
smapPsoDesc.NumRenderTargets = 0;
```

阴影 PSO 的着色器很简单——VS 只做空间变换；PS 仅用于 alpha 测试（如树叶），其他情况可以**绑定 null PS**：

```hlsl
VertexOut VS(VertexIn vin) {
    // ... 标准变换 ...
    vout.PosH = mul(posW, gViewProj);
    vout.TexC = ...;
    return vout;
}

// 仅用于 alpha 测试几何
void PS(VertexOut pin) {
    MaterialData matData = gMaterialData[gMaterialIndex];
    float4 diffuseAlbedo = matData.DiffuseAlbedo *
        gTextureMaps[matData.DiffuseMapIndex].Sample(gsamAnisotropicWrap, pin.TexC);

#ifdef ALPHA_TEST
    clip(diffuseAlbedo.a - 0.1f);
#endif
    // 注意 PS 没有返回值——只输出深度
}
```

### 20.4.5 阴影因子

阴影因子是 `[0, 1]` 的标量：

- `0`：完全在阴影中；
- `1`：完全在光照中；
- 中间值（PCF）：部分在阴影中。

```hlsl
float CalcShadowFactor(float4 shadowPosH)
{
    shadowPosH.xyz /= shadowPosH.w;
    float depth = shadowPosH.z;

    uint width, height, numMips;
    gShadowMap.GetDimensions(0, width, height, numMips);

    float dx = 1.0f / (float)width;
    float percentLit = 0.0f;

    // 3×3 box filter pattern（每次 SampleCmp 又是 4-tap）
    const float2 offsets[9] = {
        float2(-dx, -dx), float2(0.0f, -dx), float2(dx, -dx),
        float2(-dx, 0.0f), float2(0.0f, 0.0f), float2(dx, 0.0f),
        float2(-dx, +dx), float2(0.0f, +dx), float2(dx, +dx),
    };

    [unroll]
    for (int i = 0; i < 9; ++i) {
        percentLit += gShadowMap.SampleCmpLevelZero(gsamShadow,
            shadowPosH.xy + offsets[i], depth).r;
    }

    return percentLit / 9.0f;
}
```

应用到光照：

```hlsl
float3 shadowFactor = float3(1.0f, 1.0f, 1.0f);
shadowFactor[0] = CalcShadowFactor(pin.ShadowPosH);  // 只对主光做阴影

float4 directLight = ComputeLighting(gLights, mat, pin.PosW,
    bumpedNormalW, toEyeW, shadowFactor);
```

`shadowFactor` 仅影响直接光（漫反射 + 镜面反射）。**环境光不受影响**（间接光不被阴影遮挡），**反射光也不受影响**（来自环境贴图）。

### 20.4.6 阴影测试

主 pass 中需要 `d(p)` 和 `s(p)`：

```hlsl
// 顶点着色器：生成投影 UV
vout.ShadowPosH = mul(posW, gShadowTransform);

// 像素着色器：测试
float3 shadowFactor = float3(1.0f, 1.0f, 1.0f);
shadowFactor[0] = CalcShadowFactor(pin.ShadowPosH);
```

`gShadowTransform` 即前面构造的 `S = lightView * lightProj * T`，存在 pass CB 中。

### 20.4.7 渲染阴影贴图本身

Demo 在屏幕右下角绘制阴影贴图（深度图，灰度），便于调试。

---

## 20.5 大 PCF 核

大 PCF 核（如 5×5 或 9×9）产生更柔和的阴影边缘，但**会重新引入阴影粉刺**——因为相邻纹素描述的可能不是同一多边形。

**根因**：阴影测试时，原本只与 `s0`（覆盖 `p` 区域的纹素）比较是合法的；但 PCF 把相邻的 `s_{-1}, s_1` 也拿来比较——这些纹素可能描述完全不同的几何。

**解决方法**（Tuft10）：假设邻近像素与 `p` 在同一多边形上。利用 `ddx/ddy` 估计深度沿屏幕方向的变化率，按偏移补偿深度。

### 20.5.1 ddx 和 ddy

```hlsl
float ddx(float p);  // 估计 ∂p/∂x（屏幕 x 方向）
float ddy(float p);  // 估计 ∂p/∂y（屏幕 y 方向）
```

硬件以 2×2 quad 并行处理像素，通过有限差分估计导数。可用于：

- 估计颜色变化；
- 估计深度变化；
- 估计法线变化。

### 20.5.2 大核解决方案

设光空间坐标 `p = (u, v, z)`。可计算：

```
∂(u,v)/∂x  =  (du/dx, dv/dx)
∂(u,v)/∂y  =  (du/dy, dv/dy)
∂z/∂x, ∂z/∂y
```

由这些导数构造逆矩阵，让"光空间偏移 (Δu, Δv) → 屏幕空间偏移 (Δx, Δy) → 深度偏移 Δz"：

1. 已知 PCF 偏移 `(Δu, Δv)`（通过 `dx`）；
2. 用矩阵反解 `(Δx, Δy)`；
3. 用 `Δz = ∂z/∂x · Δx + ∂z/∂y · Δy` 求深度补偿。

完整实现见 DirectX 11 SDK 的 **CascadedShadowMaps11** demo（`CalculateRightAndUpTexelDepthDeltas` 与 `CalculatePCFPercentLit` 函数）。

### 20.5.3 另一种解法（Isidoro06）

把 `z = z(u, v)` 看作 `(u, v)` 的函数，用链式法则直接求 `∂z/∂u` 和 `∂z/∂v`：

```
[∂z/∂x]     [∂u/∂x  ∂v/∂x]   [∂z/∂u]
[∂z/∂y]  =  [∂u/∂y  ∂v/∂y] × [∂z/∂v]
```

求逆即可得到 `∂z/∂u, ∂z/∂v`，不需要变换到屏幕空间。

---

## 本章小结

| 主题 | 核心要点 |
|------|---------|
| **基本算法** | 两 pass：先从光视角渲染深度，再从主相机渲染并比较 |
| **正交投影** | 用于方向光；无透视除法 |
| **投影纹理** | NDC → 纹理空间矩阵 T；视锥外用 Border 寻址模式 |
| **阴影粉刺** | 由阴影贴图离散采样导致；用斜率缩放偏移修复 |
| **Peter-Panning** | 偏移过大时阴影脱离物体 |
| **PCF** | 对深度测试结果做插值（而非对深度插值）|
| **硬件加速** | `SampleCmpLevelZero` + `SamplerComparisonState` |
| **5×5+ 大核** | 引入新的粉刺问题，需 `ddx/ddy` 求自适应深度补偿 |
| **PSO 优化** | 阴影 pass 不绑定 RTV，仅深度，速度更快 |
| **着色器** | VS 标准变换；PS 仅 alpha 测试或 null |

---

## 课后练习思路

1. 实现幻灯片投影（投影纹理），分别测试透视/正交。
2. 视锥外的点用 Border 模式不接收投影。
3. 用聚光锥约束投影范围。
4. 用透视投影替换正交（适用于聚光），注意调整 slope-scaled-bias。
5. 测试不同阴影贴图分辨率（4096²、1024²、512²、256²）的画质/性能。
6. 推导"off-center"正交投影矩阵（视体不以原点为中心）。
7. 关闭斜率偏移观察阴影粉刺；过大偏移观察 Peter-Panning。
8. **点光源阴影**：用立方体贴图存深度（6 个面 90° 视野透视投影），采样时用 `TextureCube` 比较。

---

> **下一步**：第 21 章讨论**环境光遮蔽（AO）**——改进环境光项，让凹陷处更暗。
