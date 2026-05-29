# 第 18 章 立方体贴图（Cube Mapping）

---

立方体贴图本质上是**一组以特殊方式解释的 6 张纹理**——可以高效地实现**天空盒**和**反射**效果。

---

## 18.1 立方体贴图

立方体贴图存储 6 张纹理，并把它们视作一个**轴对齐立方体的 6 个面**，因此得名"立方体贴图"。每个面对应坐标轴方向（±X、±Y、±Z）。

在 Direct3D 中，立方体贴图表示为一个有 6 个元素的纹理数组：

| 索引 | 对应面 |
|------|--------|
| 0 | +X |
| 1 | -X |
| 2 | +Y |
| 3 | -Y |
| 4 | +Z |
| 5 | -Z |

**采样立方体贴图**：不再使用 2D `(u, v)` 坐标，而是用**3D 查找向量 v**——从原点出发，向量与立方体某个面的交点决定了采样纹素。向量的**长度无关紧要**，只看方向：相同方向、不同长度的两个向量采样同一个纹素。

```hlsl
TextureCube gCubeMap;
SamplerState gsamLinearWrap : register(s2);

// 像素着色器内
float3 v = float3(x, y, z);  // 某个查找向量
float4 color = gCubeMap.Sample(gsamLinearWrap, v);
```

> 查找向量应与立方体贴图位于同一坐标系。如立方体贴图相对于世界空间（即面与世界轴对齐），则查找向量应是世界空间向量。

---

## 18.2 环境贴图

立方体贴图最主要的应用是**环境贴图（Environment Mapping）**。

**生成思路**：在场景中某点放一台相机，视野角 **90°**（水平和垂直），分别朝 ±X、±Y、±Z 拍 6 张照片。6 张照片合起来正好覆盖该点周围的全部 360° 环境。把这 6 张图存进立方体贴图就是"环境贴图"。

实际工程中常用妥协：

- **少量环境贴图**：在场景的关键位置预制几张环境贴图，物体使用最近的那张；
- **省略局部物体**：环境贴图只包含远景（天空、山脉），近处物体单独处理。

### 制作环境贴图

立方体贴图本质上就是纹理数据——可以由美术预先制作。常用工具：

- **Terragen**（http://www.planetside.co.uk/）：免费个人版，可生成逼真的户外场景。
- **NVIDIA 提供的 Photoshop 插件**：保存 DDS 和立方体贴图。

合成 DDS 立方体贴图：

```bash
texassemble -cube -w 256 -h 256 -o cubemap.dds \
    lobbyxpos.jpg lobbyxneg.jpg \
    lobbyypos.jpg lobbyyneg.jpg \
    lobbyzpos.jpg lobbyzneg.jpg
```

### 18.2.1 在 Direct3D 中加载并使用立方体贴图

DDS 加载代码（`DDSTextureLoader.h/.cpp`）会自动识别立方体贴图，并创建 6 元素纹理数组：

```cpp
auto skyTex = std::make_unique<Texture>();
skyTex->Name = "skyTex";
skyTex->Filename = L"Textures/grasscube1024.dds";
ThrowIfFailed(DirectX::CreateDDSTextureFromFile12(
    md3dDevice.Get(), mCommandList.Get(),
    skyTex->Filename.c_str(),
    skyTex->Resource, skyTex->UploadHeap));
```

创建 SRV 时使用 `D3D12_SRV_DIMENSION_TEXTURECUBE`：

```cpp
D3D12_SHADER_RESOURCE_VIEW_DESC srvDesc = {};
srvDesc.Shader4ComponentMapping = D3D12_DEFAULT_SHADER_4_COMPONENT_MAPPING;
srvDesc.ViewDimension = D3D12_SRV_DIMENSION_TEXTURECUBE;
srvDesc.TextureCube.MostDetailedMip = 0;
srvDesc.TextureCube.MipLevels = skyTex->GetDesc().MipLevels;
srvDesc.TextureCube.ResourceMinLODClamp = 0.0f;
srvDesc.Format = skyTex->GetDesc().Format;

md3dDevice->CreateShaderResourceView(skyTex.Get(), &srvDesc, hDescriptor);
```

---

## 18.3 给天空贴图

用环境贴图绘制天空：**画一个巨大的球体包围整个场景**，用立方体贴图给球体表面贴图——从球心到表面点的方向向量作为查找向量。

**关键技巧**：让球**始终以相机为中心**——这样相机无论怎么移动，天空都"在无穷远处"，没法靠近。

```hlsl
struct VertexIn { float3 PosL : POSITION; /* ... */ };
struct VertexOut {
    float4 PosH : SV_POSITION;
    float3 PosL : POSITION;  // 用作立方体贴图查找向量
};

VertexOut VS(VertexIn vin) {
    VertexOut vout;

    // 用局部位置作查找向量
    vout.PosL = vin.PosL;

    // 把球的中心固定在相机位置
    float4 posW = mul(float4(vin.PosL, 1.0f), gWorld);
    posW.xyz += gEyePosW;

    // 让 z = w → z/w = 1，天空永远在远平面上
    vout.PosH = mul(posW, gViewProj).xyww;
    return vout;
}

float4 PS(VertexOut pin) : SV_Target {
    return gCubeMap.Sample(gsamLinearWrap, pin.PosL);
}
```

### 关键点

1. **`PosH = mul(..., gViewProj).xyww`**：让透视除法后 `z/w = 1`，天空永远在 NDC 远平面上（z=1）。
2. **`posW.xyz += gEyePosW`**：球以相机为中心。

### 渲染顺序与 PSO

历史上有人把天空当做"清屏"用，**第一个绘制**。现在不推荐这样做：

1. 深度/模板缓冲必须显式清除（硬件优化要求）；
2. 大部分天空会被前景遮挡 → 先画浪费 fillrate。

**现代做法：天空最后绘制，并且需要专门的 PSO 设置**：

```cpp
D3D12_GRAPHICS_PIPELINE_STATE_DESC skyPsoDesc = opaquePsoDesc;

// 相机在天空球内部，关闭背面剔除
skyPsoDesc.RasterizerState.CullMode = D3D12_CULL_MODE_NONE;

// 深度比较函数改为 LESS_EQUAL
// 否则 z=1 时无法通过深度测试（深度缓冲也被清成 1）
skyPsoDesc.DepthStencilState.DepthFunc = D3D12_COMPARISON_FUNC_LESS_EQUAL;

ThrowIfFailed(md3dDevice->CreateGraphicsPipelineState(
    &skyPsoDesc, IID_PPV_ARGS(&mPSOs["sky"])));
```

绘制时分层：

```cpp
mCommandList->SetPipelineState(mPSOs["opaque"].Get());
DrawRenderItems(mCommandList.Get(), mRitemLayer[(int)RenderLayer::Opaque]);

mCommandList->SetPipelineState(mPSOs["sky"].Get());
DrawRenderItems(mCommandList.Get(), mRitemLayer[(int)RenderLayer::Sky]);
```

---

## 18.4 模拟反射

回顾第 8 章：镜面高光是由光源经过 Fresnel 反射进入眼睛形成的。实际上光从四面八方都会撞击表面——我们之前用环境光项（ambient）近似间接光。本节用**环境贴图模拟来自周围环境的镜面反射**。

每个环境贴图纹素都可以视为一个**入射光源**。给定眼睛方向 `v = E - p`，**反射方向**：

```
r = reflect(-v, n)
```

用 `r` 采样环境贴图就得到了反射进眼睛的光。

```hlsl
const float shininess = 1.0f - roughness;

// 计算环境反射
float3 r = reflect(-toEyeW, pin.NormalW);
float4 reflectionColor = gCubeMap.Sample(gsamLinearWrap, r);
float3 fresnelFactor = SchlickFresnel(fresnelR0, pin.NormalW, r);

litColor.rgb += shininess * fresnelFactor * reflectionColor.rgb;
```

### 反射的局限性

环境贴图反射**对平面物体不准确**：反射向量 `r` 只携带方向信息，不包含位置——所以从同一表面不同点出发的反射，都采样相同的纹素。

**改进方法**：把环境贴图与一个**代理几何**（如 AABB）绑定。在像素着色器中做射线/盒求交，用交点（相对盒中心）作查找向量。

```hlsl
float3 BoxCubeMapLookup(float3 rayOrigin, float3 unitRayDir,
                        float3 boxCenter,  float3 boxExtents)
{
    // 基于 Real-Time Rendering 3rd 16.7.1 的 Slab 方法

    // 相对盒中心
    float3 p = rayOrigin - boxCenter;

    // AABB 6 个面的 t 值（向量化）
    float3 t1 = (-p + boxExtents) / unitRayDir;
    float3 t2 = (-p - boxExtents) / unitRayDir;

    float3 tmax = max(t1, t2);
    float t = min(min(tmax.x, tmax.y), tmax.z);

    // 返回相对盒中心的交点（用作查找向量）
    return p + t * unitRayDir;
}
```

---

## 18.5 动态立方体贴图

预制的环境贴图无法反映场景中运动的物体。如果场景里有动画角色或浮动物体，反射也希望"动起来"——这就要**运行时构建立方体贴图**：

> 每帧把相机放到反射物体中心，朝 6 个方向各渲染一次场景，结果存入立方体贴图。

代价是高昂：**每帧渲染场景 6 次**！所以：

- 仅对**关键物体**使用动态反射；
- 使用**较低分辨率**的立方体贴图（如 256×256）减少 fillrate 压力。

### 18.5.1 CubeRenderTarget 辅助类

```cpp
class CubeRenderTarget {
public:
    CubeRenderTarget(ID3D12Device* device, UINT width, UINT height, DXGI_FORMAT format);

    ID3D12Resource* Resource();
    CD3DX12_GPU_DESCRIPTOR_HANDLE Srv();
    CD3DX12_CPU_DESCRIPTOR_HANDLE Rtv(int faceIndex);

    D3D12_VIEWPORT Viewport() const;
    D3D12_RECT     ScissorRect() const;

    void BuildDescriptors(
        CD3DX12_CPU_DESCRIPTOR_HANDLE hCpuSrv,
        CD3DX12_GPU_DESCRIPTOR_HANDLE hGpuSrv,
        CD3DX12_CPU_DESCRIPTOR_HANDLE hCpuRtv[6]);

    void OnResize(UINT newWidth, UINT newHeight);

private:
    void BuildDescriptors();
    void BuildResource();

    ID3D12Device* md3dDevice = nullptr;
    D3D12_VIEWPORT mViewport;
    D3D12_RECT mScissorRect;

    UINT mWidth = 0;
    UINT mHeight = 0;
    DXGI_FORMAT mFormat = DXGI_FORMAT_R8G8B8A8_UNORM;

    CD3DX12_CPU_DESCRIPTOR_HANDLE mhCpuSrv;
    CD3DX12_GPU_DESCRIPTOR_HANDLE mhGpuSrv;
    CD3DX12_CPU_DESCRIPTOR_HANDLE mhCpuRtv[6];

    Microsoft::WRL::ComPtr<ID3D12Resource> mCubeMap;
};
```

### 18.5.2 构建立方体贴图资源

立方体贴图 = 6 元素的纹理数组，且要带 `ALLOW_RENDER_TARGET` 标志：

```cpp
void CubeRenderTarget::BuildResource()
{
    D3D12_RESOURCE_DESC texDesc = {};
    texDesc.Dimension = D3D12_RESOURCE_DIMENSION_TEXTURE2D;
    texDesc.Width = mWidth;
    texDesc.Height = mHeight;
    texDesc.DepthOrArraySize = 6;          // 6 个面
    texDesc.MipLevels = 1;
    texDesc.Format = mFormat;
    texDesc.SampleDesc.Count = 1;
    texDesc.Layout = D3D12_TEXTURE_LAYOUT_UNKNOWN;
    texDesc.Flags = D3D12_RESOURCE_FLAG_ALLOW_RENDER_TARGET;

    ThrowIfFailed(md3dDevice->CreateCommittedResource(
        &CD3DX12_HEAP_PROPERTIES(D3D12_HEAP_TYPE_DEFAULT),
        D3D12_HEAP_FLAG_NONE,
        &texDesc,
        D3D12_RESOURCE_STATE_GENERIC_READ,
        nullptr,
        IID_PPV_ARGS(&mCubeMap)));
}
```

### 18.5.3 描述符堆扩展

动态立方体贴图需要 **6 个 RTV** + **1 个 DSV**（深度缓冲）+ **1 个 SRV**：

```cpp
void DynamicCubeMapApp::CreateRtvAndDsvDescriptorHeaps()
{
    // +6 RTV（cube map 渲染目标）
    D3D12_DESCRIPTOR_HEAP_DESC rtvHeapDesc;
    rtvHeapDesc.NumDescriptors = SwapChainBufferCount + 6;
    rtvHeapDesc.Type  = D3D12_DESCRIPTOR_HEAP_TYPE_RTV;
    rtvHeapDesc.Flags = D3D12_DESCRIPTOR_HEAP_FLAG_NONE;
    rtvHeapDesc.NodeMask = 0;
    md3dDevice->CreateDescriptorHeap(&rtvHeapDesc, IID_PPV_ARGS(mRtvHeap.GetAddressOf()));

    // +1 DSV
    D3D12_DESCRIPTOR_HEAP_DESC dsvHeapDesc;
    dsvHeapDesc.NumDescriptors = 2;
    dsvHeapDesc.Type  = D3D12_DESCRIPTOR_HEAP_TYPE_DSV;
    md3dDevice->CreateDescriptorHeap(&dsvHeapDesc, IID_PPV_ARGS(mDsvHeap.GetAddressOf()));

    mCubeDSV = CD3DX12_CPU_DESCRIPTOR_HANDLE(
        mDsvHeap->GetCPUDescriptorHandleForHeapStart(),
        1, mDsvDescriptorSize);
}
```

### 18.5.4 构建描述符

为立方体贴图创建一个 SRV + 6 个 RTV（每面一个）：

```cpp
void CubeRenderTarget::BuildDescriptors()
{
    // SRV：整个立方体贴图
    D3D12_SHADER_RESOURCE_VIEW_DESC srvDesc = {};
    srvDesc.Shader4ComponentMapping = D3D12_DEFAULT_SHADER_4_COMPONENT_MAPPING;
    srvDesc.Format = mFormat;
    srvDesc.ViewDimension = D3D12_SRV_DIMENSION_TEXTURECUBE;
    srvDesc.TextureCube.MostDetailedMip = 0;
    srvDesc.TextureCube.MipLevels = 1;
    srvDesc.TextureCube.ResourceMinLODClamp = 0.0f;
    md3dDevice->CreateShaderResourceView(mCubeMap.Get(), &srvDesc, mhCpuSrv);

    // 每面一个 RTV
    for (int i = 0; i < 6; ++i) {
        D3D12_RENDER_TARGET_VIEW_DESC rtvDesc;
        rtvDesc.ViewDimension = D3D12_RTV_DIMENSION_TEXTURE2DARRAY;
        rtvDesc.Format = mFormat;
        rtvDesc.Texture2DArray.MipSlice = 0;
        rtvDesc.Texture2DArray.PlaneSlice = 0;
        rtvDesc.Texture2DArray.FirstArraySlice = i;  // 第 i 个面
        rtvDesc.Texture2DArray.ArraySize = 1;
        md3dDevice->CreateRenderTargetView(mCubeMap.Get(), &rtvDesc, mhCpuRtv[i]);
    }
}
```

### 18.5.5 深度缓冲

立方体贴图分辨率通常和主后台缓冲不同，需要专用深度缓冲（6 个面**共用**一个即可，因为是逐面绘制）。

### 18.5.6 视口与裁剪矩形

```cpp
mViewport    = { 0.0f, 0.0f, (float)width, (float)height, 0.0f, 1.0f };
mScissorRect = { 0, 0, (long)width, (long)height };
```

### 18.5.7 立方体贴图相机

创建 6 个相机，分别朝 ±X、±Y、±Z 方向，视野角 90°：

```cpp
Camera mCubeMapCamera[6];

void DynamicCubeMapApp::BuildCubeFaceCamera(float x, float y, float z)
{
    XMFLOAT3 center(x, y, z);

    XMFLOAT3 targets[6] = {
        XMFLOAT3(x + 1.0f, y, z),  // +X
        XMFLOAT3(x - 1.0f, y, z),  // -X
        XMFLOAT3(x, y + 1.0f, z),  // +Y
        XMFLOAT3(x, y - 1.0f, z),  // -Y
        XMFLOAT3(x, y, z + 1.0f),  // +Z
        XMFLOAT3(x, y, z - 1.0f),  // -Z
    };

    // +Y / -Y 时 up 不能是 (0,1,0)
    XMFLOAT3 ups[6] = {
        XMFLOAT3(0.0f, 1.0f,  0.0f), // +X
        XMFLOAT3(0.0f, 1.0f,  0.0f), // -X
        XMFLOAT3(0.0f, 0.0f, -1.0f), // +Y
        XMFLOAT3(0.0f, 0.0f, +1.0f), // -Y
        XMFLOAT3(0.0f, 1.0f,  0.0f), // +Z
        XMFLOAT3(0.0f, 1.0f,  0.0f), // -Z
    };

    for (int i = 0; i < 6; ++i) {
        mCubeMapCamera[i].LookAt(center, targets[i], ups[i]);
        mCubeMapCamera[i].SetLens(0.5f * XM_PI, 1.0f, 0.1f, 1000.0f);  // 90° FOV
        mCubeMapCamera[i].UpdateViewMatrix();
    }
}
```

每个面需要自己的 PassCB（view/proj 矩阵不同），所以 FrameResource 创建 7 套 PassCB（1 主相机 + 6 面）。

### 18.5.8 渲染到立方体贴图

```cpp
void DynamicCubeMapApp::DrawSceneToCubeMap()
{
    mCommandList->RSSetViewports(1, &mDynamicCubeMap->Viewport());
    mCommandList->RSSetScissorRects(1, &mDynamicCubeMap->ScissorRect());

    // 转换为 RENDER_TARGET
    mCommandList->ResourceBarrier(1, &CD3DX12_RESOURCE_BARRIER::Transition(
        mDynamicCubeMap->Resource(),
        D3D12_RESOURCE_STATE_GENERIC_READ,
        D3D12_RESOURCE_STATE_RENDER_TARGET));

    UINT passCBByteSize = d3dUtil::CalcConstantBufferByteSize(sizeof(PassConstants));

    for (int i = 0; i < 6; ++i) {
        // 清除当前面的 RTV 和共享 DSV
        mCommandList->ClearRenderTargetView(mDynamicCubeMap->Rtv(i),
            Colors::LightSteelBlue, 0, nullptr);
        mCommandList->ClearDepthStencilView(mCubeDSV,
            D3D12_CLEAR_FLAG_DEPTH | D3D12_CLEAR_FLAG_STENCIL,
            1.0f, 0, 0, nullptr);

        mCommandList->OMSetRenderTargets(1, &mDynamicCubeMap->Rtv(i),
            true, &mCubeDSV);

        // 该面对应的 PassCB（元素 1~6）
        auto passCB = mCurrFrameResource->PassCB->Resource();
        D3D12_GPU_VIRTUAL_ADDRESS passCBAddress =
            passCB->GetGPUVirtualAddress() + (1 + i) * passCBByteSize;
        mCommandList->SetGraphicsRootConstantBufferView(1, passCBAddress);

        // 绘制场景（不包括中心反射物体）
        DrawRenderItems(mCommandList.Get(), mRitemLayer[(int)RenderLayer::Opaque]);

        mCommandList->SetPipelineState(mPSOs["sky"].Get());
        DrawRenderItems(mCommandList.Get(), mRitemLayer[(int)RenderLayer::Sky]);
        mCommandList->SetPipelineState(mPSOs["opaque"].Get());
    }

    // 恢复为 GENERIC_READ 供着色器采样
    mCommandList->ResourceBarrier(1, &CD3DX12_RESOURCE_BARRIER::Transition(
        mDynamicCubeMap->Resource(),
        D3D12_RESOURCE_STATE_RENDER_TARGET,
        D3D12_RESOURCE_STATE_GENERIC_READ));
}
```

然后主渲染：

```cpp
DrawSceneToCubeMap();

// 主渲染目标
mCommandList->RSSetViewports(1, &mScreenViewport);
mCommandList->RSSetScissorRects(1, &mScissorRect);
// ... 清屏、绑定 RTV ...

// 反射物体使用动态立方体贴图
mCommandList->SetGraphicsRootDescriptorTable(3, dynamicTexDescriptor);
DrawRenderItems(mCommandList.Get(),
    mRitemLayer[(int)RenderLayer::OpaqueDynamicReflectors]);

// 其他物体使用静态背景立方体贴图
mCommandList->SetGraphicsRootDescriptorTable(3, skyTexDescriptor);
DrawRenderItems(mCommandList.Get(), mRitemLayer[(int)RenderLayer::Opaque]);

mCommandList->SetPipelineState(mPSOs["sky"].Get());
DrawRenderItems(mCommandList.Get(), mRitemLayer[(int)RenderLayer::Sky]);
```

---

## 18.6 用几何着色器一次性渲染立方体贴图

上节做法每帧需要 6 次 draw 集合调用。一种优化是用**几何着色器**——一次绘制就同时填充 6 个面。

**关键步骤**：

1. 把 RTV 和 DSV 创建为**整个纹理数组**的视图（而非单个切片）：

```cpp
D3D10_RENDER_TARGET_VIEW_DESC DescRT;
DescRT.Format = dstex.Format;
DescRT.ViewDimension = D3D10_RTV_DIMENSION_TEXTURE2DARRAY;
DescRT.Texture2DArray.FirstArraySlice = 0;
DescRT.Texture2DArray.ArraySize       = 6;  // 整个 6 元素数组
DescRT.Texture2DArray.MipSlice        = 0;
```

2. 在常量缓冲里准备 6 个视图矩阵；
3. **几何着色器把每个输入三角形复制 6 次**，并通过 `SV_RenderTargetArrayIndex` 系统值指定写入哪一面：

```hlsl
struct PS_CUBEMAP_IN {
    float4 Pos     : SV_POSITION;
    float2 Tex     : TEXCOORD0;
    uint   RTIndex : SV_RenderTargetArrayIndex;  // 关键
};

[maxvertexcount(18)]  // 6 面 × 3 顶点
void GS_CubeMap(triangle GS_CUBEMAP_IN input[3],
                inout TriangleStream<PS_CUBEMAP_IN> CubeMapStream)
{
    for (int f = 0; f < 6; ++f) {
        PS_CUBEMAP_IN output;
        output.RTIndex = f;

        for (int v = 0; v < 3; ++v) {
            output.Pos = mul(input[v].Pos, g_mViewProj[f]);
            output.Tex = input[v].Tex;
            CubeMapStream.Append(output);
        }
        CubeMapStream.RestartStrip();
    }
}
```

`SV_RenderTargetArrayIndex` 只能在几何着色器输出，且仅当 RTV 是纹理数组视图时可用。

---

## 本章小结

| 主题 | 核心要点 |
|------|---------|
| **立方体贴图** | 6 张纹理 + 3D 查找向量 |
| **HLSL 类型** | `TextureCube` + `Sample(sampler, lookupVector)` |
| **环境贴图** | 在某点拍 6 张 90°FOV 图，组成立方体贴图 |
| **天空盒** | 大球以相机为中心 + `PosH.xyww` 让 z=1 |
| **天空 PSO** | 关闭背面剔除 + `LESS_EQUAL` 深度比较 |
| **反射** | `r = reflect(-v, n)`，用 r 采样立方体贴图 |
| **盒代理几何** | 用 ray/AABB 改进反射的位置依赖 |
| **动态立方体贴图** | 每帧把场景渲染 6 次到 6 个面 |
| **6 面相机** | 90° FOV，±X/±Y/±Z 方向 |
| **几何着色器优化** | 一次绘制填 6 面，靠 `SV_RenderTargetArrayIndex` |

---

## 课后练习思路

1. 自己用 Terragen 制作一组环境贴图，用 texassemble 合成 DDS。
2. 给球体表面应用动态立方体贴图反射，并比较与静态贴图的差异。
3. 实现"盒代理"反射改进，比较平面物体的反射准确度。
4. 用几何着色器实现一次性 6 面渲染，比较性能差异。

---

> **下一步**：第 19 章将介绍**法线贴图**——通过纹理存储法线信息，让逐像素光照获得超越逐顶点的精细细节。
