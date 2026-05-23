# 第 23 章 角色动画（Character Animation）

---

第 22 章演示了**单物体**的关键帧动画。本章把这套机制升级到**角色动画**——骨骼有几十根、皮肤是连续 mesh、每个顶点被多根骨骼共同影响。最终我们要在 Direct3D 中加载并播放骨骼动画角色。

---

## 23.1 骨骼层级（Frame Hierarchy）

### 父子关系

一个角色是**树状层级**：

```
                Pelvis
                /  |  \
           LThigh Spine RThigh
              |    |     |
            LShin Upper RShin
              |    |     |
           ... 各种 spine 子节点 ...
```

**子骨骼跟随父骨骼移动**：肩动 → 大臂动 → 小臂动 → 手动。

### 数学公式

每根骨骼有自己的局部坐标系（**关节作为原点**便于旋转）。父子坐标系通过**子→父变换矩阵 `A_i`** 关联。

把骨骼 `i` 从其局部空间变到世界空间：

```
M_i = A_i · A_{i-1} · ... · A_0     (假设 0 是根，向上累乘)
```

对图 23.3 简单链：

```
M_2 = A_2 · A_1 · A_0    (hand)
M_1 = A_1 · A_0          (forearm)
M_0 = A_0                (upper arm)
```

→ 子骨骼**继承**祖先的所有变换。

> 树状层级同理——任意骨骼的世界变换 = 沿其祖先链所有 to-parent 矩阵的累积。

---

## 23.2 蒙皮 Mesh（Skinned Mesh）

### 23.2.1 定义

**蒙皮 mesh**：

- 内部是骨骼层级**骨架（skeleton）**；
- 外部是**皮肤（skin）**——单一连续的 3D mesh；
- 皮肤顶点定义在**绑定空间（bind space）**（mesh 建模时的坐标系，通常是根骨骼空间）；
- 每根骨骼影响皮肤的某部分顶点——骨骼动 → 皮肤跟着变形。

### 23.2.2 to-Root 变换的递推公式

为了高效，**从根向下**遍历层级：

```
toRoot_i = toParent_i · toRoot_p     (p 是 i 的父骨骼)
```

**优点**：

- 自顶向下，父的 toRoot 已经算好，只需一次乘法；
- 自底向上则要每根骨骼独立向上累乘，重复计算大量。

**前提**：骨骼数组必须按"父先于子"排序。例：

```
ParentIndexOfBone0: -1     ← 根
ParentIndexOfBone1: 0
ParentIndexOfBone2: 0
ParentIndexOfBone3: 2
ParentIndexOfBone4: 3
...
ParentIndexOfBone9: 8      ← 父索引 8 < 9 ✓
```

### 23.2.3 Offset Transform（偏移变换）

蒙皮顶点存在**绑定空间**，不是骨骼局部空间。必须先用 `offset_i` 把顶点从绑定空间送到骨骼 `i` 的局部空间。

### Final Transform

```
F_i = offset_i · toRoot_i
```

顶点用 `F_i` 一次完成：绑定空间 → 骨骼空间 → 根空间。

### 23.2.4 动画骨架

```cpp
// 一个动画片段（如 "walk", "run"）：每根骨骼一条动画
struct AnimationClip {
    float GetClipStartTime() const;
    float GetClipEndTime  () const;
    void  Interpolate(float t, std::vector<XMFLOAT4X4>& boneTransforms) const;

    std::vector<BoneAnimation> BoneAnimations;
};

// 整个角色的所有动画片段
std::unordered_map<std::string, AnimationClip> mAnimations;

class SkinnedData {
public:
    UINT BoneCount() const;
    float GetClipStartTime(const std::string& clipName) const;
    float GetClipEndTime  (const std::string& clipName) const;

    void Set(
        std::vector<int>& boneHierarchy,
        std::vector<XMFLOAT4X4>& boneOffsets,
        std::unordered_map<std::string, AnimationClip>& animations);

    void GetFinalTransforms(
        const std::string& clipName,
        float timePos,
        std::vector<XMFLOAT4X4>& finalTransforms) const;

private:
    std::vector<int>           mBoneHierarchy;   // 父索引表
    std::vector<XMFLOAT4X4>    mBoneOffsets;     // 偏移变换
    std::unordered_map<std::string, AnimationClip> mAnimations;
};
```

### 23.2.5 计算 Final Transform

```cpp
void SkinnedData::GetFinalTransforms(
    const std::string& clipName, float timePos,
    std::vector<XMFLOAT4X4>& finalTransforms) const
{
    UINT numBones = mBoneOffsets.size();

    // 1. 在 timePos 对所有骨骼做关键帧插值，得 toParent
    std::vector<XMFLOAT4X4> toParentTransforms(numBones);
    auto clip = mAnimations.find(clipName);
    clip->second.Interpolate(timePos, toParentTransforms);

    // 2. 自顶向下递推：toRoot_i = toParent_i · toRoot_p
    std::vector<XMFLOAT4X4> toRootTransforms(numBones);
    // 根骨骼无父，toRoot = toParent
    toRootTransforms[0] = toParentTransforms[0];

    for (UINT i = 1; i < numBones; ++i) {
        XMMATRIX toParent = XMLoadFloat4x4(&toParentTransforms[i]);
        int parentIndex = mBoneHierarchy[i];
        XMMATRIX parentToRoot = XMLoadFloat4x4(&toRootTransforms[parentIndex]);

        XMMATRIX toRoot = XMMatrixMultiply(toParent, parentToRoot);
        XMStoreFloat4x4(&toRootTransforms[i], toRoot);
    }

    // 3. final = offset · toRoot
    for (UINT i = 0; i < numBones; ++i) {
        XMMATRIX offset = XMLoadFloat4x4(&mBoneOffsets[i]);
        XMMATRIX toRoot = XMLoadFloat4x4(&toRootTransforms[i]);
        XMStoreFloat4x4(&finalTransforms[i],
            XMMatrixMultiply(offset, toRoot));
    }
}
```

---

## 23.3 顶点蒙皮（Vertex Blending）

### 核心思想

一个皮肤顶点可被**多根骨骼共同影响**，最终位置是各骨骼独立变换结果的**加权平均**：

```
v' = w0·v·F0 + w1·v·F1 + w2·v·F2 + w3·v·F3
权重和 w0+w1+w2+w3 = 1
```

每根骨骼"拉"顶点一点，按权重混合 → 关节处皮肤平滑过渡（**模拟弹性皮肤**）。

> 实践经验：每顶点 **最多 4 根骨骼影响**就够了（Möller08）。

### 顶点结构

```cpp
struct SkinnedVertex {
    XMFLOAT3 Pos;
    XMFLOAT3 Normal;
    XMFLOAT2 TexC;
    XMFLOAT4 TangentU;

    XMFLOAT3 BoneWeights;   // 只存 3 个权重，第 4 个 = 1-w0-w1-w2
    uint8_t  BoneIndices[4]; // 4 根影响骨骼在矩阵调色板中的索引
};
```

### 矩阵调色板（Matrix Palette）

GPU 端有一个**常量缓冲**，存储所有骨骼的 final transform：

```hlsl
cbuffer cbSkinned : register(b1)
{
    // 单个角色最多 96 根骨骼
    float4x4 gBoneTransforms[96];
};
```

顶点的 `BoneIndices[4]` 索引到调色板，`BoneWeights` 给出权重。

### 蒙皮顶点着色器

```hlsl
struct VertexIn {
    float3 PosL     : POSITION;
    float3 NormalL  : NORMAL;
    float2 TexC     : TEXCOORD;
    float4 TangentL : TANGENT;
#ifdef SKINNED
    float3 BoneWeights : WEIGHTS;
    uint4  BoneIndices : BONEINDICES;
#endif
};

VertexOut VS(VertexIn vin)
{
    VertexOut vout = (VertexOut)0.0f;
    MaterialData matData = gMaterialData[gMaterialIndex];

#ifdef SKINNED
    // 4 个权重：第 4 个由前 3 个推出
    float weights[4] = {0.0f, 0.0f, 0.0f, 0.0f};
    weights[0] = vin.BoneWeights.x;
    weights[1] = vin.BoneWeights.y;
    weights[2] = vin.BoneWeights.z;
    weights[3] = 1.0f - weights[0] - weights[1] - weights[2];

    float3 posL     = float3(0,0,0);
    float3 normalL  = float3(0,0,0);
    float3 tangentL = float3(0,0,0);

    // 4 根骨骼分别变换并加权混合
    for (int i = 0; i < 4; ++i) {
        // 位置（位置点 w=1）
        posL += weights[i] * mul(
            float4(vin.PosL, 1.0f),
            gBoneTransforms[vin.BoneIndices[i]]).xyz;

        // 法线（用 3x3 部分，假设无非均匀缩放）
        normalL += weights[i] * mul(
            vin.NormalL,
            (float3x3)gBoneTransforms[vin.BoneIndices[i]]);

        // 切线同理
        tangentL += weights[i] * mul(
            vin.TangentL.xyz,
            (float3x3)gBoneTransforms[vin.BoneIndices[i]]);
    }

    vin.PosL          = posL;
    vin.NormalL       = normalL;
    vin.TangentL.xyz  = tangentL;
#endif

    // 之后照常做 world / viewProj / shadow / ssao 变换
    float4 posW = mul(float4(vin.PosL, 1.0f), gWorld);
    vout.PosW = posW.xyz;
    vout.NormalW  = mul(vin.NormalL,  (float3x3)gWorld);
    vout.TangentW = mul(vin.TangentL, (float3x3)gWorld);

    vout.PosH      = mul(posW, gViewProj);
    vout.SsaoPosH  = mul(posW, gViewProjTex);
    vout.ShadowPosH= mul(posW, gShadowTransform);

    float4 texC = mul(float4(vin.TexC, 0.0f, 1.0f), gTexTransform);
    vout.TexC = mul(texC, matData.MatTransform).xy;
    return vout;
}
```

> **为什么权重只存 3 个？** 因为约束 `Σwi = 1`，第 4 个可由前 3 个反推：`w3 = 1 - w0 - w1 - w2`，节省 4 字节。

---

## 23.4 从文件加载动画数据（.m3d 格式）

`.m3d` 是本书自定义的纯文本格式，便于学习——**不追求性能**。

### 23.4.1 文件头

```
***************m3d-File-Header***************
#Materials 3
#Vertices 3121
#Triangles 4062
#Bones 44
#AnimationClips 15
```

### 23.4.2 材质块

每个材质：

```
Name: soldier_head
Diffuse: 1 1 1
Fresnel0: 0.05 0.05 0.05
Roughness: 0.5
AlphaClip: 0
MaterialTypeName: Skinned    ← 标识需要蒙皮着色器
DiffuseMap: head_diff.dds
NormalMap: head_norm.dds
```

### 23.4.3 子集表（SubsetTable）

mesh 按材质分**子集**——每子集是连续的三角形块：

```
SubsetID: 0  VertexStart: 0     VertexCount: 3915  FaceStart: 0     FaceCount: 7230
SubsetID: 1  VertexStart: 3915  VertexCount: 2984  FaceStart: 7230  FaceCount: 4449
...
```

第 `i` 个子集 ↔ 第 `i` 个材质。

### 23.4.4 顶点与三角形

```
***************Vertices**********************
Position: -14.34667 90.44742 -12.08929
Tangent: -0.3069077 0.2750875 0.9111171 1
Normal: -0.3731041 -0.9154652 0.150721
Tex-Coords: 0.21795 0.105219
BlendWeights: 0.483457 0.483457 0.0194 0.013686
BlendIndices: 3 2 39 34
...

***************Triangles*********************
0 1 2
3 4 5
6 7 8
...
```

### 23.4.5 骨骼偏移变换

```
***************BoneOffsets*******************
BoneOffset0
-0.8669753 0.4982096 0.01187624 0
0.04897417 0.1088907 -0.9928461 0
-0.4959392 -0.8601914 -0.118805 0
-10.94755 -14.61919 90.63506 1

BoneOffset1
...
```

### 23.4.6 层级表

```
***************BoneHierarchy*****************
ParentIndexOfBone0: -1
ParentIndexOfBone1: 0
ParentIndexOfBone2: 1
ParentIndexOfBone3: 2
...
```

### 23.4.7 动画片段

每个 AnimationClip 包含每根骨骼的关键帧列表：

```
AnimationClip run_loop
{
    Bone0 #Keyframes: 18
    {
        Time: 0
        Pos: 2.538344 101.6727 -0.52932
        Scale: 1 1 1
        Quat: 0.4042651 0.3919331 -0.5853591 0.5833637

        Time: 0.0666666
        ...
    }
    Bone1 #Keyframes: 18 { ... }
    ...
}

AnimationClip walk_loop { ... }
```

### 加载代码

```cpp
void M3DLoader::ReadAnimationClips(
    std::ifstream& fin,
    UINT numBones, UINT numAnimationClips,
    std::unordered_map<std::string, AnimationClip>& animations)
{
    std::string ignore;
    fin >> ignore;  // 跳过 header

    for (UINT clipIndex = 0; clipIndex < numAnimationClips; ++clipIndex) {
        std::string clipName;
        fin >> ignore >> clipName;
        fin >> ignore;  // {

        AnimationClip clip;
        clip.BoneAnimations.resize(numBones);
        for (UINT boneIndex = 0; boneIndex < numBones; ++boneIndex) {
            ReadBoneKeyframes(fin, numBones, clip.BoneAnimations[boneIndex]);
        }
        fin >> ignore;  // }

        animations[clipName] = clip;
    }
}

void M3DLoader::ReadBoneKeyframes(
    std::ifstream& fin, UINT numBones, BoneAnimation& boneAnimation)
{
    std::string ignore;
    UINT numKeyframes = 0;
    fin >> ignore >> ignore >> numKeyframes;
    fin >> ignore;  // {

    boneAnimation.Keyframes.resize(numKeyframes);
    for (UINT i = 0; i < numKeyframes; ++i) {
        float    t = 0;
        XMFLOAT3 p(0,0,0), s(1,1,1);
        XMFLOAT4 q(0,0,0,1);
        fin >> ignore >> t;
        fin >> ignore >> p.x >> p.y >> p.z;
        fin >> ignore >> s.x >> s.y >> s.z;
        fin >> ignore >> q.x >> q.y >> q.z >> q.w;

        boneAnimation.Keyframes[i].TimePos       = t;
        boneAnimation.Keyframes[i].Translation   = p;
        boneAnimation.Keyframes[i].Scale         = s;
        boneAnimation.Keyframes[i].RotationQuat  = q;
    }
    fin >> ignore;  // }
}
```

### 加载主入口

```cpp
bool M3DLoader::LoadM3d(
    const std::string& filename,
    std::vector<SkinnedVertex>& vertices,
    std::vector<USHORT>& indices,
    std::vector<Subset>& subsets,
    std::vector<M3dMaterial>& mats,
    SkinnedData& skinInfo)
{
    std::ifstream fin(filename);
    if (!fin) return false;

    UINT numMaterials = 0, numVertices = 0, numTriangles = 0;
    UINT numBones = 0, numAnimationClips = 0;
    std::string ignore;

    fin >> ignore;  // 文件头
    fin >> ignore >> numMaterials;
    fin >> ignore >> numVertices;
    fin >> ignore >> numTriangles;
    fin >> ignore >> numBones;
    fin >> ignore >> numAnimationClips;

    std::vector<XMFLOAT4X4> boneOffsets;
    std::vector<int>        boneIndexToParentIndex;
    std::unordered_map<std::string, AnimationClip> animations;

    ReadMaterials      (fin, numMaterials,      mats);
    ReadSubsetTable    (fin, numMaterials,      subsets);
    ReadSkinnedVertices(fin, numVertices,       vertices);
    ReadTriangles      (fin, numTriangles,      indices);
    ReadBoneOffsets    (fin, numBones,          boneOffsets);
    ReadBoneHierarchy  (fin, numBones,          boneIndexToParentIndex);
    ReadAnimationClips (fin, numBones, numAnimationClips, animations);

    skinInfo.Set(boneIndexToParentIndex, boneOffsets, animations);
    return true;
}
```

---

## 23.5 角色动画 Demo

### 蒙皮常量缓冲

```cpp
struct SkinnedConstants {
    XMFLOAT4X4 BoneTransforms[96];
};

std::unique_ptr<UploadBuffer<SkinnedConstants>> SkinnedCB = nullptr;
SkinnedCB = std::make_unique<UploadBuffer<SkinnedConstants>>(
    device, skinnedObjectCount, /* isConstantBuffer */ true);
```

每个**角色实例**用一个 SkinnedConstants（同一角色的多个 RenderItem 共享）。

### 角色实例（每帧推进时间）

```cpp
struct SkinnedModelInstance {
    SkinnedData*              SkinnedInfo = nullptr;
    std::vector<XMFLOAT4X4>   FinalTransforms;
    std::string               ClipName;
    float                     TimePos = 0.0f;

    void UpdateSkinnedAnimation(float dt)
    {
        TimePos += dt;
        // 循环播放
        if (TimePos > SkinnedInfo->GetClipEndTime(ClipName))
            TimePos = 0.0f;

        // 计算所有 final transform
        SkinnedInfo->GetFinalTransforms(ClipName, TimePos, FinalTransforms);
    }
};
```

### RenderItem 增加蒙皮字段

```cpp
struct RenderItem {
    // ... 既有字段 ...

    // 蒙皮常量缓冲索引（仅蒙皮项有效）
    UINT SkinnedCBIndex = -1;
    // 关联的角色实例
    SkinnedModelInstance* SkinnedModelInst = nullptr;
};
```

### 每帧更新

```cpp
void SkinnedMeshApp::UpdateSkinnedCBs(const GameTimer& gt)
{
    auto currSkinnedCB = mCurrFrameResource->SkinnedCB.get();

    mSkinnedModelInst->UpdateSkinnedAnimation(gt.DeltaTime());

    SkinnedConstants skinnedConstants;
    std::copy(
        std::begin(mSkinnedModelInst->FinalTransforms),
        std::end  (mSkinnedModelInst->FinalTransforms),
        &skinnedConstants.BoneTransforms[0]);

    currSkinnedCB->CopyData(0, skinnedConstants);
}
```

### 绘制时绑定蒙皮 CB

```cpp
if (ri->SkinnedModelInst != nullptr) {
    D3D12_GPU_VIRTUAL_ADDRESS skinnedCBAddress =
        skinnedCB->GetGPUVirtualAddress() +
        ri->SkinnedCBIndex * skinnedCBByteSize;
    cmdList->SetGraphicsRootConstantBufferView(1, skinnedCBAddress);
}
else {
    cmdList->SetGraphicsRootConstantBufferView(1, 0);
}
```

---

## 本章小结

| 主题 | 核心要点 |
|------|---------|
| **骨骼层级** | 树形父子关系；子骨骼跟随父骨骼移动 |
| **to-Root 递推** | `toRoot_i = toParent_i · toRoot_p`，自顶向下 |
| **排序前提** | 骨骼数组中父必先于子，保证 `toRoot_p` 已算完 |
| **Offset 变换** | 把顶点从绑定空间送到骨骼局部空间 |
| **Final 变换** | `F_i = offset_i · toRoot_i`，存入矩阵调色板 |
| **顶点蒙皮** | 每顶点最多 4 根骨骼，权重和为 1，加权平均 |
| **存储优化** | 只存 3 个权重，第 4 个由 `1-Σ` 推算 |
| **动画 Clip** | 每根骨骼一条 BoneAnimation；多个 Clip（walk/run/attack）共享骨架 |
| **关键帧** | 时间 + 平移 + 缩放 + 旋转四元数；位置/缩放 LERP，朝向 SLERP |
| **m3d 格式** | 文本格式：header、materials、subsets、vertices、triangles、boneOffsets、hierarchy、clips |
| **GPU 调色板** | 96 个 4×4 矩阵的 cbuffer，VS 通过 BoneIndices 索引 |

---

## 课后练习思路

1. **手动建模** + 渲染一个**线性层级**——例如球（关节）+ 圆柱（手臂）的机器人手臂。
2. **手动建模** + 渲染一个**树状层级**——使用球和圆柱组合搭建简单角色。
3. 用 **Blender**（免费）学习骨骼蒙皮建模——导出 `.x` 文件，转换成 `.m3d` 格式，在 demo 中加载运行。这是一个综合性大项目。

---

## 全书结语

到此为止，本系列教程涵盖了 **DirectX 12 从初始化到角色动画的完整流程**——

```
┌─────────────────────────────────────────────────────┐
│  数学基础（向量、矩阵、变换）→ 第 1-3 章               │
│  D3D 初始化、渲染管线、绘图        → 第 4-7 章         │
│  光照、纹理、混合、模板             → 第 8-11 章       │
│  几何/计算着色器                    → 第 12-13 章      │
│  曲面细分                          → 第 14 章          │
│  相机/实例化/拾取                  → 第 15-17 章       │
│  立方体贴图/法线贴图/阴影/SSAO       → 第 18-21 章      │
│  四元数 + 角色动画                  → 第 22-23 章 ✓    │
└─────────────────────────────────────────────────────┘
```

掌握这些之后，你已经具备**自主实现 3D 游戏渲染器**的所有核心知识。继续向更高级的话题前进：

- **PBR**（基于物理的渲染）
- **延迟渲染（Deferred Rendering）**
- **GPU 驱动渲染（GPU-driven Rendering）**
- **Ray Tracing**（D3D12 DXR）
- **多线程命令录制**

祝你在 3D 图形编程的旅途上越走越远！🚀
