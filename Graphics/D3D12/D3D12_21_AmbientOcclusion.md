# 第 21 章 环境光遮蔽（Ambient Occlusion）

---

光照模型里**环境光（Ambient）项**用来模拟间接光照——但我们一直把它当作常数。结果就是物体处于阴影中时，只有环境光参与，整个物体看起来像被均匀涂色，**完全失去 3D 立体感**。

本章介绍**环境光遮蔽（Ambient Occlusion, AO）**——为环境光提供更好的估计，让物体即使在阴影中也保留立体感。

---

## 21.1 通过光线投射实现 AO

**核心思想**：表面上一点 `p` 接收到的间接光，**与该点周围半球内被遮挡的程度成反比**。

```
        (a) 完全开阔                   (b) 部分被遮挡
        ↓ ↓ ↓ ↓                         ↓ ↓ X X
         \|/                              \|/
          p                                p
        ─────                            ──┬──
                                        几何阻挡
```

**估计方法**：随机往 `p` 的上半球方向投射 `N` 条射线，统计有 `h` 条击中几何体，则：

```
occlusion = h / N
ambientAccess = 1 - occlusion
```

> 只有交点 `q` 距离 `p` 在某阈值 `d` 之内的射线才算作遮挡（远的根本影响不到 `p`）。

我们更喜欢用 `ambientAccess`（可达性）——值越大说明该点接收到的环境光越多。

### 算法实现（CPU 预计算）

```cpp
void AmbientOcclusionApp::BuildVertexAmbientOcclusion(
    std::vector<Vertex::AmbientOcclusion>& vertices,
    const std::vector<UINT>& indices)
{
    UINT vcount = vertices.size();
    UINT tcount = indices.size() / 3;

    // 用八叉树加速射线/三角形求交
    std::vector<XMFLOAT3> positions(vcount);
    for (UINT i = 0; i < vcount; ++i)
        positions[i] = vertices[i].Pos;
    Octree octree;
    octree.Build(positions, indices);

    std::vector<int> vertexSharedCount(vcount);

    // 对每个三角形投射射线，把结果分摊到三个顶点
    for (UINT i = 0; i < tcount; ++i) {
        UINT i0 = indices[i*3+0];
        UINT i1 = indices[i*3+1];
        UINT i2 = indices[i*3+2];

        XMVECTOR v0 = XMLoadFloat3(&vertices[i0].Pos);
        XMVECTOR v1 = XMLoadFloat3(&vertices[i1].Pos);
        XMVECTOR v2 = XMLoadFloat3(&vertices[i2].Pos);

        XMVECTOR edge0 = v1 - v0;
        XMVECTOR edge1 = v2 - v0;
        XMVECTOR normal   = XMVector3Normalize(XMVector3Cross(edge0, edge1));
        XMVECTOR centroid = (v0 + v1 + v2) / 3.0f;

        // 把起点稍微沿法线偏一点，避免自相交
        centroid += 0.001f * normal;

        const int NumSampleRays = 32;
        float numUnoccluded = 0;
        for (int j = 0; j < NumSampleRays; ++j) {
            XMVECTOR randomDir = MathHelper::RandHemisphereUnitVec3(normal);
            // 射线与场景做求交
            if (!octree.RayOctreeIntersect(centroid, randomDir)) {
                numUnoccluded++;
            }
        }
        float ambientAccess = numUnoccluded / NumSampleRays;

        // 分摊到三个顶点
        vertices[i0].AmbientAccess += ambientAccess;
        vertices[i1].AmbientAccess += ambientAccess;
        vertices[i2].AmbientAccess += ambientAccess;
        vertexSharedCount[i0]++;
        vertexSharedCount[i1]++;
        vertexSharedCount[i2]++;
    }

    // 取平均
    for (UINT i = 0; i < vcount; ++i) {
        vertices[i].AmbientAccess /= vertexSharedCount[i];
    }
}
```

> **八叉树**：上千三角形时逐个测试太慢。八叉树把三角形按空间排序，能快速筛掉与射线无可能相交的三角形。

### 局限：只适合静态模型

光线投射的 AO 可以**预计算**并存为顶点属性。但：

- 动态模型（动画/变形）→ 几何变了，预计算失效；
- 即便是静态模型，预计算也要花几秒；
- 实时投射射线对每帧来说太贵。

→ 需要一个**实时近似**：屏幕空间环境光遮蔽（SSAO）。

---

## 21.2 屏幕空间环境光遮蔽（SSAO）

**策略**：每帧——

1. 把场景的**视图空间法线**渲染到一张全屏 `R16G16B16A16_FLOAT` 纹理；同时正常写深度缓冲；
2. 关掉深度，画**全屏 quad** 调用 SSAO 像素着色器；
3. 像素着色器**只用法线纹理和深度缓冲**估计每像素的遮蔽程度，输出 **SSAO 图**；
4. 渲染场景到后台缓冲时，用 SSAO 图调制**环境光项**。

### 21.2.1 法线/深度 Pass

把视图空间法线写到屏幕大小的 16F 纹理：

```hlsl
struct VertexIn {
    float3 PosL    : POSITION;
    float3 NormalL : NORMAL;
    float2 TexC    : TEXCOORD;
    float3 TangentU: TANGENT;
};

struct VertexOut {
    float4 PosH    : SV_POSITION;
    float3 NormalW : NORMAL;
    float3 TangentW: TANGENT;
    float2 TexC    : TEXCOORD;
};

VertexOut VS(VertexIn vin)
{
    VertexOut vout = (VertexOut)0.0f;
    MaterialData matData = gMaterialData[gMaterialIndex];

    vout.NormalW  = mul(vin.NormalL,  (float3x3)gWorld);
    vout.TangentW = mul(vin.TangentU, (float3x3)gWorld);

    float4 posW = mul(float4(vin.PosL, 1.0f), gWorld);
    vout.PosH = mul(posW, gViewProj);

    float4 texC = mul(float4(vin.TexC, 0.0f, 1.0f), gTexTransform);
    vout.TexC = mul(texC, matData.MatTransform).xy;
    return vout;
}

float4 PS(VertexOut pin) : SV_Target
{
    MaterialData matData = gMaterialData[gMaterialIndex];
    float4 diffuseAlbedo = matData.DiffuseAlbedo;
    uint diffuseMapIndex = matData.DiffuseMapIndex;
    diffuseAlbedo *= gTextureMaps[diffuseMapIndex].Sample(
        gsamAnisotropicWrap, pin.TexC);

#ifdef ALPHA_TEST
    clip(diffuseAlbedo.a - 0.1f);
#endif

    pin.NormalW = normalize(pin.NormalW);
    // 注意：SSAO 用插值后的顶点法线（不读法线贴图）
    float3 normalV = mul(pin.NormalW, (float3x3)gView);
    return float4(normalV, 0.0f);
}
```

### 21.2.2 SSAO Pass

```
       eye
        |\
        | \
        |  \ v ←── 从 eye 穿过近裁面到当前像素的向量
        |   \
       近裁面─p─────────  ←── 当前像素
                \
                 q  ←── p 上方半球内的随机点
                /
               r  ←── 从 eye 到 q 路径上的最近可见点
```

为了估计 `p` 处的遮蔽：

1. **重建 p 的视图空间位置**——用深度图采样的 `pz` 配合插值向量 `v`；
2. **生成 N 个随机偏移点 q**（半球内、半径有限）；
3. **找 q 对应的潜在遮蔽点 r**（投影 q 到屏幕，采样深度图）；
4. **测试 r 是否遮蔽 p**；
5. 平均所有样本得到 `occlusionSum`，输出 `ambient-access`。

#### 21.2.2.1 重建视图空间位置

```hlsl
// quad 的 6 个顶点（两个三角形覆盖全屏）
static const float2 gTexCoords[6] = {
    float2(0.0f, 1.0f), float2(0.0f, 0.0f), float2(1.0f, 0.0f),
    float2(0.0f, 1.0f), float2(1.0f, 0.0f), float2(1.0f, 1.0f)
};

VertexOut VS(uint vid : SV_VertexID)
{
    VertexOut vout;
    vout.TexC = gTexCoords[vid];

    // NDC 空间的全屏 quad
    vout.PosH = float4(2.0f*vout.TexC.x - 1.0f,
                       1.0f - 2.0f*vout.TexC.y, 0.0f, 1.0f);

    // quad 角点变换到视图空间近裁面
    float4 ph = mul(vout.PosH, gInvProj);
    vout.PosV = ph.xyz / ph.w;
    return vout;
}
```

`vout.PosV` 跨越 quad 后被插值，对每像素得到从相机指向近裁面对应点的向量 `v`。

**重建 p**：

- 投影矩阵 `gProj[2][2]=A`，`gProj[3][2]=B`，则 `z_ndc = A + B/z_view`；
- 反推 `z_view = B / (z_ndc - A)`；
- 由于 `p = t*v`，且 `pz = t*vz`，故 `t = pz/vz`，得到：

```hlsl
float NdcDepthToViewDepth(float z_ndc)
{
    return gProj[3][2] / (z_ndc - gProj[2][2]);
}

// 在 PS 里
float pz = gDepthMap.SampleLevel(gsamDepthMap, pin.TexC, 0.0f).r;
pz = NdcDepthToViewDepth(pz);
float3 p = (pz / pin.PosV.z) * pin.PosV;
```

#### 21.2.2.2 生成随机样本

随机采样会有聚簇问题（样本扎堆同一方向 → 估计偏差）。**解决方案**：

1. C++ 端预先生成 **14 个均匀分布**的偏移向量（立方体 8 角 + 6 面中心）；
2. 像素着色器里读取一个随机向量纹理，用它**反射**这 14 个固定向量——结果仍是 14 个均匀但随机化的方向。

```cpp
void Ssao::BuildOffsetVectors()
{
    // 8 个立方体角点
    mOffsets[0] = XMFLOAT4(+1.0f, +1.0f, +1.0f, 0.0f);
    mOffsets[1] = XMFLOAT4(-1.0f, -1.0f, -1.0f, 0.0f);
    mOffsets[2] = XMFLOAT4(-1.0f, +1.0f, +1.0f, 0.0f);
    mOffsets[3] = XMFLOAT4(+1.0f, -1.0f, -1.0f, 0.0f);
    mOffsets[4] = XMFLOAT4(+1.0f, +1.0f, -1.0f, 0.0f);
    mOffsets[5] = XMFLOAT4(-1.0f, -1.0f, +1.0f, 0.0f);
    mOffsets[6] = XMFLOAT4(-1.0f, +1.0f, -1.0f, 0.0f);
    mOffsets[7] = XMFLOAT4(+1.0f, -1.0f, +1.0f, 0.0f);
    // 6 个面中心
    mOffsets[8]  = XMFLOAT4(-1.0f, 0.0f, 0.0f, 0.0f);
    mOffsets[9]  = XMFLOAT4(+1.0f, 0.0f, 0.0f, 0.0f);
    mOffsets[10] = XMFLOAT4(0.0f, -1.0f, 0.0f, 0.0f);
    mOffsets[11] = XMFLOAT4(0.0f, +1.0f, 0.0f, 0.0f);
    mOffsets[12] = XMFLOAT4(0.0f, 0.0f, -1.0f, 0.0f);
    mOffsets[13] = XMFLOAT4(0.0f, 0.0f, +1.0f, 0.0f);

    // 随机长度，让样本分布在不同距离上
    for (int i = 0; i < 14; ++i) {
        float s = MathHelper::RandF(0.25f, 1.0f);
        XMVECTOR v = s * XMVector4Normalize(XMLoadFloat4(&mOffsets[i]));
        XMStoreFloat4(&mOffsets[i], v);
    }
}
```

> **对偶取点**：立方体相对面的中心、对角线，确保即使只用部分样本，分布也均匀。

#### 21.2.2.3 生成潜在遮蔽点 r

对每个随机点 `q`：

1. 投影到屏幕，得 `(qx', qy')`；
2. 采样深度图得 `rz`（沿 eye→q 方向最近的可见点深度）；
3. 由 `r = (rz/qz) * q` 重建 `r` 的视图空间完整位置。

#### 21.2.2.4 遮蔽测试

判断 `r` 是否遮蔽 `p` 取决于两个量：

| 量 | 含义 | 处理 |
|----|------|------|
| `\|pz - rz\|` | 视图空间深度距离 | 距离越远遮蔽越小；超过 `gOcclusionFadeEnd` 则不遮蔽；近到 epsilon 之内说明在同一平面，也不算 |
| `dot(n, normalize(r-p))` | 角度因子 | 防止"自遮蔽"——同一平面上的两点不该互相遮蔽 |

#### 21.2.2.5 完成计算

```hlsl
occlusionSum /= gSampleCount;
float access = 1.0f - occlusionSum;
// 增加对比度，让 SSAO 效果更明显
return saturate(pow(access, 4.0f));
```

#### 21.2.2.6 完整 SSAO 着色器

```hlsl
cbuffer cbSsao : register(b0) {
    float4x4 gProj;
    float4x4 gInvProj;
    float4x4 gProjTex;
    float4   gOffsetVectors[14];

    // 模糊用
    float4   gBlurWeights[3];
    float2   gInvRenderTargetSize;

    // 视图空间
    float gOcclusionRadius;
    float gOcclusionFadeStart;
    float gOcclusionFadeEnd;
    float gSurfaceEpsilon;
};

Texture2D gNormalMap    : register(t0);
Texture2D gDepthMap     : register(t1);
Texture2D gRandomVecMap : register(t2);

static const int gSampleCount = 14;

// 遮蔽函数：随距离线性衰减
float OcclusionFunction(float distZ)
{
    //   1.0 ─\
    //         \
    //          \
    //    ─Eps──z0──z1──→ zv
    //
    float occlusion = 0.0f;
    if (distZ > gSurfaceEpsilon) {
        float fadeLength = gOcclusionFadeEnd - gOcclusionFadeStart;
        occlusion = saturate((gOcclusionFadeEnd - distZ) / fadeLength);
    }
    return occlusion;
}

float NdcDepthToViewDepth(float z_ndc)
{
    return gProj[3][2] / (z_ndc - gProj[2][2]);
}

float4 PS(VertexOut pin) : SV_Target
{
    // p：当前像素 / n：p 处法线 / q：随机偏移点 / r：潜在遮蔽点

    // 法线和深度
    float3 n  = gNormalMap.SampleLevel(gsamPointClamp, pin.TexC, 0.0f).xyz;
    float  pz = gDepthMap.SampleLevel(gsamDepthMap, pin.TexC, 0.0f).r;
    pz = NdcDepthToViewDepth(pz);

    // 重建 p
    float3 p = (pz / pin.PosV.z) * pin.PosV;

    // 随机向量（从 [0,1] 映射到 [-1,1]）
    float3 randVec = 2.0f * gRandomVecMap.SampleLevel(
        gsamLinearWrap, 4.0f * pin.TexC, 0.0f).rgb - 1.0f;

    float occlusionSum = 0.0f;

    for (int i = 0; i < gSampleCount; ++i) {
        // 用随机向量反射固定偏移向量
        float3 offset = reflect(gOffsetVectors[i].xyz, randVec);

        // 如果偏移在 (p,n) 定义的平面后面，翻转到前面
        float flip = sign(dot(offset, n));

        // q = p 附近半球内的样本
        float3 q = p + flip * gOcclusionRadius * offset;

        // 把 q 投影到屏幕生成纹理坐标
        float4 projQ = mul(float4(q, 1.0f), gProjTex);
        projQ /= projQ.w;

        // 采样深度图获取 r 的深度
        float rz = gDepthMap.SampleLevel(gsamDepthMap, projQ.xy, 0.0f).r;
        rz = NdcDepthToViewDepth(rz);

        // 重建 r 的完整视图空间位置
        float3 r = (rz / q.z) * q;

        // 遮蔽测试
        float distZ = p.z - r.z;
        float dp = max(dot(n, normalize(r - p)), 0.0f);
        float occlusion = dp * OcclusionFunction(distZ);
        occlusionSum += occlusion;
    }

    occlusionSum /= gSampleCount;
    float access = 1.0f - occlusionSum;

    // 锐化对比度
    return saturate(pow(access, 2.0f));
}
```

> **远距离衰减**：对于深视距场景，深度缓冲精度受限可能产生瑕疵。**简单方案**：让 SSAO 随距离淡出。

### 21.2.3 模糊 Pass

只取 14 个样本，结果**噪声明显**（图 21.7）。直接取上百样本太贵——通用方案是**边缘保持模糊（双边模糊 Bilateral Blur）**：

- 普通模糊会糊掉锐利的边界（场景几何边缘）；
- 双边模糊检查**邻像素的法线和深度是否与中心相近**——相差太大就不参与平均。

```hlsl
float4 PS(VertexOut pin) : SV_Target
{
    // 解包权重数组
    float blurWeights[12] = {
        gBlurWeights[0].x, gBlurWeights[0].y, gBlurWeights[0].z, gBlurWeights[0].w,
        gBlurWeights[1].x, gBlurWeights[1].y, gBlurWeights[1].z, gBlurWeights[1].w,
        gBlurWeights[2].x, gBlurWeights[2].y, gBlurWeights[2].z, gBlurWeights[2].w,
    };

    float2 texOffset;
    if (gHorizontalBlur)
        texOffset = float2(gInvRenderTargetSize.x, 0.0f);
    else
        texOffset = float2(0.0f, gInvRenderTargetSize.y);

    // 中心永远参与
    float4 color = blurWeights[gBlurRadius] * gInputMap.SampleLevel(
        gsamPointClamp, pin.TexC, 0.0);
    float totalWeight = blurWeights[gBlurRadius];

    float3 centerNormal = gNormalMap.SampleLevel(
        gsamPointClamp, pin.TexC, 0.0f).xyz;
    float  centerDepth  = NdcDepthToViewDepth(
        gDepthMap.SampleLevel(gsamDepthMap, pin.TexC, 0.0f).r);

    for (float i = -gBlurRadius; i <= gBlurRadius; ++i) {
        if (i == 0) continue;
        float2 tex = pin.TexC + i * texOffset;

        float3 neighborNormal = gNormalMap.SampleLevel(
            gsamPointClamp, tex, 0.0f).xyz;
        float  neighborDepth  = NdcDepthToViewDepth(
            gDepthMap.SampleLevel(gsamDepthMap, tex, 0.0f).r);

        // 法线接近且深度接近 → 不是跨边界，可以采纳
        if (dot(neighborNormal, centerNormal) >= 0.8f &&
            abs(neighborDepth  - centerDepth) <= 0.2f)
        {
            float weight = blurWeights[i + gBlurRadius];
            color += weight * gInputMap.SampleLevel(
                gsamPointClamp, tex, 0.0);
            totalWeight += weight;
        }
    }

    // 总权重 ≠ 1 时归一化（被舍弃的样本要补偿）
    return color / totalWeight;
}
```

> **半分辨率渲染**：SSAO 是低频效果，渲染到后台缓冲一半大小的纹理就够了——大幅省性能。Demo 中模糊 **4 遍**（每遍水平+垂直）。

### 21.2.4 应用 SSAO

不能用 alpha 混合直接乘到后台缓冲——那会同时影响**漫反射和镜面**。SSAO 只应作用于**环境光项**：

```hlsl
// 顶点着色器：生成屏幕投影纹理坐标
vout.SsaoPosH = mul(posW, gViewProjTex);

// 像素着色器
pin.SsaoPosH /= pin.SsaoPosH.w;
float ambientAccess = gSsaoMap.Sample(
    gsamLinearClamp, pin.SsaoPosH.xy, 0.0f).r;

// 只缩放 ambient
float4 ambient = ambientAccess * gAmbientLight * diffuseAlbedo;
```

**优化深度比较**：法线 Pass 已经写过完整深度。SSAO 应用的渲染只需要在已写入像素上跑——把深度函数改成 `EQUAL`，关闭深度写入：

```cpp
opaquePsoDesc.DepthStencilState.DepthFunc       = D3D12_COMPARISON_FUNC_EQUAL;
opaquePsoDesc.DepthStencilState.DepthWriteMask  = D3D12_DEPTH_WRITE_MASK_ZERO;
```

> **效果可见性**：SSAO 的作用比较微妙——物体在阴影中时尤其明显（此时漫反射和镜面被屏蔽，只剩环境光）。没 SSAO 时阴影里的物体扁平；有 SSAO 时柱子根部、球底、骷髅头四周都会自然变暗，保留立体感。

---

## 本章小结

| 主题 | 核心要点 |
|------|---------|
| **AO 目的** | 改进环境光估计，让阴影中的物体仍有立体感 |
| **AO 原理** | 一点 `p` 的间接光与上半球内被遮蔽程度成反比 |
| **光线投射** | 预计算半球上的随机射线交点 → 静态有效，动态/实时不可行 |
| **SSAO** | 用屏幕空间法线 + 深度估计遮蔽——实时可用 |
| **三个 Pass** | ①法线/深度 ②SSAO 计算 ③双边模糊 |
| **重建位置** | `p = (pz/vz)·v`；`pz` 由 `gProj[3][2]/(z_ndc - gProj[2][2])` 得 |
| **样本分布** | 14 个均匀向量 + 随机向量纹理反射 → 均匀但随机化 |
| **遮蔽测试** | 距离衰减 + 法线/(r-p) 夹角防自遮蔽 |
| **边缘保持模糊** | 邻像素法线/深度差过大就丢弃，避免糊掉边界 |
| **应用方式** | 只缩放 ambient 项；第二遍渲染深度函数改 EQUAL |

---

## 课后练习思路

1. 研究 **KD-Tree、四叉树、八叉树**——空间数据结构如何加速射线/几何求交。
2. 把双边模糊换成**普通高斯模糊**，对比效果差异——能直观感受到边界保持的价值。
3. 思考能否在**计算着色器**上实现 SSAO——优点是共享内存可缓存深度法线，缺点是要切换计算/渲染模式。
4. 把自遮蔽防护（`dp = max(dot(n, normalize(r-p)), 0)`）去掉，复现"到处假遮蔽"的效果，加深对该项必要性的理解。

---

> **下一步**：第 22 章将介绍**四元数**——比矩阵更紧凑的旋转表示法，并支持平滑插值，是角色动画的核心数学工具。
