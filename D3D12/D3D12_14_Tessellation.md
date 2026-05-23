# 第 14 章 曲面细分

---

曲面细分（Tessellation）指渲染管线中三个相关的阶段——它们的共同任务是把几何体细分成更小的三角形，然后再对新生成的顶点做位移处理，从而在网格上增加细节。

为什么不直接建模高多边形网格？以下三点说明了细分的价值：

1. **GPU 上的动态 LOD**：根据物体到相机的距离动态调整细节。远处物体看不清细节，没必要绘制高多边形版本；靠近相机时再增加细分。
2. **物理与动画效率**：在低多边形网格上做物理/动画计算，再细分到高多边形版本，可以节省计算量。
3. **节省内存**：磁盘、内存、显存中只存低多边形资产，GPU 实时细分到高多边形版本。

细分阶段位于顶点着色器和几何着色器之间，并且是**可选阶段**。

```
顶点着色器 → 外壳着色器 (HS) → 细分器 (TS) → 域着色器 (DS) → 几何着色器
                                ↑
                          可选的细分流程
```

---

## 14.1 细分图元类型

启用细分时，提交给输入装配器的不是三角形，而是**控制点片（Patch）**。Direct3D 支持 1~32 个控制点的 patch，对应的图元拓扑类型：

```cpp
D3D_PRIMITIVE_TOPOLOGY_1_CONTROL_POINT_PATCHLIST  = 33,
D3D_PRIMITIVE_TOPOLOGY_2_CONTROL_POINT_PATCHLIST  = 34,
D3D_PRIMITIVE_TOPOLOGY_3_CONTROL_POINT_PATCHLIST  = 35,
D3D_PRIMITIVE_TOPOLOGY_4_CONTROL_POINT_PATCHLIST  = 36,
// ...
D3D_PRIMITIVE_TOPOLOGY_32_CONTROL_POINT_PATCHLIST = 64,
```

- 三角形可视为带 3 个控制点的 patch（`D3D_PRIMITIVE_3_CONTROL_POINT_PATCH`）；
- 简单四边形 patch 提交 4 个控制点（`D3D_PRIMITIVE_4_CONTROL_POINT_PATCH`）；
- 控制点数量更多时，可以用于贝塞尔曲面等数学曲面的构造。

提交 patch 时，除了使用 `IASetPrimitiveTopology` 设置图元类型，还需要在 PSO 中设置：

```cpp
opaquePsoDesc.PrimitiveTopologyType = D3D12_PRIMITIVE_TOPOLOGY_TYPE_PATCH;
```

### 14.1.1 细分与顶点着色器

启用细分后，**顶点着色器变成"控制点着色器"**——它处理的是 patch 的控制点而不是最终顶点。因此通常会在顶点着色器里对控制点做动画或物理变换（频率较低），然后再交给细分阶段做高频率的细化。

---

## 14.2 外壳着色器（Hull Shader）

外壳着色器由两个部分组成：

1. **常量外壳着色器（Constant Hull Shader）**：每个 patch 调用一次，输出**细分因子**。
2. **控制点外壳着色器（Control Point Hull Shader）**：每个输出控制点调用一次。

### 14.2.1 常量外壳着色器

常量外壳着色器每个 patch 评估一次，输出告诉细分阶段"该 patch 要细分到什么程度"的细分因子。

**四边形 patch 的均匀 3 倍细分示例**：

```hlsl
struct PatchTess {
    float EdgeTess[4]   : SV_TessFactor;        // 4 条边的细分因子
    float InsideTess[2] : SV_InsideTessFactor;  // 内部 u/v 方向的细分因子
    // 还可以输出其他每 patch 信息
};

PatchTess ConstantHS(InputPatch<VertexOut, 4> patch, uint patchID : SV_PrimitiveID)
{
    PatchTess pt;
    pt.EdgeTess[0] = 3;  // 左边
    pt.EdgeTess[1] = 3;  // 上边
    pt.EdgeTess[2] = 3;  // 右边
    pt.EdgeTess[3] = 3;  // 下边

    pt.InsideTess[0] = 3;  // u 方向（列）
    pt.InsideTess[1] = 3;  // v 方向（行）
    return pt;
}
```

**输入参数解析**：

| 参数 | 说明 |
|------|------|
| `InputPatch<VertexOut, 4>` | patch 的所有控制点（顶点着色器输出后的类型） |
| `SV_PrimitiveID` | patch ID，用于在 draw call 中唯一标识每个 patch |

**细分因子的语义**：

- **四边形 patch**：4 个边因子 + 2 个内部因子（u/v 方向独立）。
- **三角形 patch**：3 个边因子 + 1 个内部因子。

**关于细分因子的约束**：

- Direct3D 11/12 硬件支持的最大细分因子是 **64**。
- 如果所有细分因子都为 0，patch 会被**直接丢弃**，可用于实现 patch 级的视锥剔除和背面剔除。

### 关于细分量的判断

细分的目的是给网格添加细节，但不该添加用户察觉不到的细节。常用的判断指标：

| 指标 | 说明 |
|------|------|
| **距离** | 物体离相机越远，细节越少 → 远处低细分 |
| **屏幕覆盖** | 估计物体在屏幕上覆盖的像素数 → 覆盖小时低细分 |
| **朝向** | 轮廓边缘的三角形需要更细 |
| **粗糙度** | 粗糙表面需要更多细分 |

**性能建议**（来自 [Story10]）：

1. 当细分因子为 1 时（实际上没在细分），跳过细分阶段直接绘制；
2. 不要细分到三角形小于 **8 像素**——会损失性能；
3. 把所有使用细分的 draw call 批量处理（频繁开关细分代价很高）。

### 14.2.2 控制点外壳着色器

控制点外壳着色器输入一组控制点，输出一组（可能数量不同的）控制点。每个**输出控制点**调用一次。

一个典型应用是**改变曲面表示**：例如把普通三角形（3 控制点）扩展成三次贝塞尔三角形 patch（10 控制点）——这就是所谓的 N-patches 或 PN Triangles 方案。

最简单的"透传"外壳着色器：

```hlsl
struct HullOut {
    float3 PosL : POSITION;
};

[domain("quad")]
[partitioning("integer")]
[outputtopology("triangle_cw")]
[outputcontrolpoints(4)]
[patchconstantfunc("ConstantHS")]
[maxtessfactor(64.0f)]
HullOut HS(InputPatch<VertexOut, 4> p,
           uint i : SV_OutputControlPointID,
           uint patchId : SV_PrimitiveID)
{
    HullOut hout;
    hout.PosL = p[i].PosL;
    return hout;
}
```

**控制点外壳着色器的属性**：

| 属性 | 说明 |
|------|------|
| `domain` | patch 类型：`tri`、`quad`、`isoline` |
| `partitioning` | 细分模式：`integer`（整数）、`fractional_even`（偶数分数）、`fractional_odd`（奇数分数） |
| `outputtopology` | 输出三角形的环绕顺序：`triangle_cw`、`triangle_ccw`、`line` |
| `outputcontrolpoints` | 输出控制点数量。外壳着色器会被调用这么多次 |
| `patchconstantfunc` | 常量外壳着色器函数名（字符串） |
| `maxtessfactor` | 着色器使用的最大细分因子。**最高 64**，告诉驱动便于优化 |

**partitioning 模式区别**：

- **integer**：只在整数细分因子时增加/删除顶点。改变细分等级时有明显的"突变"（popping）。
- **fractional_even / fractional_odd**：新顶点在整数因子处加入，但根据小数部分**逐渐滑动**到位，提供平滑的过渡。

---

## 14.3 细分阶段

细分阶段是**完全由硬件控制的固定功能阶段**——程序员无法编程，只能通过常量外壳着色器输出的细分因子来控制细分行为。

### 14.3.1 四边形 patch 细分示例

不同的边因子和内部因子组合会产生不同的细分模式：

- 内部因子大、边因子小：内部细密，边缘简单；
- 边因子大、内部因子小：边缘细密，内部简单；
- 单一边因子大：该边过渡到内部更细的网格。

### 14.3.2 三角形 patch 细分示例

类似四边形，但因子数量不同：3 个边因子 + 1 个内部因子。

---

## 14.4 域着色器（Domain Shader）

细分阶段为细分出来的每个新顶点调用一次域着色器。**启用细分时**，域着色器扮演着"细分后顶点的顶点着色器"角色——把顶点投影到齐次裁剪空间等工作通常在这里完成。

域着色器接收：

- 常量外壳着色器输出（细分因子 + 其他每 patch 信息）；
- 细分顶点的**参数化坐标** `(u, v)`（注意：不是真实位置！）；
- 控制点外壳着色器输出的所有控制点。

我们需要用参数化坐标 + 控制点**推导出**真实的 3D 位置，最常见的方式是**双线性插值**：

```hlsl
struct DomainOut {
    float4 PosH : SV_POSITION;
};

// 细分器为每个新顶点调用一次 DS
// 启用细分后，DS 相当于"细分后的顶点着色器"
[domain("quad")]
DomainOut DS(PatchTess patchTess,
             float2 uv : SV_DomainLocation,
             const OutputPatch<HullOut, 4> quad)
{
    DomainOut dout;

    // 双线性插值得到顶点位置
    float3 v1 = lerp(quad[0].PosL, quad[1].PosL, uv.x);
    float3 v2 = lerp(quad[2].PosL, quad[3].PosL, uv.x);
    float3 p  = lerp(v1, v2, uv.y);

    float4 posW = mul(float4(p, 1.0f), gWorld);
    dout.PosH = mul(posW, gViewProj);
    return dout;
}
```

> 四边形 patch 的控制点按**行优先**排列。
>
> 三角形 patch 的域着色器类似，但接收**重心坐标** `(u, v, w)` 而非 `(u, v)`，因为贝塞尔三角形 patch 是按重心坐标定义的。

---

## 14.5 细分一个四边形 Demo

本节示例：提交一个四边形 patch 到管线，根据距离动态细分，并用"山地函数"对生成的顶点做位移。

### 顶点缓冲与几何体

```cpp
void BasicTessellationApp::BuildQuadPatchGeometry()
{
    std::array<XMFLOAT3, 4> vertices = {
        XMFLOAT3(-10.0f, 0.0f, +10.0f),
        XMFLOAT3(+10.0f, 0.0f, +10.0f),
        XMFLOAT3(-10.0f, 0.0f, -10.0f),
        XMFLOAT3(+10.0f, 0.0f, -10.0f)
    };
    std::array<std::int16_t, 4> indices = { 0, 1, 2, 3 };

    // ... 创建 VB/IB ...

    SubmeshGeometry quadSubmesh;
    quadSubmesh.IndexCount = 4;
    geo->DrawArgs["quadpatch"] = quadSubmesh;
}
```

### 渲染项

```cpp
auto quadPatchRitem = std::make_unique<RenderItem>();
quadPatchRitem->PrimitiveType =
    D3D_PRIMITIVE_TOPOLOGY_4_CONTROL_POINT_PATCHLIST;  // 关键：4 控制点 patch
quadPatchRitem->IndexCount = 4;
// ...
```

### HLSL：基于距离的细分

```hlsl
struct VertexIn { float3 PosL : POSITION; };
struct VertexOut { float3 PosL : POSITION; };

VertexOut VS(VertexIn vin) {
    VertexOut vout;
    vout.PosL = vin.PosL;
    return vout;
}

struct PatchTess {
    float EdgeTess[4]   : SV_TessFactor;
    float InsideTess[2] : SV_InsideTessFactor;
};

PatchTess ConstantHS(InputPatch<VertexOut, 4> patch,
                     uint patchID : SV_PrimitiveID)
{
    PatchTess pt;

    // patch 中心的世界坐标
    float3 centerL = 0.25f * (patch[0].PosL + patch[1].PosL +
                              patch[2].PosL + patch[3].PosL);
    float3 centerW = mul(float4(centerL, 1.0f), gWorld).xyz;
    float d = distance(centerW, gEyePosW);

    // d <= d0 时细分为 64，d >= d1 时细分为 0
    const float d0 = 20.0f;
    const float d1 = 100.0f;
    float tess = 64.0f * saturate((d1 - d) / (d1 - d0));

    pt.EdgeTess[0] = tess;
    pt.EdgeTess[1] = tess;
    pt.EdgeTess[2] = tess;
    pt.EdgeTess[3] = tess;
    pt.InsideTess[0] = tess;
    pt.InsideTess[1] = tess;
    return pt;
}

struct HullOut { float3 PosL : POSITION; };

[domain("quad")]
[partitioning("integer")]
[outputtopology("triangle_cw")]
[outputcontrolpoints(4)]
[patchconstantfunc("ConstantHS")]
[maxtessfactor(64.0f)]
HullOut HS(InputPatch<VertexOut, 4> p,
           uint i : SV_OutputControlPointID,
           uint patchId : SV_PrimitiveID)
{
    HullOut hout;
    hout.PosL = p[i].PosL;
    return hout;
}
```

### 域着色器：位移映射

仅仅细分还不够——新生成的三角形还都躺在原 patch 平面上。要让网格出现山丘感，必须在域着色器里**位移**新顶点：

```hlsl
struct DomainOut { float4 PosH : SV_POSITION; };

[domain("quad")]
DomainOut DS(PatchTess patchTess,
             float2 uv : SV_DomainLocation,
             const OutputPatch<HullOut, 4> quad)
{
    DomainOut dout;

    // 双线性插值
    float3 v1 = lerp(quad[0].PosL, quad[1].PosL, uv.x);
    float3 v2 = lerp(quad[2].PosL, quad[3].PosL, uv.x);
    float3 p  = lerp(v1, v2, uv.y);

    // 位移映射：用"山地函数"改变 y 坐标
    p.y = 0.3f * (p.z * sin(p.x) + p.x * cos(p.z));

    float4 posW = mul(float4(p, 1.0f), gWorld);
    dout.PosH = mul(posW, gViewProj);
    return dout;
}

float4 PS(DomainOut pin) : SV_Target {
    return float4(1.0f, 1.0f, 1.0f, 1.0f);
}
```

---

## 14.6 三次贝塞尔四边形 Patch

通过更多控制点可以构造平滑的曲面。本节介绍三次贝塞尔曲面——一种 4×4 控制点 patch。

### 14.6.1 贝塞尔曲线

**二次贝塞尔曲线（3 控制点 p₀, p₁, p₂）**通过反复线性插值定义：

```
p₀₁(t) = lerp(p₀, p₁, t)
p₁₂(t) = lerp(p₁, p₂, t)
p(t)   = lerp(p₀₁(t), p₁₂(t), t)
```

展开得到参数方程：

```
p(t) = (1-t)² · p₀ + 2t(1-t) · p₁ + t² · p₂
```

**三次贝塞尔曲线（4 控制点 p₀, p₁, p₂, p₃）**类似，通过三层重复插值：

```
p(t) = (1-t)³ · p₀ + 3t(1-t)² · p₁ + 3t²(1-t) · p₂ + t³ · p₃
```

这就是著名的 **Bernstein 基函数**展开。对 3 次而言：

```
B₀³(t) = (1-t)³
B₁³(t) = 3t(1-t)²
B₂³(t) = 3t²(1-t)
B₃³(t) = t³
```

### 14.6.2 三次贝塞尔曲面

考虑一个 4×4 控制点 patch。每一行的 4 个控制点定义一条三次贝塞尔曲线 `qᵢ(u)`。然后用 4 条曲线在 `u₀` 处的取值（共 4 个点）再定义一条 v 方向的贝塞尔曲线 `p(v)`。

让 u 与 v 都变化，扫出一个**三次贝塞尔曲面**：

```
p(u, v) = Σᵢ Σⱼ Bᵢ³(u) · Bⱼ³(v) · Pᵢⱼ
```

其中 i, j ∈ {0, 1, 2, 3}，Pᵢⱼ 是 16 个控制点。

**偏导数**（用于法线计算）：分别对 u 和 v 求导。`∂p/∂u × ∂p/∂v` 即为法线方向。

### 14.6.3 求值代码

```hlsl
float4 BernsteinBasis(float t) {
    float invT = 1.0f - t;
    return float4(invT * invT * invT,
                  3.0f * t * invT * invT,
                  3.0f * t * t * invT,
                  t * t * t);
}

float3 CubicBezierSum(const OutputPatch<HullOut, 16> bezpatch,
                      float4 basisU, float4 basisV)
{
    float3 sum = float3(0, 0, 0);
    sum  = basisV.x * (basisU.x*bezpatch[0].PosL  + basisU.y*bezpatch[1].PosL  +
                       basisU.z*bezpatch[2].PosL  + basisU.w*bezpatch[3].PosL);
    sum += basisV.y * (basisU.x*bezpatch[4].PosL  + basisU.y*bezpatch[5].PosL  +
                       basisU.z*bezpatch[6].PosL  + basisU.w*bezpatch[7].PosL);
    sum += basisV.z * (basisU.x*bezpatch[8].PosL  + basisU.y*bezpatch[9].PosL  +
                       basisU.z*bezpatch[10].PosL + basisU.w*bezpatch[11].PosL);
    sum += basisV.w * (basisU.x*bezpatch[12].PosL + basisU.y*bezpatch[13].PosL +
                       basisU.z*bezpatch[14].PosL + basisU.w*bezpatch[15].PosL);
    return sum;
}
```

注意 `CubicBezierSum` 不关心传入的具体基函数——传入 `BernsteinBasis(t)` 算位置，传入它的导数算切向量。

### 14.6.4 定义 patch 几何

```cpp
void BezierPatchApp::BuildQuadPatchGeometry()
{
    std::array<XMFLOAT3, 16> vertices = {
        // Row 0
        XMFLOAT3(-10.0f, -10.0f, +15.0f), XMFLOAT3(-5.0f,  0.0f, +15.0f),
        XMFLOAT3(+5.0f,   0.0f, +15.0f), XMFLOAT3(+10.0f,  0.0f, +15.0f),
        // Row 1
        XMFLOAT3(-15.0f,  0.0f, +5.0f),  XMFLOAT3(-5.0f,   0.0f, +5.0f),
        XMFLOAT3(+5.0f,  20.0f, +5.0f),  XMFLOAT3(+15.0f,  0.0f, +5.0f),
        // Row 2
        XMFLOAT3(-15.0f,  0.0f, -5.0f),  XMFLOAT3(-5.0f,   0.0f, -5.0f),
        XMFLOAT3(+5.0f,   0.0f, -5.0f),  XMFLOAT3(+15.0f,  0.0f, -5.0f),
        // Row 3
        XMFLOAT3(-10.0f, 10.0f, -15.0f), XMFLOAT3(-5.0f,   0.0f, -15.0f),
        XMFLOAT3(+5.0f,   0.0f, -15.0f), XMFLOAT3(+25.0f, 10.0f, -15.0f)
    };
    std::array<std::int16_t, 16> indices = {
        0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15
    };
    // ... 创建 VB/IB ...
}
```

渲染项使用 16 控制点的 patch 拓扑：

```cpp
quadPatchRitem->PrimitiveType =
    D3D11_PRIMITIVE_TOPOLOGY_16_CONTROL_POINT_PATCHLIST;
```

**域着色器中评估贝塞尔曲面**：

```hlsl
[domain("quad")]
DomainOut DS(PatchTess patchTess,
             float2 uv : SV_DomainLocation,
             const OutputPatch<HullOut, 16> bezPatch)
{
    DomainOut dout;

    float4 basisU = BernsteinBasis(uv.x);
    float4 basisV = BernsteinBasis(uv.y);
    float3 p = CubicBezierSum(bezPatch, basisU, basisV);

    float4 posW = mul(float4(p, 1.0f), gWorld);
    dout.PosH = mul(posW, gViewProj);
    return dout;
}
```

注：控制点不必构成均匀网格，可以自由分布。

---

## 本章小结

| 主题 | 核心要点 |
|------|---------|
| **细分阶段** | 可选的 HS → TS → DS 三阶段。HS/DS 可编程，TS 由硬件控制 |
| **优势** | GPU 动态 LOD、低频物理动画、节省内存 |
| **图元类型** | `*_CONTROL_POINT_PATCHLIST`，控制点数 1~32 |
| **常量外壳着色器** | 每 patch 评估一次，输出细分因子 |
| **控制点外壳着色器** | 每个输出控制点评估一次，可改变曲面表示 |
| **细分器** | 固定功能，根据细分因子在 patch 参数域内生成新顶点 |
| **域着色器** | 每个细分顶点评估一次，结合参数坐标和控制点计算最终位置 |
| **细分量判断** | 距离、屏幕覆盖、朝向、粗糙度 |
| **性能建议** | 因子 ≈ 1 时跳过细分；避免三角形 < 8 像素；批量绘制 |
| **partitioning** | `integer`（突变）、`fractional_even/odd`（平滑过渡） |
| **贝塞尔曲面** | 4×4 控制点 + Bernstein 基函数 → 平滑曲面 |

---

## 课后练习思路

1. 用三角形 patch 代替四边形 patch 重做 Basic Tessellation Demo。
2. 基于距离将 icosahedron 细分成球体。
3. 把 Basic Tessellation Demo 改为固定细分因子，尝试不同的边/内部因子组合。
4. 用 `fractional_even` / `fractional_odd` 替换 `integer` 分区模式，观察平滑过渡效果。
5. 推导二次贝塞尔曲面的参数方程。
6. 修改 Bezier Patch Demo 的控制点，观察曲面变化。
7. 重做 Bezier Patch Demo，改用 9 控制点（二次）贝塞尔曲面。
8. 给贝塞尔曲面加入光照——在域着色器里用偏导数计算法线（`∂p/∂u × ∂p/∂v`）。
9. 研究并实现贝塞尔三角形 patch。

---

> **下一步**：第 15 章将进入"第三部分 高级专题"，开始讨论第一人称相机系统与 Shader Model 5.1 的动态索引技术。
