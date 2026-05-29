# 第 11 章 模板缓冲

---

## 11.1 深度/模板格式和清除

深度/模板缓冲区是纹理，必须使用特定格式创建：

| 格式 | 说明 |
|------|------|
| `DXGI_FORMAT_D32_FLOAT_S8X24_UINT` | 32 位浮点深度 + 8 位模板 + 24 位填充 |
| `DXGI_FORMAT_D24_UNORM_S8_UINT` | 24 位无归一化深度 + 8 位模板（最常用） |

每帧开始时需要清除模板缓冲区：

**`ID3D12GraphicsCommandList::ClearDepthStencilView` 方法：**

```cpp
void ID3D12GraphicsCommandList::ClearDepthStencilView(
    D3D12_CPU_DESCRIPTOR_HANDLE DepthStencilView,
    D3D12_CLEAR_FLAGS           ClearFlags,
    FLOAT                       Depth,
    UINT8                       Stencil,
    UINT                        NumRects,
    const D3D12_RECT*           pRects
);
```

| 参数 | 说明 |
|------|------|
| `DepthStencilView` | 深度/模板视图描述符 |
| `ClearFlags` | `DEPTH` / `STENCIL` / 两者同时清除 |
| `Depth` | 深度清除值，`[0, 1]` 范围 |
| `Stencil` | 模板清除值，`[0, 255]` 范围 |
| `NumRects` / `pRects` | 要清除的矩形区域，`nullptr` 表示全部 |

```cpp
mCommandList->ClearDepthStencilView(
    DepthStencilView(),
    D3D12_CLEAR_FLAG_DEPTH | D3D12_CLEAR_FLAG_STENCIL,
    1.0f, 0, 0, nullptr);
```

---

## 11.2 模板测试

模板测试在输出合并阶段执行，决定是否将像素写入后台缓冲区：

```
if ( (StencilRef & StencilReadMask)  op  (Value & StencilReadMask) )
    接受像素
else
    拒绝像素
```

| 操作数 | 说明 |
|--------|------|
| `StencilRef` | 应用程序定义的模板参考值 |
| `StencilReadMask` | 读取掩码，默认 `0xff` |
| `Value` | 模板缓冲区中当前像素的值 |
| `op` | 比较函数 |

**`D3D12_COMPARISON_FUNC` 枚举：**

```cpp
typedef enum D3D12_COMPARISON_FUNC {
    D3D12_COMPARISON_NEVER         = 1,  // 总是失败
    D3D12_COMPARISON_LESS          = 2,  // <
    D3D12_COMPARISON_EQUAL         = 3,  // ==
    D3D12_COMPARISON_LESS_EQUAL    = 4,  // <=
    D3D12_COMPARISON_GREATER       = 5,  // >
    D3D12_COMPARISON_NOT_EQUAL     = 6,  // !=
    D3D12_COMPARISON_GREATER_EQUAL = 7,  // >=
    D3D12_COMPARISON_ALWAYS        = 8   // 总是通过
} D3D12_COMPARISON_FUNC;
```

模板测试失败的像素不会写入深度缓冲区。

---

## 11.3 描述深度/模板状态

深度/模板状态通过 `D3D12_DEPTH_STENCIL_DESC` 描述，是 PSO 的一部分。

**`D3D12_DEPTH_STENCIL_DESC` 结构体：**

```cpp
typedef struct D3D12_DEPTH_STENCIL_DESC {
    BOOL                 DepthEnable;
    D3D12_DEPTH_WRITE_MASK DepthWriteMask;
    D3D12_COMPARISON_FUNC DepthFunc;
    BOOL                 StencilEnable;
    UINT8                StencilReadMask;
    UINT8                StencilWriteMask;
    D3D12_DEPTH_STENCILOP_DESC FrontFace;
    D3D12_DEPTH_STENCILOP_DESC BackFace;
} D3D12_DEPTH_STENCIL_DESC;
```

### 11.3.1 深度设置

| 成员 | 说明 |
|------|------|
| `DepthEnable` | `true` 启用深度测试。禁用后绘制顺序决定覆盖关系 |
| `DepthWriteMask` | `ALL` 允许深度写入，`ZERO` 禁止深度写入（但仍执行深度测试） |
| `DepthFunc` | 深度测试比较函数，通常为 `LESS` |

### 11.3.2 模板设置

| 成员 | 说明 |
|------|------|
| `StencilEnable` | 启用/禁用模板测试 |
| `StencilReadMask` | 模板测试读取掩码，默认 `0xff` |
| `StencilWriteMask` | 模板写入掩码，默认 `0xff` |
| `FrontFace` / `BackFace` | 正面/背面三角形的模板操作 |

**`D3D12_DEPTH_STENCILOP_DESC` 结构体：**

```cpp
typedef struct D3D12_DEPTH_STENCILOP_DESC {
    D3D12_STENCIL_OP      StencilFailOp;
    D3D12_STENCIL_OP      StencilDepthFailOp;
    D3D12_STENCIL_OP      StencilPassOp;
    D3D12_COMPARISON_FUNC StencilFunc;
} D3D12_DEPTH_STENCILOP_DESC;
```

| 成员 | 说明 |
|------|------|
| `StencilFailOp` | 模板测试失败时的操作 |
| `StencilDepthFailOp` | 模板通过但深度失败时的操作 |
| `StencilPassOp` | 模板和深度都通过时的操作 |
| `StencilFunc` | 模板测试比较函数 |

**`D3D12_STENCIL_OP` 枚举：**

| 值 | 操作 |
|-----|------|
| `KEEP` | 保持原值 |
| `ZERO` | 设为 0 |
| `REPLACE` | 替换为 `StencilRef` |
| `INCR_SAT` | 递增，超过最大值时钳制 |
| `DECR_SAT` | 递减，低于 0 时钳制 |
| `INVERT` | 位取反 |
| `INCR` | 递增，超过最大值时回绕到 0 |
| `DECR` | 递减，低于 0 时回绕到最大值 |

### 11.3.3 创建和绑定深度/模板状态

填充 `D3D12_DEPTH_STENCIL_DESC` 后赋值给 PSO 的 `DepthStencilState` 字段。模板参考值通过 `OMSetStencilRef` 设置：

```cpp
void ID3D12GraphicsCommandList::OMSetStencilRef(UINT StencilRef);
```

---

## 11.4 实现平面镜像

### 11.4.1 算法概述

平面镜像的实现步骤：

1. **绘制常规场景**：地板、墙壁、物体（不绘制镜子）。不修改模板缓冲区。
2. **清除模板缓冲区**为 0。
3. **仅将镜子绘制到模板缓冲区**：
   - 禁用颜色写入（`RenderTargetWriteMask = 0`）
   - 禁用深度写入（`DepthWriteMask = ZERO`）
   - 模板测试设为 `ALWAYS`，通过时执行 `REPLACE` 操作将值设为 `StencilRef = 1`
   - 深度失败时执行 `KEEP`
   - 结果：模板缓冲区中仅可见镜子区域为 1，其余为 0
4. **绘制反射物体**：
   - 模板测试设为 `EQUAL`，`StencilRef = 1`
   - 仅在模板值为 1 的区域（即可见镜子区域）绘制反射
5. **绘制镜子本身**：
   - 使用透明混合（如 30% 不透明度）让反射能够透过镜子显示

### 11.4.2 镜像深度/模板状态定义

```cpp
// PSO：标记镜子模板
CD3DX12_BLEND_DESC mirrorBlendState(D3D12_DEFAULT);
mirrorBlendState.RenderTarget[0].RenderTargetWriteMask = 0;

D3D12_DEPTH_STENCIL_DESC mirrorDSS;
mirrorDSS.DepthEnable      = true;
mirrorDSS.DepthWriteMask   = D3D12_DEPTH_WRITE_MASK_ZERO;
mirrorDSS.DepthFunc        = D3D12_COMPARISON_FUNC_LESS;
mirrorDSS.StencilEnable    = true;
mirrorDSS.StencilReadMask  = 0xff;
mirrorDSS.StencilWriteMask = 0xff;
mirrorDSS.FrontFace.StencilFailOp      = D3D12_STENCIL_OP_KEEP;
mirrorDSS.FrontFace.StencilDepthFailOp = D3D12_STENCIL_OP_KEEP;
mirrorDSS.FrontFace.StencilPassOp      = D3D12_STENCIL_OP_REPLACE;
mirrorDSS.FrontFace.StencilFunc        = D3D12_COMPARISON_FUNC_ALWAYS;
mirrorDSS.BackFace = mirrorDSS.FrontFace;

D3D12_GRAPHICS_PIPELINE_STATE_DESC markMirrorsPsoDesc = opaquePsoDesc;
markMirrorsPsoDesc.BlendState       = mirrorBlendState;
markMirrorsPsoDesc.DepthStencilState = mirrorDSS;

// PSO：绘制反射（仅进入镜子区域）
D3D12_DEPTH_STENCIL_DESC reflectionsDSS;
reflectionsDSS.DepthEnable      = true;
reflectionsDSS.DepthWriteMask   = D3D12_DEPTH_WRITE_MASK_ALL;
reflectionsDSS.DepthFunc        = D3D12_COMPARISON_FUNC_LESS;
reflectionsDSS.StencilEnable    = true;
reflectionsDSS.StencilReadMask  = 0xff;
reflectionsDSS.StencilWriteMask = 0xff;
reflectionsDSS.FrontFace.StencilFailOp      = D3D12_STENCIL_OP_KEEP;
reflectionsDSS.FrontFace.StencilDepthFailOp = D3D12_STENCIL_OP_KEEP;
reflectionsDSS.FrontFace.StencilPassOp      = D3D12_STENCIL_OP_KEEP;
reflectionsDSS.FrontFace.StencilFunc        = D3D12_COMPARISON_FUNC_EQUAL;
reflectionsDSS.BackFace = reflectionsDSS.FrontFace;

D3D12_GRAPHICS_PIPELINE_STATE_DESC drawReflectionsPsoDesc = opaquePsoDesc;
drawReflectionsPsoDesc.DepthStencilState = reflectionsDSS;
// 反射后法线朝内，需要反转正面定义
drawReflectionsPsoDesc.RasterizerState.FrontCounterClockwise = true;
```

### 11.4.3 绘制场景

```cpp
// 1. 绘制不透明物体
DrawRenderItems(mCommandList.Get(), mRitemLayer[(int)RenderLayer::Opaque]);

// 2. 标记镜子模板区域
mCommandList->OMSetStencilRef(1);
mCommandList->SetPipelineState(mPSOs["markStencilMirrors"].Get());
DrawRenderItems(mCommandList.Get(), mRitemLayer[(int)RenderLayer::Mirrors]);

// 3. 绘制反射（使用反射后的光照常量）
mCommandList->SetGraphicsRootConstantBufferView(2,
    passCB->GetGPUVirtualAddress() + 1 * passCBByteSize);
mCommandList->SetPipelineState(mPSOs["drawStencilReflections"].Get());
DrawRenderItems(mCommandList.Get(), mRitemLayer[(int)RenderLayer::Reflected]);

// 4. 恢复并绘制镜子（透明）
mCommandList->SetGraphicsRootConstantBufferView(2, passCB->GetGPUVirtualAddress());
mCommandList->OMSetStencilRef(0);
mCommandList->SetPipelineState(mPSOs["transparent"].Get());
DrawRenderItems(mCommandList.Get(), mRitemLayer[(int)RenderLayer::Transparent]);
```

反射光照常量需要单独计算（光源方向也需要关于镜面对称反射）：

```cpp
void UpdateReflectedPassCB(const GameTimer& gt) {
    mReflectedPassCB = mMainPassCB;
    XMVECTOR mirrorPlane = XMVectorSet(0.0f, 0.0f, 1.0f, 0.0f);  // xy 平面
    XMMATRIX R = XMMatrixReflect(mirrorPlane);

    for (int i = 0; i < 3; ++i) {
        XMVECTOR lightDir = XMLoadFloat3(&mMainPassCB.Lights[i].Direction);
        XMVECTOR reflectedLightDir = XMVector3TransformNormal(lightDir, R);
        XMStoreFloat3(&mReflectedPassCB.Lights[i].Direction, reflectedLightDir);
    }

    auto currPassCB = mCurrFrameResource->PassCB.get();
    currPassCB->CopyData(1, mReflectedPassCB);  // 存储在索引 1
}
```

### 11.4.4 环绕顺序与反射

反射后三角形的环绕顺序不会改变，导致面法线朝内。需要在反射绘制 PSO 中反转正面定义：

```cpp
drawReflectionsPsoDesc.RasterizerState.FrontCounterClockwise = true;
```

---

## 11.5 实现平面阴影

### 11.5.1 平行光阴影矩阵

对于平面 `(n, d)` 和平行光方向 `L`，顶点 `p` 的阴影投影：

```cpp
XMMATRIX S = XMMatrixShadow(shadowPlane, toMainLight);
```

`XMMatrixShadow` 函数自动处理：若 `w = 0` 表示平行光，若 `w = 1` 表示点光源。

### 11.5.2 防止双重混合

阴影几何体可能重叠，导致同一像素被多次混合而变暗（**双重混合**）。使用模板缓冲区防止：

```cpp
// PSO 配置：绘制阴影
D3D12_DEPTH_STENCIL_DESC shadowDSS;
shadowDSS.DepthEnable      = true;
shadowDSS.DepthWriteMask   = D3D12_DEPTH_WRITE_MASK_ALL;
shadowDSS.DepthFunc        = D3D12_COMPARISON_FUNC_LESS;
shadowDSS.StencilEnable    = true;
shadowDSS.StencilReadMask  = 0xff;
shadowDSS.StencilWriteMask = 0xff;
shadowDSS.FrontFace.StencilFailOp      = D3D12_STENCIL_OP_KEEP;
shadowDSS.FrontFace.StencilDepthFailOp = D3D12_STENCIL_OP_KEEP;
shadowDSS.FrontFace.StencilPassOp      = D3D12_STENCIL_OP_INCR;
shadowDSS.FrontFace.StencilFunc        = D3D12_COMPARISON_FUNC_EQUAL;
shadowDSS.BackFace = shadowDSS.FrontFace;
```

算法：
1. 阴影区域的模板值已清除为 0
2. 模板测试设为 `EQUAL`，`StencilRef = 0`
3. 通过时递增模板值到 1
4. 同一像素第二次绘制时模板值为 1，测试失败，不会重复绘制

```cpp
// 绘制阴影
mCommandList->OMSetStencilRef(0);
mCommandList->SetPipelineState(mPSOs["shadow"].Get());
DrawRenderItems(mCommandList.Get(), mRitemLayer[(int)RenderLayer::Shadow]);
```

阴影材质为 50% 透明黑色：

```cpp
auto shadowMat = std::make_unique<Material>();
shadowMat->DiffuseAlbedo = XMFLOAT4(0.0f, 0.0f, 0.0f, 0.5f);
shadowMat->Roughness = 0.0f;
```

**防止 z-fighting**：将阴影沿法线方向略微偏移：

```cpp
XMMATRIX shadowOffsetY = XMMatrixTranslation(0.0f, 0.001f, 0.0f);
XMStoreFloat4x4(&mShadowedSkullRitem->World, skullWorld * S * shadowOffsetY);
```
