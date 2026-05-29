# 第 9 章 纹理映射

---

## 9.1 纹理和资源回顾

纹理本质上是一个数据元素矩阵。2D 纹理最常见的用途是存储 2D 图像数据，但纹理的用途远不止于此——例如在法线贴图中，每个元素存储的是一个 3D 向量而不是颜色。

Direct3D 支持 1D、2D 和 3D 纹理，统一由 `ID3D12Resource` 接口表示。纹理与缓冲区资源的不同之处在于：
- 纹理可以有 mipmap 层级
- GPU 可以对纹理执行特殊操作（过滤、多重采样）
- 纹理资源的数据格式受限（由 `DXGI_FORMAT` 描述），而缓冲区可以存储任意数据

纹理可以绑定到渲染管线的不同阶段，常见用法包括：
- 作为**渲染目标**（RTV）：Direct3D 向纹理中绘制
- 作为**着色器资源**（SRV）：在着色器中采样纹理

一个纹理可以同时具备这两种用途（但不同时），需要创建两个不同的描述符：一个 RTV 描述符和一个 SRV 描述符。本章只关注将纹理作为着色器资源使用。

---

## 9.2 纹理坐标

Direct3D 使用 `(u, v)` 纹理坐标系统：
- **u 轴**：水平方向，从左到右，`[0, 1]`
- **v 轴**：垂直方向，从上到下，`[0, 1]`

归一化到 `[0, 1]` 的好处是维度无关：`(0.5, 0.5)` 始终表示纹理中心，无论纹理实际像素尺寸是 `256×256` 还是 `2048×2048`。

每个 3D 三角形的顶点对应纹理空间中的一个 2D 顶点，从而在 3D 三角形和 2D 纹理三角形之间建立映射关系。光栅化时，顶点纹理坐标在三角形内部进行插值。

```cpp
struct Vertex {
    DirectX::XMFLOAT3 Pos;
    DirectX::XMFLOAT3 Normal;
    DirectX::XMFLOAT2 TexC;  // 纹理坐标
};

std::vector<D3D12_INPUT_ELEMENT_DESC> mInputLayout = {
    { "POSITION", 0, DXGI_FORMAT_R32G32B32_FLOAT, 0, 0,
      D3D12_INPUT_CLASSIFICATION_PER_VERTEX_DATA, 0 },
    { "NORMAL",   0, DXGI_FORMAT_R32G32B32_FLOAT, 0, 12,
      D3D12_INPUT_CLASSIFICATION_PER_VERTEX_DATA, 0 },
    { "TEXCOORD", 0, DXGI_FORMAT_R32G32_FLOAT,    0, 24,
      D3D12_INPUT_CLASSIFICATION_PER_VERTEX_DATA, 0 }
};
```

**纹理图集（Texture Atlas）**：将多个不相关的图像放在一张大纹理上，不同物体通过不同的纹理坐标引用各自的子纹理区域，可以减少纹理切换和绘制调用开销。

---

## 9.3 纹理数据源

游戏纹理通常由美术师在 Photoshop 等工具中制作，保存为 BMP、DDS、TGA、PNG 等格式。对于实时图形，**DDS（DirectDraw Surface）格式**是首选，因为：
- 支持 GPU 原生理解的多种像素格式
- 支持 GPU 原生解压的压缩格式
- 支持 mipmap、纹理数组、立方体贴图、体纹理

### 9.3.1 DDS 格式概述

DDS 支持多种像素格式（由 `DXGI_FORMAT` 描述）。常用格式：

| 格式 | 说明 |
|------|------|
| `DXGI_FORMAT_B8G8R8A8_UNORM` | 低动态范围图像 |
| `DXGI_FORMAT_R16G16B16A16_FLOAT` | 高动态范围图像 |
| `DXGI_FORMAT_BC1_UNORM` | 块压缩，3 通道 + 1 位 Alpha |
| `DXGI_FORMAT_BC2_UNORM` | 块压缩，3 通道 + 4 位 Alpha |
| `DXGI_FORMAT_BC3_UNORM` | 块压缩，3 通道 + 8 位 Alpha |
| `DXGI_FORMAT_BC4_UNORM` | 块压缩，单通道（灰度） |
| `DXGI_FORMAT_BC5_UNORM` | 块压缩，双通道 |
| `DXGI_FORMAT_BC6_UF16` | 块压缩，HDR 数据 |
| `DXGI_FORMAT_BC7_UNORM` | 块压缩，高质量 RGBA，适合法线贴图 |

**块压缩（Block Compression, BC）**：
- 压缩纹理在 GPU 内存中以压缩状态存储，使用时由 GPU 实时解压
- DDS 文件本身也更小，节省磁盘空间
- 压缩纹理**不能**用作渲染目标
- 块压缩算法以 `4×4` 像素块为单位工作，因此纹理尺寸必须是 4 的倍数

### 9.3.2 创建 DDS 文件

1. **NVIDIA Photoshop 插件**：支持导出 DDS，可选择 `DXGI_FORMAT`、生成 mipmap
2. **texconv 命令行工具**：微软提供的转换工具，支持格式转换、缩放、mipmap 生成等

```bash
# 将 BMP 转换为 BC3 压缩的 DDS，生成 10 级 mipmap
texconv -m 10 -f BC3_UNORM treeArray.bmp
```

3. **texassemble**：用于创建纹理数组、体纹理、立方体贴图的 DDS 文件

---

## 9.4 创建和启用纹理

### 9.4.1 加载 DDS 文件

使用 `CreateDDSTextureFromFile12` 函数（位于 `Common/DDSTextureLoader.h/.cpp`）加载 DDS 文件：

```cpp
HRESULT DirectX::CreateDDSTextureFromFile12(
    _In_  ID3D12Device* device,
    _In_  ID3D12GraphicsCommandList* cmdList,
    _In_z_ const wchar_t* szFileName,
    _Out_ Microsoft::WRL::ComPtr<ID3D12Resource>& texture,
    _Out_ Microsoft::WRL::ComPtr<ID3D12Resource>& textureUploadHeap
);
```

| 参数 | 说明 |
|------|------|
| `device` | D3D12 设备指针 |
| `cmdList` | 用于提交 GPU 命令的命令列表 |
| `szFileName` | 图像文件名 |
| `texture` | 返回创建的纹理资源（默认堆） |
| `textureUploadHeap` | 返回用于上传的临时堆资源，GPU 完成复制前不能销毁 |

```cpp
struct Texture {
    std::string Name;
    std::wstring Filename;
    Microsoft::WRL::ComPtr<ID3D12Resource> Resource = nullptr;
    Microsoft::WRL::ComPtr<ID3D12Resource> UploadHeap = nullptr;
};

std::unordered_map<std::string, std::unique_ptr<Texture>> mTextures;

void CrateApp::LoadTextures() {
    auto woodCrateTex = std::make_unique<Texture>();
    woodCrateTex->Name = "woodCrateTex";
    woodCrateTex->Filename = L"Textures/WoodCrate01.dds";

    ThrowIfFailed(DirectX::CreateDDSTextureFromFile12(
        md3dDevice.Get(), mCommandList.Get(),
        woodCrateTex->Filename.c_str(),
        woodCrateTex->Resource,
        woodCrateTex->UploadHeap));

    mTextures[woodCrateTex->Name] = std::move(woodCrateTex);
}
```

### 9.4.2 SRV 描述符堆

纹理资源创建后，需要创建 SRV 描述符，以便在着色器中使用。首先需要创建描述符堆：

```cpp
D3D12_DESCRIPTOR_HEAP_DESC srvHeapDesc = {};
srvHeapDesc.NumDescriptors = 3;  // 假设有 3 个纹理
srvHeapDesc.Type = D3D12_DESCRIPTOR_HEAP_TYPE_CBV_SRV_UAV;
srvHeapDesc.Flags = D3D12_DESCRIPTOR_HEAP_FLAG_SHADER_VISIBLE;

ThrowIfFailed(md3dDevice->CreateDescriptorHeap(
    &srvHeapDesc, IID_PPV_ARGS(&mSrvDescriptorHeap)));
```

### 9.4.3 创建 SRV 描述符

**`D3D12_SHADER_RESOURCE_VIEW_DESC` 结构体：**

```cpp
typedef struct D3D12_SHADER_RESOURCE_VIEW_DESC {
    DXGI_FORMAT                     Format;
    D3D12_SRV_DIMENSION             ViewDimension;
    UINT                            Shader4ComponentMapping;
    union {
        D3D12_BUFFER_SRV        Buffer;
        D3D12_TEX1D_SRV         Texture1D;
        D3D12_TEX1D_ARRAY_SRV   Texture1DArray;
        D3D12_TEX2D_SRV         Texture2D;
        D3D12_TEX2D_ARRAY_SRV   Texture2DArray;
        D3D12_TEX2DMS_SRV       Texture2DMS;
        D3D12_TEX2DMS_ARRAY_SRV Texture2DMSArray;
        D3D12_TEX3D_SRV         Texture3D;
        D3D12_TEXCUBE_SRV       TextureCube;
        D3D12_TEXCUBE_ARRAY_SRV TextureCubeArray;
    };
} D3D12_SHADER_RESOURCE_VIEW_DESC;
```

**`D3D12_TEX2D_SRV` 结构体：**

```cpp
typedef struct D3D12_TEX2D_SRV {
    UINT  MostDetailedMip;
    UINT  MipLevels;
    UINT  PlaneSlice;
    FLOAT ResourceMinLODClamp;
} D3D12_TEX2D_SRV;
```

| 成员 | 说明 |
|------|------|
| `Format` | 资源格式。创建时为 typeless 则此处必须指定具体类型 |
| `ViewDimension` | 资源维度：`TEXTURE2D`、`TEXTURE1D`、`TEXTURE3D`、`TEXTURECUBE` 等 |
| `Shader4ComponentMapping` | 组件重映射。通常使用 `D3D12_DEFAULT_SHADER_4_COMPONENT_MAPPING` |
| `MostDetailedMip` | 最详细的 mipmap 层级索引（0 为最高分辨率） |
| `MipLevels` | 要查看的 mipmap 数量。设为 `-1` 表示从 `MostDetailedMip` 到最后一级 |
| `ResourceMinLODClamp` | 最小可访问 mipmap 级别。`0.0` 表示全部可访问 |

```cpp
CD3DX12_CPU_DESCRIPTOR_HANDLE hDescriptor(
    mSrvDescriptorHeap->GetCPUDescriptorHandleForHeapStart());

D3D12_SHADER_RESOURCE_VIEW_DESC srvDesc = {};
srvDesc.Shader4ComponentMapping = D3D12_DEFAULT_SHADER_4_COMPONENT_MAPPING;
srvDesc.Format = texture->GetDesc().Format;
srvDesc.ViewDimension = D3D12_SRV_DIMENSION_TEXTURE2D;
srvDesc.Texture2D.MostDetailedMip = 0;
srvDesc.Texture2D.MipLevels = texture->GetDesc().MipLevels;
srvDesc.Texture2D.ResourceMinLODClamp = 0.0f;

md3dDevice->CreateShaderResourceView(texture.Get(), &srvDesc, hDescriptor);
```

### 9.4.4 绑定纹理到管线

材质中添加纹理索引：

```cpp
struct Material {
    // ...
    int DiffuseSrvHeapIndex = -1;  // SRV 堆中的索引
    // ...
};
```

绘制时绑定纹理：

```cpp
void CrateApp::DrawRenderItems(
    ID3D12GraphicsCommandList* cmdList,
    const std::vector<RenderItem*>& ritems)
{
    // ...
    for (size_t i = 0; i < ritems.size(); ++i) {
        auto ri = ritems[i];
        // ...

        // 获取纹理 SRV 句柄
        CD3DX12_GPU_DESCRIPTOR_HANDLE tex(
            mSrvDescriptorHeap->GetGPUDescriptorHandleForHeapStart());
        tex.Offset(ri->Mat->DiffuseSrvHeapIndex, mCbvSrvDescriptorSize);

        cmdList->SetGraphicsRootDescriptorTable(0, tex);   // 纹理
        cmdList->SetGraphicsRootConstantBufferView(1, objCBAddress);
        cmdList->SetGraphicsRootConstantBufferView(3, matCBAddress);
        cmdList->DrawIndexedInstanced(...);
    }
}
```

像素着色器中将纹理采样的漫反射反照率与材质常量缓冲区的反照率相乘：

```hlsl
float4 texDiffuseAlbedo = gDiffuseMap.Sample(gsamLinearWrap, pin.TexC);
float4 diffuseAlbedo = texDiffuseAlbedo * gDiffuseAlbedo;
```

通常 `gDiffuseAlbedo` 设为 `(1,1,1,1)` 以不改变纹理颜色，但可以通过微调实现色调调整（如稍微偏蓝）。

---

## 9.5 纹理过滤

纹理坐标 `(u, v)` 通常不会恰好落在 texel 中心。GPU 需要通过**过滤**来近似 texel 之间的颜色值。

### 9.5.1 放大（Magnification）

**放大**发生在少数 texel 覆盖大量像素时（例如相机靠近纹理表面）。GPU 支持两种插值方法：

| 过滤方式 | 说明 | 别名 |
|---------|------|------|
| **点过滤（Point Filtering）** | 使用最近的 texel 值，产生块状外观 | 常数插值 / 最近邻采样 |
| **线性过滤（Linear Filtering）** | 对相邻 texel 进行线性插值，更平滑 | 双线性插值（2D） |

**双线性插值**：给定纹理坐标落在四个 texel 之间时，先在 u 方向做两次 1D 线性插值，再在 v 方向做一次 1D 线性插值。

### 9.5.2 缩小（Minification）

**缩小**发生在大量 texel 覆盖少量像素时（例如相机远离纹理表面）。除了点/线性过滤外，还需要 **Mipmap** 技术。

**Mipmap**：在初始化时预计算纹理的缩小版本链，每级尺寸减半，直到 `1×1`。

```
原始: 256x256 (Level 0)
      128x128 (Level 1)
       64x64  (Level 2)
       32x32  (Level 3)
       ...
        1x1   (Level 8)
```

运行时 GPU 根据屏幕几何体的投影分辨率选择最合适的 mipmap 层级：

| Mipmap 过滤 | 说明 |
|------------|------|
| **点 mipmap 过滤** | 选择最接近的单个 mipmap 层级 |
| **线性 mipmap 过滤** | 选择两个最接近的层级，分别采样后线性插值（三线性过滤） |

### 9.5.3 各向异性过滤

**各向异性过滤（Anisotropic Filtering）** 用于改善多边形法线与视线方向夹角很大时（例如几乎垂直于视平面的地面）的纹理模糊问题。这是质量最高但开销最大的过滤方式。

---

## 9.6 寻址模式

当纹理坐标超出 `[0, 1]` 范围时，Direct3D 通过**寻址模式**定义如何处理：

**`D3D12_TEXTURE_ADDRESS_MODE` 枚举：**

```cpp
typedef enum D3D12_TEXTURE_ADDRESS_MODE {
    D3D12_TEXTURE_ADDRESS_MODE_WRAP        = 1,  // 重复（平铺）
    D3D12_TEXTURE_ADDRESS_MODE_MIRROR      = 2,  // 镜像
    D3D12_TEXTURE_ADDRESS_MODE_CLAMP       = 3,  // 钳制
    D3D12_TEXTURE_ADDRESS_MODE_BORDER      = 4,  // 边框颜色
    D3D12_TEXTURE_ADDRESS_MODE_MIRROR_ONCE = 5   // 镜像一次
} D3D12_TEXTURE_ADDRESS_MODE;
```

| 模式 | 行为 | 典型用途 |
|------|------|---------|
| **Wrap（重复）** | 纹理在每个整数边界重复 | 平铺纹理（砖墙、草地） |
| **Clamp（钳制）** | 超出范围的坐标映射到边界 texel | UI 元素、避免边缘重复 |
| **Mirror（镜像）** | 纹理在每个整数边界镜像翻转 | 减少重复感 |
| **Border（边框）** | 超出范围的坐标映射到指定的边框颜色 | 特殊效果 |

**平铺（Tiling）** 是最常用的技术：通过将纹理坐标缩放到 `[0, n]` 范围，使纹理在表面上重复 `n×n` 次，从而在不增加纹理数据的情况下提高表面分辨率。平铺时需要纹理是**无缝的**（seamless），否则重复边界会很明显。

---

## 9.7 采样器对象

过滤方式和寻址模式由**采样器对象（Sampler Object）**定义。应用程序通常需要多个采样器来以不同方式采样纹理。

### 9.7.1 创建采样器

采样器描述符存储在**采样器描述符堆**中：

```cpp
D3D12_DESCRIPTOR_HEAP_DESC samplerHeapDesc = {};
samplerHeapDesc.NumDescriptors = 1;
samplerHeapDesc.Type = D3D12_DESCRIPTOR_HEAP_TYPE_SAMPLER;
samplerHeapDesc.Flags = D3D12_DESCRIPTOR_HEAP_FLAG_SHADER_VISIBLE;

ComPtr<ID3D12DescriptorHeap> mSamplerDescriptorHeap;
ThrowIfFailed(md3dDevice->CreateDescriptorHeap(
    &samplerHeapDesc, IID_PPV_ARGS(&mSamplerDescriptorHeap)));
```

**`D3D12_SAMPLER_DESC` 结构体：**

```cpp
typedef struct D3D12_SAMPLER_DESC {
    D3D12_FILTER               Filter;
    D3D12_TEXTURE_ADDRESS_MODE AddressU;
    D3D12_TEXTURE_ADDRESS_MODE AddressV;
    D3D12_TEXTURE_ADDRESS_MODE AddressW;
    FLOAT                      MipLODBias;
    UINT                       MaxAnisotropy;
    D3D12_COMPARISON_FUNC      ComparisonFunc;
    FLOAT                      BorderColor[4];
    FLOAT                      MinLOD;
    FLOAT                      MaxLOD;
} D3D12_SAMPLER_DESC;
```

| 成员 | 说明 |
|------|------|
| `Filter` | 过滤模式 |
| `AddressU/V/W` | u/v/w 方向的寻址模式 |
| `MipLODBias` | mipmap LOD 偏移，默认 `0.0` |
| `MaxAnisotropy` | 最大各向异性值（1-16），仅对各向异性过滤有效 |
| `ComparisonFunc` | 比较函数，用于阴影映射等特殊应用，默认 `ALWAYS` |
| `BorderColor` | 边框颜色，仅对 `BORDER` 寻址模式有效 |
| `MinLOD` / `MaxLOD` | 最小/最大可访问 mipmap 级别 |

**常用 `D3D12_FILTER` 值：**

| 过滤值 | Min | Mag | Mip |
|--------|-----|-----|-----|
| `MIN_MAG_MIP_POINT` | 点 | 点 | 点 |
| `MIN_MAG_LINEAR_MIP_POINT` | 线性 | 线性 | 点 |
| `MIN_MAG_MIP_LINEAR` | 线性 | 线性 | 线性（三线性） |
| `ANISOTROPIC` | 各向异性 | 各向异性 | 各向异性 |

```cpp
D3D12_SAMPLER_DESC samplerDesc = {};
samplerDesc.Filter = D3D12_FILTER_MIN_MAG_MIP_LINEAR;
samplerDesc.AddressU = D3D12_TEXTURE_ADDRESS_MODE_WRAP;
samplerDesc.AddressV = D3D12_TEXTURE_ADDRESS_MODE_WRAP;
samplerDesc.AddressW = D3D12_TEXTURE_ADDRESS_MODE_WRAP;
samplerDesc.MinLOD = 0;
samplerDesc.MaxLOD = D3D12_FLOAT32_MAX;
samplerDesc.MipLODBias = 0.0f;
samplerDesc.MaxAnisotropy = 1;
samplerDesc.ComparisonFunc = D3D12_COMPARISON_FUNC_ALWAYS;

md3dDevice->CreateSampler(&samplerDesc,
    mSamplerDescriptorHeap->GetCPUDescriptorHandleForHeapStart());
```

绑定采样器到根签名：

```cpp
commandList->SetGraphicsRootDescriptorTable(1,
    mSamplerDescriptorHeap->GetGPUDescriptorHandleForHeapStart());
```

### 9.7.2 静态采样器

由于图形应用通常只需要少量采样器，Direct3D 提供了**静态采样器**快捷方式，无需创建采样器堆。

**`D3D12_STATIC_SAMPLER_DESC` 结构体**：与 `D3D12_SAMPLER_DESC` 类似，但：
- 边框颜色受限（只能是透明黑、不透明黑、不透明白）
- 包含着色器寄存器、寄存器空间、着色器可见性字段
- 最多可定义 2032 个静态采样器

```cpp
std::array<const CD3DX12_STATIC_SAMPLER_DESC, 6> GetStaticSamplers() {
    const CD3DX12_STATIC_SAMPLER_DESC pointWrap(
        0, D3D12_FILTER_MIN_MAG_MIP_POINT,
        D3D12_TEXTURE_ADDRESS_MODE_WRAP,
        D3D12_TEXTURE_ADDRESS_MODE_WRAP,
        D3D12_TEXTURE_ADDRESS_MODE_WRAP);

    const CD3DX12_STATIC_SAMPLER_DESC pointClamp(
        1, D3D12_FILTER_MIN_MAG_MIP_POINT,
        D3D12_TEXTURE_ADDRESS_MODE_CLAMP,
        D3D12_TEXTURE_ADDRESS_MODE_CLAMP,
        D3D12_TEXTURE_ADDRESS_MODE_CLAMP);

    const CD3DX12_STATIC_SAMPLER_DESC linearWrap(
        2, D3D12_FILTER_MIN_MAG_MIP_LINEAR,
        D3D12_TEXTURE_ADDRESS_MODE_WRAP,
        D3D12_TEXTURE_ADDRESS_MODE_WRAP,
        D3D12_TEXTURE_ADDRESS_MODE_WRAP);

    const CD3DX12_STATIC_SAMPLER_DESC linearClamp(
        3, D3D12_FILTER_MIN_MAG_MIP_LINEAR,
        D3D12_TEXTURE_ADDRESS_MODE_CLAMP,
        D3D12_TEXTURE_ADDRESS_MODE_CLAMP,
        D3D12_TEXTURE_ADDRESS_MODE_CLAMP);

    const CD3DX12_STATIC_SAMPLER_DESC anisotropicWrap(
        4, D3D12_FILTER_ANISOTROPIC,
        D3D12_TEXTURE_ADDRESS_MODE_WRAP,
        D3D12_TEXTURE_ADDRESS_MODE_WRAP,
        D3D12_TEXTURE_ADDRESS_MODE_WRAP,
        0.0f, 8);  // mipLODBias, maxAnisotropy

    const CD3DX12_STATIC_SAMPLER_DESC anisotropicClamp(
        5, D3D12_FILTER_ANISOTROPIC,
        D3D12_TEXTURE_ADDRESS_MODE_CLAMP,
        D3D12_TEXTURE_ADDRESS_MODE_CLAMP,
        D3D12_TEXTURE_ADDRESS_MODE_CLAMP,
        0.0f, 8);

    return { pointWrap, pointClamp, linearWrap,
             linearClamp, anisotropicWrap, anisotropicClamp };
}

void BuildRootSignature() {
    CD3DX12_DESCRIPTOR_RANGE texTable;
    texTable.Init(D3D12_DESCRIPTOR_RANGE_TYPE_SRV, 1, 0);

    CD3DX12_ROOT_PARAMETER slotRootParameter[4];
    slotRootParameter[0].InitAsDescriptorTable(1, &texTable,
        D3D12_SHADER_VISIBILITY_PIXEL);  // texture
    slotRootParameter[1].InitAsConstantBufferView(0);  // per-object
    slotRootParameter[2].InitAsConstantBufferView(1);  // per-pass
    slotRootParameter[3].InitAsConstantBufferView(2);  // per-material

    auto staticSamplers = GetStaticSamplers();

    CD3DX12_ROOT_SIGNATURE_DESC rootSigDesc(
        4, slotRootParameter,
        (UINT)staticSamplers.size(), staticSamplers.data(),
        D3D12_ROOT_SIGNATURE_FLAG_ALLOW_INPUT_ASSEMBLER_INPUT_LAYOUT);

    // ... serialize and create
}
```

---

## 9.8 在着色器中采样纹理

HLSL 中纹理和采样器的声明：

```hlsl
Texture2D gDiffuseMap : register(t0);

SamplerState gsamPointWrap        : register(s0);
SamplerState gsamPointClamp       : register(s1);
SamplerState gsamLinearWrap       : register(s2);
SamplerState gsamLinearClamp      : register(s3);
SamplerState gsamAnisotropicWrap  : register(s4);
SamplerState gsamAnisotropicClamp : register(s5);
```

**`Texture2D::Sample` 方法：**

```hlsl
float4 Texture2D::Sample(SamplerState S, float2 Location);
```

| 参数 | 说明 |
|------|------|
| `S` | 采样器状态对象，定义过滤和寻址模式 |
| `Location` | 纹理坐标 `(u, v)` |

```hlsl
float4 PS(VertexOut pin) : SV_Target {
    float4 diffuseAlbedo = gDiffuseMap.Sample(gsamAnisotropicWrap, pin.TexC)
                         * gDiffuseAlbedo;
    // ... 光照计算
}
```

---

## 9.9 Crate Demo

### 9.9.1 指定纹理坐标

`GeometryGenerator::CreateBox` 为立方体每个面生成纹理坐标，使整张贴图映射到每个面上：

```cpp
// 前面
v[0] = Vertex(-w2, -h2, -d2, ..., 0.0f, 1.0f);
v[1] = Vertex(-w2, +h2, -d2, ..., 0.0f, 0.0f);
v[2] = Vertex(+w2, +h2, -d2, ..., 1.0f, 0.0f);
v[3] = Vertex(+w2, -h2, -d2, ..., 1.0f, 1.0f);

// 后面（注意纹理坐标方向）
v[4] = Vertex(-w2, -h2, +d2, ..., 1.0f, 1.0f);
v[5] = Vertex(+w2, -h2, +d2, ..., 0.0f, 1.0f);
v[6] = Vertex(+w2, +h2, +d2, ..., 0.0f, 0.0f);
v[7] = Vertex(-w2, +h2, +d2, ..., 1.0f, 0.0f);
```

### 9.9.2 创建纹理

```cpp
void CrateApp::LoadTextures() {
    auto woodCrateTex = std::make_unique<Texture>();
    woodCrateTex->Name = "woodCrateTex";
    woodCrateTex->Filename = L"Textures/WoodCrate01.dds";

    ThrowIfFailed(DirectX::CreateDDSTextureFromFile12(
        md3dDevice.Get(), mCommandList.Get(),
        woodCrateTex->Filename.c_str(),
        woodCrateTex->Resource, woodCrateTex->UploadHeap));

    mTextures[woodCrateTex->Name] = std::move(woodCrateTex);
}
```

### 9.9.3 更新 HLSL

```hlsl
Texture2D gDiffuseMap : register(t0);
SamplerState gsamAnisotropicWrap : register(s4);

cbuffer cbPerObject : register(b0) {
    float4x4 gWorld;
    float4x4 gTexTransform;
};

cbuffer cbMaterial : register(b2) {
    float4 gDiffuseAlbedo;
    float3 gFresnelR0;
    float  gRoughness;
    float4x4 gMatTransform;
};

struct VertexIn {
    float3 PosL    : POSITION;
    float3 NormalL : NORMAL;
    float2 TexC    : TEXCOORD;
};

struct VertexOut {
    float4 PosH    : SV_POSITION;
    float3 PosW    : POSITION;
    float3 NormalW : NORMAL;
    float2 TexC    : TEXCOORD;
};

VertexOut VS(VertexIn vin) {
    VertexOut vout = (VertexOut)0.0f;

    float4 posW = mul(float4(vin.PosL, 1.0f), gWorld);
    vout.PosW = posW.xyz;
    vout.NormalW = mul(vin.NormalL, (float3x3)gWorld);
    vout.PosH = mul(posW, gViewProj);

    float4 texC = mul(float4(vin.TexC, 0.0f, 1.0f), gTexTransform);
    vout.TexC = mul(texC, gMatTransform).xy;

    return vout;
}

float4 PS(VertexOut pin) : SV_Target {
    float4 diffuseAlbedo = gDiffuseMap.Sample(gsamAnisotropicWrap, pin.TexC)
                         * gDiffuseAlbedo;

    pin.NormalW = normalize(pin.NormalW);
    float3 toEyeW = normalize(gEyePosW - pin.PosW);

    float4 ambient = gAmbientLight * diffuseAlbedo;

    const float shininess = 1.0f - gRoughness;
    Material mat = { diffuseAlbedo, gFresnelR0, shininess };
    float3 shadowFactor = 1.0f;

    float4 directLight = ComputeLighting(gLights, mat, pin.PosW,
                                         pin.NormalW, toEyeW, shadowFactor);

    float4 litColor = ambient + directLight;
    litColor.a = diffuseAlbedo.a;
    return litColor;
}
```

---

## 9.10 纹理变换

纹理坐标可以像其他 2D 点一样进行平移、旋转和缩放。两个变换矩阵的作用：
- `gTexTransform`：对象级别的纹理变换
- `gMatTransform`：材质级别的纹理变换（如水流动画）

常见用途：
1. **缩放纹理坐标**：将 `[0,1]` 缩放到 `[0,5]`，使纹理平铺 5 次
2. **平移纹理坐标**：随时间移动，产生云层飘动、水流效果
3. **旋转纹理坐标**：用于粒子效果（如旋转火球）

```hlsl
float4 texC = mul(float4(vin.TexC, 0.0f, 1.0f), gTexTransform);
vout.TexC = mul(texC, gMatTransform).xy;
```

---

## 9.11 带纹理的地形和波浪演示

### 9.11.1 网格纹理坐标生成

对于 `m×n` 的网格，第 `(i, j)` 个顶点的纹理坐标为：

```
u = j / (n - 1)
v = i / (m - 1)
```

这样纹理被拉伸覆盖整个网格。

```cpp
float du = 1.0f / (n - 1);
float dv = 1.0f / (m - 1);

for (uint32 i = 0; i < m; ++i) {
    for (uint32 j = 0; j < n; ++j) {
        // ...
        meshData.Vertices[i * n + j].TexC.x = j * du;
        meshData.Vertices[i * n + j].TexC.y = i * dv;
    }
}
```

### 9.11.2 纹理平铺

地形网格很大，如果拉伸单张贴图会产生严重的放大瑕疵。解决方法是将纹理坐标缩放 5 倍，配合 Wrap 寻址模式使纹理平铺 5×5 次：

```cpp
auto gridRitem = std::make_unique<RenderItem>();
gridRitem->World = MathHelper::Identity4x4();
XMStoreFloat4x4(&gridRitem->TexTransform,
    XMMatrixScaling(5.0f, 5.0f, 1.0f));
```

### 9.11.3 纹理动画

通过每帧微量平移纹理坐标实现水面流动效果：

```cpp
void TexWavesApp::AnimateMaterials(const GameTimer& gt) {
    auto waterMat = mMaterials["water"].get();

    float& tu = waterMat->MatTransform(3, 0);
    float& tv = waterMat->MatTransform(3, 1);

    tu += 0.1f * gt.DeltaTime();
    tv += 0.02f * gt.DeltaTime();

    if (tu >= 1.0f) tu -= 1.0f;
    if (tv >= 1.0f) tv -= 1.0f;

    waterMat->NumFramesDirty = gNumFrameResources;
}
```

使用 Wrap 寻址模式配合无缝纹理，可以无限循环平移纹理坐标。
