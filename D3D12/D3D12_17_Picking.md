# 第 17 章 拾取（Picking）

---

**拾取问题**：给定鼠标在屏幕上的 2D 坐标，确定用户选择了哪个 3D 物体（或者哪个三角形）。

通常我们是把 3D 顶点变换到屏幕空间；拾取则是**反过来**——把屏幕坐标变换回 3D 空间。但 2D 屏幕点不能唯一对应 3D 点（很多 3D 点会投影到同一屏幕点），这种歧义通常用"取离相机最近的物体"来解决。

**算法步骤**：

1. 给定屏幕点 `s`，找到投影窗口上对应的点 `p`；
2. 在视图空间构造**拾取射线**——从视图空间原点出发，穿过 `p`；
3. 把射线和待测物体变换到同一坐标系；
4. 找出射线与场景中物体的交点，最近的就是被选中的。

---

## 17.1 屏幕到投影窗口的变换

视口矩阵把 NDC 坐标变换到屏幕空间。简化情形（视口覆盖整个后台缓冲，深度范围 [0,1]）下，公式：

```
x_screen = (NDC_x + 1) * 0.5 * Width
y_screen = (1 - NDC_y) * 0.5 * Height
```

**反向求解 NDC**：

```
NDC_x = 2 * x_screen / Width - 1
NDC_y = -2 * y_screen / Height + 1
```

NDC 到**视图空间**的反变换：投影时 `NDC_x = x_v / (r * d)`，所以反推 `x_v = NDC_x * r`。由于投影矩阵的 `P(0,0) = 1 / (r * tan(fovY/2))`，可以直接用：

```
x_v = NDC_x / P(0, 0)
y_v = NDC_y / P(1, 1)
```

射线穿过点 `(x_v, y_v, 1)`（用 z=1 而不是 z=d，因为方向相同）。

```cpp
void PickingApp::Pick(int sx, int sy)
{
    XMFLOAT4X4 P = mCamera.GetProj4x4f();

    // 视图空间中的拾取射线
    float vx = (+2.0f * sx / mClientWidth  - 1.0f) / P(0, 0);
    float vy = (-2.0f * sy / mClientHeight + 1.0f) / P(1, 1);

    XMVECTOR rayOrigin = XMVectorSet(0.0f, 0.0f, 0.0f, 1.0f);
    XMVECTOR rayDir    = XMVectorSet(vx,   vy,   1.0f, 0.0f);

    // ... 后续变换到物体局部空间并测试
}
```

> 视图空间中相机位于原点，所以射线起点 `rayOrigin = (0, 0, 0)`。

---

## 17.2 世界/局部空间拾取射线

视图空间的射线 `r_v(t) = q + tu`。视图矩阵 `V` 把世界 → 视图，所以**逆 V** 把视图 → 世界。世界空间的拾取射线：

```
r_w(t) = V⁻¹ * q + t * V⁻¹ * u
```

- **q 是点**（w=1，受平移影响）；
- **u 是方向向量**（w=0，不受平移影响）。

但更常见的是**把射线变换到物体的局部空间**——因为 mesh 通常在局部空间中定义。物体的世界矩阵 `W`，那 `W⁻¹` 把世界 → 物体局部。每个物体都有自己的局部空间，所以**每次测试都要变换一次射线**。

> **为什么不把 mesh 顶点变换到世界空间？** 一个 mesh 可能有上千个顶点，全部变换太贵。变换射线只需变换 2 个三维量。

```cpp
// 初始：假设没有选中
mPickedRitem->Visible = false;

for (auto ri : mRitemLayer[(int)RenderLayer::Opaque]) {
    auto geo = ri->Geo;
    if (ri->Visible == false) continue;

    XMMATRIX V = mCamera.GetView();
    XMMATRIX invView = XMMatrixInverse(&XMMatrixDeterminant(V), V);

    XMMATRIX W = XMLoadFloat4x4(&ri->World);
    XMMATRIX invWorld = XMMatrixInverse(&XMMatrixDeterminant(W), W);

    // 视图空间 → 物体局部空间
    XMMATRIX toLocal = XMMatrixMultiply(invView, invWorld);

    rayOrigin = XMVector3TransformCoord (rayOrigin, toLocal);  // 点（w=1）
    rayDir    = XMVector3TransformNormal(rayDir,    toLocal);  // 向量（w=0）

    // 求交需要单位方向
    rayDir = XMVector3Normalize(rayDir);

    // ... 求交测试
}
```

| 函数 | 隐含 w | 用途 |
|------|--------|------|
| `XMVector3TransformCoord` | w = 1 | 变换**点** |
| `XMVector3TransformNormal` | w = 0 | 变换**向量** |

---

## 17.3 射线/Mesh 求交

把射线变换到 mesh 的局部空间后，对 mesh 的**每个三角形**做射线/三角形求交测试。如果有命中，记录最近的那个。

**第一步：先做粗筛——射线与 AABB 求交**（类似视锥剔除的思路），命中再做精细测试：

```cpp
float tmin = 0.0f;
if (ri->Bounds.Intersects(rayOrigin, rayDir, tmin))
{
    // 拿到 CPU 侧的顶点/索引数据（GPU buffer 不能直接读）
    auto vertices = (Vertex*)geo->VertexBufferCPU->GetBufferPointer();
    auto indices  = (std::uint32_t*)geo->IndexBufferCPU->GetBufferPointer();
    UINT triCount = ri->IndexCount / 3;

    tmin = MathHelper::Infinity;
    for (UINT i = 0; i < triCount; ++i) {
        UINT i0 = indices[i*3 + 0];
        UINT i1 = indices[i*3 + 1];
        UINT i2 = indices[i*3 + 2];

        XMVECTOR v0 = XMLoadFloat3(&vertices[i0].Pos);
        XMVECTOR v1 = XMLoadFloat3(&vertices[i1].Pos);
        XMVECTOR v2 = XMLoadFloat3(&vertices[i2].Pos);

        float t = 0.0f;
        if (TriangleTests::Intersects(rayOrigin, rayDir, v0, v1, v2, t)) {
            if (t < tmin) {
                tmin = t;
                UINT pickedTriangle = i;

                // 用一个特殊的 render-item 高亮被选三角形
                mPickedRitem->Visible           = true;
                mPickedRitem->IndexCount        = 3;
                mPickedRitem->BaseVertexLocation = 0;
                mPickedRitem->World             = ri->World;
                mPickedRitem->StartIndexLocation = 3 * pickedTriangle;
                mPickedRitem->NumFramesDirty = gNumFrameResources;
            }
        }
    }
}
```

> **CPU 侧顶点数据**：注意拾取需要 `MeshGeometry::VertexBufferCPU`——GPU 顶点缓冲是不能 CPU 读取的。常见做法是同时保留 CPU 副本（供拾取、碰撞、视锥剔除用）。为了节省内存，CPU 副本可以是更简化的版本。

### 17.3.1 射线/AABB 求交

```cpp
bool XM_CALLCONV BoundingBox::Intersects(
    FXMVECTOR Origin,      // 射线起点
    FXMVECTOR Direction,   // 射线方向（必须单位长度）
    float& Dist) const;    // 输出：t 值
```

返回 `true` 表示射线击中盒子。`Dist` 输出最近交点的 `t` 值，对应交点 `p = Origin + t * Direction`。

> 先做 AABB 求交可以提前拒绝大量 mesh，避免遍历每个三角形——这与视锥剔除的优化思路一致。

### 17.3.2 射线/球求交

```cpp
bool XM_CALLCONV BoundingSphere::Intersects(
    FXMVECTOR Origin, FXMVECTOR Direction, float& Dist) const;
```

**推导**：球面 `|p - c| = r`。代入射线 `r(t) = q + t*u`：

```
|q + tu - c|² = r²
```

展开后是关于 `t` 的二次方程，方向是单位向量时一次项系数 `a = u·u = 1`：

```
t² + 2(u·(q-c))t + (|q-c|² - r²) = 0
```

- 判别式 < 0：错过球；
- 判别式 = 0：相切于一点；
- 判别式 > 0：两个交点，取较小的正根。

### 17.3.3 射线/三角形求交

```cpp
bool XM_CALLCONV TriangleTests::Intersects(
    FXMVECTOR Origin,    // 射线起点
    FXMVECTOR Direction, // 射线方向（单位长度）
    FXMVECTOR V0,        // 三角形顶点 v0
    GXMVECTOR V1,        // 顶点 v1
    HXMVECTOR V2,        // 顶点 v2
    float& Dist);        // 输出 t
```

**算法**：设射线 `r(t) = q + tu`，三角形参数化 `T(u, v) = v0 + u(v1-v0) + v(v2-v0)`，其中 `u >= 0, v >= 0, u+v <= 1`。

要解 `r(t) = T(u, v)`：

```
q + tu = v0 + u(v1-v0) + v(v2-v0)
```

令 `e1 = v1-v0, e2 = v2-v0, m = q-v0`，转化为线性方程组，用 Cramer 法则求解：

```
[t]                    [(m × e1) · e2]
[u]  =  1/det(A)  *    [(u × e2) · m ]
[v]                    [(m × e1) · u ]
```

DirectX 库的实现是基于 Möller-Trumbore 算法的优化版本。

---

## 17.4 Demo 应用

Demo 加载一个汽车模型，用户右键点击选中一个三角形，被选三角形以**高亮材质**绘制。

需要一个 `mPickedRitem` 来表示被选三角形，但**初始化时无法填满**（不知道哪个三角形会被选中）。所以加一个 `Visible` 字段，被选时才设为 `true`。

```cpp
RenderItem* mPickedRitem;  // 缓存在 PickingApp 类中

if (TriangleTests::Intersects(rayOrigin, rayDir, v0, v1, v2, t)) {
    if (t < tmin) {
        tmin = t;
        UINT pickedTriangle = i;

        mPickedRitem->Visible           = true;
        mPickedRitem->IndexCount        = 3;
        mPickedRitem->BaseVertexLocation = 0;
        mPickedRitem->World             = ri->World;
        mPickedRitem->StartIndexLocation = 3 * pickedTriangle;
        mPickedRitem->NumFramesDirty = gNumFrameResources;
    }
}
```

### 高亮渲染

被选三角形会被绘制两次：一次普通材质，一次高亮材质。第二次绘制时：

- **深度比较函数改为 `LESS_EQUAL`**——否则深度测试会失败（深度值相等，使用 `LESS` 时通不过）；
- 启用**透明混合**让高亮颜色叠加。

```cpp
DrawRenderItems(mCommandList.Get(), mRitemLayer[(int)RenderLayer::Opaque]);

mCommandList->SetPipelineState(mPSOs["highlight"].Get());
DrawRenderItems(mCommandList.Get(), mRitemLayer[(int)RenderLayer::Highlight]);
```

---

## 本章小结

| 主题 | 核心要点 |
|------|---------|
| **拾取目的** | 根据屏幕鼠标位置确定被选 3D 物体/三角形 |
| **拾取射线** | 从视图空间原点出发，穿过投影窗口对应点 |
| **NDC ↔ 视图空间** | 用投影矩阵的对角元素简化反推 |
| **射线变换** | 起点用 `TransformCoord`（w=1）、方向用 `TransformNormal`（w=0） |
| **变换策略** | 把射线变换到物体局部空间，而非把 mesh 顶点变换到世界 |
| **粗筛-精测** | 先 Ray/AABB 测试，命中再做 Ray/Triangle 测试 |
| **CPU 副本** | 拾取需要 CPU 端的顶点/索引数据（GPU buffer 无法读取） |
| **高亮渲染** | 深度比较改 `LESS_EQUAL`，配合透明混合 |

---

## 课后练习思路

1. 把 Picking Demo 改用包围球而非 AABB 做粗筛。
2. 研究 Ray/AABB 求交算法（Slab 方法）。
3. 上千物体时，仍要做大量 Ray/包围体测试——研究**八叉树（Octree）**或 BVH 如何加速空间查询。同样的策略也能加速视锥剔除。

---

> **下一步**：第 18 章将进入**立方体贴图**——可以高效贴图天空盒，也能模拟环境反射。
