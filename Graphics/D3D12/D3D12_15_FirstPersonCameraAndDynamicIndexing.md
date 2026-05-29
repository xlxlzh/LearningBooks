# 第 15 章 第一人称相机与动态索引

---

本章涵盖两个独立的短主题：

1. 设计一个**第一人称相机系统**，替换之前 demo 中使用的轨道（orbital）相机。
2. 介绍 **Shader Model 5.1** 新增的**动态索引**特性——在着色器中动态访问纹理数组（`Texture2D gDiffuseMap[n]`）。这类似第 12 章用过的 `Texture2DArray`，但数组里的纹理可以是不同尺寸和格式，更灵活。

---

## 15.1 视图变换回顾

视图空间是与相机绑定的坐标系：相机位于原点，视线沿 +Z 轴方向（左手系），+X 指向相机右侧，+Y 指向相机上方。

设 `Q_W`、`u_W`、`v_W`、`w_W` 分别表示视图空间的原点和三个坐标轴在世界空间中的齐次坐标，则**视图空间到世界空间**的变换矩阵：

```
       | u.x  u.y  u.z  0 |
W   =  | v.x  v.y  v.z  0 |
       | w.x  w.y  w.z  0 |
       | Q.x  Q.y  Q.z  1 |
```

我们需要的是**逆变换**——从世界空间到视图空间，即 `W⁻¹`。由于世界变换可分解为 `W = RT`（先旋转后平移），逆矩阵有简洁形式：

```
              | u.x      v.x      w.x      0 |
V = W⁻¹  =   | u.y      v.y      w.y      0 |
              | u.z      v.z      w.z      0 |
              | -Q·u     -Q·v     -Q·w     1 |
```

注意：变换坐标不是真的"移动"物体，只是改变了参考坐标系。

---

## 15.2 Camera 类

`Camera` 类封装了相机相关代码。核心数据：

- **位置 (Position)** + **三个基底向量 (Right / Up / Look)**：定义视图坐标系。
- **视锥属性**：相当于相机的"镜头"——视野角、近/远平面。

```cpp
class Camera {
public:
    Camera();
    ~Camera();

    // 位置访问
    DirectX::XMVECTOR GetPosition() const;
    DirectX::XMFLOAT3 GetPosition3f() const;
    void SetPosition(float x, float y, float z);
    void SetPosition(const DirectX::XMFLOAT3& v);

    // 基底向量访问
    DirectX::XMVECTOR GetRight() const;
    DirectX::XMVECTOR GetUp() const;
    DirectX::XMVECTOR GetLook() const;

    // 视锥属性
    float GetNearZ() const;
    float GetFarZ() const;
    float GetAspect() const;
    float GetFovY() const;
    float GetFovX() const;

    // 近/远平面尺寸（视图空间）
    float GetNearWindowWidth() const;
    float GetNearWindowHeight() const;
    float GetFarWindowWidth() const;
    float GetFarWindowHeight() const;

    // 设置视锥
    void SetLens(float fovY, float aspect, float zn, float zf);

    // 通过 LookAt 设置相机姿态
    void LookAt(DirectX::FXMVECTOR pos,
                DirectX::FXMVECTOR target,
                DirectX::FXMVECTOR worldUp);

    // 视图/投影矩阵
    DirectX::XMMATRIX GetView() const;
    DirectX::XMMATRIX GetProj() const;

    // 相机平移
    void Strafe(float d);  // 横移
    void Walk(float d);    // 前后

    // 相机旋转
    void Pitch(float angle);    // 俯仰
    void RotateY(float angle);  // 偏航

    // 更新视图矩阵
    void UpdateViewMatrix();

private:
    DirectX::XMFLOAT3 mPosition = { 0.0f, 0.0f, 0.0f };
    DirectX::XMFLOAT3 mRight    = { 1.0f, 0.0f, 0.0f };
    DirectX::XMFLOAT3 mUp       = { 0.0f, 1.0f, 0.0f };
    DirectX::XMFLOAT3 mLook     = { 0.0f, 0.0f, 1.0f };

    // 视锥属性缓存
    float mNearZ = 0.0f;
    float mFarZ  = 0.0f;
    float mAspect = 0.0f;
    float mFovY = 0.0f;
    float mNearWindowHeight = 0.0f;
    float mFarWindowHeight  = 0.0f;

    bool mViewDirty = true;

    // 矩阵缓存
    DirectX::XMFLOAT4X4 mView = MathHelper::Identity4x4();
    DirectX::XMFLOAT4X4 mProj = MathHelper::Identity4x4();
};
```

`Camera.h` / `Camera.cpp` 文件位于 `Common` 目录。

---

## 15.3 关键方法实现

### 15.3.1 XMVECTOR 与 XMFLOAT3 返回变体

为了方便调用，许多 getter 同时提供 `XMVECTOR` 和 `XMFLOAT3` 版本：

```cpp
XMVECTOR Camera::GetPosition() const {
    return XMLoadFloat3(&mPosition);
}

XMFLOAT3 Camera::GetPosition3f() const {
    return mPosition;
}
```

### 15.3.2 SetLens

可把视锥看作相机的"镜头"。`SetLens` 缓存视锥属性并构建投影矩阵：

```cpp
void Camera::SetLens(float fovY, float aspect, float zn, float zf)
{
    mFovY = fovY;
    mAspect = aspect;
    mNearZ = zn;
    mFarZ = zf;

    mNearWindowHeight = 2.0f * mNearZ * tanf(0.5f * mFovY);
    mFarWindowHeight  = 2.0f * mFarZ  * tanf(0.5f * mFovY);

    XMMATRIX P = XMMatrixPerspectiveFovLH(mFovY, mAspect, mNearZ, mFarZ);
    XMStoreFloat4x4(&mProj, P);
}
```

### 15.3.3 派生视锥信息

垂直视野角已缓存，可推导出水平视野角、近/远平面的宽度和高度：

```cpp
float Camera::GetFovX() const {
    float halfWidth = 0.5f * GetNearWindowWidth();
    return 2.0f * atan(halfWidth / mNearZ);
}

float Camera::GetNearWindowWidth() const {
    return mAspect * mNearWindowHeight;
}

float Camera::GetNearWindowHeight() const {
    return mNearWindowHeight;
}

float Camera::GetFarWindowWidth() const {
    return mAspect * mFarWindowHeight;
}

float Camera::GetFarWindowHeight() const {
    return mFarWindowHeight;
}
```

### 15.3.4 相机变换

第一人称相机的四个基本操作（忽略碰撞检测）：

| 操作 | 实现 |
|------|------|
| **前后移动 (Walk)** | 沿 Look 向量平移位置 |
| **左右横移 (Strafe)** | 沿 Right 向量平移位置 |
| **俯仰 (Pitch)** | 绕 Right 向量旋转 Look 和 Up |
| **偏航 (RotateY)** | 绕世界 Y 轴旋转所有基底向量 |

```cpp
void Camera::Walk(float d) {
    // mPosition += d * mLook
    XMVECTOR s = XMVectorReplicate(d);
    XMVECTOR l = XMLoadFloat3(&mLook);
    XMVECTOR p = XMLoadFloat3(&mPosition);
    XMStoreFloat3(&mPosition, XMVectorMultiplyAdd(s, l, p));
}

void Camera::Strafe(float d) {
    // mPosition += d * mRight
    XMVECTOR s = XMVectorReplicate(d);
    XMVECTOR r = XMLoadFloat3(&mRight);
    XMVECTOR p = XMLoadFloat3(&mPosition);
    XMStoreFloat3(&mPosition, XMVectorMultiplyAdd(s, r, p));
}

void Camera::Pitch(float angle) {
    // 绕 Right 向量旋转 Up 和 Look
    XMMATRIX R = XMMatrixRotationAxis(XMLoadFloat3(&mRight), angle);
    XMStoreFloat3(&mUp,   XMVector3TransformNormal(XMLoadFloat3(&mUp),   R));
    XMStoreFloat3(&mLook, XMVector3TransformNormal(XMLoadFloat3(&mLook), R));
}

void Camera::RotateY(float angle) {
    // 绕世界 Y 轴旋转所有基底向量
    XMMATRIX R = XMMatrixRotationY(angle);
    XMStoreFloat3(&mRight, XMVector3TransformNormal(XMLoadFloat3(&mRight), R));
    XMStoreFloat3(&mUp,    XMVector3TransformNormal(XMLoadFloat3(&mUp),    R));
    XMStoreFloat3(&mLook,  XMVector3TransformNormal(XMLoadFloat3(&mLook),  R));
}
```

### 15.3.5 构建视图矩阵

`UpdateViewMatrix` 包含两步：

1. **重新正交单位化** Right、Up、Look 向量。多次旋转后浮点误差会累积，让基底不再正交。
2. 把基底和位置代入视图矩阵公式。

```cpp
void Camera::UpdateViewMatrix() {
    if (mViewDirty) {
        XMVECTOR R = XMLoadFloat3(&mRight);
        XMVECTOR U = XMLoadFloat3(&mUp);
        XMVECTOR L = XMLoadFloat3(&mLook);
        XMVECTOR P = XMLoadFloat3(&mPosition);

        // 重新正交单位化基底
        L = XMVector3Normalize(L);
        U = XMVector3Normalize(XMVector3Cross(L, R));
        // U、L 已正交单位，无需归一化叉积
        R = XMVector3Cross(U, L);

        // 视图矩阵第 4 行的 -Q·R, -Q·U, -Q·L
        float x = -XMVectorGetX(XMVector3Dot(P, R));
        float y = -XMVectorGetX(XMVector3Dot(P, U));
        float z = -XMVectorGetX(XMVector3Dot(P, L));

        XMStoreFloat3(&mRight, R);
        XMStoreFloat3(&mUp,    U);
        XMStoreFloat3(&mLook,  L);

        mView(0, 0) = mRight.x;  mView(1, 0) = mRight.y;
        mView(2, 0) = mRight.z;  mView(3, 0) = x;
        mView(0, 1) = mUp.x;     mView(1, 1) = mUp.y;
        mView(2, 1) = mUp.z;     mView(3, 1) = y;
        mView(0, 2) = mLook.x;   mView(1, 2) = mLook.y;
        mView(2, 2) = mLook.z;   mView(3, 2) = z;
        mView(0, 3) = 0.0f;      mView(1, 3) = 0.0f;
        mView(2, 3) = 0.0f;      mView(3, 3) = 1.0f;

        mViewDirty = false;
    }
}
```

---

## 15.4 Camera Demo 说明

替换旧的轨道相机变量（`mPhi`、`mTheta`、`mRadius`、`mView`、`mProj`），新增：

```cpp
Camera mCamera;
```

**窗口大小变化时**：

```cpp
void CameraApp::OnResize()
{
    D3DApp::OnResize();
    mCamera.SetLens(0.25f * MathHelper::Pi, AspectRatio(), 1.0f, 1000.0f);
}
```

**键盘控制**：

```cpp
void CameraApp::UpdateScene(float dt)
{
    if (GetAsyncKeyState('W') & 0x8000) mCamera.Walk(10.0f * dt);
    if (GetAsyncKeyState('S') & 0x8000) mCamera.Walk(-10.0f * dt);
    if (GetAsyncKeyState('A') & 0x8000) mCamera.Strafe(-10.0f * dt);
    if (GetAsyncKeyState('D') & 0x8000) mCamera.Strafe(10.0f * dt);
}
```

**鼠标控制**：

```cpp
void CameraAndDynamicIndexingApp::OnMouseMove(WPARAM btnState, int x, int y)
{
    if ((btnState & MK_LBUTTON) != 0)
    {
        // 每像素对应 0.25 度
        float dx = XMConvertToRadians(0.25f * static_cast<float>(x - mLastMousePos.x));
        float dy = XMConvertToRadians(0.25f * static_cast<float>(y - mLastMousePos.y));

        mCamera.Pitch(dy);
        mCamera.RotateY(dx);
    }
    mLastMousePos.x = x;
    mLastMousePos.y = y;
}
```

**渲染时获取矩阵**：

```cpp
mCamera.UpdateViewMatrix();
XMMATRIX view = mCamera.GetView();
XMMATRIX proj = mCamera.GetProj();
```

---

## 15.5 动态索引（Dynamic Indexing）

动态索引允许在着色器中**动态访问资源数组**——本节我们用一个纹理数组示范。索引可以来自：

1. 常量缓冲中的元素；
2. 系统 ID（`SV_PrimitiveID`、`SV_VertexID`、`SV_DispatchThreadID`、`SV_InstanceID`）；
3. 某个计算的结果；
4. 纹理采样的结果；
5. 顶点结构的某个字段。

**最简单的语法**：

```hlsl
cbuffer cbPerDrawIndex : register(b0) {
    int gDiffuseTexIndex;
};
Texture2D gDiffuseMap[4] : register(t0);

float4 texValue = gDiffuseMap[gDiffuseTexIndex].Sample(gsamLinearWrap, pin.TexC);
```

### 优化策略

目标：**减少每个 render-item 需要设置的描述符数量**——以前每个 render-item 都要设置 ObjectCB、MaterialCB、纹理 SRV。

**新方案**：

1. **把所有材质放进一个 StructuredBuffer**——而不是每个材质一个常量缓冲。一次绑定，所有材质对着色器可见。
2. **ObjectCB 加一个 `MaterialIndex` 字段**——用于在着色器里索引材质 buffer。
3. **所有纹理 SRV 一次性绑定到根签名**（而不是每个 render-item 绑一个）。
4. **材质数据加一个 `DiffuseMapIndex` 字段**——指向材质要用的纹理。

这样：每个 render-item 只需要设置 ObjectCB，渲染时通过 `MaterialIndex → MaterialData → DiffuseMapIndex → 纹理` 链式查找。

### 材质 StructuredBuffer

```cpp
struct MaterialData {
    DirectX::XMFLOAT4 DiffuseAlbedo = { 1.0f, 1.0f, 1.0f, 1.0f };
    DirectX::XMFLOAT3 FresnelR0     = { 0.01f, 0.01f, 0.01f };
    float Roughness                 = 64.0f;
    DirectX::XMFLOAT4X4 MatTransform = MathHelper::Identity4x4();

    UINT DiffuseMapIndex = 0;
    UINT MaterialPad0;
    UINT MaterialPad1;
    UINT MaterialPad2;
};

MaterialBuffer = std::make_unique<UploadBuffer<MaterialData>>(
    device, materialCount, false);
```

由于材质需要每帧更新，使用 **Upload 堆**而非 Default 堆。

### 根签名

```cpp
CD3DX12_DESCRIPTOR_RANGE texTable;
texTable.Init(D3D12_DESCRIPTOR_RANGE_TYPE_SRV, 4, 0, 0);

CD3DX12_ROOT_PARAMETER slotRootParameter[4];
// 性能 TIP：按访问频率从高到低排序
slotRootParameter[0].InitAsConstantBufferView(0);  // 每 render-item ObjCB
slotRootParameter[1].InitAsConstantBufferView(1);  // 每 pass CB
slotRootParameter[2].InitAsShaderResourceView(0, 1);  // 材质 StructuredBuffer
slotRootParameter[3].InitAsDescriptorTable(1, &texTable,
    D3D12_SHADER_VISIBILITY_PIXEL);  // 纹理数组
```

### 渲染流程

```cpp
void CameraAndDynamicIndexingApp::Draw(const GameTimer& gt)
{
    // ... 每帧绑定 PassCB、材质 buffer、纹理数组 ...

    auto matBuffer = mCurrFrameResource->MaterialBuffer->Resource();
    mCommandList->SetGraphicsRootShaderResourceView(2,
        matBuffer->GetGPUVirtualAddress());

    // 一次性绑定所有纹理（只需指定描述符表第一个槽位）
    mCommandList->SetGraphicsRootDescriptorTable(3,
        mSrvDescriptorHeap->GetGPUDescriptorHandleForHeapStart());

    DrawRenderItems(mCommandList.Get(), mOpaqueRitems);
}

void DrawRenderItems(ID3D12GraphicsCommandList* cmdList,
                     const std::vector<RenderItem*>& ritems)
{
    for (auto ri : ritems) {
        // 每 render-item 只设置 ObjCB
        cmdList->SetGraphicsRootConstantBufferView(0, objCBAddress);
        cmdList->DrawIndexedInstanced(ri->IndexCount, 1,
            ri->StartIndexLocation, ri->BaseVertexLocation, 0);
    }
}
```

**ObjectConstants 新增字段**：

```cpp
struct ObjectConstants {
    DirectX::XMFLOAT4X4 World         = MathHelper::Identity4x4();
    DirectX::XMFLOAT4X4 TexTransform  = MathHelper::Identity4x4();
    UINT MaterialIndex;   // 新增：用于查找材质 buffer
    UINT ObjPad0;
    UINT ObjPad1;
    UINT ObjPad2;
};
```

### HLSL 完整示例

```hlsl
#include "LightingUtil.hlsl"

struct MaterialData {
    float4 DiffuseAlbedo;
    float3 FresnelR0;
    float  Roughness;
    float4x4 MatTransform;
    uint DiffuseMapIndex;
    uint MatPad0;
    uint MatPad1;
    uint MatPad2;
};

// 纹理数组（SM 5.1+），数组中可以是不同尺寸/格式的纹理
Texture2D gDiffuseMap[4] : register(t0);

// 注意：放在 space1 防止与纹理数组（占 t0~t3 的 space0）冲突
StructuredBuffer<MaterialData> gMaterialData : register(t0, space1);

SamplerState gsamLinearWrap : register(s2);

cbuffer cbPerObject : register(b0) {
    float4x4 gWorld;
    float4x4 gTexTransform;
    uint     gMaterialIndex;
    // ... pad ...
};

cbuffer cbPass : register(b1) { /* ... */ };

struct VertexIn  { float3 PosL : POSITION; float3 NormalL : NORMAL; float2 TexC : TEXCOORD; };
struct VertexOut { float4 PosH : SV_POSITION; float3 PosW : POSITION; float3 NormalW : NORMAL; float2 TexC : TEXCOORD; };

VertexOut VS(VertexIn vin) {
    VertexOut vout = (VertexOut)0.0f;

    // 用 MaterialIndex 查找当前材质
    MaterialData matData = gMaterialData[gMaterialIndex];

    float4 posW = mul(float4(vin.PosL, 1.0f), gWorld);
    vout.PosW = posW.xyz;
    vout.NormalW = mul(vin.NormalL, (float3x3)gWorld);
    vout.PosH = mul(posW, gViewProj);

    float4 texC = mul(float4(vin.TexC, 0.0f, 1.0f), gTexTransform);
    vout.TexC = mul(texC, matData.MatTransform).xy;
    return vout;
}

float4 PS(VertexOut pin) : SV_Target {
    MaterialData matData = gMaterialData[gMaterialIndex];
    float4 diffuseAlbedo = matData.DiffuseAlbedo;
    float3 fresnelR0     = matData.FresnelR0;
    float  roughness     = matData.Roughness;
    uint   diffuseTexIndex = matData.DiffuseMapIndex;

    // 关键：动态索引纹理数组
    diffuseAlbedo *= gDiffuseMap[diffuseTexIndex].Sample(gsamLinearWrap, pin.TexC);

    pin.NormalW = normalize(pin.NormalW);
    float3 toEyeW = normalize(gEyePosW - pin.PosW);
    float4 ambient = gAmbientLight * diffuseAlbedo;

    Material mat = { diffuseAlbedo, fresnelR0, roughness };
    float4 directLight = ComputeDirectLighting(gLights, mat, pin.PosW, pin.NormalW, toEyeW);

    float4 litColor = ambient + directLight;
    litColor.a = diffuseAlbedo.a;
    return litColor;
}
```

### Register Space

```hlsl
StructuredBuffer<MaterialData> gMaterialData : register(t0, space1);
```

如果不显式指定，默认是 `space0`。**Space 是用来防止资源寄存器冲突的额外维度**：

```hlsl
Texture2D gDiffuseMap : register(t0, space0);
Texture2D gNormalMap  : register(t0, space1);  // 同样是 t0，但不冲突
Texture2D gShadowMap  : register(t0, space2);
```

对资源数组特别有用：`Texture2D gDiffuseMap[4]` 占据 `t0~t3`（space0），让 `gMaterialData` 放进 space1 就避免了手动数寄存器。

### 动态索引的其他用途

1. **合并相邻几何**：把几个用不同纹理的相邻 mesh 合成一个 render-item，纹理索引存在顶点结构里。
2. **单 pass 多纹理混合**：不同尺寸和格式的纹理在一次绘制中混合。
3. **带不同材质/纹理的实例化绘制**：用 `SV_InstanceID` 作为索引（第 16 章会用到）。

---

## 本章小结

| 主题 | 核心要点 |
|------|---------|
| **相机坐标系** | 位置（世界坐标）+ 三正交单位基底（Right/Up/Look） |
| **Camera 类** | 把视图与投影封装为一体，"镜头"即视锥 |
| **相机移动** | Walk/Strafe 平移；Pitch 绕 Right、RotateY 绕世界 Y |
| **重新正交化** | 多次旋转后需要重新正交单位化基底，避免误差累积 |
| **动态索引** | SM 5.1+ 支持，可在着色器中用变量索引纹理数组 |
| **优化思路** | 把材质放 StructuredBuffer，纹理全帧绑定，每 render-item 只设 ObjCB |
| **Register Space** | 增加维度避免资源寄存器冲突，特别适合资源数组 |

---

## 课后练习思路

1. 给出世界空间和视图空间的基底与原点，用点积推导视图矩阵的形式。
2. 给相机加上 **roll**（绕 Look 向量旋转）——飞行游戏中很常用。
3. 把 5 个有不同纹理的盒子合成一个 render-item：在顶点结构里加纹理索引，PS 用它选择纹理，一次 draw call 绘制 5 个盒子。

---

> **下一步**：第 16 章将利用动态索引技术实现**硬件实例化**，并讨论**视锥剔除**优化。
