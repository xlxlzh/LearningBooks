# 第 16 章 实例化与视锥剔除

---

本章学习两个互补的优化技术：

- **实例化（Instancing）**：在同一次 draw call 中绘制同一物体的多个副本（位置/朝向/材质各异）。
- **视锥剔除（Frustum Culling）**：通过 CPU 端简单测试，提前丢弃视锥体外的整组三角形。

---

## 16.1 硬件实例化

实例化指**多次绘制同一物体，但每个副本可以有不同的位置、旋转、缩放、材质、纹理**。常见用例：

- 几种树木重复多次构成森林；
- 几个小行星模型重复构建小行星带；
- 几个角色模型重复构成人群。

如果为每个副本都复制一份顶点和索引数据，会严重浪费内存。更好的做法是只保存一份几何数据（局部空间），然后多次绘制，每次用不同的世界矩阵和材质。

但即使这样，**每次绘制仍有 API 开销**：状态切换、设置常量缓冲、调用 Draw 等。D3D12 比 D3D11 已经大幅减少了 API 开销，但批量绘制依然有性能优势。

> **为什么关心 API 开销？** D3D11 时代很多游戏是 **CPU 瓶颈**（CPU 是瓶颈而非 GPU），原因正是大量独立 draw call 带来的状态切换开销。引擎通常用各种**批处理**技术（见 [Wloka03]）来缓解。硬件实例化就是 API 层面的批处理工具。

### 16.1.1 绘制实例化数据

其实之前所有 demo 都在调用实例化 API（实例数为 1）：

```cpp
cmdList->DrawIndexedInstanced(
    ri->IndexCount,
    1,                        // ← InstanceCount：实例数
    ri->StartIndexLocation,
    ri->BaseVertexLocation,
    0);
```

把第二个参数改成 10，几何就会绘制 10 次。但仅仅画 10 次相同的东西没什么用——还需要给每个实例传不同的数据。

### 16.1.2 实例数据

旧版本（D3D9/11）通常用 **per-instance 顶点缓冲**（`D3D12_INPUT_CLASSIFICATION_PER_INSTANCE_DATA`）传实例数据。D3D12 仍支持，但有更现代的做法：

**用 StructuredBuffer 存放所有实例的数据**——比如要绘制 100 个实例，就建一个有 100 个元素的 StructuredBuffer，每个元素是 per-instance 数据。然后在顶点着色器里用 `SV_InstanceID` 系统值索引这个 buffer。

```hlsl
struct InstanceData {
    float4x4 World;
    float4x4 TexTransform;
    uint MaterialIndex;
    uint InstPad0;
    uint InstPad1;
    uint InstPad2;
};

Texture2D gDiffuseMap[7] : register(t0);

// 放到 space1 防止与纹理数组冲突（纹理数组占 t0~t6 of space0）
StructuredBuffer<InstanceData> gInstanceData : register(t0, space1);
StructuredBuffer<MaterialData> gMaterialData : register(t1, space1);

struct VertexOut {
    float4 PosH    : SV_POSITION;
    float3 PosW    : POSITION;
    float3 NormalW : NORMAL;
    float2 TexC    : TEXCOORD;

    // nointerpolation：索引不允许在三角形内插值
    nointerpolation uint MatIndex : MATINDEX;
};

VertexOut VS(VertexIn vin, uint instanceID : SV_InstanceID)
{
    VertexOut vout = (VertexOut)0.0f;

    // 取本实例的数据
    InstanceData instData = gInstanceData[instanceID];
    float4x4 world        = instData.World;
    float4x4 texTransform = instData.TexTransform;
    uint matIndex         = instData.MaterialIndex;
    vout.MatIndex = matIndex;

    // 通过材质索引取材质
    MaterialData matData = gMaterialData[matIndex];

    float4 posW = mul(float4(vin.PosL, 1.0f), world);
    vout.PosW = posW.xyz;
    vout.NormalW = mul(vin.NormalL, (float3x3)world);
    vout.PosH = mul(posW, gViewProj);

    float4 texC = mul(float4(vin.TexC, 0.0f, 1.0f), texTransform);
    vout.TexC = mul(texC, matData.MatTransform).xy;
    return vout;
}

float4 PS(VertexOut pin) : SV_Target {
    MaterialData matData = gMaterialData[pin.MatIndex];

    // 动态索引纹理数组
    float4 diffuseAlbedo = matData.DiffuseAlbedo *
        gDiffuseMap[matData.DiffuseMapIndex].Sample(gsamLinearWrap, pin.TexC);

    // ... 标准光照 ...
    return litColor;
}
```

注意：**没有 per-object 常量缓冲了**——per-object 数据全部从 InstanceBuffer 取。一次 draw call 就可以绘制大量带不同材质/纹理的物体。

**对应的根签名**：

```cpp
CD3DX12_DESCRIPTOR_RANGE texTable;
texTable.Init(D3D12_DESCRIPTOR_RANGE_TYPE_SRV, 7, 0, 0);

CD3DX12_ROOT_PARAMETER slotRootParameter[4];
// 性能 TIP：从最频繁到最不频繁
slotRootParameter[0].InitAsShaderResourceView(0, 1);  // InstanceData
slotRootParameter[1].InitAsShaderResourceView(1, 1);  // MaterialData
slotRootParameter[2].InitAsConstantBufferView(0);     // PassCB
slotRootParameter[3].InitAsDescriptorTable(1, &texTable,
    D3D12_SHADER_VISIBILITY_PIXEL);                   // 纹理数组
```

**渲染流程**：

```cpp
void InstancingAndCullingApp::Draw(const GameTimer& gt)
{
    // ... 每帧绑定 ...
    auto matBuffer = mCurrFrameResource->MaterialBuffer->Resource();
    mCommandList->SetGraphicsRootShaderResourceView(1,
        matBuffer->GetGPUVirtualAddress());

    auto passCB = mCurrFrameResource->PassCB->Resource();
    mCommandList->SetGraphicsRootConstantBufferView(2,
        passCB->GetGPUVirtualAddress());

    mCommandList->SetGraphicsRootDescriptorTable(3,
        mSrvDescriptorHeap->GetGPUDescriptorHandleForHeapStart());

    DrawRenderItems(mCommandList.Get(), mOpaqueRitems);
}

void DrawRenderItems(ID3D12GraphicsCommandList* cmdList,
                     const std::vector<RenderItem*>& ritems)
{
    for (auto ri : ritems) {
        cmdList->IASetVertexBuffers(0, 1, &ri->Geo->VertexBufferView());
        cmdList->IASetIndexBuffer(&ri->Geo->IndexBufferView());
        cmdList->IASetPrimitiveTopology(ri->PrimitiveType);

        // 设置该 render-item 的实例缓冲
        auto instanceBuffer = mCurrFrameResource->InstanceBuffer->Resource();
        cmdList->SetGraphicsRootShaderResourceView(0,
            instanceBuffer->GetGPUVirtualAddress());

        cmdList->DrawIndexedInstanced(
            ri->IndexCount, ri->InstanceCount,
            ri->StartIndexLocation, ri->BaseVertexLocation, 0);
    }
}
```

### 16.1.3 创建实例缓冲

CPU 侧数据结构（与原来的 per-object 常量缓冲很相似）：

```cpp
struct InstanceData {
    DirectX::XMFLOAT4X4 World        = MathHelper::Identity4x4();
    DirectX::XMFLOAT4X4 TexTransform = MathHelper::Identity4x4();
    UINT MaterialIndex;
    UINT InstancePad0;
    UINT InstancePad1;
    UINT InstancePad2;
};

struct RenderItem {
    // ...
    std::vector<InstanceData> Instances;  // 所有实例的数据
    UINT InstanceCount = 0;               // 可见实例数（剔除后）
    // ...
};
```

实例 buffer 是 **upload 堆上的 StructuredBuffer**，方便每帧更新（CPU 写入可见实例的数据，剔除掉的不写入）。

```cpp
struct FrameResource {
    Microsoft::WRL::ComPtr<ID3D12CommandAllocator> CmdListAlloc;

    std::unique_ptr<UploadBuffer<PassConstants>>  PassCB;
    std::unique_ptr<UploadBuffer<MaterialData>>   MaterialBuffer;
    std::unique_ptr<UploadBuffer<InstanceData>>   InstanceBuffer;

    UINT64 Fence = 0;
};

FrameResource::FrameResource(ID3D12Device* device,
                             UINT passCount,
                             UINT maxInstanceCount,
                             UINT materialCount)
{
    // ...
    PassCB         = std::make_unique<UploadBuffer<PassConstants>>(device, passCount, true);
    MaterialBuffer = std::make_unique<UploadBuffer<MaterialData>>(device, materialCount, false);

    // InstanceBuffer 不是常量缓冲，最后一个参数传 false
    InstanceBuffer = std::make_unique<UploadBuffer<InstanceData>>(device, maxInstanceCount, false);
}
```

---

## 16.2 包围体与视锥

实现视锥剔除前，需要熟悉视锥和各种**包围体（Bounding Volume）**的数学表示。包围体是用简单几何形状近似一个物体的体积——虽然只是近似，但数学表达简单、计算高效。

### 16.2.1 DirectXCollision 库

`DirectXCollision.h` 是 DirectX Math 的一部分，提供常用几何求交测试：射线/三角形、射线/盒、盒/盒、盒/平面、盒/视锥、球/视锥等。

### 16.2.2 包围盒（AABB / OBB）

**轴对齐包围盒（AABB, Axis-Aligned Bounding Box）**：紧贴 mesh 的盒子，且面平行于坐标轴。

- 表示方式 1：最小点 `v_min` + 最大点 `v_max`（遍历所有顶点取分量极值）。
- 表示方式 2：中心 `c` + 范围向量 `e`（到 6 个面的距离）。

DirectX collision 库使用**中心/范围**表示：

```cpp
struct BoundingBox {
    static const size_t CORNER_COUNT = 8;
    XMFLOAT3 Center;    // 盒子中心
    XMFLOAT3 Extents;   // 到每个面的距离
    // ...
};
```

转换公式：`c = 0.5 * (vmin + vmax)`，`e = 0.5 * (vmax - vmin)`。

**手动计算 mesh 的 AABB**：

```cpp
XMFLOAT3 vMinf3(+MathHelper::Infinity, +MathHelper::Infinity, +MathHelper::Infinity);
XMFLOAT3 vMaxf3(-MathHelper::Infinity, -MathHelper::Infinity, -MathHelper::Infinity);
XMVECTOR vMin = XMLoadFloat3(&vMinf3);
XMVECTOR vMax = XMLoadFloat3(&vMaxf3);

for (UINT i = 0; i < vcount; ++i) {
    XMVECTOR P = XMLoadFloat3(&vertices[i].Pos);
    vMin = XMVectorMin(vMin, P);
    vMax = XMVectorMax(vMax, P);
}

BoundingBox bounds;
XMStoreFloat3(&bounds.Center,  0.5f * (vMin + vMax));
XMStoreFloat3(&bounds.Extents, 0.5f * (vMax - vMin));
```

**用 DirectX collision 库便捷构造**：

```cpp
BoundingBox box;
BoundingBox::CreateFromPoints(
    box, vertices.size(),
    &vertices[0].Pos,           // 第一个顶点位置的地址
    sizeof(Vertex::Basic32));   // 步长（到下一个顶点位置的偏移）
```

> **注意**：需要在 CPU 端保留一份 mesh 的顶点列表副本（如 `std::vector`），因为 GPU 上的顶点缓冲 CPU 无法直接读取。这份副本同样会用于拾取（第 17 章）和碰撞检测。

#### 16.2.2.1 旋转后的 AABB

局部空间的 AABB 经过世界变换（含旋转）后，在世界空间中**不再是 AABB**，而是**OBB（Oriented Bounding Box，有向包围盒）**。

应对方案：

1. **每次重新计算世界空间的 AABB**：可能比原物体大很多（变"胖"了）。
2. **把求交测试在物体局部空间内进行**：把视锥/射线变换到局部空间，再与原 AABB 测试（推荐）。
3. **使用 OBB**：保留盒子的朝向信息。

OBB 在 DirectX collision 库中表示为：

```cpp
struct BoundingOrientedBox {
    static const size_t CORNER_COUNT = 8;
    XMFLOAT3 Center;
    XMFLOAT3 Extents;
    XMFLOAT4 Orientation;  // 单位四元数表示盒子→世界的旋转
    // ...
};
```

注：四元数是表示旋转的另一种数学对象，第 22 章会详细介绍。

### 16.2.3 包围球

**包围球**用中心点和半径表示。一种构造方法：先算 AABB，把 AABB 的中心作为球心，再遍历所有顶点取最远距离作为半径。

```cpp
struct BoundingSphere {
    XMFLOAT3 Center;
    float Radius;
    // ...
};
```

> **缩放问题**：局部空间的包围球经过非均匀缩放后可能不再紧贴 mesh。补偿方法是用最大缩放分量缩放半径。最简单的策略是统一所有 mesh 的建模比例，避免缩放。

### 16.2.4 视锥（Frustum）

视锥可用**六个平面**（左/右/上/下/近/远）来定义——这些平面都假设**朝内**，所以视锥就是六个平面正半空间的交集。

DirectX collision 库的视锥表示：

```cpp
struct BoundingFrustum {
    static const size_t CORNER_COUNT = 8;

    XMFLOAT3 Origin;       // 视锥原点
    XMFLOAT4 Orientation;  // 四元数表示朝向

    float RightSlope;      // +X 方向斜率 (X/Z)
    float LeftSlope;       // -X 方向斜率
    float TopSlope;        // +Y 方向斜率 (Y/Z)
    float BottomSlope;     // -Y 方向斜率
    float Near, Far;       // 近/远平面 Z 距离
    // ...
};
```

#### 16.2.4.1 从投影矩阵构造视锥

NDC 空间中视锥是一个标准盒子 `[-1,1] × [-1,1] × [0,1]`。把 NDC 空间中 6 个特征点变换回视图空间，就能恢复视锥参数：

```cpp
inline void XM_CALLCONV BoundingFrustum::CreateFromMatrix(
    BoundingFrustum& Out, FXMMATRIX Projection)
{
    // NDC 空间中视锥的特征点
    static XMVECTORF32 HomogenousPoints[6] = {
        {  1.0f,  0.0f, 1.0f, 1.0f },  // right（在远平面）
        { -1.0f,  0.0f, 1.0f, 1.0f },  // left
        {  0.0f,  1.0f, 1.0f, 1.0f },  // top
        {  0.0f, -1.0f, 1.0f, 1.0f },  // bottom
        {  0.0f,  0.0f, 0.0f, 1.0f },  // near
        {  0.0f,  0.0f, 1.0f, 1.0f }   // far
    };

    XMVECTOR Determinant;
    XMMATRIX matInverse = XMMatrixInverse(&Determinant, Projection);

    // 变换到视图空间
    XMVECTOR Points[6];
    for (size_t i = 0; i < 6; ++i)
        Points[i] = XMVector4Transform(HomogenousPoints[i], matInverse);

    Out.Origin      = XMFLOAT3(0.0f, 0.0f, 0.0f);
    Out.Orientation = XMFLOAT4(0.0f, 0.0f, 0.0f, 1.0f);

    // 计算斜率
    Points[0] = Points[0] * XMVectorReciprocal(XMVectorSplatZ(Points[0]));
    Points[1] = Points[1] * XMVectorReciprocal(XMVectorSplatZ(Points[1]));
    Points[2] = Points[2] * XMVectorReciprocal(XMVectorSplatZ(Points[2]));
    Points[3] = Points[3] * XMVectorReciprocal(XMVectorSplatZ(Points[3]));

    Out.RightSlope  = XMVectorGetX(Points[0]);
    Out.LeftSlope   = XMVectorGetX(Points[1]);
    Out.TopSlope    = XMVectorGetY(Points[2]);
    Out.BottomSlope = XMVectorGetY(Points[3]);

    Out.Near = XMVectorGetZ(Points[4] * XMVectorReciprocal(XMVectorSplatW(Points[4])));
    Out.Far  = XMVectorGetZ(Points[5] * XMVectorReciprocal(XMVectorSplatW(Points[5])));
}
```

#### 16.2.4.2 视锥/包围球求交

**逻辑**：如果存在某个视锥平面 L，使得球**完全在 L 的负半空间**，则球在视锥外；否则球与视锥相交。

球/平面测试：设球心 c、半径 r、平面方程 `n·x + d = 0`，符号距离 `k = n·c + d`：

| 条件 | 含义 |
|------|------|
| `k > r` | 球完全在平面**正**半空间（视锥内侧） |
| `k < -r` | 球完全在平面**负**半空间（视锥外侧）→ 整个测试可以提前结束 |
| `\|k\| <= r` | 球与平面相交 |

```cpp
enum ContainmentType {
    DISJOINT   = 0,  // 完全在视锥外
    INTERSECTS = 1,  // 部分相交
    CONTAINS   = 2,  // 完全在视锥内
};

ContainmentType BoundingFrustum::Contains(const BoundingSphere& sphere) const;
```

#### 16.2.4.3 视锥/AABB 求交

类似球的情况，简化为 6 次 AABB/平面测试。对每个平面，找出**与平面法线最对齐的对角线 PQ**：

- 若 P 在平面前面 → Q 也在前面 → 盒子在前面；
- 若 Q 在平面后面 → P 也在后面 → 盒子在后面；
- 若 P 在后、Q 在前 → 相交。

```cpp
// 找出与平面法线最对齐的对角线方向
for (int j = 0; j < 3; ++j) {
    if (planeNormal[j] >= 0.0f) {
        P[j] = box.minPt[j];
        Q[j] = box.maxPt[j];
    } else {
        P[j] = box.maxPt[j];
        Q[j] = box.minPt[j];
    }
}
```

```cpp
ContainmentType BoundingFrustum::Contains(const BoundingBox& box) const;
```

---

## 16.3 视锥剔除

回想第 5 章：硬件会在裁剪阶段**自动丢弃视锥外的三角形**。但是！这些三角形仍需经过：

- API draw call 开销；
- 顶点着色器；
- 可能的细分、几何着色器；
- 最终在裁剪阶段被丢弃。

如果场景有上百万三角形，浪费的计算量巨大。

**视锥剔除的核心思想**：在 CPU 端用包围体做粗粒度的剔除——如果物体的包围体不与视锥相交，根本就不提交到 GPU。

> 假设相机视野角 90°、远平面无穷远，视锥只占世界体积的 **1/6**——也就是说理论上 5/6 的物体都可以被剔除。实际场景中通常视野角更小、远平面有限，可以剔除更多。

### Demo：5×5×5 骷髅头网格

Demo 中绘制一个 125 个实例的骷髅头网格。AABB 在局部空间中计算（每个骷髅头共用一份）。

每帧 `UpdateInstanceData` 时遍历所有实例，把视锥变换到实例的局部空间后做求交：

```cpp
XMMATRIX view = mCamera.GetView();
XMMATRIX invView = XMMatrixInverse(&XMMatrixDeterminant(view), view);

auto currInstanceBuffer = mCurrFrameResource->InstanceBuffer.get();
for (auto& e : mAllRitems) {
    const auto& instanceData = e->Instances;

    int visibleInstanceCount = 0;
    for (UINT i = 0; i < (UINT)instanceData.size(); ++i) {
        XMMATRIX world         = XMLoadFloat4x4(&instanceData[i].World);
        XMMATRIX texTransform  = XMLoadFloat4x4(&instanceData[i].TexTransform);
        XMMATRIX invWorld      = XMMatrixInverse(&XMMatrixDeterminant(world), world);

        // 视图空间 → 物体局部空间
        XMMATRIX viewToLocal = XMMatrixMultiply(invView, invWorld);

        // 把视锥变换到物体局部空间
        BoundingFrustum localSpaceFrustum;
        mCamFrustum.Transform(localSpaceFrustum, viewToLocal);

        // 在局部空间做视锥/AABB 测试
        if (localSpaceFrustum.Contains(e->Bounds) != DirectX::DISJOINT) {
            InstanceData data;
            XMStoreFloat4x4(&data.World, XMMatrixTranspose(world));
            XMStoreFloat4x4(&data.TexTransform, XMMatrixTranspose(texTransform));
            data.MaterialIndex = instanceData[i].MaterialIndex;

            // 只把可见实例写入 StructuredBuffer
            currInstanceBuffer->CopyData(visibleInstanceCount++, data);
        }
    }
    e->InstanceCount = visibleInstanceCount;
}
```

绘制时只画可见实例：

```cpp
cmdList->DrawIndexedInstanced(
    ri->IndexCount, ri->InstanceCount,  // 用剔除后的实例数
    ri->StartIndexLocation, ri->BaseVertexLocation, 0);
```

**性能对比**：

- 关闭剔除：125 个骷髅头全部提交 → 帧时间 ~33 ms。
- 开启剔除：只画 11~13 个可见的 → 帧时间减半。

每个骷髅头 ~60K 三角形，一次 AABB/视锥测试就能拒掉 60K 三角形——这就是视锥剔除的价值。

---

## 本章小结

| 主题 | 核心要点 |
|------|---------|
| **实例化目的** | 节省 API 开销，一次 draw call 绘制多副本 |
| **现代实现** | StructuredBuffer 存 InstanceData + `SV_InstanceID` 索引 |
| **配合动态索引** | 不同实例可以使用不同材质和纹理 |
| **DrawIndexedInstanced** | 第二个参数 InstanceCount 控制实例数 |
| **包围体类型** | AABB、OBB、Sphere、Frustum |
| **AABB 旋转** | 局部空间 AABB 经世界变换会变成 OBB；最好把视锥变换到局部空间做测试 |
| **DirectXCollision** | 提供包围体类型 + 变换 + 求交测试 |
| **视锥剔除** | CPU 端用包围体测试，避免向 GPU 提交不可见物体 |

---

## 课后练习思路

1. 把 Instancing and Culling Demo 改用包围球代替包围盒。
2. 推导世界空间下的 6 个视锥平面方程（提示：从 view-projection 矩阵推导）。
3. 阅读 `DirectXCollision.h`，熟悉所有求交函数。
4. 实现 OBB / 平面求交测试：找出 OBB 在平面法线方向上的投影长度。

---

> **下一步**：第 17 章将研究"反向"问题——给定鼠标点击的屏幕位置，确定用户点选了哪个 3D 物体（**拾取**）。
