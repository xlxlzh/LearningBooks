# 第 8 章 光照

---

## 8.1 光与材质的交互

使用光照时，不再直接指定顶点颜色，而是定义**材质（Material）**和**光源（Light）**，然后通过**光照方程**计算顶点颜色。这能产生比直接着色更真实的效果。

材质可以看作是决定光如何与物体表面交互的属性集合，包括：
- 表面反射和吸收的光颜色
- 表面下方的折射率
- 表面光滑程度
- 透明度

通过指定材质属性，可以模拟木头、石头、玻璃、金属、水等真实世界的表面。

### 局部光照与全局光照

**局部光照模型（Local Illumination）**：每个对象独立地被照亮，只考虑光源直接发出的光，忽略从其他物体反射过来的间接光。这是本书（以及大多数实时应用）采用的方式。

**全局光照模型（Global Illumination）**：同时考虑光源直接发出的光和从场景中其他物体反射/折射后的间接光。全局光照能生成接近照片级的真实感，但计算成本极高，通常不适合实时游戏。近似全局光照是活跃的研究领域，常见方法包括：预计算静态物体的间接光照、体素全局光照（VXGI）等。

---

## 8.2 法线向量

**面法线（Face Normal）** 是垂直于多边形所在平面的单位向量，描述多边形的朝向。

**表面法线（Surface Normal）** 是垂直于表面上某点切平面的单位向量，描述该点的朝向。

为了进行光照计算，需要知道三角形网格表面上每一点的法线。实践中，只在**顶点**处指定法线（称为**顶点法线**），然后在光栅化阶段通过插值获得三角形内部各点的近似法线（回顾 5.10.3 节）。这种在像素级别插值法线并进行光照计算的方法称为**逐像素光照（Phong Shading）**。更便宜但精度较低的方法是**逐顶点光照（Gouraud Shading）**——在顶点着色器中计算光照，然后将结果插值到像素。

### 8.2.1 计算法线向量

三角形面法线可通过两条边向量的叉积并归一化得到：

```cpp
XMVECTOR ComputeNormal(FXMVECTOR p0, FXMVECTOR p1, FXMVECTOR p2) {
    XMVECTOR u = p1 - p0;
    XMVECTOR v = p2 - p0;
    return XMVector3Normalize(XMVector3Cross(u, v));
}
```

对于三角形网格，顶点法线通常通过**法线平均**近似：对于共享某顶点的所有多边形，计算它们的面法线，然后求和并归一化。

```cpp
// 伪代码：法线平均
for (每个三角形) {
    Vector3 faceNormal = Cross(e0, e1);
    mVertices[i0].normal += faceNormal;
    mVertices[i1].normal += faceNormal;
    mVertices[i2].normal += faceNormal;
}
for (每个顶点 v) {
    mVertices[i].normal = Normalize(mVertices[i].normal);
}
```

更精细的方案可以使用加权平均，例如按多边形面积加权。

### 8.2.2 变换法线向量

对点/向量使用非均匀缩放变换矩阵 `A` 时，法线向量不能直接用 `A` 变换——变换后的法线可能不再与变换后的切平面正交。

正确的变换矩阵是 **A 的逆-转置矩阵（Inverse-Transpose）**：

```
B = (A⁻¹)ᵀ
```

如果 `A` 是正交矩阵（`Aᵀ = A⁻¹`），则 `B = A`，无需特殊处理。

**`MathHelper::InverseTranspose` 实现：**

```cpp
static XMMATRIX InverseTranspose(CXMMATRIX M) {
    XMMATRIX A = M;
    // 清除平移分量（法线是向量，不受平移影响；
    // 但为避免与其他矩阵拼接时出错，这里做预防措施）
    A.r[3] = XMVectorSet(0.0f, 0.0f, 0.0f, 1.0f);
    XMVECTOR det = XMMatrixDeterminant(A);
    return XMMatrixTranspose(XMMatrixInverse(&det, A));
}
```

变换后的法线可能不再为单位长度，需要在着色器中重新归一化。

---

## 8.3 光照计算中的重要向量

| 向量 | 符号 | 定义 | 说明 |
|------|------|------|------|
| 视线向量 | **v** | `normalize(E - p)` | 从表面点 `p` 指向眼睛位置 `E` 的单位向量 |
| 光向量 | **L** | 指向光源的单位向量 | 与入射光方向 **I** 相反 |
| 法线 | **n** | 表面法线 | 单位向量，垂直于表面切平面 |
| 反射向量 | **r** | `reflect(I, n)` | 入射光关于法线的反射方向 |
| 半角向量 | **h** | `normalize(L + v)` | 光向量与视线向量的平分方向 |

反射向量可通过 HLSL 内置函数 `reflect(I, n)` 计算。

---

## 8.4 兰伯特余弦定律

光照强度与光线入射角度有关。当光线垂直照射表面时最亮，斜射时变暗。

兰伯特余弦定律表明：表面接收到的辐照度（irradiance）与 `cos(θ)` 成正比，其中 `θ` 是法线 **n** 与光向量 **L** 的夹角。

```
max(L · n, 0)
```

当 `L · n < 0` 时光照在表面背面，取 0。

---

## 8.5 漫反射光照

漫反射模拟光进入介质内部、在其中散射、部分被吸收、剩余部分从表面向各个方向均匀散射出去的现象（次表面散射的简化模型）。

漫反射光的颜色由以下因素决定：
- `B_L`：入射光颜色/强度
- `m_d`：材质的漫反射反照率（Diffuse Albedo），范围 `[0, 1]`

漫反射计算公式：

```
c_d = max(L · n, 0) ⊗ B_L ⊗ m_d
```

其中 `⊗` 表示分量乘法。漫反射是**视角无关的**——无论从哪个方向看，表面颜色都一样。

---

## 8.6 环境光

局部光照模型忽略了间接光（从其他物体反射过来的光）。为了弥补这一缺陷，引入**环境光项**：

```
c_a = A_L ⊗ m_d
```

- `A_L`：环境光颜色（间接光的总量）
- `m_d`：漫反射反照率（用同一参数表示对环境光的反射比例）

环境光没有真实的物理计算基础，它只是将物体均匀提亮一点，模拟经过多次散射后从四面八方均匀到达的间接光。

---

## 8.7 镜面反射光照

当光到达两种不同折射率介质的界面时，一部分反射、一部分折射进入介质。这种反射称为**镜面反射（Specular Reflection）**，对应的反射光叫**镜面光**。

对于不透明物体，折射光进入介质内部并产生漫反射。因此人眼看到的反射光是**镜面反射光和漫反射光的组合**。与漫反射不同，镜面反射是**视角相关的**——只有特定方向才能看到明亮的镜面高光。

### 8.7.1 菲涅尔效应

菲涅尔方程描述了入射光中被反射的比例 `R_F`，取值范围 `[0, 1]`。`R_F` 取决于：
1. 材质属性 `R_F(0°)`（垂直入射时的反射率）
2. 入射角 `θ_i`（法线与光向量的夹角）

实时渲染中通常使用 **Schlick 近似**：

```
R_F(θ_i) = R_F(0°) + (1 - R_F(0°)) × (1 - cos(θ_i))⁵
```

常见材质的 `R_F(0°)`：

| 材质 | `R_F(0°)`（RGB） |
|------|------------------|
| 水 | `(0.02, 0.02, 0.02)` |
| 玻璃 | `(0.04, 0.04, 0.04)` |
| 塑料 | `(0.04, 0.04, 0.04)` |
| 金 | `(1.00, 0.71, 0.29)` |
| 银 | `(0.95, 0.93, 0.88)` |
| 铜 | `(0.95, 0.64, 0.54)` |
| 铁 | `(0.56, 0.57, 0.58)` |

**菲涅尔效应的关键观察**：当 `θ_i → 90°`（掠射角）时，反射率趋近于 1。例如看池塘时，垂直往下看能看到水底（反射弱），看向地平线则看到强烈的水面反射。

### 8.7.2 粗糙度

真实物体的表面在微观尺度上是粗糙的。完美的镜面没有粗糙度，所有微法线都与宏观法线一致。粗糙度增加时，微法线偏离宏观法线，导致反射光扩散成一个**镜面瓣（Specular Lobe）**。

**微表面模型**将微观表面建模为大量微小平面（微面元）。对于给定的视线 **v** 和光向量 **L**，只有法线为 **h = normalize(L + v)** 的微面元才能将光反射进眼睛。

粗糙度通过以下分布函数建模：

```
D(h) = cosᵐ(θ_h) = (n · h)ᵐ
```

其中 `m` 控制粗糙度：`m` 越大表面越光滑，镜面瓣越窄越亮；`m` 越小表面越粗糙，镜面瓣越宽越暗。

为保持能量守恒，需要归一化因子：

```
S(h) = (m + 8) / 8 × (n · h)ᵐ
```

### 完整的镜面反射公式

综合菲涅尔效应和粗糙度，镜面反射光为：

```
c_s = max(L · n, 0) ⊗ B_L ⊗ R_F(θ_h) ⊗ S(h)
```

其中 `θ_h` 是半角向量 **h** 与光向量 **L** 的夹角。

---

## 8.8 光照模型总结

综合环境光、漫反射光和镜面反射光，本书采用的完整光照方程为：

```
c = c_a + c_d + c_s

  = A_L ⊗ m_d
  + max(L · n, 0) ⊗ B_L ⊗ m_d
  + max(L · n, 0) ⊗ B_L ⊗ R_F(θ_h) ⊗ (m + 8) / 8 × (n · h)ᵐ
```

| 分量 | 说明 |
|------|------|
| `A_L` | 环境光颜色 |
| `B_L` | 入射直接光颜色/强度 |
| `m_d` | 漫反射反照率 |
| `L` | 光向量（指向光源） |
| `n` | 表面法线 |
| `h` | 半角向量 |
| `R_F(θ_h)` | 菲涅尔反射率（Schlick 近似） |
| `m` | 粗糙度指数（由归一化粗糙度推导） |
| `θ_h` | 半角向量与光向量的夹角 |

**图 8.21 的效果分解**：
- (a) 仅环境光：物体均匀提亮
- (b) 环境光 + 漫反射：因兰伯特定律产生明暗过渡
- (c) 环境光 + 漫反射 + 镜面反射：出现镜面高光

---

## 8.9 材质实现

### 8.9.1 材质数据结构

```cpp
// C++ 端：d3dUtil.h
struct Material {
    std::string Name;
    int MatCBIndex = -1;           // 材质常量缓冲区索引
    int DiffuseSrvHeapIndex = -1;  // 漫反射纹理 SRV 索引（后续章节使用）
    int NumFramesDirty = gNumFrameResources;

    DirectX::XMFLOAT4 DiffuseAlbedo = { 1.0f, 1.0f, 1.0f, 1.0f };
    DirectX::XMFLOAT3 FresnelR0     = { 0.01f, 0.01f, 0.01f };
    float              Roughness      = 0.25f;
    DirectX::XMFLOAT4X4 MatTransform = MathHelper::Identity4x4();
};

// GPU 端常量缓冲区数据
struct MaterialConstants {
    DirectX::XMFLOAT4 DiffuseAlbedo = { 1.0f, 1.0f, 1.0f, 1.0f };
    DirectX::XMFLOAT3 FresnelR0     = { 0.01f, 0.01f, 0.01f };
    float              Roughness      = 0.25f;
    DirectX::XMFLOAT4X4 MatTransform = MathHelper::Identity4x4();
};
```

| 成员 | 说明 |
|------|------|
| `DiffuseAlbedo` | 漫反射反照率，RGB 分量表示反射比例，Alpha 分量用于透明度 |
| `FresnelR0` | 垂直入射时的菲涅尔反射率，决定材质的整体反射特性 |
| `Roughness` | 归一化粗糙度 `[0, 1]`，`0` 为完全光滑，`1` 为最粗糙。光泽度 = `1 - Roughness` |
| `MatTransform` | 材质变换矩阵（纹理变换用，后续章节介绍） |

建模真实材质需要合理设置 `DiffuseAlbedo` 和 `FresnelR0`，并进行艺术调整。例如金属导体吸收所有折射光，因此 `DiffuseAlbedo` 应接近零，但由于简化的光照模型，给一个很小的非零值通常效果更好。

### 8.9.2 材质更新

与对象常量类似，材质也使用脏标记机制：

```cpp
void LitWavesApp::BuildMaterials() {
    auto grass = std::make_unique<Material>();
    grass->Name = "grass";
    grass->MatCBIndex = 0;
    grass->DiffuseAlbedo = XMFLOAT4(0.2f, 0.6f, 0.2f, 1.0f);
    grass->FresnelR0 = XMFLOAT3(0.01f, 0.01f, 0.01f);
    grass->Roughness = 0.125f;

    auto water = std::make_unique<Material>();
    water->Name = "water";
    water->MatCBIndex = 1;
    water->DiffuseAlbedo = XMFLOAT4(0.0f, 0.2f, 0.6f, 1.0f);
    water->FresnelR0 = XMFLOAT3(0.1f, 0.1f, 0.1f);
    water->Roughness = 0.0f;

    mMaterials["grass"] = std::move(grass);
    mMaterials["water"] = std::move(water);
}

void LitWavesApp::UpdateMaterialCBs(const GameTimer& gt) {
    auto currMaterialCB = mCurrFrameResource->MaterialCB.get();

    for (auto& e : mMaterials) {
        Material* mat = e.second.get();
        if (mat->NumFramesDirty > 0) {
            MaterialConstants matConstants;
            matConstants.DiffuseAlbedo = mat->DiffuseAlbedo;
            matConstants.FresnelR0     = mat->FresnelR0;
            matConstants.Roughness     = mat->Roughness;
            currMaterialCB->CopyData(mat->MatCBIndex, matConstants);
            mat->NumFramesDirty--;
        }
    }
}
```

### 8.9.3 带材质的绘制

```cpp
void LitWavesApp::DrawRenderItems(
    ID3D12GraphicsCommandList* cmdList,
    const std::vector<RenderItem*>& ritems)
{
    UINT objCBByteSize = d3dUtil::CalcConstantBufferByteSize(sizeof(ObjectConstants));
    UINT matCBByteSize = d3dUtil::CalcConstantBufferByteSize(sizeof(MaterialConstants));

    auto objectCB = mCurrFrameResource->ObjectCB->Resource();
    auto matCB    = mCurrFrameResource->MaterialCB->Resource();

    for (size_t i = 0; i < ritems.size(); ++i) {
        auto ri = ritems[i];
        cmdList->IASetVertexBuffers(0, 1, &ri->Geo->VertexBufferView());
        cmdList->IASetIndexBuffer(&ri->Geo->IndexBufferView());
        cmdList->IASetPrimitiveTopology(ri->PrimitiveType);

        D3D12_GPU_VIRTUAL_ADDRESS objCBAddress = objectCB->GetGPUVirtualAddress()
                                               + ri->ObjCBIndex * objCBByteSize;
        D3D12_GPU_VIRTUAL_ADDRESS matCBAddress = matCB->GetGPUVirtualAddress()
                                               + ri->Mat->MatCBIndex * matCBByteSize;

        cmdList->SetGraphicsRootConstantBufferView(0, objCBAddress);
        cmdList->SetGraphicsRootConstantBufferView(1, matCBAddress);
        cmdList->DrawIndexedInstanced(ri->IndexCount, 1,
            ri->StartIndexLocation, ri->BaseVertexLocation, 0);
    }
}
```

---

## 8.10 平行光（方向光）

平行光模拟距离非常远的光源（如太阳），所有入射光线可近似为平行。由于光源极远，可以忽略距离衰减，只需指定光的方向和强度。

定义方向光只需要一个**方向向量**（指向光源，与光线传播方向相反）。

---

## 8.11 点光源

点光源向四面八方发射光线，物理原型是灯泡。对于表面上任意点 `P`，光向量为：

```
L = normalize(Q - P)
```

其中 `Q` 是点光源位置。

点光源与平行光的唯一区别就是光向量的计算方式：点光源的光向量因点而异，平行光的光向量恒定。

### 8.11.1 衰减

物理上光强随距离按**平方反比定律**衰减：

```
I(d) = I_0 / d²
```

本书示例采用更简单的**线性衰减**：

```
att(d) = saturate((falloffEnd - d) / (falloffEnd - falloffStart))
```

| 参数 | 说明 |
|------|------|
| `d` | 表面点到光源的距离 |
| `falloffStart` | 开始衰减的距离（此前光强不变） |
| `falloffEnd` | 光强降为 0 的距离 |

当 `d >= falloffEnd` 时，该点不受光源影响。这为着色器提供了早期退出优化的机会。

---

## 8.12 聚光灯

聚光灯有位置 `Q`、瞄准方向 `d`，并通过一个锥形区域辐射光线。物理原型是手电筒。

光向量计算与点光源相同：

```
L = normalize(Q - P)
```

判断点 `P` 是否在聚光灯锥内：计算 `-L` 与 `d` 的夹角 `φ`，若 `φ < φ_max` 则在锥内。

聚光灯强度还需要乘以**聚光灯因子**：

```
k_spot = max(dot(-L, d), 0)^s
```

其中 `s` 控制锥的锐利程度：`s` 越大锥越锐利，边缘过渡越硬。

聚光灯的计算成本最高：需要计算距离（含平方根）、衰减、以及额外的 `k_spot`。

---

## 8.13 HLSL 光照实现

### 8.13.1 光源结构

```cpp
// C++ 端
struct Light {
    DirectX::XMFLOAT3 Strength;     // 光颜色/强度
    float             FalloffStart; // 点/聚光灯：衰减开始距离
    DirectX::XMFLOAT3 Direction;    // 平行/聚光灯：光方向
    float             FalloffEnd;   // 点/聚光灯：衰减结束距离
    DirectX::XMFLOAT3 Position;     // 点/聚光灯：光源位置
    float             SpotPower;    // 聚光灯：锥锐利度
};
```

对应的 HLSL 结构：

```hlsl
struct Light {
    float3 Strength;
    float  FalloffStart; // point/spot only
    float3 Direction;
    float  FalloffEnd;   // point/spot only
    float3 Position;
    float  SpotPower;    // spot only
};
```

**HLSL 结构体打包规则**：元素按 4D 向量打包，单个元素不能跨越两个 4D 向量。上述 `Light` 结构恰好打包为 3 个 4D 向量：

```
Vector 1: (Strength.x, Strength.y, Strength.z, FalloffStart)
Vector 2: (Direction.x, Direction.y, Direction.z, FalloffEnd)
Vector 3: (Position.x, Position.y, Position.z, SpotPower)
```

C++ 和 HLSL 结构体布局必须匹配，否则 `memcpy` 上传数据时会出现错误。

### 8.13.2 公共辅助函数

```hlsl
// LightingUtil.hlsl

// 线性衰减
float CalcAttenuation(float d, float falloffStart, float falloffEnd) {
    return saturate((falloffEnd - d) / (falloffEnd - falloffStart));
}

// Schlick 菲涅尔近似
float3 SchlickFresnel(float3 R0, float3 normal, float3 lightVec) {
    float cosIncidentAngle = saturate(dot(normal, lightVec));
    float f0 = 1.0f - cosIncidentAngle;
    float3 reflectPercent = R0 + (1.0f - R0) * (f0 * f0 * f0 * f0 * f0);
    return reflectPercent;
}

struct Material {
    float4 DiffuseAlbedo;
    float3 FresnelR0;
    float  Shininess;  // Shininess = 1 - roughness
};

// Blinn-Phong 光照计算
float3 BlinnPhong(float3 lightStrength, float3 lightVec,
                  float3 normal, float3 toEye, Material mat)
{
    const float m = mat.Shininess * 256.0f;
    float3 halfVec = normalize(toEye + lightVec);

    float roughnessFactor = (m + 8.0f) * pow(max(dot(halfVec, normal), 0.0f), m) / 8.0f;
    float3 fresnelFactor  = SchlickFresnel(mat.FresnelR0, halfVec, lightVec);

    float3 specAlbedo = fresnelFactor * roughnessFactor;

    // 将高光值缩放到 [0,1] 范围，避免 LDR 渲染中硬截断
    specAlbedo = specAlbedo / (specAlbedo + 1.0f);

    return (mat.DiffuseAlbedo.rgb + specAlbedo) * lightStrength;
}
```

**关于 HDR/LDR**：本书示例使用低动态范围（LDR）渲染，颜色值被截断到 `[0, 1]`。高光值可能超过 1，通过 `specAlbedo / (specAlbedo + 1.0f)` 可以产生更柔和的高光过渡。高动态范围（HDR）渲染使用浮点渲染目标允许颜色超出 `[0, 1]`，然后通过色调映射（tonemapping）压缩到显示范围。

### 8.13.3 方向光计算

```hlsl
float3 ComputeDirectionalLight(Light L, Material mat,
                               float3 normal, float3 toEye)
{
    // 光向量与光传播方向相反
    float3 lightVec = -L.Direction;

    // 兰伯特余弦定律
    float ndotl = max(dot(lightVec, normal), 0.0f);
    float3 lightStrength = L.Strength * ndotl;

    return BlinnPhong(lightStrength, lightVec, normal, toEye, mat);
}
```

### 8.13.4 点光源计算

```hlsl
float3 ComputePointLight(Light L, Material mat,
                         float3 pos, float3 normal, float3 toEye)
{
    float3 lightVec = L.Position - pos;
    float d = length(lightVec);

    // 超出范围则提前退出
    if (d > L.FalloffEnd)
        return 0.0f;

    lightVec /= d;  // 归一化

    float ndotl = max(dot(lightVec, normal), 0.0f);
    float3 lightStrength = L.Strength * ndotl;

    // 距离衰减
    float att = CalcAttenuation(d, L.FalloffStart, L.FalloffEnd);
    lightStrength *= att;

    return BlinnPhong(lightStrength, lightVec, normal, toEye, mat);
}
```

### 8.13.5 聚光灯计算

```hlsl
float3 ComputeSpotLight(Light L, Material mat,
                        float3 pos, float3 normal, float3 toEye)
{
    float3 lightVec = L.Position - pos;
    float d = length(lightVec);

    if (d > L.FalloffEnd)
        return 0.0f;

    lightVec /= d;

    float ndotl = max(dot(lightVec, normal), 0.0f);
    float3 lightStrength = L.Strength * ndotl;

    // 距离衰减
    float att = CalcAttenuation(d, L.FalloffStart, L.FalloffEnd);
    lightStrength *= att;

    // 聚光灯锥形衰减
    float spotFactor = pow(max(dot(-lightVec, L.Direction), 0.0f), L.SpotPower);
    lightStrength *= spotFactor;

    return BlinnPhong(lightStrength, lightVec, normal, toEye, mat);
}
```

### 8.13.6 多光源累加

光照是**可叠加的**。场景中最多支持 16 个光源（可按需调整），按类型分组存储在数组中。

```hlsl
#ifndef NUM_DIR_LIGHTS
    #define NUM_DIR_LIGHTS 1
#endif
#ifndef NUM_POINT_LIGHTS
    #define NUM_POINT_LIGHTS 0
#endif
#ifndef NUM_SPOT_LIGHTS
    #define NUM_SPOT_LIGHTS 0
#endif

#define MaxLights 16

cbuffer cbPass : register(b2) {
    // ...
    float4 gAmbientLight;
    Light  gLights[MaxLights];
};

float4 ComputeLighting(Light gLights[MaxLights], Material mat,
                       float3 pos, float3 normal, float3 toEye,
                       float3 shadowFactor)
{
    float3 result = 0.0f;
    int i = 0;

#if (NUM_DIR_LIGHTS > 0)
    for (i = 0; i < NUM_DIR_LIGHTS; ++i) {
        result += shadowFactor[i] *
            ComputeDirectionalLight(gLights[i], mat, normal, toEye);
    }
#endif

#if (NUM_POINT_LIGHTS > 0)
    for (i = NUM_DIR_LIGHTS; i < NUM_DIR_LIGHTS + NUM_POINT_LIGHTS; ++i) {
        result += ComputePointLight(gLights[i], mat, pos, normal, toEye);
    }
#endif

#if (NUM_SPOT_LIGHTS > 0)
    for (i = NUM_DIR_LIGHTS + NUM_POINT_LIGHTS;
         i < NUM_DIR_LIGHTS + NUM_POINT_LIGHTS + NUM_SPOT_LIGHTS; ++i)
    {
        result += ComputeSpotLight(gLights[i], mat, pos, normal, toEye);
    }
#endif

    return float4(result, 0.0f);
}
```

通过 `#define` 控制每种光源数量，着色器只计算实际需要的光源数。`shadowFactor` 在阴影章节之前设为 `(1, 1, 1)`。

### 8.13.7 主着色器文件

```hlsl
// Default.hlsl
#include "LightingUtil.hlsl"

cbuffer cbPerObject : register(b0) {
    float4x4 gWorld;
};

cbuffer cbMaterial : register(b1) {
    float4 gDiffuseAlbedo;
    float3 gFresnelR0;
    float  gRoughness;
    float4x4 gMatTransform;
};

cbuffer cbPass : register(b2) {
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
    float4   gAmbientLight;
    Light    gLights[MaxLights];
};

struct VertexIn {
    float3 PosL    : POSITION;
    float3 NormalL : NORMAL;
};

struct VertexOut {
    float4 PosH  : SV_POSITION;
    float3 PosW  : POSITION;
    float3 NormalW : NORMAL;
};

VertexOut VS(VertexIn vin) {
    VertexOut vout = (VertexOut)0.0f;

    float4 posW = mul(float4(vin.PosL, 1.0f), gWorld);
    vout.PosW   = posW.xyz;

    // 假设非均匀缩放；否则需要使用世界矩阵的逆-转置
    vout.NormalW = mul(vin.NormalL, (float3x3)gWorld);

    vout.PosH = mul(posW, gViewProj);
    return vout;
}

float4 PS(VertexOut pin) : SV_Target {
    // 插值后的法线可能非单位长度，重新归一化
    pin.NormalW = normalize(pin.NormalW);

    float3 toEyeW = normalize(gEyePosW - pin.PosW);

    // 间接光照（环境光）
    float4 ambient = gAmbientLight * gDiffuseAlbedo;

    // 直接光照
    const float shininess = 1.0f - gRoughness;
    Material mat = { gDiffuseAlbedo, gFresnelR0, shininess };
    float3 shadowFactor = 1.0f;

    float4 directLight = ComputeLighting(gLights, mat, pin.PosW,
                                         pin.NormalW, toEyeW, shadowFactor);

    float4 litColor = ambient + directLight;
    litColor.a = gDiffuseAlbedo.a;

    return litColor;
}
```

---

## 8.14 光照演示

### 8.14.1 顶点格式

光照计算需要表面法线。顶点结构体中添加 `Normal` 字段，不再指定顶点颜色（颜色由光照方程生成）：

```cpp
// C++
struct Vertex {
    DirectX::XMFLOAT3 Pos;
    DirectX::XMFLOAT3 Normal;
};

// HLSL
struct VertexIn {
    float3 PosL    : POSITION;
    float3 NormalL : NORMAL;
};

// 输入布局
mInputLayout = {
    { "POSITION", 0, DXGI_FORMAT_R32G32B32_FLOAT, 0, 0,
      D3D12_INPUT_CLASSIFICATION_PER_VERTEX_DATA, 0 },
    { "NORMAL",   0, DXGI_FORMAT_R32G32B32_FLOAT, 0, 12,
      D3D12_INPUT_CLASSIFICATION_PER_VERTEX_DATA, 0 }
};
```

### 8.14.2 法线计算

`GeometryGenerator` 生成的形状已经带有顶点法线。但地形网格的高度被动态修改，需要自行计算法线。

对于函数 `y = f(x, z)` 表示的曲面，法线可通过偏导数直接计算：

```
T_x = (1, ∂f/∂x, 0)   // x 方向切线
T_z = (0, ∂f/∂z, 1)   // z 方向切线
n = T_z × T_x = (-∂f/∂x, 1, -∂f/∂z)
```

地形高度函数：

```
f(x, z) = 0.3(z·sin(0.1x) + x·cos(0.1z))
```

偏导数为：

```
∂f/∂x = 0.03·z·cos(0.1x) + 0.3·cos(0.1z)
∂f/∂z = 0.3·sin(0.1x) - 0.03·x·sin(0.1z)
```

```cpp
XMFLOAT3 LitWavesApp::GetHillsNormal(float x, float z) const {
    // n = (-df/dx, 1, -df/dz)
    XMFLOAT3 n(
        -0.03f*z*cosf(0.1f*x) - 0.3f*cosf(0.1f*z),
         1.0f,
        -0.3f*sinf(0.1f*x) + 0.03f*x*sinf(0.1f*z)
    );
    XMVECTOR unitNormal = XMVector3Normalize(XMLoadFloat3(&n));
    XMStoreFloat3(&n, unitNormal);
    return n;
}
```

水面法线通过有限差分法近似计算（因为水波没有解析公式）。

### 8.14.3 更新光照方向

光照演示使用一个方向光模拟太阳。用户通过方向键旋转太阳位置。

用球坐标 `(1, θ, φ)` 表示太阳方向（半径固定为 1，因为太阳假设在无限远处）：

```cpp
float mSunTheta = 1.25f * XM_PI;
float mSunPhi   = XM_PIDIV4;

void LitWavesApp::OnKeyboardInput(const GameTimer& gt) {
    const float dt = gt.DeltaTime();
    if (GetAsyncKeyState(VK_LEFT)  & 0x8000) mSunTheta -= 1.0f * dt;
    if (GetAsyncKeyState(VK_RIGHT) & 0x8000) mSunTheta += 1.0f * dt;
    if (GetAsyncKeyState(VK_UP)    & 0x8000) mSunPhi   -= 1.0f * dt;
    if (GetAsyncKeyState(VK_DOWN)  & 0x8000) mSunPhi   += 1.0f * dt;

    mSunPhi = MathHelper::Clamp(mSunPhi, 0.1f, XM_PIDIV2);
}

void LitWavesApp::UpdateMainPassCB(const GameTimer& gt) {
    // ...
    XMVECTOR lightDir = MathHelper::SphericalToCartesian(1.0f, mSunTheta, mSunPhi);
    XMStoreFloat3(&mMainPassCB.Lights[0].Direction, lightDir);
    mMainPassCB.Lights[0].Strength = { 0.8f, 0.8f, 0.7f };
    // ...
}
```

将光源数组放入通道常量缓冲区意味着每轮渲染通道最多支持 16 个光源。对于小型演示足够，但对于大型游戏世界（可能有数百个光源）不够。解决方案包括：将光源数组移到逐对象常量缓冲区（只绑定影响该对象的光源）、使用延迟渲染（Deferred Rendering）或 Forward+ 渲染。

### 8.14.4 根签名更新

光照引入了新的材质常量缓冲区 `cbMaterial`。根签名需要增加一个根描述符：

```cpp
CD3DX12_ROOT_PARAMETER slotRootParameter[3];
slotRootParameter[0].InitAsConstantBufferView(0); // per-object:  register(b0)
slotRootParameter[1].InitAsConstantBufferView(1); // per-material: register(b1)
slotRootParameter[2].InitAsConstantBufferView(2); // per-pass:     register(b2)

CD3DX12_ROOT_SIGNATURE_DESC rootSigDesc(3, slotRootParameter, 0, nullptr,
    D3D12_ROOT_SIGNATURE_FLAG_ALLOW_INPUT_ASSEMBLER_INPUT_LAYOUT);
```
