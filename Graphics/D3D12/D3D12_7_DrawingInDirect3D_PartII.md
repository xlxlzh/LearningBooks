# 第 7 章 Direct3D 中的绘制（Part II）

---

## 7.1 帧资源

回顾第 4 章，CPU 和 GPU 是并行工作的。CPU 构建并提交命令列表，GPU 在命令队列中处理命令。目标是让两者都保持忙碌，充分利用系统硬件资源。

在此之前，示例程序每帧都在末尾调用 `FlushCommandQueue`，强制 CPU 等待 GPU 完成当前帧的所有命令。这种同步是必要的，原因有二：

1. **命令分配器不能重置**：如果 CPU 在 GPU 还没处理完第 `n` 帧命令时，就进入第 `n+1` 帧并重置命令分配器，那么 GPU 正在执行的命令会被清除。
2. **常量缓冲区不能覆盖**：如果 CPU 在第 `n+1` 帧覆盖常量缓冲区数据，但 GPU 还没执行完引用该常量缓冲区的第 `n` 帧绘制命令，那么 GPU 将使用错误的数据。

然而，每帧都等待 GPU 完成是低效的：
- 每帧开始时，GPU 没有命令可处理（因为 CPU 之前清空了队列）。
- 每帧结束时，CPU 在等待 GPU 完成处理。

**帧资源（Frame Resources）** 就是解决这个问题的方案：创建一个**循环数组**（通常包含 3 个元素），存储每帧 CPU 需要修改的资源。CPU 在帧 `n` 时，从循环数组中获取下一个可用的（即 GPU 不再使用的）帧资源，更新资源并构建命令列表。这样 GPU 可以在处理前几帧的同时，CPU 提前准备后续帧的命令。

### FrameResource 结构体

```cpp
struct FrameResource {
public:
    FrameResource(ID3D12Device* device, UINT passCount, UINT objectCount);
    FrameResource(const FrameResource& rhs) = delete;
    FrameResource& operator=(const FrameResource& rhs) = delete;
    ~FrameResource();

    // 每个帧资源都需要自己的命令分配器
    Microsoft::WRL::ComPtr<ID3D12CommandAllocator> CmdListAlloc;

    // 每个帧资源都需要自己的常量缓冲区
    std::unique_ptr<UploadBuffer<PassConstants>>  PassCB  = nullptr;
    std::unique_ptr<UploadBuffer<ObjectConstants>> ObjectCB = nullptr;

    // Fence 值，标记到这一点的命令。用于检查 GPU 是否仍在使用这些帧资源
    UINT64 Fence = 0;
};

FrameResource::FrameResource(ID3D12Device* device, UINT passCount, UINT objectCount) {
    ThrowIfFailed(device->CreateCommandAllocator(
        D3D12_COMMAND_LIST_TYPE_DIRECT,
        IID_PPV_ARGS(CmdListAlloc.GetAddressOf())));

    PassCB   = std::make_unique<UploadBuffer<PassConstants>>(device, passCount, true);
    ObjectCB = std::make_unique<UploadBuffer<ObjectConstants>>(device, objectCount, true);
}
```

应用程序维护一个帧资源数组和当前帧索引：

```cpp
static const int gNumFrameResources = 3;
std::vector<std::unique_ptr<FrameResource>> mFrameResources;
FrameResource* mCurrFrameResource = nullptr;
int mCurrFrameResourceIndex = 0;

void ShapesApp::BuildFrameResources() {
    for (int i = 0; i < gNumFrameResources; ++i) {
        mFrameResources.push_back(std::make_unique<FrameResource>(
            md3dDevice.Get(), 1, (UINT)mAllRitems.size()));
    }
}
```

### Update 中的帧资源循环算法

```cpp
void ShapesApp::Update(const GameTimer& gt) {
    // 循环到下一个帧资源
    mCurrFrameResourceIndex = (mCurrFrameResourceIndex + 1) % gNumFrameResources;
    mCurrFrameResource = mFrameResources[mCurrFrameResourceIndex].get();

    // 检查 GPU 是否已完成该帧资源对应的命令
    if (mCurrFrameResource->Fence != 0 &&
        mCommandQueue->GetCompletedValue() < mCurrFrameResource->Fence) {
        HANDLE eventHandle = CreateEventEx(nullptr, false, false, EVENT_ALL_ACCESS);
        ThrowIfFailed(mCommandQueue->SetEventOnFenceCompletion(
            mCurrFrameResource->Fence, eventHandle));
        WaitForSingleObject(eventHandle, INFINITE);
        CloseHandle(eventHandle);
    }

    // ... 更新 mCurrFrameResource 中的资源（如常量缓冲区）
}
```

### Draw 中的 Fence 标记

```cpp
void ShapesApp::Draw(const GameTimer& gt) {
    // ... 构建并提交命令列表

    // 推进 Fence 值以标记到这一点的命令
    mCurrFrameResource->Fence = ++mCurrentFence;

    // 在命令队列中添加设置新 Fence 点的指令
    // 由于这是在 GPU 时间线上，新 Fence 点只有在 GPU 处理完前面所有命令后才会设置
    mCommandQueue->Signal(mFence.Get(), mCurrentFence);

    // GPU 可能仍在处理前几帧的命令，但这没关系，
    // 因为我们不会触碰与那些帧关联的帧资源
}
```

帧资源并不能完全消除等待。如果 CPU 比 GPU 快太多，CPU 最终还是要等待 GPU 追上。但这正是期望的情况——说明 GPU 被充分利用了，多余的 CPU 周期可以用于游戏的其他部分（如 AI、物理、游戏逻辑）。帧资源的价值在于**保持 GPU 持续有工作可做**：当 GPU 处理帧 `n` 时，CPU 可以提前构建帧 `n+1` 和 `n+2` 的命令。

---

## 7.2 渲染项

绘制一个对象需要设置多个参数：绑定顶点/索引缓冲区、绑定对象常量、设置图元类型、指定 `DrawIndexedInstanced` 参数等。随着场景中对象增多，有必要创建一个轻量级结构体来存储提交一次完整绘制调用所需的数据。

```cpp
struct RenderItem {
    RenderItem() = default;

    // 世界矩阵，定义对象局部空间到世界空间的变换
    XMFLOAT4X4 World = MathHelper::Identity4x4();

    // 脏标记：对象数据已改变，需要更新常量缓冲区。
    // 因为有多个 FrameResource，每个都需要更新，
    // 所以修改时应设为 gNumFrameResources
    int NumFramesDirty = gNumFrameResources;

    // 该渲染项在 ObjectCB 中的索引
    UINT ObjCBIndex = -1;

    // 关联的几何体（多个渲染项可共享同一几何体）
    MeshGeometry* Geo = nullptr;

    // 图元拓扑
    D3D12_PRIMITIVE_TOPOLOGY PrimitiveType = D3D_PRIMITIVE_TOPOLOGY_TRIANGLELIST;

    // DrawIndexedInstanced 参数
    UINT IndexCount         = 0;
    UINT StartIndexLocation = 0;
    int  BaseVertexLocation = 0;
};
```

| 成员 | 说明 |
|------|------|
| `World` | 对象的世界变换矩阵，控制位置、朝向和缩放 |
| `NumFramesDirty` | 脏标记。当修改对象数据时设为 `gNumFrameResources`，表示所有帧资源的常量缓冲区都需要更新。每次更新后递减，为 0 时停止更新 |
| `ObjCBIndex` | 该渲染项在对象常量缓冲区数组中的索引 |
| `Geo` | 指向 `MeshGeometry` 的指针，多个渲染项可共享同一几何体（实例化） |
| `PrimitiveType` | 图元拓扑类型 |
| `IndexCount` / `StartIndexLocation` / `BaseVertexLocation` | 直接传给 `DrawIndexedInstanced` 的参数 |

应用程序按绘制方式维护渲染项列表：

```cpp
// 所有渲染项
std::vector<std::unique_ptr<RenderItem>> mAllRitems;

// 按 PSO 分类的渲染项指针列表
std::vector<RenderItem*> mOpaqueRitems;
std::vector<RenderItem*> mTransparentRitems;
```

---

## 7.3 通道常量

前面的 `FrameResource` 引入了一个新的常量缓冲区 `PassCB`，用于存储每轮渲染通道中固定不变的数据，例如：

- 观察者位置
- 视图矩阵和投影矩阵
- 渲染目标尺寸
- 游戏时间信息

这样就将常量数据按**更新频率**分组：
- **通道常量（Pass Constants）**：每轮渲染通道更新一次。
- **对象常量（Object Constants）**：仅当对象的世界矩阵变化时更新。

**HLSL 中的常量缓冲区定义：**

```hlsl
cbuffer cbPerObject : register(b0) {
    float4x4 gWorld;
};

cbuffer cbPass : register(b1) {
    float4x4 gView;
    float4x4 gInvView;
    float4x4 gProj;
    float4x4 gInvProj;
    float4x4 gViewProj;
    float4x4 gInvViewProj;
    float3   gEyePosW;
    float    cbPerObjectPad1;
    float2   gRenderTargetSize;
    float2   gInvRenderTargetSize;
    float    gNearZ;
    float    gFarZ;
    float    gTotalTime;
    float    gDeltaTime;
};
```

顶点着色器相应更新为：

```hlsl
VertexOut VS(VertexIn vin) {
    VertexOut vout;
    float4 posW = mul(float4(vin.PosL, 1.0f), gWorld);
    vout.PosH   = mul(posW, gViewProj);
    vout.Color  = vin.Color;
    return vout;
}
```

每个顶点多一次向量-矩阵乘法在现代 GPU 上开销可以忽略不计。

### 更新对象常量缓冲区

```cpp
void ShapesApp::UpdateObjectCBs(const GameTimer& gt) {
    auto currObjectCB = mCurrFrameResource->ObjectCB.get();

    for (auto& e : mAllRitems) {
        // 只有常量发生变化时才更新
        if (e->NumFramesDirty > 0) {
            XMMATRIX world = XMLoadFloat4x4(&e->World);

            ObjectConstants objConstants;
            XMStoreFloat4x4(&objConstants.World, XMMatrixTranspose(world));
            currObjectCB->CopyData(e->ObjCBIndex, objConstants);

            // 下一个 FrameResource 也需要更新
            e->NumFramesDirty--;
        }
    }
}
```

### 更新通道常量缓冲区

```cpp
void ShapesApp::UpdateMainPassCB(const GameTimer& gt) {
    XMMATRIX view    = XMLoadFloat4x4(&mView);
    XMMATRIX proj    = XMLoadFloat4x4(&mProj);
    XMMATRIX viewProj = XMMatrixMultiply(view, proj);

    XMStoreFloat4x4(&mMainPassCB.View,        XMMatrixTranspose(view));
    XMStoreFloat4x4(&mMainPassCB.InvView,     XMMatrixTranspose(XMMatrixInverse(&XMMatrixDeterminant(view), view)));
    XMStoreFloat4x4(&mMainPassCB.Proj,        XMMatrixTranspose(proj));
    XMStoreFloat4x4(&mMainPassCB.InvProj,     XMMatrixTranspose(XMMatrixInverse(&XMMatrixDeterminant(proj), proj)));
    XMStoreFloat4x4(&mMainPassCB.ViewProj,    XMMatrixTranspose(viewProj));
    XMStoreFloat4x4(&mMainPassCB.InvViewProj, XMMatrixTranspose(XMMatrixInverse(&XMMatrixDeterminant(viewProj), viewProj)));

    mMainPassCB.EyePosW             = mEyePos;
    mMainPassCB.RenderTargetSize    = XMFLOAT2((float)mClientWidth, (float)mClientHeight);
    mMainPassCB.InvRenderTargetSize = XMFLOAT2(1.0f / mClientWidth, 1.0f / mClientHeight);
    mMainPassCB.NearZ               = 1.0f;
    mMainPassCB.FarZ                = 1000.0f;
    mMainPassCB.TotalTime           = gt.TotalTime();
    mMainPassCB.DeltaTime           = gt.DeltaTime();

    auto currPassCB = mCurrFrameResource->PassCB.get();
    currPassCB->CopyData(0, mMainPassCB);
}
```

由于着色器期望的资源变了，根签名也需要更新以支持两个描述符表：

```cpp
CD3DX12_DESCRIPTOR_RANGE cbvTable0;
cbvTable0.Init(D3D12_DESCRIPTOR_RANGE_TYPE_CBV, 1, 0);  // per-object: register(b0)

CD3DX12_DESCRIPTOR_RANGE cbvTable1;
cbvTable1.Init(D3D12_DESCRIPTOR_RANGE_TYPE_CBV, 1, 1);  // per-pass:   register(b1)

CD3DX12_ROOT_PARAMETER slotRootParameter[2];
slotRootParameter[0].InitAsDescriptorTable(1, &cbvTable0);  // object CBV
slotRootParameter[1].InitAsDescriptorTable(1, &cbvTable1);  // pass CBV

CD3DX12_ROOT_SIGNATURE_DESC rootSigDesc(2, slotRootParameter, 0, nullptr,
    D3D12_ROOT_SIGNATURE_FLAG_ALLOW_INPUT_ASSEMBLER_INPUT_LAYOUT);
```

**性能提示**：着色器中常量缓冲区的数量建议保持在 5 个以下。

---

## 7.4 形状几何体

本节介绍如何用程序生成椭球体、球体、圆柱体和圆锥体的几何数据。这些形状在绘制天空穹顶、调试可视化、碰撞检测和延迟渲染中很有用。

`GeometryGenerator` 类（定义在 `GeometryGenerator.h/.cpp`）是用于生成简单几何形状的辅助工具类。它生成的数据存放在系统内存中，需要手动复制到顶点/索引缓冲区。

```cpp
class GeometryGenerator {
public:
    using uint16 = std::uint16_t;
    using uint32 = std::uint32_t;

    struct Vertex {
        DirectX::XMFLOAT3 Position;
        DirectX::XMFLOAT3 Normal;
        DirectX::XMFLOAT3 TangentU;
        DirectX::XMFLOAT2 TexC;
    };

    struct MeshData {
        std::vector<Vertex>     Vertices;
        std::vector<uint32>     Indices32;

        std::vector<uint16>& GetIndices16() {
            if (mIndices16.empty()) {
                mIndices16.resize(Indices32.size());
                for (size_t i = 0; i < Indices32.size(); ++i)
                    mIndices16[i] = static_cast<uint16>(Indices32[i]);
            }
            return mIndices16;
        }

    private:
        std::vector<uint16> mIndices16;
    };

    // 生成方法
    MeshData CreateBox(float width, float height, float depth, uint32 numSubdivisions);
    MeshData CreateGrid(float width, float depth, uint32 m, uint32 n);
    MeshData CreateCylinder(float bottomRadius, float topRadius, float height,
                            uint32 sliceCount, uint32 stackCount);
    MeshData CreateSphere(float radius, uint32 sliceCount, uint32 stackCount);
    MeshData CreateGeosphere(float radius, uint32 numSubdivisions);
};
```

### 7.4.1 圆柱体网格生成

圆柱体由底面半径、顶面半径、高度、切片数（sliceCount）和堆叠数（stackCount）定义。分为三部分：侧面、顶盖、底盖。

**侧面几何体**：

圆柱体以原点为中心，沿 y 轴方向。所有顶点位于 `stackCount + 1` 个环上，每个环有 `sliceCount` 个唯一顶点。环的半径增量为 `radiusStep = (topRadius - bottomRadius) / stackCount`。

```cpp
GeometryGenerator::MeshData GeometryGenerator::CreateCylinder(
    float bottomRadius, float topRadius, float height,
    uint32 sliceCount, uint32 stackCount)
{
    MeshData meshData;

    float stackHeight = height / stackCount;
    float radiusStep  = (topRadius - bottomRadius) / stackCount;
    uint32 ringCount  = stackCount + 1;

    // 从底部到顶部逐环生成顶点
    for (uint32 i = 0; i < ringCount; ++i) {
        float y = -0.5f * height + i * stackHeight;
        float r = bottomRadius + i * radiusStep;
        float dTheta = 2.0f * XM_PI / sliceCount;

        for (uint32 j = 0; j <= sliceCount; ++j) {
            Vertex vertex;
            float c = cosf(j * dTheta);
            float s = sinf(j * dTheta);

            vertex.Position = XMFLOAT3(r * c, y, r * s);
            vertex.TexC.x   = (float)j / sliceCount;
            vertex.TexC.y   = 1.0f - (float)i / stackCount;
            vertex.TangentU = XMFLOAT3(-s, 0.0f, c);

            float dr = bottomRadius - topRadius;
            XMFLOAT3 bitangent(dr * c, -height, dr * s);
            XMVECTOR T = XMLoadFloat3(&vertex.TangentU);
            XMVECTOR B = XMLoadFloat3(&bitangent);
            XMStoreFloat3(&vertex.Normal, XMVector3Normalize(XMVector3Cross(T, B)));

            meshData.Vertices.push_back(vertex);
        }
    }

    // 每个 slice 在每个 stack 上形成一个 quad（两个三角形）
    uint32 ringVertexCount = sliceCount + 1;
    for (uint32 i = 0; i < stackCount; ++i) {
        for (uint32 j = 0; j < sliceCount; ++j) {
            meshData.Indices32.push_back(i       * ringVertexCount + j);
            meshData.Indices32.push_back((i + 1) * ringVertexCount + j);
            meshData.Indices32.push_back((i + 1) * ringVertexCount + j + 1);

            meshData.Indices32.push_back(i       * ringVertexCount + j);
            meshData.Indices32.push_back((i + 1) * ringVertexCount + j + 1);
            meshData.Indices32.push_back(i       * ringVertexCount + j + 1);
        }
    }

    BuildCylinderTopCap(...);
    BuildCylinderBottomCap(...);
    return meshData;
}
```

每个环的首尾顶点位置相同但纹理坐标不同，因此需要重复顶点以确保纹理正确映射。

**顶盖/底盖几何体**：

顶盖通过将顶部环的三角形收缩到中心点来近似圆形。底盖类似。

### 7.4.2 球体网格生成

球体由半径、切片数和堆叠数定义。算法与圆柱体类似，但半径随环的变化基于三角函数而非线性插值。

```cpp
GeometryGenerator::MeshData GeometryGenerator::CreateSphere(
    float radius, uint32 sliceCount, uint32 stackCount);
```

通过对球体应用非均匀缩放的世界变换，可以将其变为椭球体。

### 7.4.3 测地线球体生成

普通球体的三角形面积不均匀。测地线球体（Geosphere）使用面积和边长几乎相等的三角形来逼近球面。

生成方法：
1. 从一个**二十面体（icosahedron）**开始。
2. 将每个三角形细分为 4 个相等的小三角形（取各边中点）。
3. 将新顶点投影到球面上（归一化后乘以半径）。
4. 重复步骤 2-3 `numSubdivisions` 次。

```cpp
GeometryGenerator::MeshData GeometryGenerator::CreateGeosphere(
    float radius, uint32 numSubdivisions)
{
    numSubdivisions = std::min<uint32>(numSubdivisions, 6u);

    // 二十面体的 12 个顶点
    const float X = 0.525731f;
    const float Z = 0.850651f;
    XMFLOAT3 pos[12] = {
        XMFLOAT3(-X, 0.0f,  Z), XMFLOAT3(X, 0.0f,  Z),
        XMFLOAT3(-X, 0.0f, -Z), XMFLOAT3(X, 0.0f, -Z),
        XMFLOAT3(0.0f,  Z,  X), XMFLOAT3(0.0f,  Z, -X),
        XMFLOAT3(0.0f, -Z,  X), XMFLOAT3(0.0f, -Z, -X),
        XMFLOAT3( Z,  X, 0.0f), XMFLOAT3(-Z,  X, 0.0f),
        XMFLOAT3( Z, -X, 0.0f), XMFLOAT3(-Z, -X, 0.0f)
    };

    // 二十面体的 20 个面（60 个索引）
    uint32 k[60] = {
        1,4,0,  4,9,0,  4,5,9,  8,5,4,  1,8,4,
        1,10,8, 10,3,8,  8,3,5,  3,2,5,  3,7,2,
        3,10,7, 10,6,7,  6,11,7, 6,0,11, 6,1,0,
        10,1,6, 11,0,9,  2,11,9, 5,2,9,  11,2,7
    };

    MeshData meshData;
    meshData.Vertices.resize(12);
    meshData.Indices32.assign(&k[0], &k[60]);
    for (uint32 i = 0; i < 12; ++i)
        meshData.Vertices[i].Position = pos[i];

    // 细分
    for (uint32 i = 0; i < numSubdivisions; ++i)
        Subdivide(meshData);

    // 投影到球面
    for (uint32 i = 0; i < meshData.Vertices.size(); ++i) {
        XMVECTOR n = XMVector3Normalize(
            XMLoadFloat3(&meshData.Vertices[i].Position));
        XMVECTOR p = radius * n;
        XMStoreFloat3(&meshData.Vertices[i].Position, p);
        XMStoreFloat3(&meshData.Vertices[i].Normal,   n);
        // ... 计算纹理坐标和切线
    }

    return meshData;
}
```

---

## 7.5 Shapes Demo

### 7.5.1 顶点和索引缓冲区

在 Shapes Demo 中，所有几何体的顶点和索引被合并到一个大的顶点缓冲区和索引缓冲区中。绘制时通过 `DrawIndexedInstanced` 的参数来指定要绘制的子区域。

```cpp
void ShapesApp::BuildShapeGeometry() {
    GeometryGenerator geoGen;
    auto box      = geoGen.CreateBox(1.5f, 0.5f, 1.5f, 3);
    auto grid     = geoGen.CreateGrid(20.0f, 30.0f, 60, 40);
    auto sphere   = geoGen.CreateSphere(0.5f, 20, 20);
    auto cylinder = geoGen.CreateCylinder(0.5f, 0.3f, 3.0f, 20, 20);

    // 计算每个对象在合并缓冲区中的偏移
    UINT boxVertexOffset      = 0;
    UINT gridVertexOffset     = (UINT)box.Vertices.size();
    UINT sphereVertexOffset   = gridVertexOffset + (UINT)grid.Vertices.size();
    UINT cylinderVertexOffset = sphereVertexOffset + (UINT)sphere.Vertices.size();

    UINT boxIndexOffset      = 0;
    UINT gridIndexOffset     = (UINT)box.Indices32.size();
    UINT sphereIndexOffset   = gridIndexOffset + (UINT)grid.Indices32.size();
    UINT cylinderIndexOffset = sphereIndexOffset + (UINT)sphere.Indices32.size();

    // 定义 SubmeshGeometry
    SubmeshGeometry boxSubmesh;
    boxSubmesh.IndexCount         = (UINT)box.Indices32.size();
    boxSubmesh.StartIndexLocation = boxIndexOffset;
    boxSubmesh.BaseVertexLocation = boxVertexOffset;
    // ... 类似定义 grid、sphere、cylinder

    // 提取顶点数据并合并
    std::vector<Vertex> vertices(totalVertexCount);
    // ... 将 box、grid、sphere、cylinder 的顶点复制到 vertices

    std::vector<std::uint16_t> indices;
    indices.insert(indices.end(), std::begin(box.GetIndices16()),      std::end(box.GetIndices16()));
    indices.insert(indices.end(), std::begin(grid.GetIndices16()),     std::end(grid.GetIndices16()));
    indices.insert(indices.end(), std::begin(sphere.GetIndices16()),   std::end(sphere.GetIndices16()));
    indices.insert(indices.end(), std::begin(cylinder.GetIndices16()), std::end(cylinder.GetIndices16()));

    // 创建 VertexBufferGPU、IndexBufferGPU ...
    geo->DrawArgs["box"]      = boxSubmesh;
    geo->DrawArgs["grid"]     = gridSubmesh;
    geo->DrawArgs["sphere"]   = sphereSubmesh;
    geo->DrawArgs["cylinder"] = cylinderSubmesh;

    mGeometries[geo->Name] = std::move(geo);
}
```

为了便于管理大量资源，本书后续示例使用 `unordered_map` 通过名称查找对象：

```cpp
std::unordered_map<std::string, std::unique_ptr<MeshGeometry>> mGeometries;
std::unordered_map<std::string, ComPtr<ID3DBlob>>              mShaders;
std::unordered_map<std::string, ComPtr<ID3D12PipelineState>>   mPSOs;
```

### 7.5.2 渲染项

场景中每个可见对象对应一个 `RenderItem`。多个对象可以共享同一 `MeshGeometry`，通过不同的世界矩阵实现实例化绘制，节省内存。

```cpp
void ShapesApp::BuildRenderItems() {
    auto boxRitem = std::make_unique<RenderItem>();
    XMStoreFloat4x4(&boxRitem->World,
        XMMatrixScaling(2.0f, 2.0f, 2.0f) *
        XMMatrixTranslation(0.0f, 0.5f, 0.0f));
    boxRitem->ObjCBIndex         = 0;
    boxRitem->Geo                = mGeometries["shapeGeo"].get();
    boxRitem->PrimitiveType      = D3D_PRIMITIVE_TOPOLOGY_TRIANGLELIST;
    boxRitem->IndexCount         = boxRitem->Geo->DrawArgs["box"].IndexCount;
    boxRitem->StartIndexLocation = boxRitem->Geo->DrawArgs["box"].StartIndexLocation;
    boxRitem->BaseVertexLocation = boxRitem->Geo->DrawArgs["box"].BaseVertexLocation;
    mAllRitems.push_back(std::move(boxRitem));

    // ... 创建 grid、columns、spheres 等渲染项

    for (auto& e : mAllRitems)
        mOpaqueRitems.push_back(e.get());
}
```

### 7.5.3 帧资源和常量缓冲区视图

每个 `FrameResource` 包含一个 `PassCB` 和每个渲染项的 `ObjectCB`。如果有 3 个帧资源和 `n` 个渲染项，则需要 `3n` 个对象 CBV 和 `3` 个通道 CBV，总共 `3(n+1)` 个描述符。

```cpp
void ShapesApp::BuildDescriptorHeaps() {
    UINT objCount = (UINT)mOpaqueRitems.size();

    // 每个对象的每个帧资源需要一个 CBV + 每个帧资源一个 Pass CBV
    UINT numDescriptors = (objCount + 1) * gNumFrameResources;

    // Pass CBV 在描述符堆末尾的偏移
    mPassCbvOffset = objCount * gNumFrameResources;

    D3D12_DESCRIPTOR_HEAP_DESC cbvHeapDesc;
    cbvHeapDesc.NumDescriptors = numDescriptors;
    cbvHeapDesc.Type           = D3D12_DESCRIPTOR_HEAP_TYPE_CBV_SRV_UAV;
    cbvHeapDesc.Flags          = D3D12_DESCRIPTOR_HEAP_FLAG_SHADER_VISIBLE;
    cbvHeapDesc.NodeMask       = 0;

    ThrowIfFailed(md3dDevice->CreateDescriptorHeap(
        &cbvHeapDesc, IID_PPV_ARGS(&mCbvHeap)));
}
```

描述符堆的布局如下：
- 描述符 `0 ~ n-1`：第 0 个帧资源的对象 CBV
- 描述符 `n ~ 2n-1`：第 1 个帧资源的对象 CBV
- 描述符 `2n ~ 3n-1`：第 2 个帧资源的对象 CBV
- 描述符 `3n ~ 3n+2`：第 0/1/2 个帧资源的 Pass CBV

**`GetDescriptorHandleIncrementSize` 方法：**

```cpp
UINT ID3D12Device::GetDescriptorHandleIncrementSize(
    D3D12_DESCRIPTOR_HEAP_TYPE DescriptorHeapType
);
```

| 参数 | 说明 |
|------|------|
| `DescriptorHeapType` | 描述符堆类型 |

| 返回值 | 说明 |
|--------|------|
| `UINT` | 指定类型描述符的增量大小（以字节为单位），硬件相关 |

创建堆时需要缓存这个值，以便在堆中偏移到不同描述符：

```cpp
mRtvDescriptorSize      = md3dDevice->GetDescriptorHandleIncrementSize(
                              D3D12_DESCRIPTOR_HEAP_TYPE_RTV);
mDsvDescriptorSize      = md3dDevice->GetDescriptorHandleIncrementSize(
                              D3D12_DESCRIPTOR_HEAP_TYPE_DSV);
mCbvSrvUavDescriptorSize = md3dDevice->GetDescriptorHandleIncrementSize(
                              D3D12_DESCRIPTOR_HEAP_TYPE_CBV_SRV_UAV);
```

**`CD3DX12_CPU_DESCRIPTOR_HANDLE::Offset` 方法：**

```cpp
// 方式一：偏移指定字节数
void Offset(INT OffsetInDescriptors, UINT DescriptorIncrementSize);

// 方式二：直接偏移字节数（较少使用）
void Offset(INT OffsetInDescriptors);
```

**创建所有 CBV 的代码：**

```cpp
void ShapesApp::BuildConstantBufferViews() {
    UINT objCBByteSize = d3dUtil::CalcConstantBufferByteSize(sizeof(ObjectConstants));
    UINT objCount = (UINT)mOpaqueRitems.size();

    // 对象 CBV
    for (int frameIndex = 0; frameIndex < gNumFrameResources; ++frameIndex) {
        auto objectCB = mFrameResources[frameIndex]->ObjectCB->Resource();
        for (UINT i = 0; i < objCount; ++i) {
            D3D12_GPU_VIRTUAL_ADDRESS cbAddress = objectCB->GetGPUVirtualAddress();
            cbAddress += i * objCBByteSize;

            int heapIndex = frameIndex * objCount + i;
            auto handle = CD3DX12_CPU_DESCRIPTOR_HANDLE(
                mCbvHeap->GetCPUDescriptorHandleForHeapStart());
            handle.Offset(heapIndex, mCbvSrvUavDescriptorSize);

            D3D12_CONSTANT_BUFFER_VIEW_DESC cbvDesc;
            cbvDesc.BufferLocation = cbAddress;
            cbvDesc.SizeInBytes    = objCBByteSize;

            md3dDevice->CreateConstantBufferView(&cbvDesc, handle);
        }
    }

    // Pass CBV（最后 3 个描述符）
    UINT passCBByteSize = d3dUtil::CalcConstantBufferByteSize(sizeof(PassConstants));
    for (int frameIndex = 0; frameIndex < gNumFrameResources; ++frameIndex) {
        auto passCB = mFrameResources[frameIndex]->PassCB->Resource();
        D3D12_GPU_VIRTUAL_ADDRESS cbAddress = passCB->GetGPUVirtualAddress();

        int heapIndex = mPassCbvOffset + frameIndex;
        auto handle = CD3DX12_CPU_DESCRIPTOR_HANDLE(
            mCbvHeap->GetCPUDescriptorHandleForHeapStart());
        handle.Offset(heapIndex, mCbvSrvUavDescriptorSize);

        D3D12_CONSTANT_BUFFER_VIEW_DESC cbvDesc;
        cbvDesc.BufferLocation = cbAddress;
        cbvDesc.SizeInBytes    = passCBByteSize;

        md3dDevice->CreateConstantBufferView(&cbvDesc, handle);
    }
}
```

### 7.5.4 绘制场景

绘制时需要正确计算描述符堆中 CBV 的偏移。

```cpp
void ShapesApp::DrawRenderItems(
    ID3D12GraphicsCommandList* cmdList,
    const std::vector<RenderItem*>& ritems)
{
    UINT objCBByteSize = d3dUtil::CalcConstantBufferByteSize(sizeof(ObjectConstants));

    for (size_t i = 0; i < ritems.size(); ++i) {
        auto ri = ritems[i];

        cmdList->IASetVertexBuffers(0, 1, &ri->Geo->VertexBufferView());
        cmdList->IASetIndexBuffer(&ri->Geo->IndexBufferView());
        cmdList->IASetPrimitiveTopology(ri->PrimitiveType);

        // 计算 CBV 在描述符堆中的索引
        UINT cbvIndex = mCurrFrameResourceIndex * (UINT)mOpaqueRitems.size()
                      + ri->ObjCBIndex;
        auto cbvHandle = CD3DX12_GPU_DESCRIPTOR_HANDLE(
            mCbvHeap->GetGPUDescriptorHandleForHeapStart());
        cbvHandle.Offset(cbvIndex, mCbvSrvUavDescriptorSize);

        cmdList->SetGraphicsRootDescriptorTable(0, cbvHandle);
        cmdList->DrawIndexedInstanced(
            ri->IndexCount, 1,
            ri->StartIndexLocation, ri->BaseVertexLocation, 0);
    }
}
```

主 `Draw` 方法：

```cpp
void ShapesApp::Draw(const GameTimer& gt) {
    auto cmdListAlloc = mCurrFrameResource->CmdListAlloc;
    ThrowIfFailed(cmdListAlloc->Reset());
    ThrowIfFailed(mCommandList->Reset(
        cmdListAlloc.Get(), mPSOs["opaque"].Get()));

    // ... 设置视口、裁剪矩形、资源屏障、清除缓冲区、设置 OM 目标

    ID3D12DescriptorHeap* descriptorHeaps[] = { mCbvHeap.Get() };
    mCommandList->SetDescriptorHeaps(_countof(descriptorHeaps), descriptorHeaps);
    mCommandList->SetGraphicsRootSignature(mRootSignature.Get());

    // 绑定通道 CBV（每轮绘制只设置一次）
    int passCbvIndex = mPassCbvOffset + mCurrFrameResourceIndex;
    auto passCbvHandle = CD3DX12_GPU_DESCRIPTOR_HANDLE(
        mCbvHeap->GetGPUDescriptorHandleForHeapStart());
    passCbvHandle.Offset(passCbvIndex, mCbvSrvUavDescriptorSize);
    mCommandList->SetGraphicsRootDescriptorTable(1, passCbvHandle);

    // 绘制所有不透明对象
    DrawRenderItems(mCommandList.Get(), mOpaqueRitems);

    // ... 资源屏障转换、Present、Signal Fence
}
```

---

## 7.6 深入根签名

根签名定义了绘制调用前需要绑定到管线的资源，以及这些资源映射到哪个着色器寄存器。创建 PSO 时，根签名与着色器程序的组合会被验证。

### 7.6.1 根参数

根签名由**根参数数组**定义。根参数有三种类型：

1. **描述符表（Descriptor Table）**：引用描述符堆中一段连续范围的描述符。
2. **根描述符（Root Descriptor / Inline Descriptor）**：直接在根签名中设置描述符，不需要描述符堆。仅支持常量缓冲区的 CBV 和缓冲区的 SRV/UAV，**不支持纹理 SRV**。
3. **根常量（Root Constants）**：直接在根签名中绑定一组 32 位常量值。

**性能限制**：根签名总共不能超过 **64 DWORD**。

| 根参数类型 | 开销（DWORD） |
|-----------|--------------|
| 描述符表 | 1 |
| 根描述符 | 2 |
| 根常量 | 1 每 32 位常量 |

**`D3D12_ROOT_PARAMETER` 结构体：**

```cpp
typedef struct D3D12_ROOT_PARAMETER {
    D3D12_ROOT_PARAMETER_TYPE ParameterType;
    union {
        D3D12_ROOT_DESCRIPTOR_TABLE DescriptorTable;
        D3D12_ROOT_CONSTANTS        Constants;
        D3D12_ROOT_DESCRIPTOR       Descriptor;
    };
    D3D12_SHADER_VISIBILITY ShaderVisibility;
} D3D12_ROOT_PARAMETER;
```

| 成员 | 说明 |
|------|------|
| `ParameterType` | 根参数类型 |
| `DescriptorTable/Constants/Descriptor` | 根据类型填充的联合体成员 |
| `ShaderVisibility` | 该根参数对哪些着色器可见 |

**`D3D12_ROOT_PARAMETER_TYPE` 枚举：**

```cpp
typedef enum D3D12_ROOT_PARAMETER_TYPE {
    D3D12_ROOT_PARAMETER_TYPE_DESCRIPTOR_TABLE = 0,
    D3D12_ROOT_PARAMETER_TYPE_32BIT_CONSTANTS  = 1,
    D3D12_ROOT_PARAMETER_TYPE_CBV              = 2,
    D3D12_ROOT_PARAMETER_TYPE_SRV              = 3,
    D3D12_ROOT_PARAMETER_TYPE_UAV              = 4
} D3D12_ROOT_PARAMETER_TYPE;
```

**`D3D12_SHADER_VISIBILITY` 枚举：**

```cpp
typedef enum D3D12_SHADER_VISIBILITY {
    D3D12_SHADER_VISIBILITY_ALL      = 0,
    D3D12_SHADER_VISIBILITY_VERTEX   = 1,
    D3D12_SHADER_VISIBILITY_HULL     = 2,
    D3D12_SHADER_VISIBILITY_DOMAIN   = 3,
    D3D12_SHADER_VISIBILITY_GEOMETRY = 4,
    D3D12_SHADER_VISIBILITY_PIXEL    = 5
} D3D12_SHADER_VISIBILITY;
```

限制可见性可能带来优化机会。例如，如果某个资源只被像素着色器使用，可以指定 `D3D12_SHADER_VISIBILITY_PIXEL`。

### 7.6.2 描述符表

描述符表根参数通过 `D3D12_ROOT_DESCRIPTOR_TABLE` 定义。

**`D3D12_ROOT_DESCRIPTOR_TABLE` 结构体：**

```cpp
typedef struct D3D12_ROOT_DESCRIPTOR_TABLE {
    UINT                         NumDescriptorRanges;
    const D3D12_DESCRIPTOR_RANGE* pDescriptorRanges;
} D3D12_ROOT_DESCRIPTOR_TABLE;
```

**`D3D12_DESCRIPTOR_RANGE` 结构体：**

```cpp
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
| `RangeType` | 描述符范围类型：SRV、UAV、CBV、SAMPLER |
| `NumDescriptors` | 范围内描述符数量 |
| `BaseShaderRegister` | 绑定的基础着色器寄存器。例如 `NumDescriptors=3`、`BaseShaderRegister=1`、`RangeType=CBV`，则绑定到 `b1`、`b2`、`b3` |
| `RegisterSpace` | 寄存器空间，默认 `0`。同一编号但不同空间的寄存器不冲突 |
| `OffsetInDescriptorsFromTableStart` | 从表起始位置的偏移。使用 `D3D12_DESCRIPTOR_RANGE_OFFSET_APPEND` 让 Direct3D 自动计算 |

**混合类型的描述符表示例**：

```cpp
CD3DX12_DESCRIPTOR_RANGE descRange[3];
descRange[0].Init(D3D12_DESCRIPTOR_RANGE_TYPE_CBV, 2, 0, 0, 0);  // 2 CBVs
descRange[1].Init(D3D12_DESCRIPTOR_RANGE_TYPE_SRV, 3, 0, 0, 2);  // 3 SRVs
descRange[2].Init(D3D12_DESCRIPTOR_RANGE_TYPE_UAV, 1, 0, 0, 5);  // 1 UAV

slotRootParameter[0].InitAsDescriptorTable(3, descRange, D3D12_SHADER_VISIBILITY_ALL);
```

应用程序需要在描述符堆中绑定一段连续的 6 个描述符（2 CBV + 3 SRV + 1 UAV）。

### 7.6.3 根描述符

根描述符直接在根签名中设置资源虚拟地址，**不需要描述符堆**。

**`D3D12_ROOT_DESCRIPTOR` 结构体：**

```cpp
typedef struct D3D12_ROOT_DESCRIPTOR {
    UINT ShaderRegister;
    UINT RegisterSpace;
} D3D12_ROOT_DESCRIPTOR;
```

绑定根描述符的方法：

```cpp
UINT objCBByteSize = d3dUtil::CalcConstantBufferByteSize(sizeof(ObjectConstants));
D3D12_GPU_VIRTUAL_ADDRESS objCBAddress = objectCB->GetGPUVirtualAddress();
objCBAddress += ri->ObjCBIndex * objCBByteSize;

cmdList->SetGraphicsRootConstantBufferView(0, objCBAddress);
```

**`ID3D12GraphicsCommandList::SetGraphicsRootConstantBufferView` 方法：**

```cpp
void ID3D12GraphicsCommandList::SetGraphicsRootConstantBufferView(
    UINT                        RootParameterIndex,
    D3D12_GPU_VIRTUAL_ADDRESS   BufferLocation
);
```

| 参数 | 说明 |
|------|------|
| `RootParameterIndex` | 根参数索引 |
| `BufferLocation` | 常量缓冲区资源的 GPU 虚拟地址 |

类似的方法还有：
- `SetGraphicsRootShaderResourceView` — 设置根 SRV
- `SetGraphicsRootUnorderedAccessView` — 设置根 UAV

### 7.6.4 根常量

根常量直接在根签名中存储 32 位常量值，同样**不需要描述符堆**。

**`D3D12_ROOT_CONSTANTS` 结构体：**

```cpp
typedef struct D3D12_ROOT_CONSTANTS {
    UINT ShaderRegister;
    UINT RegisterSpace;
    UINT Num32BitValues;
} D3D12_ROOT_CONSTANTS;
```

| 成员 | 说明 |
|------|------|
| `ShaderRegister` | 映射到的常量缓冲区寄存器 |
| `RegisterSpace` | 寄存器空间 |
| `Num32BitValues` | 32 位常量值的数量 |

从着色器角度看，根常量仍然映射到一个常量缓冲区：

```hlsl
cbuffer cbSettings : register(b0) {
    int   gBlurRadius;
    float w0, w1, w2, w3, w4, w5, w6, w7, w8, w9, w10;
};
```

**`ID3D12GraphicsCommandList::SetGraphicsRoot32BitConstants` 方法：**

```cpp
void ID3D12GraphicsCommandList::SetGraphicsRoot32BitConstants(
    UINT        RootParameterIndex,
    UINT        Num32BitValuesToSet,
    const void* pSrcData,
    UINT        DestOffsetIn32BitValues
);
```

| 参数 | 说明 |
|------|------|
| `RootParameterIndex` | 根参数索引 |
| `Num32BitValuesToSet` | 要设置的 32 位值数量 |
| `pSrcData` | 数据源指针 |
| `DestOffsetIn32BitValues` | 目标偏移（以 32 位值为单位） |

```cpp
// 设置根常量到 register b0
cmdList->SetGraphicsRoot32BitConstants(0, 1, &blurRadius, 0);
cmdList->SetGraphicsRoot32BitConstants(0, (UINT)weights.size(), weights.data(), 1);
```

### 7.6.5 复杂根签名示例

考虑一个需要以下资源的着色器：

```hlsl
Texture2D gDiffuseMap : register(t0);

cbuffer cbPerObject : register(b0) { float4x4 gWorld; float4x4 gTexTransform; };
cbuffer cbPass    : register(b1) { /* 视图/投影矩阵等 */ };
cbuffer cbMaterial: register(b2) { float4 gDiffuseAlbedo; float3 gFresnelR0; float gRoughness; float4x4 gMatTransform; };
```

对应的根签名：

```cpp
CD3DX12_DESCRIPTOR_RANGE texTable;
texTable.Init(D3D12_DESCRIPTOR_RANGE_TYPE_SRV, 1, 0);  // register t0

CD3DX12_ROOT_PARAMETER slotRootParameter[4];
// 性能提示：按更新频率从高到低排序
slotRootParameter[0].InitAsDescriptorTable(1, &texTable, D3D12_SHADER_VISIBILITY_PIXEL);
slotRootParameter[1].InitAsConstantBufferView(0);  // register b0
slotRootParameter[2].InitAsConstantBufferView(1);  // register b1
slotRootParameter[3].InitAsConstantBufferView(2);  // register b2

CD3DX12_ROOT_SIGNATURE_DESC rootSigDesc(4, slotRootParameter, 0, nullptr,
    D3D12_ROOT_SIGNATURE_FLAG_ALLOW_INPUT_ASSEMBLER_INPUT_LAYOUT);
```

### 7.6.6 根参数版本控制

根参数的值（实际参数）在每次绘制调用之间可以更改。硬件会自动为每次绘制调用保存根参数的**快照**，因此无需担心覆盖问题。

```cpp
for (size_t i = 0; i < mRitems.size(); ++i) {
    const auto& ri = mRitems[i];

    int cbvOffset = mCurrFrameResourceIndex * (int)mRitems.size() + ri.CbIndex;
    cbvHandle.Offset(cbvOffset, mCbvSrvDescriptorSize);

    cmdList->SetGraphicsRootDescriptorTable(0, cbvHandle);
    cmdList->DrawIndexedInstanced(...);
}
```

每次 `DrawIndexedInstanced` 会使用调用时根参数的当前状态。

**性能提示**：
- 根签名应尽量保持**小**。根参数越大，每帧需要保存的快照就越大。
- 根参数应按**更新频率从高到低**排序。
- 尽量**避免切换根签名**。可以在多个 PSO 之间共享同一个"超级"根签名，即使某些着色器不使用根签名中的所有参数。

---

## 7.7 Land and Waves Demo

### 7.7.1 生成网格顶点

函数 `y = f(x, z)` 的图像是一个曲面。通过在 xz 平面构建网格，对每个网格点应用高度函数，可以得到地形表面。

一个 `m × n` 顶点的网格有 `(m-1) × (n-1)` 个单元格（quad），每个单元格由两个三角形覆盖，共 `2(m-1)(n-1)` 个三角形。网格宽度为 `w`、深度为 `d` 时，x 方向单元格间距为 `dx = w/(n-1)`，z 方向为 `dz = d/(m-1)`。

```cpp
GeometryGenerator::MeshData GeometryGenerator::CreateGrid(
    float width, float depth, uint32 m, uint32 n)
{
    MeshData meshData;
    uint32 vertexCount = m * n;
    uint32 faceCount   = (m - 1) * (n - 1) * 2;

    float halfWidth = 0.5f * width;
    float halfDepth = 0.5f * depth;
    float dx = width  / (n - 1);
    float dz = depth  / (m - 1);
    float du = 1.0f   / (n - 1);
    float dv = 1.0f   / (m - 1);

    meshData.Vertices.resize(vertexCount);
    for (uint32 i = 0; i < m; ++i) {
        float z = halfDepth - i * dz;
        for (uint32 j = 0; j < n; ++j) {
            float x = -halfWidth + j * dx;
            meshData.Vertices[i * n + j].Position  = XMFLOAT3(x, 0.0f, z);
            meshData.Vertices[i * n + j].Normal    = XMFLOAT3(0.0f, 1.0f, 0.0f);
            meshData.Vertices[i * n + j].TangentU  = XMFLOAT3(1.0f, 0.0f, 0.0f);
            meshData.Vertices[i * n + j].TexC.x    = j * du;
            meshData.Vertices[i * n + j].TexC.y    = i * dv;
        }
    }
    // ... 生成索引
    return meshData;
}
```

### 7.7.2 生成网格索引

逐行遍历每个 quad，计算两个三角形的索引：

```
第 (i,j) 个 quad 的顶点索引：

A = i*n + j       B = i*n + (j+1)
C = (i+1)*n + j   D = (i+1)*n + (j+1)

三角形 1: A, B, C
三角形 2: C, B, D
```

```cpp
meshData.Indices32.resize(faceCount * 3);
uint32 k = 0;
for (uint32 i = 0; i < m - 1; ++i) {
    for (uint32 j = 0; j < n - 1; ++j) {
        meshData.Indices32[k]   = i * n + j;
        meshData.Indices32[k+1] = i * n + j + 1;
        meshData.Indices32[k+2] = (i + 1) * n + j;

        meshData.Indices32[k+3] = (i + 1) * n + j;
        meshData.Indices32[k+4] = i * n + j + 1;
        meshData.Indices32[k+5] = (i + 1) * n + j + 1;
        k += 6;
    }
}
```

### 7.7.3 应用高度函数

创建网格后，对每个顶点应用高度函数，并根据高度设置顶点颜色：

```cpp
float LandAndWavesApp::GetHillsHeight(float x, float z) const {
    return 0.3f * (z * sinf(0.1f * x) + x * cosf(0.1f * z));
}

void LandAndWavesApp::BuildLandGeometry() {
    GeometryGenerator geoGen;
    auto grid = geoGen.CreateGrid(160.0f, 160.0f, 50, 50);

    std::vector<Vertex> vertices(grid.Vertices.size());
    for (size_t i = 0; i < grid.Vertices.size(); ++i) {
        auto& p = grid.Vertices[i].Position;
        vertices[i].Pos = p;
        vertices[i].Pos.y = GetHillsHeight(p.x, p.z);

        // 根据高度设置颜色
        if (vertices[i].Pos.y < -10.0f)
            vertices[i].Color = XMFLOAT4(1.0f, 0.96f, 0.62f, 1.0f);  // 沙滩
        else if (vertices[i].Pos.y < 5.0f)
            vertices[i].Color = XMFLOAT4(0.48f, 0.77f, 0.46f, 1.0f);  // 浅绿
        else if (vertices[i].Pos.y < 12.0f)
            vertices[i].Color = XMFLOAT4(0.1f, 0.48f, 0.19f, 1.0f);   // 深绿
        else if (vertices[i].Pos.y < 20.0f)
            vertices[i].Color = XMFLOAT4(0.45f, 0.39f, 0.34f, 1.0f);  // 深褐
        else
            vertices[i].Color = XMFLOAT4(1.0f, 1.0f, 1.0f, 1.0f);     // 白雪
    }
    // ... 创建 GPU 资源
}
```

### 7.7.4 根 CBV

Land and Waves Demo 改用**根描述符**绑定常量缓冲区，从而不再需要 CBV 描述符堆。

根签名变化：

```cpp
CD3DX12_ROOT_PARAMETER slotRootParameter[2];
slotRootParameter[0].InitAsConstantBufferView(0);  // per-object: register(b0)
slotRootParameter[1].InitAsConstantBufferView(1);  // per-pass:   register(b1)

CD3DX12_ROOT_SIGNATURE_DESC rootSigDesc(2, slotRootParameter, 0, nullptr,
    D3D12_ROOT_SIGNATURE_FLAG_ALLOW_INPUT_ASSEMBLER_INPUT_LAYOUT);
```

绘制代码变化：

```cpp
void LandAndWavesApp::Draw(const GameTimer& gt) {
    // ...
    // 绑定通道 CBV（每轮只设置一次）
    auto passCB = mCurrFrameResource->PassCB->Resource();
    mCommandList->SetGraphicsRootConstantBufferView(1, passCB->GetGPUVirtualAddress());

    DrawRenderItems(mCommandList.Get(), mRitemLayer[(int)RenderLayer::Opaque]);
    // ...
}

void LandAndWavesApp::DrawRenderItems(
    ID3D12GraphicsCommandList* cmdList,
    const std::vector<RenderItem*>& ritems)
{
    UINT objCBByteSize = d3dUtil::CalcConstantBufferByteSize(sizeof(ObjectConstants));
    auto objectCB = mCurrFrameResource->ObjectCB->Resource();

    for (size_t i = 0; i < ritems.size(); ++i) {
        auto ri = ritems[i];
        cmdList->IASetVertexBuffers(0, 1, &ri->Geo->VertexBufferView());
        cmdList->IASetIndexBuffer(&ri->Geo->IndexBufferView());
        cmdList->IASetPrimitiveTopology(ri->PrimitiveType);

        D3D12_GPU_VIRTUAL_ADDRESS objCBAddress = objectCB->GetGPUVirtualAddress();
        objCBAddress += ri->ObjCBIndex * objCBByteSize;
        cmdList->SetGraphicsRootConstantBufferView(0, objCBAddress);

        cmdList->DrawIndexedInstanced(ri->IndexCount, 1,
            ri->StartIndexLocation, ri->BaseVertexLocation, 0);
    }
}
```

### 7.7.5 动态顶点缓冲区

之前的示例将顶点存储在**默认缓冲区**中，适用于静态几何体。如果顶点数据需要**每帧更新**（如波浪模拟、粒子系统），则需要**动态顶点缓冲区**。

动态顶点缓冲区本质上就是一个 `UploadBuffer`，但存储的是顶点而不是常量：

```cpp
std::unique_ptr<UploadBuffer<Vertex>> WavesVB = nullptr;
WavesVB = std::make_unique<UploadBuffer<Vertex>>(device, waveVertCount, false);
```

**关键：动态顶点缓冲区也必须是帧资源**，否则可能在 GPU 还没处理完上一帧时就覆盖数据。

每帧更新波浪顶点：

```cpp
void LandAndWavesApp::UpdateWaves(const GameTimer& gt) {
    // 每 0.25 秒产生一个随机波浪扰动
    static float t_base = 0.0f;
    if ((mTimer.TotalTime() - t_base) >= 0.25f) {
        t_base += 0.25f;
        int i = MathHelper::Rand(4, mWaves->RowCount() - 5);
        int j = MathHelper::Rand(4, mWaves->ColumnCount() - 5);
        float r = MathHelper::RandF(0.2f, 0.5f);
        mWaves->Disturb(i, j, r);
    }

    // 更新波浪模拟
    mWaves->Update(gt.DeltaTime());

    // 将新顶点数据上传到当前帧的动态顶点缓冲区
    auto currWavesVB = mCurrFrameResource->WavesVB.get();
    for (int i = 0; i < mWaves->VertexCount(); ++i) {
        Vertex v;
        v.Pos   = mWaves->Position(i);
        v.Color = XMFLOAT4(DirectX::Colors::Blue);
        currWavesVB->CopyData(i, v);
    }

    // 将波浪渲染项的顶点缓冲区指向当前帧的动态缓冲区
    mWavesRitem->Geo->VertexBufferGPU = currWavesVB->Resource();
}
```

动态缓冲区有 CPU 到 GPU 的数据传输开销，因此**静态缓冲区优先于动态缓冲区**。Direct3D 提供了多种减少对动态缓冲区依赖的技术：

1. **顶点着色器动画**：简单的动画可在顶点着色器中完成。
2. **GPU 上的波模拟**：通过渲染到纹理、计算着色器和顶点纹理取样的组合，完全在 GPU 上实现波浪模拟。
3. **几何着色器**：允许 GPU 创建或销毁图元，无需 CPU 参与。
4. **曲面细分阶段**：允许 GPU 动态增加几何细节。

索引缓冲区也可以是动态的。但在 Land and Waves Demo 中，三角拓扑不变，只有顶点高度变化，因此只需要动态顶点缓冲区。
