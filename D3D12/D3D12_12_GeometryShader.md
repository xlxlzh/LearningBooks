# 第 12 章 几何着色器

---

## 12.1 几何着色器编程

几何着色器是位于顶点着色器和像素着色器之间的可选阶段。与逐顶点处理的顶点着色器不同，几何着色器以**完整图元**为输入（点、线或三角形），可以输出零个、一个或多个图元。

几何着色器的最大优势是**可以创建或销毁几何体**。输入图元可以扩展为其他图元，也可以根据条件选择不输出任何图元。输出图元类型可以与输入不同。

### 12.1.1 基本语法

```hlsl
[maxvertexcount(N)]
void ShaderName(
    PrimitiveType InputVertexType InputName[NumElements],
    inout StreamOutputObject<OutputVertexType> OutputName)
{
    // 几何着色器体...
}
```

| 属性/参数 | 说明 |
|----------|------|
| `[maxvertexcount(N)]` | 单次调用最多输出的顶点数。应尽可能小以提升性能 |
| `PrimitiveType` | 输入图元类型：`point`、`line`、`triangle`、`lineadj`、`triangleadj` |
| `InputName[NumElements]` | 输入顶点数组：点=1，线=2，三角形=3，线邻接=4，三角邻接=6 |
| `StreamOutputObject` | 输出流类型：`PointStream`、`LineStream`、`TriangleStream` |

**输出流方法**：

```hlsl
// 向输出流追加顶点
void StreamOutputObject<OutputVertexType>::Append(OutputVertexType v);

// 结束当前条带，开始新条带（用于模拟列表而非条带）
void StreamOutputObject<OutputVertexType>::RestartStrip();
```

### 12.1.2 输入图元类型

| 类型 | 说明 |
|------|------|
| `point` | 点图元 |
| `line` | 线图元（列表或条带） |
| `triangle` | 三角形图元（列表或条带） |
| `lineadj` | 带邻接信息的线 |
| `triangleadj` | 带邻接信息的三角形 |

注意：几何着色器不区分列表（list）和条带（strip），总是接收完整图元。

### 12.1.3 示例：三角形细分

```hlsl
struct VertexOut {
    float3 PosL    : POSITION;
    float3 NormalL : NORMAL;
    float2 Tex     : TEXCOORD;
};

struct GeoOut {
    float4 PosH    : SV_POSITION;
    float3 PosW    : POSITION;
    float3 NormalW : NORMAL;
    float2 Tex     : TEXCOORD;
};

void Subdivide(VertexOut inVerts[3], out VertexOut outVerts[6]) {
    // 计算边中点
    VertexOut m[3];
    m[0].PosL = 0.5f * (inVerts[0].PosL + inVerts[1].PosL);
    m[1].PosL = 0.5f * (inVerts[1].PosL + inVerts[2].PosL);
    m[2].PosL = 0.5f * (inVerts[2].PosL + inVerts[0].PosL);

    // 投影到单位球面并计算法线
    m[0].PosL = normalize(m[0].PosL); m[0].NormalL = m[0].PosL;
    m[1].PosL = normalize(m[1].PosL); m[1].NormalL = m[1].PosL;
    m[2].PosL = normalize(m[2].PosL); m[2].NormalL = m[2].PosL;

    // 插值纹理坐标
    m[0].Tex = 0.5f * (inVerts[0].Tex + inVerts[1].Tex);
    m[1].Tex = 0.5f * (inVerts[1].Tex + inVerts[2].Tex);
    m[2].Tex = 0.5f * (inVerts[2].Tex + inVerts[0].Tex);

    outVerts[0] = inVerts[0];
    outVerts[1] = m[0];
    outVerts[2] = m[2];
    outVerts[3] = m[1];
    outVerts[4] = inVerts[2];
    outVerts[5] = inVerts[1];
}

[maxvertexcount(8)]
void GS(triangle VertexOut gin[3], inout TriangleStream<GeoOut> triStream) {
    VertexOut v[6];
    Subdivide(gin, v);

    GeoOut gout[6];
    [unroll] for (int i = 0; i < 6; ++i) {
        gout[i].PosW    = mul(float4(v[i].PosL, 1.0f), gWorld).xyz;
        gout[i].NormalW = mul(v[i].NormalL, (float3x3)gWorldInvTranspose);
        gout[i].PosH    = mul(float4(v[i].PosL, 1.0f), gWorldViewProj);
        gout[i].Tex     = v[i].Tex;
    }

    // 输出条带 1（底部三个三角形）
    [unroll] for (int j = 0; j < 5; ++j)
        triStream.Append(gout[j]);
    triStream.RestartStrip();

    // 输出条带 2（顶部三角形）
    triStream.Append(gout[1]);
    triStream.Append(gout[5]);
    triStream.Append(gout[3]);
}
```

### 12.1.4 PSO 设置

几何着色器在 PSO 中通过 `GS` 字段绑定：

```cpp
D3D12_GRAPHICS_PIPELINE_STATE_DESC treeSpritePsoDesc = opaquePsoDesc;
treeSpritePsoDesc.GS = {
    reinterpret_cast<BYTE*>(mShaders["treeSpriteGS"]->GetBufferPointer()),
    mShaders["treeSpriteGS"]->GetBufferSize()
};
```

编译时使用 `gs_5_0` 目标：

```cpp
mShaders["treeSpriteGS"] = d3dUtil::CompileShader(
    L"Shaders\\TreeSprite.hlsl", nullptr, "GS", "gs_5_0");
```

---

## 12.2 树木公告板演示

### 12.2.1 公告板概述

当树木很远时，使用公告板（Billboard）技术提高效率：用一张带树木图片的矩形代替完整的 3D 模型。关键要求是**公告板必须始终面向相机**。

假设 y 轴向上，xz 平面为地面，公告板通常与 y 轴对齐，仅在 xz 平面内旋转面向相机。

给定公告板中心位置 `C` 和相机位置 `E`：

```
look = normalize(E - C)
look.y = 0  // y轴对齐，投影到xz平面
right = cross(up, look)  // up = (0, 1, 0)
```

公告板四边形顶点：

```hlsl
v[0] = C + halfWidth * right - halfHeight * up;
v[1] = C + halfWidth * right + halfHeight * up;
v[2] = C - halfWidth * right - halfHeight * up;
v[3] = C - halfWidth * right + halfHeight * up;
```

**几何着色器 vs CPU 实现**：
- CPU 方式：每个公告板需要 4 个顶点，相机移动时需更新动态顶点缓冲区
- GS 方式：每个公告板只需 1 个点顶点，使用静态缓冲区，GS 负责扩展和面朝相机

### 12.2.2 顶点结构

```cpp
struct TreeSpriteVertex {
    XMFLOAT3 Pos;   // 世界空间中心位置
    XMFLOAT2 Size;  // 世界空间宽高
};

mTreeSpriteInputLayout = {
    { "POSITION", 0, DXGI_FORMAT_R32G32B32_FLOAT, 0, 0,
      D3D12_INPUT_CLASSIFICATION_PER_VERTEX_DATA, 0 },
    { "SIZE", 0, DXGI_FORMAT_R32G32_FLOAT, 0, 12,
      D3D12_INPUT_CLASSIFICATION_PER_VERTEX_DATA, 0 }
};
```

### 12.2.3 完整 HLSL

```hlsl
struct VertexIn {
    float3 PosW  : POSITION;
    float2 SizeW : SIZE;
};

struct VertexOut {
    float3 CenterW : POSITION;
    float2 SizeW   : SIZE;
};

struct GeoOut {
    float4 PosH    : SV_POSITION;
    float3 PosW    : POSITION;
    float3 NormalW : NORMAL;
    float2 TexC    : TEXCOORD;
    uint   PrimID  : SV_PrimitiveID;
};

VertexOut VS(VertexIn vin) {
    VertexOut vout;
    vout.CenterW = vin.PosW;
    vout.SizeW   = vin.SizeW;
    return vout;
}

[maxvertexcount(4)]
void GS(point VertexOut gin[1],
        uint primID : SV_PrimitiveID,
        inout TriangleStream<GeoOut> triStream)
{
    // 计算局部坐标系：y轴对齐，面向相机
    float3 up   = float3(0.0f, 1.0f, 0.0f);
    float3 look = gEyePosW - gin[0].CenterW;
    look.y = 0.0f;           // 投影到xz平面
    look = normalize(look);  // 视图方向
    float3 right = cross(up, look);

    float halfWidth  = 0.5f * gin[0].SizeW.x;
    float halfHeight = 0.5f * gin[0].SizeW.y;

    float4 v[4];
    v[0] = float4(gin[0].CenterW + halfWidth * right - halfHeight * up, 1.0f);
    v[1] = float4(gin[0].CenterW + halfWidth * right + halfHeight * up, 1.0f);
    v[2] = float4(gin[0].CenterW - halfWidth * right - halfHeight * up, 1.0f);
    v[3] = float4(gin[0].CenterW - halfWidth * right + halfHeight * up, 1.0f);

    float2 texC[4] = {
        float2(0.0f, 1.0f), float2(0.0f, 0.0f),
        float2(1.0f, 1.0f), float2(1.0f, 0.0f)
    };

    GeoOut gout;
    [unroll] for (int i = 0; i < 4; ++i) {
        gout.PosH    = mul(v[i], gViewProj);
        gout.PosW    = v[i].xyz;
        gout.NormalW = look;      // 公告板法线朝向相机
        gout.TexC    = texC[i];
        gout.PrimID  = primID;
        triStream.Append(gout);
    }
}

float4 PS(GeoOut pin) : SV_Target {
    float3 uvw = float3(pin.TexC, pin.PrimID % 3);
    float4 diffuseAlbedo = gTreeMapArray.Sample(gsamAnisotropicWrap, uvw)
                         * gDiffuseAlbedo;

#ifdef ALPHA_TEST
    clip(diffuseAlbedo.a - 0.1f);
#endif

    pin.NormalW = normalize(pin.NormalW);
    // ... 光照计算
}
```

---

## 12.3 纹理数组

### 12.3.1 概述

纹理数组在单个资源中存储多张纹理。HLSL 中使用 `Texture2DArray` 类型。相比 `Texture2D TexArray[4]` 的索引方式，纹理数组通常性能更好。

C++ 中通过 `ID3D12Resource` 创建，设置 `DepthOrArraySize` 为数组元素数量。

### 12.3.2 采样纹理数组

需要三个纹理坐标：`(u, v, index)`，其中 `index` 是数组索引。

```hlsl
float3 uvw = float3(pin.TexC, pin.PrimID % 3);
float4 diffuseAlbedo = gTreeMapArray.Sample(gsamAnisotropicWrap, uvw);
```

### 12.3.3 创建纹理数组 DDS

使用 `texassemble` 工具：

```bash
# 从多个图像创建纹理数组
texassemble -array -o treeArray.dds t0.dds t1.dds t2.dds t3.dds

# 生成mipmap并转换格式
texconv -m 10 -f BC3_UNORM treeArray.dds
```

### 12.3.4 纹理子资源

| 术语 | 说明 |
|------|------|
| **Array Slice** | 纹理数组中的一个元素及其完整 mipmap 链 |
| **Mip Slice** | 纹理数组中某一特定 mipmap 级别的所有纹理 |
| **Subresource** | 纹理数组中单个元素的一个 mipmap 级别 |

线性子资源索引计算：

```cpp
inline UINT D3D12CalcSubresource(
    UINT MipSlice, UINT ArraySlice, UINT PlaneSlice,
    UINT MipLevels, UINT ArraySize)
{
    return MipSlice + ArraySlice * MipLevels
         + PlaneSlice * MipLevels * ArraySize;
}
```

---

## 12.4 SV_PrimitiveID 和 SV_VertexID

**`SV_PrimitiveID`**：输入装配器自动为每个图元生成 ID。第一个图元为 0，第二个为 1，依此类推。若存在几何着色器，必须在 GS 签名中声明；否则可直接在 PS 中使用。

```hlsl
[maxvertexcount(4)]
void GS(point VertexOut gin[1],
        uint primID : SV_PrimitiveID,
        inout TriangleStream<GeoOut> triStream)
{
    // primID 自动传入，可传递给像素着色器
    gout.PrimID = primID;
}
```

**`SV_VertexID`**：输入装配器自动为每个顶点生成 ID。

```hlsl
VertexOut VS(VertexIn vin, uint vertID : SV_VertexID) {
    // vertID: Draw() 时为 0,1,...,n-1；DrawIndexed() 时为索引值
}
```

---

## 12.5 Alpha-to-Coverage

公告板边缘的 `clip()` 函数会产生块状伪影。虽然可以用透明度混合解决，但需要排序且会造成大量 overdraw。

**Alpha-to-Coverage** 技术：在启用 MSAA 时，硬件根据像素着色器输出的 Alpha 值计算子像素覆盖率。

例如 4X MSAA，若 Alpha = 0.5，则约 2/4 子像素被视为覆盖，产生平滑边缘。

启用方式：

```cpp
D3D12_BLEND_DESC blendDesc = {};
blendDesc.AlphaToCoverageEnable = true;  // 启用
// ... 其余混合设置
```

建议：对于树叶、栅栏等 Alpha 遮罩纹理，始终使用 alpha-to-coverage（需要启用 MSAA）。
