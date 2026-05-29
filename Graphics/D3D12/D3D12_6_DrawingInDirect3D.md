# 第 6 章 Direct3D 中的绘制（Part I）

---

## 6.1 顶点和输入布局

在 Direct3D 中，顶点不仅可以存储空间位置，还可以携带颜色、法线、纹理坐标、切线等额外数据。开发者需要首先定义一个**顶点结构体**，然后用**输入布局描述（Input Layout Description）**告诉 Direct3D 这个结构体的内存布局。

### 6.1.1 自定义顶点格式

定义顶点结构体时，根据渲染需求选择需要的数据字段。以下是一个同时包含位置和颜色的顶点类型：

```cpp
struct Vertex {
    DirectX::XMFLOAT3 Pos;    // 顶点位置 (x, y, z)
    DirectX::XMFLOAT4 Color;  // 顶点颜色 (r, g, b, a)
};
```

另一个更复杂的例子，包含位置、法线、切线和两组纹理坐标：

```cpp
struct Vertex {
    DirectX::XMFLOAT3 Pos;      // 位置
    DirectX::XMFLOAT3 Normal;   // 法线
    DirectX::XMFLOAT3 TangentU; // 切线
    DirectX::XMFLOAT2 TexC;     // 纹理坐标
};
```

### 6.1.2 输入布局描述

Direct3D 使用 `D3D12_INPUT_ELEMENT_DESC` 结构体数组来描述顶点结构体的每个字段。

**`D3D12_INPUT_ELEMENT_DESC` 结构体定义：**

```cpp
typedef struct D3D12_INPUT_ELEMENT_DESC {
    LPCSTR                     SemanticName;
    UINT                       SemanticIndex;
    DXGI_FORMAT                Format;
    UINT                       InputSlot;
    UINT                       AlignedByteOffset;
    D3D12_INPUT_CLASSIFICATION InputSlotClass;
    UINT                       InstanceDataStepRate;
} D3D12_INPUT_ELEMENT_DESC;
```

| 参数 | 类型 | 说明 |
|------|------|------|
| `SemanticName` | `LPCSTR` | **语义名称**，用于将顶点元素与着色器输入参数匹配。常用值：`"POSITION"`、`"COLOR"`、`"NORMAL"`、`"TEXCOORD"`、`"TANGENT"` |
| `SemanticIndex` | `UINT` | **语义索引**，当同一语义有多个时使用。例如两组纹理坐标分别用 `TEXCOORD0` 和 `TEXCOORD1`，对应索引 `0` 和 `1` |
| `Format` | `DXGI_FORMAT` | **数据格式**，描述该元素的数据类型和分量数。例如 `DXGI_FORMAT_R32G32B32_FLOAT` 表示 3 个 32 位浮点数 |
| `InputSlot` | `UINT` | **输入槽索引**（0-15），指定该元素从哪个输入槽获取数据。默认使用槽 0 |
| `AlignedByteOffset` | `UINT` | **对齐字节偏移**，该元素相对于顶点结构体起始地址的偏移量。可以使用 `D3D12_APPEND_ALIGNED_ELEMENT` 让 Direct3D 自动计算偏移 |
| `InputSlotClass` | `D3D12_INPUT_CLASSIFICATION` | **输入分类**，通常为 `D3D12_INPUT_PER_VERTEX_DATA`（逐顶点数据）。另一个是 `D3D12_INPUT_PER_INSTANCE_DATA`（实例化绘制时使用） |
| `InstanceDataStepRate` | `UINT` | **实例化步进率**。对于逐顶点数据设为 `0`。实例化绘制时设为每多少个实例更新一次该数据 |

**`D3D12_INPUT_CLASSIFICATION` 枚举：**

```cpp
typedef enum D3D12_INPUT_CLASSIFICATION {
    D3D12_INPUT_CLASSIFICATION_PER_VERTEX_DATA   = 0,
    D3D12_INPUT_CLASSIFICATION_PER_INSTANCE_DATA = 1
} D3D12_INPUT_CLASSIFICATION;
```

**示例：为 `Vertex { XMFLOAT3 Pos; XMFLOAT4 Color; }` 创建输入布局描述**

```cpp
std::vector<D3D12_INPUT_ELEMENT_DESC> mInputLayout = {
    {
        "POSITION",                          // SemanticName
        0,                                   // SemanticIndex
        DXGI_FORMAT_R32G32B32_FLOAT,         // Format: 3×float
        0,                                   // InputSlot
        0,                                   // AlignedByteOffset
        D3D12_INPUT_CLASSIFICATION_PER_VERTEX_DATA,
        0
    },
    {
        "COLOR",                             // SemanticName
        0,                                   // SemanticIndex
        DXGI_FORMAT_R32G32B32A32_FLOAT,      // Format: 4×float
        0,                                   // InputSlot
        12,                                  // AlignedByteOffset: 3×4=12
        D3D12_INPUT_CLASSIFICATION_PER_VERTEX_DATA,
        0
    }
};
```

使用 `D3D12_APPEND_ALIGNED_ELEMENT` 自动计算偏移量：

```cpp
std::vector<D3D12_INPUT_ELEMENT_DESC> mInputLayout = {
    { "POSITION", 0, DXGI_FORMAT_R32G32B32_FLOAT,    0, 0,  D3D12_INPUT_CLASSIFICATION_PER_VERTEX_DATA, 0 },
    { "COLOR",    0, DXGI_FORMAT_R32G32B32A32_FLOAT, 0, 12, D3D12_INPUT_CLASSIFICATION_PER_VERTEX_DATA, 0 }
};
// 或自动计算：
std::vector<D3D12_INPUT_ELEMENT_DESC> mInputLayout = {
    { "POSITION", 0, DXGI_FORMAT_R32G32B32_FLOAT,    0, 0,                             D3D12_INPUT_CLASSIFICATION_PER_VERTEX_DATA, 0 },
    { "COLOR",    0, DXGI_FORMAT_R32G32B32A32_FLOAT, 0, D3D12_APPEND_ALIGNED_ELEMENT, D3D12_INPUT_CLASSIFICATION_PER_VERTEX_DATA, 0 }
};
```

---

## 6.2 顶点缓冲区

为了让 GPU 访问顶点数组，需要将顶点数据放入一种称为**缓冲区（Buffer）**的 GPU 资源中。存储顶点的缓冲区称为**顶点缓冲区（Vertex Buffer）**。缓冲区比纹理简单：它不是多维的，没有 mipmap、过滤或多重采样。

### 6.2.1 创建缓冲区资源

缓冲区通过 `ID3D12Device::CreateCommittedResource` 创建，需要填充 `D3D12_RESOURCE_DESC` 描述结构体。

**`CD3DX12_RESOURCE_DESC::Buffer` 辅助方法：**

```cpp
static inline CD3DX12_RESOURCE_DESC Buffer(
    UINT64 width,
    D3D12_RESOURCE_FLAGS flags = D3D12_RESOURCE_FLAG_NONE,
    UINT64 alignment = 0
);
```

| 参数 | 说明 |
|------|------|
| `width` | 缓冲区大小，以字节为单位 |
| `flags` | 资源标志，通常为 `D3D12_RESOURCE_FLAG_NONE` |
| `alignment` | 对齐方式，缓冲区的默认值为 `0` |

对于静态几何体（每帧不变化的几何体，如树木、建筑、地形、角色），顶点缓冲区应放入**默认堆（`D3D12_HEAP_TYPE_DEFAULT`）**以获得最佳性能。但 CPU 无法直接写入默认堆，因此初始化时需要借助一个中间的上传缓冲区（`D3D12_HEAP_TYPE_UPLOAD`）。

### 6.2.2 上传缓冲区辅助函数

由于每次创建默认缓冲区都需要重复创建上传缓冲区、复制数据、状态转换等步骤，可以封装一个工具函数：

```cpp
Microsoft::WRL::ComPtr<ID3D12Resource> d3dUtil::CreateDefaultBuffer(
    ID3D12Device* device,
    ID3D12GraphicsCommandList* cmdList,
    const void* initData,
    UINT64 byteSize,
    Microsoft::WRL::ComPtr<ID3D12Resource>& uploadBuffer
);
```

| 参数 | 说明 |
|------|------|
| `device` | D3D12 设备接口指针 |
| `cmdList` | 用于录制复制命令的命令列表 |
| `initData` | 指向系统内存中初始化数据的指针 |
| `byteSize` | 要复制的数据字节数 |
| `uploadBuffer` | 输出参数，返回创建的上传缓冲区（调用者需保持其存活直到 GPU 执行完复制命令） |

**`D3D12_SUBRESOURCE_DATA` 结构体：**

```cpp
typedef struct D3D12_SUBRESOURCE_DATA {
    const void* pData;
    LONG_PTR    RowPitch;
    LONG_PTR    SlicePitch;
} D3D12_SUBRESOURCE_DATA;
```

| 成员 | 说明 |
|------|------|
| `pData` | 指向系统内存数据数组的指针 |
| `RowPitch` | 对于缓冲区，表示要复制的数据字节大小 |
| `SlicePitch` | 对于缓冲区，表示要复制的数据字节大小 |

**创建顶点缓冲区的完整示例：**

```cpp
Vertex vertices[] = {
    { XMFLOAT3(-1.0f, -1.0f, -1.0f), XMFLOAT4(Colors::White)   },
    { XMFLOAT3(-1.0f, +1.0f, -1.0f), XMFLOAT4(Colors::Black)   },
    { XMFLOAT3(+1.0f, +1.0f, -1.0f), XMFLOAT4(Colors::Red)     },
    { XMFLOAT3(+1.0f, -1.0f, -1.0f), XMFLOAT4(Colors::Green)   },
    { XMFLOAT3(-1.0f, -1.0f, +1.0f), XMFLOAT4(Colors::Blue)    },
    { XMFLOAT3(-1.0f, +1.0f, +1.0f), XMFLOAT4(Colors::Yellow)  },
    { XMFLOAT3(+1.0f, +1.0f, +1.0f), XMFLOAT4(Colors::Cyan)    },
    { XMFLOAT3(+1.0f, -1.0f, +1.0f), XMFLOAT4(Colors::Magenta) }
};

const UINT64 vbByteSize = 8 * sizeof(Vertex);

ComPtr<ID3D12Resource> VertexBufferGPU = nullptr;
ComPtr<ID3D12Resource> VertexBufferUploader = nullptr;

VertexBufferGPU = d3dUtil::CreateDefaultBuffer(
    md3dDevice.Get(),
    mCommandList.Get(),
    vertices,
    vbByteSize,
    VertexBufferUploader
);
```

### 6.2.3 顶点缓冲区视图

创建顶点缓冲区后，需要创建一个**顶点缓冲区视图（Vertex Buffer View）**才能将其绑定到管线。与 RTV 不同，顶点缓冲区视图**不需要描述符堆**。

**`D3D12_VERTEX_BUFFER_VIEW` 结构体：**

```cpp
typedef struct D3D12_VERTEX_BUFFER_VIEW {
    D3D12_GPU_VIRTUAL_ADDRESS BufferLocation;
    UINT                      SizeInBytes;
    UINT                      StrideInBytes;
} D3D12_VERTEX_BUFFER_VIEW;
```

| 成员 | 说明 |
|------|------|
| `BufferLocation` | 顶点缓冲区资源的 GPU 虚拟地址，通过 `ID3D12Resource::GetGPUVirtualAddress()` 获取 |
| `SizeInBytes` | 从 `BufferLocation` 开始查看的缓冲区字节数 |
| `StrideInBytes` | 每个顶点元素的大小，以字节为单位 |

**创建并绑定顶点缓冲区视图：**

```cpp
D3D12_VERTEX_BUFFER_VIEW vbv;
vbv.BufferLocation = VertexBufferGPU->GetGPUVirtualAddress();
vbv.StrideInBytes  = sizeof(Vertex);
vbv.SizeInBytes    = vbByteSize;

D3D12_VERTEX_BUFFER_VIEW vertexBuffers[1] = { vbv };
mCommandList->IASetVertexBuffers(0, 1, vertexBuffers);
```

### 6.2.4 IASetVertexBuffers

**`ID3D12GraphicsCommandList::IASetVertexBuffers` 方法：**

```cpp
void ID3D12GraphicsCommandList::IASetVertexBuffers(
    UINT                        StartSlot,
    UINT                        NumBuffers,
    const D3D12_VERTEX_BUFFER_VIEW* pViews
);
```

| 参数 | 说明 |
|------|------|
| `StartSlot` | 开始绑定顶点缓冲区的输入槽索引，范围为 0-15 |
| `NumBuffers` | 要绑定的顶点缓冲区数量。如果从槽 `k` 开始绑定 `n` 个缓冲区，则绑定到槽 `k, k+1, ..., k+n-1` |
| `pViews` | 指向顶点缓冲区视图数组首元素的指针 |

顶点缓冲区绑定到输入槽后，会一直保持在那个槽中，直到被显式更改。如果场景中有多个顶点缓冲区，可以按如下方式组织代码：

```cpp
// 使用顶点缓冲区 1 绘制
mCommandList->IASetVertexBuffers(0, 1, &vbView1);
/* ... 绘制使用 vb1 的对象 ... */

// 切换到顶点缓冲区 2
mCommandList->IASetVertexBuffers(0, 1, &vbView2);
/* ... 绘制使用 vb2 的对象 ... */
```

### 6.2.5 DrawInstanced

绑定顶点缓冲区只是让顶点准备好进入管线，真正的绘制命令由 `DrawInstanced` 发出。

**`ID3D12GraphicsCommandList::DrawInstanced` 方法：**

```cpp
void ID3D12GraphicsCommandList::DrawInstanced(
    UINT VertexCountPerInstance,
    UINT InstanceCount,
    UINT StartVertexLocation,
    UINT StartInstanceLocation
);
```

| 参数 | 说明 |
|------|------|
| `VertexCountPerInstance` | 每个实例要绘制的顶点数量 |
| `InstanceCount` | 实例数量。普通绘制设为 `1` |
| `StartVertexLocation` | 顶点缓冲区中开始绘制的第一个顶点的索引（从 0 开始） |
| `StartInstanceLocation` | 实例偏移量，普通绘制设为 `0` |

`VertexCountPerInstance` 和 `StartVertexLocation` 共同定义了顶点缓冲区中要绘制的一个连续子区间：

```
顶点缓冲区:
[ V0 ][ V1 ][ V2 ][ V3 ][ V4 ][ V5 ][ V6 ][ V7 ]
  ^                    ^
  |____________________|
StartVertexLocation=0  VertexCountPerInstance=5
绘制 V0, V1, V2, V3, V4
```

**注意**：`DrawInstanced` 不指定图元类型。图元拓扑通过 `IASetPrimitiveTopology` 单独设置：

```cpp
mCommandList->IASetPrimitiveTopology(D3D_PRIMITIVE_TOPOLOGY_TRIANGLELIST);
```

---

## 6.3 索引和索引缓冲区

与顶点类似，为了让 GPU 访问索引数组，需要将索引数据放入**索引缓冲区（Index Buffer）**。由于 `d3dUtil::CreateDefaultBuffer` 使用 `void*` 作为通用数据指针，因此完全可以用同一个函数创建索引缓冲区。

### 6.3.1 索引缓冲区视图

索引缓冲区视图由 `D3D12_INDEX_BUFFER_VIEW` 结构体描述，同样不需要描述符堆。

**`D3D12_INDEX_BUFFER_VIEW` 结构体：**

```cpp
typedef struct D3D12_INDEX_BUFFER_VIEW {
    D3D12_GPU_VIRTUAL_ADDRESS BufferLocation;
    UINT                      SizeInBytes;
    DXGI_FORMAT               Format;
} D3D12_INDEX_BUFFER_VIEW;
```

| 成员 | 说明 |
|------|------|
| `BufferLocation` | 索引缓冲区资源的 GPU 虚拟地址 |
| `SizeInBytes` | 从 `BufferLocation` 开始查看的缓冲区字节数 |
| `Format` | 索引格式，必须是 `DXGI_FORMAT_R16_UINT`（16 位索引）或 `DXGI_FORMAT_R32_UINT`（32 位索引）。16 位索引节省内存和带宽，仅在需要更大范围时才使用 32 位 |

### 6.3.2 IASetIndexBuffer

**`ID3D12GraphicsCommandList::IASetIndexBuffer` 方法：**

```cpp
void ID3D12GraphicsCommandList::IASetIndexBuffer(
    const D3D12_INDEX_BUFFER_VIEW* pView
);
```

| 参数 | 说明 |
|------|------|
| `pView` | 指向 `D3D12_INDEX_BUFFER_VIEW` 的指针。设为 `nullptr` 可解除绑定 |

**创建并绑定索引缓冲区的完整示例：**

```cpp
std::uint16_t indices[] = {
    // 前面
    0, 1, 2,   0, 2, 3,
    // 后面
    4, 6, 5,   4, 7, 6,
    // 左面
    4, 5, 1,   4, 1, 0,
    // 右面
    3, 2, 6,   3, 6, 7,
    // 顶面
    1, 5, 6,   1, 6, 2,
    // 底面
    4, 0, 3,   4, 3, 7
};

const UINT ibByteSize = 36 * sizeof(std::uint16_t);

ComPtr<ID3D12Resource> IndexBufferGPU = nullptr;
ComPtr<ID3D12Resource> IndexBufferUploader = nullptr;

IndexBufferGPU = d3dUtil::CreateDefaultBuffer(
    md3dDevice.Get(), mCommandList.Get(),
    indices, ibByteSize, IndexBufferUploader);

D3D12_INDEX_BUFFER_VIEW ibv;
ibv.BufferLocation = IndexBufferGPU->GetGPUVirtualAddress();
ibv.Format         = DXGI_FORMAT_R16_UINT;
ibv.SizeInBytes    = ibByteSize;

mCommandList->IASetIndexBuffer(&ibv);
```

### 6.3.3 DrawIndexedInstanced

使用索引时必须用 `DrawIndexedInstanced` 代替 `DrawInstanced`。

**`ID3D12GraphicsCommandList::DrawIndexedInstanced` 方法：**

```cpp
void ID3D12GraphicsCommandList::DrawIndexedInstanced(
    UINT IndexCountPerInstance,
    UINT InstanceCount,
    UINT StartIndexLocation,
    INT  BaseVertexLocation,
    UINT StartInstanceLocation
);
```

| 参数 | 说明 |
|------|------|
| `IndexCountPerInstance` | 每个实例要绘制的索引数量 |
| `InstanceCount` | 实例数量，普通绘制设为 `1` |
| `StartIndexLocation` | 索引缓冲区中开始读取的起始索引位置 |
| `BaseVertexLocation` | 一个整数偏移量，在执行绘制前加到每个索引值上。用于将多个模型的索引合并到一个全局缓冲区时修正索引 |
| `StartInstanceLocation` | 实例偏移量，普通绘制设为 `0` |

### 6.3.4 BaseVertexLocation 的用途

假设场景中有球体、盒子和圆柱体三个对象。最初每个对象有自己的局部顶点缓冲区和局部索引缓冲区，索引值相对于各自的局部顶点缓冲区。

如果将三个对象的顶点和索引分别拼接成一个**全局顶点缓冲区**和一个**全局索引缓冲区**，索引值就不再正确了——因为它们存储的是相对于局部顶点缓冲区的位置，而非全局缓冲区。

解决方法是将每个对象的首顶点在全局缓冲区中的位置作为 `BaseVertexLocation` 传入：

```cpp
// 球体：索引从 0 开始，顶点从 0 开始
mCmdList->DrawIndexedInstanced(numSphereIndices, 1, 0, 0, 0);

// 盒子：索引从 firstBoxIndex 开始，顶点偏移 firstBoxVertexPos
mCmdList->DrawIndexedInstanced(numBoxIndices, 1, firstBoxIndex, firstBoxVertexPos, 0);

// 圆柱体：索引从 firstCylIndex 开始，顶点偏移 firstCylVertexPos
mCmdList->DrawIndexedInstanced(numCylIndices, 1, firstCylIndex, firstCylVertexPos, 0);
```

这样就不需要在 CPU 端重新计算所有索引值，而是让 GPU 在绘制时自动加上偏移量。

---

## 6.4 顶点着色器示例

顶点着色器是在 GPU 上执行的程序，每个被绘制的顶点都会经过它。以下是一个简单的顶点着色器实现：

```hlsl
cbuffer cbPerObject : register(b0) {
    float4x4 gWorldViewProj;
};

void VS(float3 iPosL : POSITION,
        float4 iColor : COLOR,
        out float4 oPosH : SV_POSITION,
        out float4 oColor : COLOR)
{
    // 将顶点从局部空间变换到齐次裁剪空间
    oPosH = mul(float4(iPosL, 1.0f), gWorldViewProj);
    
    // 将顶点颜色直接传给像素着色器
    oColor = iColor;
}
```

### 6.4.1 HLSL 语法要点

着色器使用 **HLSL（High Level Shading Language）** 编写，语法与 C++ 类似。顶点着色器是一个函数，函数名可以任意取（这里叫 `VS`）。

| 特性 | 说明 |
|------|------|
| 输入参数 | 前两个参数是输入参数，对应 C++ 端定义的顶点结构体字段 |
| 输出参数 | 带有 `out` 关键字的参数是输出参数。HLSL 不支持引用或指针，所以用 `out` 返回多个值 |
| 语义（Semantic） | `: POSITION`、`: COLOR`、`: SV_POSITION` 等是语义，用于匹配顶点元素与着色器参数、以及着色器输出与下一阶段输入 |
| `SV_POSITION` | 系统值语义，表示顶点在齐次裁剪空间中的位置。GPU 需要知道这个值，因为它参与裁剪、深度测试和光栅化 |
| `mul` | 内置函数，用于向量-矩阵乘法。支持多种重载形式（向量×矩阵、矩阵×矩阵等） |
| `float4(...)` | 向量构造语法。`float4(iPosL, 1.0f)` 等价于 `float4(iPosL.x, iPosL.y, iPosL.z, 1.0f)` |

### 6.4.2 使用结构体的等价写法

上述顶点着色器可以用输入/输出结构体改写，效果完全相同：

```hlsl
cbuffer cbPerObject : register(b0) {
    float4x4 gWorldViewProj;
};

struct VertexIn {
    float3 PosL : POSITION;
    float4 Color : COLOR;
};

struct VertexOut {
    float4 PosH : SV_POSITION;
    float4 Color : COLOR;
};

VertexOut VS(VertexIn vin) {
    VertexOut vout;
    vout.PosH  = mul(float4(vin.PosL, 1.0f), gWorldViewProj);
    vout.Color = vin.Color;
    return vout;
}
```

**重要规则**：如果没有几何着色器，顶点着色器必须使用 `SV_POSITION` 语义输出齐次裁剪空间的位置。如果有几何着色器，则可以将这个责任推迟到几何着色器。

顶点着色器（或几何着色器）**不做透视除法**，它只负责投影矩阵乘法部分。透视除法由硬件在后续阶段自动完成。

### 6.4.3 输入布局与输入签名的链接验证

输入布局描述（`D3D12_INPUT_ELEMENT_DESC` 数组）与顶点着色器的输入签名之间存在链接关系。如果传入的顶点数据没有提供着色器期望的所有输入，会产生错误。

例如以下情况是**不兼容的**：

```cpp
// C++ 端：顶点只有位置和颜色
struct Vertex { XMFLOAT3 Pos; XMFLOAT4 Color; };
D3D12_INPUT_ELEMENT_DESC desc[] = {
    {"POSITION", 0, DXGI_FORMAT_R32G32B32_FLOAT,    0, 0, D3D12_INPUT_PER_VERTEX_DATA, 0},
    {"COLOR",    0, DXGI_FORMAT_R32G32B32A32_FLOAT, 0, 12, D3D12_INPUT_PER_VERTEX_DATA, 0}
};

// HLSL 端：着色器期望位置、颜色、法线
struct VertexIn {
    float3 PosL   : POSITION;
    float4 Color  : COLOR;
    float3 Normal : NORMAL;  // <-- 缺少这个输入！
};
```

但顶点数据**可以提供着色器不需要的额外数据**，这是允许的：

```cpp
// C++ 端：顶点有位置、颜色、法线
struct Vertex { XMFLOAT3 Pos; XMFLOAT4 Color; XMFLOAT3 Normal; };
D3D12_INPUT_ELEMENT_DESC desc[] = {
    {"POSITION", 0, DXGI_FORMAT_R32G32B32_FLOAT,    0, 0,  D3D12_INPUT_PER_VERTEX_DATA, 0},
    {"COLOR",    0, DXGI_FORMAT_R32G32B32A32_FLOAT, 0, 12, D3D12_INPUT_PER_VERTEX_DATA, 0},
    {"NORMAL",   0, DXGI_FORMAT_R32G32B32_FLOAT,    0, 28, D3D12_INPUT_PER_VERTEX_DATA, 0}
};

// HLSL 端：着色器只使用位置和颜色
struct VertexIn {
    float3 PosL  : POSITION;
    float4 Color : COLOR;
    // 没有 NORMAL，但没问题，多余的顶点数据会被忽略
};
```

输入布局描述和顶点着色器在创建 PSO 时会进行兼容性验证。

---

## 6.5 像素着色器示例

光栅化阶段会对顶点着色器输出的顶点属性进行插值，将插值后的结果作为输入传给像素着色器（回顾 5.10.3 节）。

像素着色器也是逐片元执行的函数，其任务是根据输入计算该片元的颜色值。注意片元不一定能最终写入后台缓冲区——它可能被裁剪、被深度测试剔除、或被模板测试丢弃。因此一个像素位置可能有多个片元候选者。

```hlsl
cbuffer cbPerObject : register(b0) {
    float4x4 gWorldViewProj;
};

struct VertexOut {
    float4 PosH  : SV_POSITION;
    float4 Color : COLOR;
};

float4 PS(VertexOut pin) : SV_Target {
    return pin.Color;
}
```

| 特性 | 说明 |
|------|------|
| `SV_Target` | 返回值语义，表示该返回值应写入渲染目标。`SV_Target0` 的简写 |
| 输入匹配 | 像素着色器的输入必须**精确匹配**顶点着色器的输出（语义和类型都要对应） |
| `clip()` | HLSL 提供的 `clip` 函数可以丢弃片元。如果在像素着色器中调用，会禁用 early-z 优化 |

**Early-Z 优化**：作为一种硬件优化，片元可能在进入像素着色器之前就被深度测试剔除。但如果像素着色器修改了深度值，则必须执行像素着色器，因为修改后的深度事先未知。

---

## 6.6 常量缓冲区

常量缓冲区是一种 GPU 资源（`ID3D12Resource`），其数据可以被着色器程序引用。例如前面的顶点着色器中的 `cbuffer`：

```hlsl
cbuffer cbPerObject : register(b0) {
    float4x4 gWorldViewProj;
};
```

这声明了一个名为 `cbPerObject` 的常量缓冲区，存储世界-视图-投影组合矩阵。在 HLSL 中，`float4x4` 表示 4×4 矩阵，类似地还有 `float3x4`、`float2x2` 等。

### 6.6.1 创建常量缓冲区

与顶点/索引缓冲区不同，常量缓冲区通常需要**每帧由 CPU 更新**（例如相机移动时需要更新视图矩阵）。因此常量缓冲区应创建在**上传堆（`D3D12_HEAP_TYPE_UPLOAD`）**中，而不是默认堆。

常量缓冲区有一个特殊的硬件要求：**大小必须是 256 字节的整数倍**。

**`d3dUtil::CalcConstantBufferByteSize` 辅助函数：**

```cpp
UINT d3dUtil::CalcConstantBufferByteSize(UINT byteSize) {
    // 常量缓冲区必须是 256 的整数倍
    // 通过加 255 再屏蔽低 8 位实现向上取整到最近的 256 倍数
    return (byteSize + 255) & ~255;
}
```

**示例：创建容纳 `NumElements` 个常量缓冲区的上传缓冲区**

```cpp
struct ObjectConstants {
    DirectX::XMFLOAT4X4 WorldViewProj = MathHelper::Identity4x4();
};

UINT elementByteSize = d3dUtil::CalcConstantBufferByteSize(sizeof(ObjectConstants));

ComPtr<ID3D12Resource> mUploadCBuffer;
device->CreateCommittedResource(
    &CD3DX12_HEAP_PROPERTIES(D3D12_HEAP_TYPE_UPLOAD),
    D3D12_HEAP_FLAG_NONE,
    &CD3DX12_RESOURCE_DESC::Buffer(elementByteSize * NumElements),
    D3D12_RESOURCE_STATE_GENERIC_READ,
    nullptr,
    IID_PPV_ARGS(&mUploadCBuffer)
);
```

HLSL 中的 `cbuffer` 结构会被**隐式填充**到 256 字节边界，因此不需要手动在 HLSL 中添加填充字段。

Direct3D 12 还引入了 Shader Model 5.1 的替代语法：

```hlsl
struct ObjectConstants {
    float4x4 gWorldViewProj;
    uint matIndex;
};
ConstantBuffer<ObjectConstants> gObjConstants : register(b0);

// 访问方式：
uint index = gObjConstants.matIndex;
```

### 6.6.2 更新常量缓冲区

由于常量缓冲区在上传堆中创建，CPU 可以直接更新其内容。

**`ID3D12Resource::Map` 方法：**

```cpp
HRESULT ID3D12Resource::Map(
    UINT  Subresource,
    const D3D12_RANGE* pReadRange,
    void** ppData
);
```

| 参数 | 说明 |
|------|------|
| `Subresource` | 要映射的子资源索引。对于缓冲区只有 1 个子资源，因此设为 `0` |
| `pReadRange` | 指向 `D3D12_RANGE` 的指针，描述要映射的内存范围。设为 `nullptr` 表示映射整个资源 |
| `ppData` | 输出参数，返回指向映射数据的指针 |

**`ID3D12Resource::Unmap` 方法：**

```cpp
void ID3D12Resource::Unmap(
    UINT  Subresource,
    const D3D12_RANGE* pWrittenRange
);
```

**示例：映射、更新、解除映射**

```cpp
ComPtr<ID3D12Resource> mUploadBuffer;
BYTE* mMappedData = nullptr;

// 映射资源
mUploadBuffer->Map(0, nullptr, reinterpret_cast<void**>(&mMappedData));

// 复制数据
memcpy(mMappedData, &data, dataSizeInBytes);

// 解除映射
if (mUploadBuffer != nullptr) {
    mUploadBuffer->Unmap(0, nullptr);
    mMappedData = nullptr;
}
```

### 6.6.3 UploadBuffer 辅助类

为了方便管理上传缓冲区，可以封装一个模板类 `UploadBuffer`：

```cpp
template<typename T>
class UploadBuffer {
public:
    UploadBuffer(ID3D12Device* device, UINT elementCount, bool isConstantBuffer);
    UploadBuffer(const UploadBuffer& rhs) = delete;
    UploadBuffer& operator=(const UploadBuffer& rhs) = delete;
    ~UploadBuffer();

    ID3D12Resource* Resource() const;
    void CopyData(int elementIndex, const T& data);

private:
    Microsoft::WRL::ComPtr<ID3D12Resource> mUploadBuffer;
    BYTE* mMappedData = nullptr;
    UINT mElementByteSize = 0;
    bool mIsConstantBuffer = false;
};
```

| 模板参数/成员 | 说明 |
|------|------|
| `T` | 缓冲区中每个元素的数据类型 |
| `elementCount` | 元素数量 |
| `isConstantBuffer` | 如果为 `true`，会自动将每个元素的大小向上取整到 256 的倍数 |
| `CopyData(index, data)` | 将数据复制到第 `index` 个元素的位置 |

构造函数内部会创建上传堆资源并立即调用 `Map`，析构函数会自动调用 `Unmap`。使用示例：

```cpp
// 创建容纳 1 个 ObjectConstants 的常量缓冲区
std::unique_ptr<UploadBuffer<ObjectConstants>> mObjectCB =
    std::make_unique<UploadBuffer<ObjectConstants>>(md3dDevice.Get(), 1, true);

// 每帧更新数据
ObjectConstants objConstants;
XMStoreFloat4x4(&objConstants.WorldViewProj, XMMatrixTranspose(worldViewProj));
mObjectCB->CopyData(0, objConstants);
```

**重要注意事项**：虽然映射后不需要立即解除映射，但**不能在 GPU 正在使用该资源时向其写入数据**，因此需要配合同步机制。

### 6.6.4 常量缓冲区描述符

回忆 4.1.6 节，资源通过**描述符（Descriptor）**绑定到渲染管线。常量缓冲区需要**常量缓冲区视图（CBV）**，存储在 `D3D12_DESCRIPTOR_HEAP_TYPE_CBV_SRV_UAV` 类型的描述符堆中。

**创建 CBV/SRV/UAV 描述符堆：**

```cpp
D3D12_DESCRIPTOR_HEAP_DESC cbvHeapDesc;
cbvHeapDesc.NumDescriptors = 1;
cbvHeapDesc.Type           = D3D12_DESCRIPTOR_HEAP_TYPE_CBV_SRV_UAV;
cbvHeapDesc.Flags          = D3D12_DESCRIPTOR_HEAP_FLAG_SHADER_VISIBLE;
cbvHeapDesc.NodeMask       = 0;

ComPtr<ID3D12DescriptorHeap> mCbvHeap;
md3dDevice->CreateDescriptorHeap(&cbvHeapDesc, IID_PPV_ARGS(&mCbvHeap));
```

注意这里的 `D3D12_DESCRIPTOR_HEAP_FLAG_SHADER_VISIBLE` 标志，表示该堆中的描述符可以被着色器程序访问。

**`D3D12_CONSTANT_BUFFER_VIEW_DESC` 结构体：**

```cpp
typedef struct D3D12_CONSTANT_BUFFER_VIEW_DESC {
    D3D12_GPU_VIRTUAL_ADDRESS BufferLocation;
    UINT                      SizeInBytes;
} D3D12_CONSTANT_BUFFER_VIEW_DESC;
```

| 成员 | 说明 |
|------|------|
| `BufferLocation` | 常量缓冲区资源的 GPU 虚拟地址 |
| `SizeInBytes` | 视图大小，必须是 **256 的整数倍** |

**`ID3D12Device::CreateConstantBufferView` 方法：**

```cpp
void ID3D12Device::CreateConstantBufferView(
    const D3D12_CONSTANT_BUFFER_VIEW_DESC* pDesc,
    D3D12_CPU_DESCRIPTOR_HANDLE          DestDescriptor
);
```

**创建 CBV 的完整示例：**

```cpp
// 获取缓冲区起始地址
D3D12_GPU_VIRTUAL_ADDRESS cbAddress = mObjectCB->Resource()->GetGPUVirtualAddress();

// 偏移到第 i 个对象的常量缓冲区
int boxCBufIndex = 0;
cbAddress += boxCBufIndex * objCBByteSize;

D3D12_CONSTANT_BUFFER_VIEW_DESC cbvDesc;
cbvDesc.BufferLocation = cbAddress;
cbvDesc.SizeInBytes    = d3dUtil::CalcConstantBufferByteSize(sizeof(ObjectConstants));

// 在描述符堆的第一个位置创建 CBV
md3dDevice->CreateConstantBufferView(
    &cbvDesc,
    mCbvHeap->GetCPUDescriptorHandleForHeapStart()
);
```

如果 `SizeInBytes` 或偏移量不是 256 的倍数，调试层会报错：

```
D3D12 ERROR: SizeInBytes of 64 is invalid. Device requires SizeInBytes be a multiple of 256.
```

### 6.6.5 根签名和描述符表

不同的着色器程序通常需要不同的资源输入。根签名（Root Signature）定义了着色器程序期望的资源以及这些资源映射到哪个着色器寄存器。

如果把着色器程序看作函数，输入资源看作函数参数，那么根签名就是**函数签名**。

**根签名相关接口和结构：**

```cpp
// 描述符范围：描述符堆中的一段连续描述符
typedef struct D3D12_DESCRIPTOR_RANGE {
    D3D12_DESCRIPTOR_RANGE_TYPE RangeType;
    UINT                        NumDescriptors;
    UINT                        BaseShaderRegister;
    UINT                        RegisterSpace;
    UINT                        OffsetInDescriptorsFromTableStart;
} D3D12_DESCRIPTOR_RANGE;
```

| 成员 | 说明 |
|------|------|
| `RangeType` | 描述符范围类型：`D3D12_DESCRIPTOR_RANGE_TYPE_CBV`、`SRV`、`UAV`、`SAMPLER` |
| `NumDescriptors` | 该范围中包含的描述符数量 |
| `BaseShaderRegister` | 绑定的基础着色器寄存器编号（例如 `0` 对应 `register(b0)`） |
| `RegisterSpace` | 寄存器空间，通常设为 `0` |
| `OffsetInDescriptorsFromTableStart` | 从表起始位置的偏移，通常设为 `D3D12_DESCRIPTOR_RANGE_OFFSET_APPEND` |

**`D3D12_DESCRIPTOR_RANGE_TYPE` 枚举：**

```cpp
typedef enum D3D12_DESCRIPTOR_RANGE_TYPE {
    D3D12_DESCRIPTOR_RANGE_TYPE_SRV     = 0,  // 着色器资源视图
    D3D12_DESCRIPTOR_RANGE_TYPE_UAV     = 1,  // 无序访问视图
    D3D12_DESCRIPTOR_RANGE_TYPE_CBV     = 2,  // 常量缓冲区视图
    D3D12_DESCRIPTOR_RANGE_TYPE_SAMPLER = 3   // 采样器
} D3D12_DESCRIPTOR_RANGE_TYPE;
```

**创建根签名的示例代码（单个 CBV 描述符表）：**

```cpp
// 1. 定义根参数（描述符表）
CD3DX12_ROOT_PARAMETER slotRootParameter[1];

CD3DX12_DESCRIPTOR_RANGE cbvTable;
cbvTable.Init(
    D3D12_DESCRIPTOR_RANGE_TYPE_CBV,  // 类型：CBV
    1,                                // 描述符数量
    0                                 // 绑定到 register(b0)
);

slotRootParameter[0].InitAsDescriptorTable(1, &cbvTable);

// 2. 定义根签名描述
CD3DX12_ROOT_SIGNATURE_DESC rootSigDesc(
    1, slotRootParameter,
    0, nullptr,
    D3D12_ROOT_SIGNATURE_FLAG_ALLOW_INPUT_ASSEMBLER_INPUT_LAYOUT
);

// 3. 序列化根签名
ComPtr<ID3DBlob> serializedRootSig = nullptr;
ComPtr<ID3DBlob> errorBlob = nullptr;
HRESULT hr = D3D12SerializeRootSignature(
    &rootSigDesc,
    D3D_ROOT_SIGNATURE_VERSION_1,
    serializedRootSig.GetAddressOf(),
    errorBlob.GetAddressOf()
);

if (errorBlob != nullptr) {
    ::OutputDebugStringA((char*)errorBlob->GetBufferPointer());
}
ThrowIfFailed(hr);

// 4. 创建根签名
ThrowIfFailed(md3dDevice->CreateRootSignature(
    0,
    serializedRootSig->GetBufferPointer(),
    serializedRootSig->GetBufferSize(),
    IID_PPV_ARGS(&mRootSignature)
));
```

**`ID3D12GraphicsCommandList::SetGraphicsRootDescriptorTable` 方法：**

```cpp
void ID3D12GraphicsCommandList::SetGraphicsRootDescriptorTable(
    UINT                        RootParameterIndex,
    D3D12_GPU_DESCRIPTOR_HANDLE BaseDescriptor
);
```

| 参数 | 说明 |
|------|------|
| `RootParameterIndex` | 要设置的根参数索引 |
| `BaseDescriptor` | 指向描述符堆中描述符表的 GPU 句柄 |

**绑定根签名和描述符表的完整流程：**

```cpp
// 设置根签名
mCommandList->SetGraphicsRootSignature(mRootSignature.Get());

// 设置描述符堆
ID3D12DescriptorHeap* descriptorHeaps[] = { mCbvHeap.Get() };
mCommandList->SetDescriptorHeaps(_countof(descriptorHeaps), descriptorHeaps);

// 设置描述符表（绑定到根参数 0）
mCommandList->SetGraphicsRootDescriptorTable(
    0,
    mCbvHeap->GetGPUDescriptorHandleForHeapStart()
);
```

**性能提示**：
- 根签名应尽量保持**最小化**。
- 尽量减少每帧修改根签名的次数。
- 切换根签名后，**所有现有绑定都会丢失**，需要重新绑定新根签名期望的所有资源。

---

## 6.7 编译着色器

着色器代码可以在**运行时编译**，也可以在**离线状态编译**。

### 6.7.1 运行时编译

**`D3DCompileFromFile` 函数：**

```cpp
HRESULT D3DCompileFromFile(
    LPCWSTR                pFileName,
    const D3D_SHADER_MACRO* pDefines,
    ID3DInclude*           pInclude,
    LPCSTR                 pEntrypoint,
    LPCSTR                 pTarget,
    UINT                   Flags1,
    UINT                   Flags2,
    ID3DBlob**             ppCode,
    ID3DBlob**             ppErrorMsgs
);
```

| 参数 | 说明 |
|------|------|
| `pFileName` | HLSL 源文件的宽字符路径 |
| `pDefines` | 可选的着色器宏定义数组，用于条件编译。不需要时设为 `nullptr` |
| `pInclude` | 可选的包含接口。使用 `D3D_COMPILE_STANDARD_FILE_INCLUDE` 表示标准文件包含 |
| `pEntrypoint` | 着色器入口函数名称，例如 `"VS"` 或 `"PS"` |
| `pTarget` | 编译目标版本字符串，例如 `"vs_5_0"`（顶点着色器 Shader Model 5.0）或 `"ps_5_0"` |
| `Flags1` | 编译标志。常用：`D3DCOMPILE_DEBUG`（启用调试信息）、`D3DCOMPILE_SKIP_OPTIMIZATION`（跳过优化） |
| `Flags2` | 次要编译标志，通常设为 `0` |
| `ppCode` | 输出参数，返回包含编译后字节码的 `ID3DBlob` |
| `ppErrorMsgs` | 输出参数，返回编译错误/警告信息。如果没有错误，可能为 `nullptr` |

**常用编译标志（`Flags1`）：**

| 标志 | 说明 |
|------|------|
| `D3DCOMPILE_DEBUG` | 插入调试信息 |
| `D3DCOMPILE_SKIP_OPTIMIZATION` | 跳过优化，保留更多调试信息 |
| `D3DCOMPILE_PACK_MATRIX_ROW_MAJOR` | 矩阵按行主序打包 |
| `D3DCOMPILE_PACK_MATRIX_COLUMN_MAJOR` | 矩阵按列主序打包（默认） |
| `D3DCOMPILE_ENABLE_STRICTNESS` | 强制严格遵循 Shader Model 规范 |

**`ID3DBlob` 接口：**

```cpp
interface ID3DBlob : IUnknown {
    LPVOID GetBufferPointer();  // 返回数据指针
    SIZE_T GetBufferSize();     // 返回数据字节大小
};
```

**运行时编译的辅助函数示例：**

```cpp
ComPtr<ID3DBlob> d3dUtil::CompileShader(
    const std::wstring& filename,
    const D3D_SHADER_MACRO* defines,
    const std::string& entrypoint,
    const std::string& target)
{
    UINT compileFlags = 0;
#if defined(DEBUG) || defined(_DEBUG)
    compileFlags = D3DCOMPILE_DEBUG | D3DCOMPILE_SKIP_OPTIMIZATION;
#endif

    ComPtr<ID3DBlob> byteCode = nullptr;
    ComPtr<ID3DBlob> errors;

    HRESULT hr = D3DCompileFromFile(
        filename.c_str(),
        defines,
        D3D_COMPILE_STANDARD_FILE_INCLUDE,
        entrypoint.c_str(),
        target.c_str(),
        compileFlags,
        0,
        &byteCode,
        &errors
    );

    if (errors != nullptr) {
        OutputDebugStringA((char*)errors->GetBufferPointer());
    }
    ThrowIfFailed(hr);
    return byteCode;
}
```

### 6.7.2 离线编译

运行时编译之外，还可以使用 FXC 工具在构建时离线编译着色器。离线编译的优点：

1. 复杂着色器的编译可能耗时较长，离线编译可以**缩短加载时间**。
2. 在构建阶段就能看到编译错误，比运行时更方便。
3. Windows 8 Store 应用**必须使用**离线编译。

编译后的着色器通常使用 `.cso`（Compiled Shader Object）扩展名。

**FXC 命令行示例（Debug）：**

```bash
fxc "color.hlsl" /Od /Zi /T vs_5_0 /E "VS" /Fo "color_vs.cso" /Fc "color_vs.asm"
fxc "color.hlsl" /Od /Zi /T ps_5_0 /E "PS" /Fo "color_ps.cso" /Fc "color_ps.asm"
```

**FXC 命令行示例（Release）：**

```bash
fxc "color.hlsl" /T vs_5_0 /E "VS" /Fo "color_vs.cso" /Fc "color_vs.asm"
fxc "color.hlsl" /T ps_5_0 /E "PS" /Fo "color_ps.cso" /Fc "color_ps.asm"
```

| 参数 | 说明 |
|------|------|
| `/Od` | 禁用优化 |
| `/Zi` | 启用调试信息 |
| `/T` | 指定目标配置文件，如 `vs_5_0` |
| `/E` | 指定入口函数名 |
| `/Fo` | 指定输出文件（编译后的字节码） |
| `/Fc` | 指定汇编输出文件 |

**加载已编译的 .cso 文件：**

```cpp
ComPtr<ID3DBlob> d3dUtil::LoadBinary(const std::wstring& filename) {
    std::ifstream fin(filename, std::ios::binary);
    fin.seekg(0, std::ios_base::end);
    std::ifstream::pos_type size = (int)fin.tellg();
    fin.seekg(0, std::ios_base::beg);

    ComPtr<ID3DBlob> blob;
    ThrowIfFailed(D3DCreateBlob(size, blob.GetAddressOf()));
    fin.read((char*)blob->GetBufferPointer(), size);
    fin.close();

    return blob;
}

// 使用：
ComPtr<ID3DBlob> mvsByteCode = d3dUtil::LoadBinary(L"Shaders\\color_vs.cso");
ComPtr<ID3DBlob> mpsByteCode = d3dUtil::LoadBinary(L"Shaders\\color_ps.cso");
```

### 6.7.3 使用 Visual Studio 编译着色器

Visual Studio 可以识别项目中的 `.hlsl` 文件，并提供集成编译选项（HLSL 编译器属性页）。但是 VS 的集成支持有两个局限：

1. 每个 `.hlsl` 文件**只能包含一个着色器程序**（无法同时存放顶点和像素着色器）。
2. 无法对同一个着色器使用不同的预处理器指令编译出多个变体。

因此本书示例通常使用 FXC 命令行或运行时编译，而非 VS 集成编译。

---

## 6.8 光栅化状态

渲染管线中有些部分是可编程的，有些部分只能配置。光栅化状态组由 `D3D12_RASTERIZER_DESC` 结构体描述，用于配置光栅化阶段。

**`D3D12_RASTERIZER_DESC` 结构体：**

```cpp
typedef struct D3D12_RASTERIZER_DESC {
    D3D12_FILL_MODE                       FillMode;
    D3D12_CULL_MODE                       CullMode;
    BOOL                                  FrontCounterClockwise;
    INT                                   DepthBias;
    FLOAT                                 DepthBiasClamp;
    FLOAT                                 SlopeScaledDepthBias;
    BOOL                                  DepthClipEnable;
    BOOL                                  ScissorEnable;
    BOOL                                  MultisampleEnable;
    BOOL                                  AntialiasedLineEnable;
    UINT                                  ForcedSampleCount;
    D3D12_CONSERVATIVE_RASTERIZATION_MODE ConservativeRaster;
} D3D12_RASTERIZER_DESC;
```

| 成员 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `FillMode` | `D3D12_FILL_MODE` | `D3D12_FILL_SOLID` | 填充模式：`SOLID`（实心）或 `WIREFRAME`（线框） |
| `CullMode` | `D3D12_CULL_MODE` | `D3D12_CULL_BACK` | 剔除模式：`NONE`（不剔除）、`BACK`（剔除背面）、`FRONT`（剔除正面） |
| `FrontCounterClockwise` | `BOOL` | `FALSE` | 为 `FALSE` 时，顺时针三角形为正面；为 `TRUE` 时，逆时针为正面 |
| `DepthBias` | `INT` | `0` | 深度偏移值，用于阴影映射等 |
| `DepthBiasClamp` | `FLOAT` | `0.0f` | 最大深度偏移钳制值 |
| `SlopeScaledDepthBias` | `FLOAT` | `0.0f` | 斜率缩放深度偏移 |
| `DepthClipEnable` | `BOOL` | `TRUE` | 是否启用深度裁剪（近平面/远平面裁剪） |
| `ScissorEnable` | `BOOL` | `FALSE` | 是否启用裁剪矩形测试 |
| `MultisampleEnable` | `BOOL` | `FALSE` | 是否启用多重采样抗锯齿 |
| `AntialiasedLineEnable` | `BOOL` | `FALSE` | 是否启用线条反锯齿 |
| `ForcedSampleCount` | `UINT` | `0` | 强制采样数量 |
| `ConservativeRaster` | `D3D12_CONSERVATIVE_RASTERIZATION_MODE` | `OFF` | 保守光栅化模式 |

**`D3D12_FILL_MODE` 枚举：**

```cpp
typedef enum D3D12_FILL_MODE {
    D3D12_FILL_WIREFRAME = 2,
    D3D12_FILL_SOLID     = 3
} D3D12_FILL_MODE;
```

**`D3D12_CULL_MODE` 枚举：**

```cpp
typedef enum D3D12_CULL_MODE {
    D3D12_CULL_NONE = 1,
    D3D12_CULL_FRONT = 2,
    D3D12_CULL_BACK  = 3
} D3D12_CULL_MODE;
```

**示例：创建线框模式并禁用背面剔除的光栅化状态：**

```cpp
CD3DX12_RASTERIZER_DESC rsDesc(D3D12_DEFAULT);
rsDesc.FillMode = D3D12_FILL_WIREFRAME;
rsDesc.CullMode = D3D12_CULL_NONE;
```

`CD3DX12_RASTERIZER_DESC` 是辅助类，构造函数 `CD3DX12_RASTERIZER_DESC(D3D12_DEFAULT)` 将所有成员初始化为默认值。

---

## 6.9 管线状态对象

前面已经介绍了输入布局描述、顶点/像素着色器、光栅化状态的配置方法，但还没有说明如何将这些对象真正绑定到图形管线。Direct3D 12 将大部分管线状态封装在一个称为**管线状态对象（Pipeline State Object, PSO）**的聚合结构中，由 `ID3D12PipelineState` 接口表示。

**`D3D12_GRAPHICS_PIPELINE_STATE_DESC` 结构体：**

```cpp
typedef struct D3D12_GRAPHICS_PIPELINE_STATE_DESC {
    ID3D12RootSignature*         pRootSignature;
    D3D12_SHADER_BYTECODE        VS;
    D3D12_SHADER_BYTECODE        PS;
    D3D12_SHADER_BYTECODE        DS;
    D3D12_SHADER_BYTECODE        HS;
    D3D12_SHADER_BYTECODE        GS;
    D3D12_STREAM_OUTPUT_DESC     StreamOutput;
    D3D12_BLEND_DESC             BlendState;
    UINT                         SampleMask;
    D3D12_RASTERIZER_DESC        RasterizerState;
    D3D12_DEPTH_STENCIL_DESC     DepthStencilState;
    D3D12_INPUT_LAYOUT_DESC      InputLayout;
    D3D12_PRIMITIVE_TOPOLOGY_TYPE PrimitiveTopologyType;
    UINT                         NumRenderTargets;
    DXGI_FORMAT                  RTVFormats[8];
    DXGI_FORMAT                  DSVFormat;
    DXGI_SAMPLE_DESC             SampleDesc;
    UINT                         NodeMask;
    D3D12_CACHED_PIPELINE_STATE  CachedPSO;
    D3D12_PIPELINE_STATE_FLAGS   Flags;
} D3D12_GRAPHICS_PIPELINE_STATE_DESC;
```

| 成员 | 说明 |
|------|------|
| `pRootSignature` | 与此 PSO 绑定的根签名指针。根签名必须与 PSO 中指定的着色器兼容 |
| `VS` | 顶点着色器字节码，由 `D3D12_SHADER_BYTECODE` 描述 |
| `PS` | 像素着色器字节码 |
| `DS` | 域着色器（Tessellation 相关，后续章节介绍） |
| `HS` | 外壳着色器（Tessellation 相关，后续章节介绍） |
| `GS` | 几何着色器（后续章节介绍） |
| `StreamOutput` | 流输出配置，目前可以清零 |
| `BlendState` | 混合状态。暂时使用 `CD3DX12_BLEND_DESC(D3D12_DEFAULT)` |
| `SampleMask` | 32 位采样掩码，默认 `0xffffffff` 不禁用任何采样 |
| `RasterizerState` | 光栅化状态配置 |
| `DepthStencilState` | 深度/模板状态。暂时使用 `CD3DX12_DEPTH_STENCIL_DESC(D3D12_DEFAULT)` |
| `InputLayout` | 输入布局描述，包含 `D3D12_INPUT_ELEMENT_DESC` 数组及其元素数量 |
| `PrimitiveTopologyType` | 图元拓扑类型，不是具体的拓扑，而是类型分类 |
| `NumRenderTargets` | 同时使用的渲染目标数量 |
| `RTVFormats[8]` | 渲染目标格式数组，支持多渲染目标（MRT） |
| `DSVFormat` | 深度/模板缓冲区格式 |
| `SampleDesc` | 多重采样描述，需与渲染目标设置匹配 |

**`D3D12_SHADER_BYTECODE` 结构体：**

```cpp
typedef struct D3D12_SHADER_BYTECODE {
    const BYTE* pShaderBytecode;
    SIZE_T      BytecodeLength;
} D3D12_SHADER_BYTECODE;
```

**`D3D12_INPUT_LAYOUT_DESC` 结构体：**

```cpp
typedef struct D3D12_INPUT_LAYOUT_DESC {
    const D3D12_INPUT_ELEMENT_DESC* pInputElementDescs;
    UINT                            NumElements;
} D3D12_INPUT_LAYOUT_DESC;
```

**`D3D12_PRIMITIVE_TOPOLOGY_TYPE` 枚举：**

```cpp
typedef enum D3D12_PRIMITIVE_TOPOLOGY_TYPE {
    D3D12_PRIMITIVE_TOPOLOGY_TYPE_UNDEFINED = 0,
    D3D12_PRIMITIVE_TOPOLOGY_TYPE_POINT     = 1,
    D3D12_PRIMITIVE_TOPOLOGY_TYPE_LINE      = 2,
    D3D12_PRIMITIVE_TOPOLOGY_TYPE_TRIANGLE  = 3,
    D3D12_PRIMITIVE_TOPOLOGY_TYPE_PATCH     = 4
} D3D12_PRIMITIVE_TOPOLOGY_TYPE;
```

注意 `PrimitiveTopologyType` 是**拓扑类型**（分类），而 `IASetPrimitiveTopology` 设置的是**具体拓扑**（如 `TRIANGLELIST`、`TRIANGLESTRIP` 等）。两者的区别：

| 设置位置 | 类型 | 示例值 |
|----------|------|--------|
| PSO 中 `PrimitiveTopologyType` | `D3D12_PRIMITIVE_TOPOLOGY_TYPE_*` | `TRIANGLE` |
| 命令列表 `IASetPrimitiveTopology` | `D3D_PRIMITIVE_TOPOLOGY_*` | `TRIANGLELIST` |

**`ID3D12Device::CreateGraphicsPipelineState` 方法：**

```cpp
HRESULT ID3D12Device::CreateGraphicsPipelineState(
    const D3D12_GRAPHICS_PIPELINE_STATE_DESC* pDesc,
    REFIID                                    riid,
    void**                                    ppPipelineState
);
```

**创建 PSO 的完整示例：**

```cpp
D3D12_GRAPHICS_PIPELINE_STATE_DESC psoDesc;
ZeroMemory(&psoDesc, sizeof(D3D12_GRAPHICS_PIPELINE_STATE_DESC));

psoDesc.InputLayout           = { mInputLayout.data(), (UINT)mInputLayout.size() };
psoDesc.pRootSignature        = mRootSignature.Get();
psoDesc.VS                    = { reinterpret_cast<BYTE*>(mvsByteCode->GetBufferPointer()), mvsByteCode->GetBufferSize() };
psoDesc.PS                    = { reinterpret_cast<BYTE*>(mpsByteCode->GetBufferPointer()), mpsByteCode->GetBufferSize() };
psoDesc.RasterizerState       = CD3DX12_RASTERIZER_DESC(D3D12_DEFAULT);
psoDesc.BlendState            = CD3DX12_BLEND_DESC(D3D12_DEFAULT);
psoDesc.DepthStencilState     = CD3DX12_DEPTH_STENCIL_DESC(D3D12_DEFAULT);
psoDesc.SampleMask            = UINT_MAX;
psoDesc.PrimitiveTopologyType = D3D12_PRIMITIVE_TOPOLOGY_TYPE_TRIANGLE;
psoDesc.NumRenderTargets      = 1;
psoDesc.RTVFormats[0]         = mBackBufferFormat;
psoDesc.SampleDesc.Count      = m4xMsaaState ? 4 : 1;
psoDesc.SampleDesc.Quality    = m4xMsaaState ? (m4xMsaaQuality - 1) : 0;
psoDesc.DSVFormat             = mDepthStencilFormat;

ComPtr<ID3D12PipelineState> mPSO;
ThrowIfFailed(md3dDevice->CreateGraphicsPipelineState(&psoDesc, IID_PPV_ARGS(&mPSO)));
```

### 6.9.1 PSO 设计原理

Direct3D 12 将管线状态聚合到 PSO 中是为了**性能**。在 D3D11 中，这些渲染状态是单独设置的，但状态之间存在关联性——修改一个状态可能导致驱动需要重新编程硬件的其他相关状态。驱动通常会延迟到绘制调用时才真正设置硬件状态，但这需要运行时的额外簿记开销。

在 D3D12 中，由于所有状态作为聚合体一次性指定，驱动可以在**初始化时**就生成所有硬件状态编程代码，因为整个管线状态已知且已验证兼容。

**PSO 使用注意事项：**
- PSO 应在**初始化阶段**创建，因为验证和创建可能耗时较长。
- 按需运行时创建也可以（首次引用时创建并缓存到哈希表中）。
- 不是所有状态都在 PSO 中。视口（Viewport）和裁剪矩形（Scissor Rect）是独立设置的，因为它们可以高效地单独变更。
- Direct3D 是状态机：PSO 绑定后会一直保持，直到被显式覆盖（或命令列表被重置）。
- **尽量减少 PSO 切换**：将使用相同 PSO 的对象放在一起绘制，不要每个绘制调用都切换 PSO。

**切换 PSO 的代码模式：**

```cpp
// Reset 时可指定初始 PSO
mCommandList->Reset(mDirectCmdListAlloc.Get(), mPSO1.Get());
/* ... 绘制使用 PSO1 的对象 ... */

// 切换 PSO
mCommandList->SetPipelineState(mPSO2.Get());
/* ... 绘制使用 PSO2 的对象 ... */

// 再切换
mCommandList->SetPipelineState(mPSO3.Get());
/* ... 绘制使用 PSO3 的对象 ... */
```

---

## 6.10 几何辅助结构

将顶点缓冲区和索引缓冲区组合在一起，并附带相关元数据，对于管理几何数据非常有帮助。`MeshGeometry` 结构体定义在 `d3dUtil.h` 中，在本书中广泛使用。

```cpp
// 定义 MeshGeometry 中某个子几何体的范围
struct SubmeshGeometry {
    UINT IndexCount         = 0;     // 索引数量
    UINT StartIndexLocation = 0;     // 起始索引位置
    INT  BaseVertexLocation = 0;     // 基础顶点偏移
    DirectX::BoundingBox Bounds;     // 包围盒（后续章节使用）
};

struct MeshGeometry {
    std::string Name;

    // CPU 内存副本（使用 Blob 存储，因为格式可能是通用的）
    Microsoft::WRL::ComPtr<ID3DBlob> VertexBufferCPU = nullptr;
    Microsoft::WRL::ComPtr<ID3DBlob> IndexBufferCPU  = nullptr;

    // GPU 资源
    Microsoft::WRL::ComPtr<ID3D12Resource> VertexBufferGPU = nullptr;
    Microsoft::WRL::ComPtr<ID3D12Resource> IndexBufferGPU  = nullptr;

    // 上传缓冲区（初始化完成后可以释放）
    Microsoft::WRL::ComPtr<ID3D12Resource> VertexBufferUploader = nullptr;
    Microsoft::WRL::ComPtr<ID3D12Resource> IndexBufferUploader  = nullptr;

    // 缓冲区元数据
    UINT        VertexByteStride     = 0;
    UINT        VertexBufferByteSize = 0;
    DXGI_FORMAT IndexFormat          = DXGI_FORMAT_R16_UINT;
    UINT        IndexBufferByteSize  = 0;

    // 一个 MeshGeometry 可以存储多个子网格
    std::unordered_map<std::string, SubmeshGeometry> DrawArgs;

    // 返回顶点缓冲区视图
    D3D12_VERTEX_BUFFER_VIEW VertexBufferView() const {
        D3D12_VERTEX_BUFFER_VIEW vbv;
        vbv.BufferLocation = VertexBufferGPU->GetGPUVirtualAddress();
        vbv.StrideInBytes  = VertexByteStride;
        vbv.SizeInBytes    = VertexBufferByteSize;
        return vbv;
    }

    // 返回索引缓冲区视图
    D3D12_INDEX_BUFFER_VIEW IndexBufferView() const {
        D3D12_INDEX_BUFFER_VIEW ibv;
        ibv.BufferLocation = IndexBufferGPU->GetGPUVirtualAddress();
        ibv.Format         = IndexFormat;
        ibv.SizeInBytes    = IndexBufferByteSize;
        return ibv;
    }

    // 释放上传缓冲区（在 GPU 完成上传后调用）
    void DisposeUploaders() {
        VertexBufferUploader = nullptr;
        IndexBufferUploader  = nullptr;
    }
};
```

`MeshGeometry` 的设计优点：
- 将顶点/索引的 CPU 副本、GPU 资源、上传缓冲区集中管理。
- 缓存了视图需要的所有元数据（步长、大小、格式）。
- 通过 `DrawArgs` 支持多个子网格存储在同一个顶点/索引缓冲区中（参考 6.3.4 节的合并技术）。
- 提供 `VertexBufferView()` 和 `IndexBufferView()` 方法直接获取视图结构体。
- `DisposeUploaders()` 可在 GPU 完成上传后释放中间上传缓冲区，节省内存。

---

## 6.11 Box Demo（盒子演示）

本节将本章的所有知识点整合到一个完整的程序中，渲染一个带颜色的旋转立方体。

### 6.11.1 着色器代码 color.hlsl

```hlsl
cbuffer cbPerObject : register(b0) {
    float4x4 gWorldViewProj;
};

struct VertexIn {
    float3 Pos   : POSITION;
    float4 Color : COLOR;
};

struct VertexOut {
    float4 PosH  : SV_POSITION;
    float4 Color : COLOR;
};

VertexOut VS(VertexIn vin) {
    VertexOut vout;
    vout.PosH  = mul(float4(vin.Pos, 1.0f), gWorldViewProj);
    vout.Color = vin.Color;
    return vout;
}

float4 PS(VertexOut pin) : SV_Target {
    return pin.Color;
}
```

### 6.11.2 C++ 应用程序 BoxApp.cpp

```cpp
#include "../../Common/d3dApp.h"
#include "../../Common/MathHelper.h"
#include "../../Common/UploadBuffer.h"

using Microsoft::WRL::ComPtr;
using namespace DirectX;
using namespace DirectX::PackedVector;

// 顶点结构
struct Vertex {
    XMFLOAT3 Pos;
    XMFLOAT4 Color;
};

// 常量缓冲区数据（必须 256 对齐，由 UploadBuffer 处理）
struct ObjectConstants {
    XMFLOAT4X4 WorldViewProj = MathHelper::Identity4x4();
};

class BoxApp : public D3DApp {
public:
    BoxApp(HINSTANCE hInstance);
    BoxApp(const BoxApp& rhs) = delete;
    BoxApp& operator=(const BoxApp& rhs) = delete;
    ~BoxApp();

    virtual bool Initialize() override;

private:
    virtual void OnResize() override;
    virtual void Update(const GameTimer& gt) override;
    virtual void Draw(const GameTimer& gt) override;

    virtual void OnMouseDown(WPARAM btnState, int x, int y) override;
    virtual void OnMouseUp(WPARAM btnState, int x, int y) override;
    virtual void OnMouseMove(WPARAM btnState, int x, int y) override;

    void BuildDescriptorHeaps();
    void BuildConstantBuffers();
    void BuildRootSignature();
    void BuildShadersAndInputLayout();
    void BuildBoxGeometry();
    void BuildPSO();

private:
    ComPtr<ID3D12RootSignature> mRootSignature = nullptr;
    ComPtr<ID3D12DescriptorHeap> mCbvHeap = nullptr;

    std::unique_ptr<UploadBuffer<ObjectConstants>> mObjectCB = nullptr;
    std::unique_ptr<MeshGeometry> mBoxGeo = nullptr;

    ComPtr<ID3DBlob> mvsByteCode = nullptr;
    ComPtr<ID3DBlob> mpsByteCode = nullptr;
    std::vector<D3D12_INPUT_ELEMENT_DESC> mInputLayout;

    ComPtr<ID3D12PipelineState> mPSO = nullptr;

    XMFLOAT4X4 mWorld = MathHelper::Identity4x4();
    XMFLOAT4X4 mView  = MathHelper::Identity4x4();
    XMFLOAT4X4 mProj  = MathHelper::Identity4x4();

    float mTheta  = 1.5f * XM_PI;
    float mPhi    = XM_PIDIV4;
    float mRadius = 5.0f;

    POINT mLastMousePos;
};

int WINAPI WinMain(HINSTANCE hInstance, HINSTANCE prevInstance,
                   PSTR cmdLine, int showCmd) {
#if defined(DEBUG) | defined(_DEBUG)
    _CrtSetDbgFlag(_CRTDBG_ALLOC_MEM_DF | _CRTDBG_LEAK_CHECK_DF);
#endif

    try {
        BoxApp theApp(hInstance);
        if (!theApp.Initialize())
            return 0;
        return theApp.Run();
    } catch (DxException& e) {
        MessageBox(nullptr, e.ToString().c_str(), L"HR Failed", MB_OK);
        return 0;
    }
}

BoxApp::BoxApp(HINSTANCE hInstance) : D3DApp(hInstance) { }
BoxApp::~BoxApp() { }

bool BoxApp::Initialize() {
    if (!D3DApp::Initialize())
        return false;

    // 重置命令列表以准备录制初始化命令
    ThrowIfFailed(mCommandList->Reset(mDirectCmdListAlloc.Get(), nullptr));

    BuildDescriptorHeaps();
    BuildConstantBuffers();
    BuildRootSignature();
    BuildShadersAndInputLayout();
    BuildBoxGeometry();
    BuildPSO();

    // 执行初始化命令
    ThrowIfFailed(mCommandList->Close());
    ID3D12CommandList* cmdsLists[] = { mCommandList.Get() };
    mCommandQueue->ExecuteCommandLists(_countof(cmdsLists), cmdsLists);

    // 等待初始化完成
    FlushCommandQueue();

    return true;
}

void BoxApp::OnResize() {
    D3DApp::OnResize();

    // 窗口大小变化时更新投影矩阵
    XMMATRIX P = XMMatrixPerspectiveFovLH(
        0.25f * MathHelper::Pi,
        AspectRatio(),
        1.0f,
        1000.0f);
    XMStoreFloat4x4(&mProj, P);
}

void BoxApp::Update(const GameTimer& gt) {
    // 球坐标转笛卡尔坐标
    float x = mRadius * sinf(mPhi) * cosf(mTheta);
    float z = mRadius * sinf(mPhi) * sinf(mTheta);
    float y = mRadius * cosf(mPhi);

    // 构建观察矩阵
    XMVECTOR pos    = XMVectorSet(x, y, z, 1.0f);
    XMVECTOR target = XMVectorZero();
    XMVECTOR up     = XMVectorSet(0.0f, 1.0f, 0.0f, 0.0f);
    XMMATRIX view   = XMMatrixLookAtLH(pos, target, up);
    XMStoreFloat4x4(&mView, view);

    XMMATRIX world       = XMLoadFloat4x4(&mWorld);
    XMMATRIX proj        = XMLoadFloat4x4(&mProj);
    XMMATRIX worldViewProj = world * view * proj;

    // 更新常量缓冲区（HLSL 默认列主序矩阵，需要转置）
    ObjectConstants objConstants;
    XMStoreFloat4x4(&objConstants.WorldViewProj, XMMatrixTranspose(worldViewProj));
    mObjectCB->CopyData(0, objConstants);
}

void BoxApp::Draw(const GameTimer& gt) {
    // 复用与命令录制关联的内存
    ThrowIfFailed(mDirectCmdListAlloc->Reset());
    ThrowIfFailed(mCommandList->Reset(mDirectCmdListAlloc.Get(), mPSO.Get()));

    mCommandList->RSSetViewports(1, &mScreenViewport);
    mCommandList->RSSetScissorRects(1, &mScissorRect);

    // 状态转换：Present -> Render Target
    mCommandList->ResourceBarrier(1, &CD3DX12_RESOURCE_BARRIER::Transition(
        CurrentBackBuffer(),
        D3D12_RESOURCE_STATE_PRESENT,
        D3D12_RESOURCE_STATE_RENDER_TARGET));

    // 清除后台缓冲区和深度缓冲区
    mCommandList->ClearRenderTargetView(
        CurrentBackBufferView(),
        Colors::LightSteelBlue,
        0, nullptr);
    mCommandList->ClearDepthStencilView(
        DepthStencilView(),
        D3D12_CLEAR_FLAG_DEPTH | D3D12_CLEAR_FLAG_STENCIL,
        1.0f, 0, 0, nullptr);

    // 设置渲染目标
    mCommandList->OMSetRenderTargets(
        1, &CurrentBackBufferView(), true, &DepthStencilView());

    // 设置描述符堆和根签名
    ID3D12DescriptorHeap* descriptorHeaps[] = { mCbvHeap.Get() };
    mCommandList->SetDescriptorHeaps(_countof(descriptorHeaps), descriptorHeaps);
    mCommandList->SetGraphicsRootSignature(mRootSignature.Get());

    // 绑定顶点/索引缓冲区
    mCommandList->IASetVertexBuffers(0, 1, &mBoxGeo->VertexBufferView());
    mCommandList->IASetIndexBuffer(&mBoxGeo->IndexBufferView());
    mCommandList->IASetPrimitiveTopology(D3D_PRIMITIVE_TOPOLOGY_TRIANGLELIST);

    // 绑定常量缓冲区描述符表并绘制
    mCommandList->SetGraphicsRootDescriptorTable(
        0, mCbvHeap->GetGPUDescriptorHandleForHeapStart());

    mCommandList->DrawIndexedInstanced(
        mBoxGeo->DrawArgs["box"].IndexCount,
        1, 0, 0, 0);

    // 状态转换：Render Target -> Present
    mCommandList->ResourceBarrier(1, &CD3DX12_RESOURCE_BARRIER::Transition(
        CurrentBackBuffer(),
        D3D12_RESOURCE_STATE_RENDER_TARGET,
        D3D12_RESOURCE_STATE_PRESENT));

    // 完成命令录制
    ThrowIfFailed(mCommandList->Close());

    // 提交命令列表到队列执行
    ID3D12CommandList* cmdsLists[] = { mCommandList.Get() };
    mCommandQueue->ExecuteCommandLists(_countof(cmdsLists), cmdsLists);

    // 呈现（交换前后台缓冲区）
    ThrowIfFailed(mSwapChain->Present(0, 0));
    mCurrBackBuffer = (mCurrBackBuffer + 1) % SwapChainBufferCount;

    // 等待帧命令完成（简单但低效的做法，后续章节会优化）
    FlushCommandQueue();
}

void BoxApp::OnMouseDown(WPARAM btnState, int x, int y) {
    mLastMousePos.x = x;
    mLastMousePos.y = y;
    SetCapture(mhMainWnd);
}

void BoxApp::OnMouseUp(WPARAM btnState, int x, int y) {
    ReleaseCapture();
}

void BoxApp::OnMouseMove(WPARAM btnState, int x, int y) {
    if ((btnState & MK_LBUTTON) != 0) {
        // 左键：旋转
        float dx = XMConvertToRadians(
            0.25f * static_cast<float>(x - mLastMousePos.x));
        float dy = XMConvertToRadians(
            0.25f * static_cast<float>(y - mLastMousePos.y));

        mTheta += dx;
        mPhi   += dy;
        mPhi = MathHelper::Clamp(mPhi, 0.1f, MathHelper::Pi - 0.1f);
    } else if ((btnState & MK_RBUTTON) != 0) {
        // 右键：缩放
        float dx = 0.005f * static_cast<float>(x - mLastMousePos.x);
        float dy = 0.005f * static_cast<float>(y - mLastMousePos.y);
        mRadius += dx - dy;
        mRadius = MathHelper::Clamp(mRadius, 3.0f, 15.0f);
    }

    mLastMousePos.x = x;
    mLastMousePos.y = y;
}

void BoxApp::BuildDescriptorHeaps() {
    D3D12_DESCRIPTOR_HEAP_DESC cbvHeapDesc;
    cbvHeapDesc.NumDescriptors = 1;
    cbvHeapDesc.Type           = D3D12_DESCRIPTOR_HEAP_TYPE_CBV_SRV_UAV;
    cbvHeapDesc.Flags          = D3D12_DESCRIPTOR_HEAP_FLAG_SHADER_VISIBLE;
    cbvHeapDesc.NodeMask       = 0;

    ThrowIfFailed(md3dDevice->CreateDescriptorHeap(
        &cbvHeapDesc, IID_PPV_ARGS(&mCbvHeap)));
}

void BoxApp::BuildConstantBuffers() {
    mObjectCB = std::make_unique<UploadBuffer<ObjectConstants>>(
        md3dDevice.Get(), 1, true);

    UINT objCBByteSize = d3dUtil::CalcConstantBufferByteSize(sizeof(ObjectConstants));

    D3D12_GPU_VIRTUAL_ADDRESS cbAddress = mObjectCB->Resource()->GetGPUVirtualAddress();
    int boxCBufIndex = 0;
    cbAddress += boxCBufIndex * objCBByteSize;

    D3D12_CONSTANT_BUFFER_VIEW_DESC cbvDesc;
    cbvDesc.BufferLocation = cbAddress;
    cbvDesc.SizeInBytes    = objCBByteSize;

    md3dDevice->CreateConstantBufferView(
        &cbvDesc,
        mCbvHeap->GetCPUDescriptorHandleForHeapStart());
}

void BoxApp::BuildRootSignature() {
    CD3DX12_ROOT_PARAMETER slotRootParameter[1];

    CD3DX12_DESCRIPTOR_RANGE cbvTable;
    cbvTable.Init(D3D12_DESCRIPTOR_RANGE_TYPE_CBV, 1, 0);
    slotRootParameter[0].InitAsDescriptorTable(1, &cbvTable);

    CD3DX12_ROOT_SIGNATURE_DESC rootSigDesc(
        1, slotRootParameter,
        0, nullptr,
        D3D12_ROOT_SIGNATURE_FLAG_ALLOW_INPUT_ASSEMBLER_INPUT_LAYOUT);

    ComPtr<ID3DBlob> serializedRootSig = nullptr;
    ComPtr<ID3DBlob> errorBlob = nullptr;

    HRESULT hr = D3D12SerializeRootSignature(
        &rootSigDesc,
        D3D_ROOT_SIGNATURE_VERSION_1,
        serializedRootSig.GetAddressOf(),
        errorBlob.GetAddressOf());

    if (errorBlob != nullptr) {
        ::OutputDebugStringA((char*)errorBlob->GetBufferPointer());
    }
    ThrowIfFailed(hr);

    ThrowIfFailed(md3dDevice->CreateRootSignature(
        0,
        serializedRootSig->GetBufferPointer(),
        serializedRootSig->GetBufferSize(),
        IID_PPV_ARGS(&mRootSignature)));
}

void BoxApp::BuildShadersAndInputLayout() {
    mvsByteCode = d3dUtil::CompileShader(
        L"Shaders\\color.hlsl", nullptr, "VS", "vs_5_0");
    mpsByteCode = d3dUtil::CompileShader(
        L"Shaders\\color.hlsl", nullptr, "PS", "ps_5_0");

    mInputLayout = {
        { "POSITION", 0, DXGI_FORMAT_R32G32B32_FLOAT,
          0, 0, D3D12_INPUT_CLASSIFICATION_PER_VERTEX_DATA, 0 },
        { "COLOR", 0, DXGI_FORMAT_R32G32B32A32_FLOAT,
          0, 12, D3D12_INPUT_CLASSIFICATION_PER_VERTEX_DATA, 0 }
    };
}

void BoxApp::BuildBoxGeometry() {
    std::array<Vertex, 8> vertices = {
        Vertex({ XMFLOAT3(-1.0f, -1.0f, -1.0f), XMFLOAT4(Colors::White)   }),
        Vertex({ XMFLOAT3(-1.0f, +1.0f, -1.0f), XMFLOAT4(Colors::Black)   }),
        Vertex({ XMFLOAT3(+1.0f, +1.0f, -1.0f), XMFLOAT4(Colors::Red)     }),
        Vertex({ XMFLOAT3(+1.0f, -1.0f, -1.0f), XMFLOAT4(Colors::Green)   }),
        Vertex({ XMFLOAT3(-1.0f, -1.0f, +1.0f), XMFLOAT4(Colors::Blue)    }),
        Vertex({ XMFLOAT3(-1.0f, +1.0f, +1.0f), XMFLOAT4(Colors::Yellow)  }),
        Vertex({ XMFLOAT3(+1.0f, +1.0f, +1.0f), XMFLOAT4(Colors::Cyan)    }),
        Vertex({ XMFLOAT3(+1.0f, -1.0f, +1.0f), XMFLOAT4(Colors::Magenta) })
    };

    std::array<std::uint16_t, 36> indices = {
        // 前面
        0, 1, 2,   0, 2, 3,
        // 后面
        4, 6, 5,   4, 7, 6,
        // 左面
        4, 5, 1,   4, 1, 0,
        // 右面
        3, 2, 6,   3, 6, 7,
        // 顶面
        1, 5, 6,   1, 6, 2,
        // 底面
        4, 0, 3,   4, 3, 7
    };

    const UINT vbByteSize = (UINT)vertices.size() * sizeof(Vertex);
    const UINT ibByteSize = (UINT)indices.size() * sizeof(std::uint16_t);

    mBoxGeo = std::make_unique<MeshGeometry>();
    mBoxGeo->Name = "boxGeo";

    // 创建 CPU 副本
    ThrowIfFailed(D3DCreateBlob(vbByteSize, &mBoxGeo->VertexBufferCPU));
    CopyMemory(mBoxGeo->VertexBufferCPU->GetBufferPointer(),
               vertices.data(), vbByteSize);
    ThrowIfFailed(D3DCreateBlob(ibByteSize, &mBoxGeo->IndexBufferCPU));
    CopyMemory(mBoxGeo->IndexBufferCPU->GetBufferPointer(),
               indices.data(), ibByteSize);

    // 创建 GPU 资源
    mBoxGeo->VertexBufferGPU = d3dUtil::CreateDefaultBuffer(
        md3dDevice.Get(), mCommandList.Get(),
        vertices.data(), vbByteSize, mBoxGeo->VertexBufferUploader);
    mBoxGeo->IndexBufferGPU = d3dUtil::CreateDefaultBuffer(
        md3dDevice.Get(), mCommandList.Get(),
        indices.data(), ibByteSize, mBoxGeo->IndexBufferUploader);

    mBoxGeo->VertexByteStride     = sizeof(Vertex);
    mBoxGeo->VertexBufferByteSize = vbByteSize;
    mBoxGeo->IndexFormat          = DXGI_FORMAT_R16_UINT;
    mBoxGeo->IndexBufferByteSize  = ibByteSize;

    SubmeshGeometry submesh;
    submesh.IndexCount         = (UINT)indices.size();
    submesh.StartIndexLocation = 0;
    submesh.BaseVertexLocation = 0;
    mBoxGeo->DrawArgs["box"] = submesh;
}

void BoxApp::BuildPSO() {
    D3D12_GRAPHICS_PIPELINE_STATE_DESC psoDesc;
    ZeroMemory(&psoDesc, sizeof(D3D12_GRAPHICS_PIPELINE_STATE_DESC));

    psoDesc.InputLayout = { mInputLayout.data(), (UINT)mInputLayout.size() };
    psoDesc.pRootSignature = mRootSignature.Get();
    psoDesc.VS = {
        reinterpret_cast<BYTE*>(mvsByteCode->GetBufferPointer()),
        mvsByteCode->GetBufferSize()
    };
    psoDesc.PS = {
        reinterpret_cast<BYTE*>(mpsByteCode->GetBufferPointer()),
        mpsByteCode->GetBufferSize()
    };
    psoDesc.RasterizerState       = CD3DX12_RASTERIZER_DESC(D3D12_DEFAULT);
    psoDesc.BlendState            = CD3DX12_BLEND_DESC(D3D12_DEFAULT);
    psoDesc.DepthStencilState     = CD3DX12_DEPTH_STENCIL_DESC(D3D12_DEFAULT);
    psoDesc.SampleMask            = UINT_MAX;
    psoDesc.PrimitiveTopologyType = D3D12_PRIMITIVE_TOPOLOGY_TYPE_TRIANGLE;
    psoDesc.NumRenderTargets      = 1;
    psoDesc.RTVFormats[0]         = mBackBufferFormat;
    psoDesc.SampleDesc.Count      = m4xMsaaState ? 4 : 1;
    psoDesc.SampleDesc.Quality    = m4xMsaaState ? (m4xMsaaQuality - 1) : 0;
    psoDesc.DSVFormat             = mDepthStencilFormat;

    ThrowIfFailed(md3dDevice->CreateGraphicsPipelineState(
        &psoDesc, IID_PPV_ARGS(&mPSO)));
}
```

### 6.11.3 程序结构说明

**初始化流程（`Initialize`）：**

1. 调用基类 `D3DApp::Initialize()` 完成 D3D12 设备和交换链的初始化。
2. 重置命令列表，开始录制初始化命令。
3. 按顺序构建：描述符堆 → 常量缓冲区 → 根签名 → 着色器和输入布局 → 几何体 → PSO。
4. 关闭命令列表，提交执行，调用 `FlushCommandQueue()` 等待 GPU 完成。

**每帧更新（`Update`）：**

1. 根据球坐标 `(mRadius, mTheta, mPhi)` 计算相机位置。
2. 用 `XMMatrixLookAtLH` 构建观察矩阵。
3. 计算 `world * view * proj` 组合矩阵。
4. **转置矩阵**后复制到常量缓冲区（HLSL 默认使用列主序矩阵，而 C++ 中的 `XMFLOAT4X4` 是行主序，需要转置以匹配）。

**每帧绘制（`Draw`）：**

1. 重置命令分配器和命令列表（复用内存）。
2. 设置视口和裁剪矩形。
3. 资源屏障：后台缓冲区从 `PRESENT` 转换到 `RENDER_TARGET`。
4. 清除后台缓冲区（淡钢蓝色）和深度缓冲区。
5. 设置 OM 渲染目标。
6. 设置 CBV 描述符堆和根签名。
7. 绑定顶点缓冲区、索引缓冲区、图元拓扑。
8. 设置根描述符表，绑定常量缓冲区。
9. 调用 `DrawIndexedInstanced` 绘制 36 个索引（12 个三角形）。
10. 资源屏障：后台缓冲区从 `RENDER_TARGET` 转换回 `PRESENT`。
11. 关闭并提交命令列表，调用 `Present` 呈现。
12. 调用 `FlushCommandQueue` 等待 GPU 完成（简单但低效，后续章节优化）。

**鼠标交互：**

- 按住左键拖动：以球坐标旋转相机（改变 `mTheta` 和 `mPhi`）。
- 按住右键拖动：拉近/拉远相机（改变 `mRadius`）。
- `mPhi` 被钳制在 `[0.1, Pi - 0.1]` 之间，防止相机到达极点时产生翻转。
