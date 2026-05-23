# 第 10 章 混合

---

## 10.1 混合方程

混合（Blending）允许将当前正在光栅化的像素（**源像素**，Source）与后台缓冲区中已存在的像素（**目标像素**，Destination）按一定规则组合。

Direct3D 的 RGB 混合方程：

```
C = C_src ⊗ F_src ⊕ C_dst ⊗ F_dst
```

Alpha 混合方程（独立配置）：

```
A = A_src · F_src ⊕ A_dst · F_dst
```

| 符号 | 说明 |
|------|------|
| `C_src` / `A_src` | 像素着色器输出的颜色 / Alpha |
| `C_dst` / `A_dst` | 渲染目标中已有的颜色 / Alpha |
| `F_src` / `F_dst` | 源 / 目标混合因子 |
| `⊗` | 分量乘法 |
| `⊕` | 混合操作（加、减、取最小/最大等） |

RGB 与 Alpha 使用独立方程，以便分别控制。

---

## 10.2 混合操作

**`D3D12_BLEND_OP` 枚举：**

| 值 | 操作 |
|-----|------|
| `ADD` | `C_src ⊗ F_src + C_dst ⊗ F_dst` |
| `SUBTRACT` | `C_src ⊗ F_src - C_dst ⊗ F_dst` |
| `REV_SUBTRACT` | `C_dst ⊗ F_dst - C_src ⊗ F_src` |
| `MIN` | `min(C_src, C_dst)` — 忽略混合因子 |
| `MAX` | `max(C_src, C_dst)` — 忽略混合因子 |

**逻辑操作（`D3D12_LOGIC_OP`）**：

Direct3D 12 还支持使用位逻辑运算符代替传统混合方程。可选值包括：`CLEAR`、`SET`、`COPY`、`COPY_INVERTED`、`NOOP`、`INVERT`、`AND`、`OR`、`XOR` 等。

使用逻辑操作时：
- 不能与常规混合同时启用
- 渲染目标格式必须是 UINT 类型（如 `R8G8B8A8_UINT`）

---

## 10.3 混合因子

**`D3D12_BLEND` 枚举常用值：**

| 混合因子 | RGB 值 | Alpha 值 | 说明 |
|---------|--------|---------|------|
| `ZERO` | `(0, 0, 0)` | `0` | 零 |
| `ONE` | `(1, 1, 1)` | `1` | 一 |
| `SRC_COLOR` | `(r_s, g_s, b_s)` | — | 源颜色 |
| `INV_SRC_COLOR` | `(1-r_s, 1-g_s, 1-b_s)` | — | 源颜色反相 |
| `SRC_ALPHA` | `(a_s, a_s, a_s)` | `a_s` | 源 Alpha |
| `INV_SRC_ALPHA` | `(1-a_s, 1-a_s, 1-a_s)` | `1-a_s` | 源 Alpha 反相 |
| `DEST_ALPHA` | `(a_d, a_d, a_d)` | `a_d` | 目标 Alpha |
| `INV_DEST_ALPHA` | `(1-a_d, 1-a_d, 1-a_d)` | `1-a_d` | 目标 Alpha 反相 |
| `DEST_COLOR` | `(r_d, g_d, b_d)` | — | 目标颜色 |
| `INV_DEST_COLOR` | `(1-r_d, 1-g_d, 1-b_d)` | — | 目标颜色反相 |
| `SRC_ALPHA_SAT` | `(a_s', a_s', a_s')` | `a_s'` | `a_s' = clamp(a_s, 0, 1)` |
| `BLEND_FACTOR` | 由 `OMSetBlendFactor` 设置的颜色 | 用户指定的常量因子 |
| `INV_BLEND_FACTOR` | 上述颜色的反相 | |

**`OMSetBlendFactor` 方法：**

```cpp
void ID3D12GraphicsCommandList::OMSetBlendFactor(const FLOAT BlendFactor[4]);
```

传入 `nullptr` 恢复默认值 `(1, 1, 1, 1)`。

---

## 10.4 混合状态

混合状态是 PSO 的一部分，通过 `D3D12_BLEND_DESC` 描述。

**`D3D12_BLEND_DESC` 结构体：**

```cpp
typedef struct D3D12_BLEND_DESC {
    BOOL                           AlphaToCoverageEnable;
    BOOL                           IndependentBlendEnable;
    D3D12_RENDER_TARGET_BLEND_DESC RenderTarget[8];
} D3D12_BLEND_DESC;
```

| 成员 | 说明 |
|------|------|
| `AlphaToCoverageEnable` | 启用 alpha-to-coverage，用于渲染树叶/栅栏等半透明边缘，需要开启多重采样 |
| `IndependentBlendEnable` | 为 `true` 时，8 个渲染目标可分别配置混合；为 `false` 时全部使用 `RenderTarget[0]` |
| `RenderTarget[8]` | 每个渲染目标的混合配置 |

**`D3D12_RENDER_TARGET_BLEND_DESC` 结构体：**

```cpp
typedef struct D3D12_RENDER_TARGET_BLEND_DESC {
    BOOL                 BlendEnable;
    BOOL                 LogicOpEnable;
    D3D12_BLEND          SrcBlend;
    D3D12_BLEND          DestBlend;
    D3D12_BLEND_OP       BlendOp;
    D3D12_BLEND          SrcBlendAlpha;
    D3D12_BLEND          DestBlendAlpha;
    D3D12_BLEND_OP       BlendOpAlpha;
    D3D12_LOGIC_OP       LogicOp;
    UINT8                RenderTargetWriteMask;
} D3D12_RENDER_TARGET_BLEND_DESC;
```

| 成员 | 说明 |
|------|------|
| `BlendEnable` / `LogicOpEnable` | 启用常规混合或逻辑操作（不能同时启用） |
| `SrcBlend` / `DestBlend` | RGB 源/目标混合因子 |
| `BlendOp` | RGB 混合操作 |
| `SrcBlendAlpha` / `DestBlendAlpha` | Alpha 源/目标混合因子 |
| `BlendOpAlpha` | Alpha 混合操作 |
| `LogicOp` | 逻辑操作类型 |
| `RenderTargetWriteMask` | 颜色通道写入掩码 |

**`D3D12_COLOR_WRITE_ENABLE` 枚举：**

```cpp
typedef enum D3D12_COLOR_WRITE_ENABLE {
    D3D12_COLOR_WRITE_ENABLE_RED   = 1,
    D3D12_COLOR_WRITE_ENABLE_GREEN = 2,
    D3D12_COLOR_WRITE_ENABLE_BLUE  = 4,
    D3D12_COLOR_WRITE_ENABLE_ALPHA = 8,
    D3D12_COLOR_WRITE_ENABLE_ALL   = RED | GREEN | BLUE | ALPHA
} D3D12_COLOR_WRITE_ENABLE;
```

**创建透明混合 PSO 示例：**

```cpp
D3D12_GRAPHICS_PIPELINE_STATE_DESC transparentPsoDesc = opaquePsoDesc;

D3D12_RENDER_TARGET_BLEND_DESC transparencyBlendDesc;
transparencyBlendDesc.BlendEnable            = true;
transparencyBlendDesc.LogicOpEnable          = false;
transparencyBlendDesc.SrcBlend               = D3D12_BLEND_SRC_ALPHA;
transparencyBlendDesc.DestBlend              = D3D12_BLEND_INV_SRC_ALPHA;
transparencyBlendDesc.BlendOp                = D3D12_BLEND_OP_ADD;
transparencyBlendDesc.SrcBlendAlpha          = D3D12_BLEND_ONE;
transparencyBlendDesc.DestBlendAlpha         = D3D12_BLEND_ZERO;
transparencyBlendDesc.BlendOpAlpha           = D3D12_BLEND_OP_ADD;
transparencyBlendDesc.LogicOp                = D3D12_LOGIC_OP_NOOP;
transparencyBlendDesc.RenderTargetWriteMask  = D3D12_COLOR_WRITE_ENABLE_ALL;

transparentPsoDesc.BlendState.RenderTarget[0] = transparencyBlendDesc;

ThrowIfFailed(md3dDevice->CreateGraphicsPipelineState(
    &transparentPsoDesc, IID_PPV_ARGS(&mPSOs["transparent"])));
```

混合会消耗额外 per-pixel 计算，只在需要时启用。

---

## 10.5 混合示例

### 10.5.1 不写入颜色

```
F_src = ZERO, F_dst = ONE, BlendOp = ADD
=> C = 0 ⊗ C_src + 1 ⊗ C_dst = C_dst
```

等价于设置 `RenderTargetWriteMask = 0`，不写入任何颜色通道。

### 10.5.2 加法 / 减法混合

```cpp
// 加法：C = C_src + C_dst
SrcBlend = D3D12_BLEND_ONE;
DestBlend = D3D12_BLEND_ONE;
BlendOp = D3D12_BLEND_OP_ADD;

// 减法：C = C_dst - C_src
SrcBlend = D3D12_BLEND_ONE;
DestBlend = D3D12_BLEND_ONE;
BlendOp = D3D12_BLEND_OP_SUBTRACT;
```

加法混合使图像变亮，常用于光晕、粒子效果。

### 10.5.3 乘法混合

```cpp
// C = C_src ⊗ C_dst
SrcBlend = D3D12_BLEND_ZERO;
DestBlend = D3D12_BLEND_SRC_COLOR;
BlendOp = D3D12_BLEND_OP_ADD;
```

### 10.5.4 透明度混合

```cpp
SrcBlend = D3D12_BLEND_SRC_ALPHA;
DestBlend = D3D12_BLEND_INV_SRC_ALPHA;
BlendOp = D3D12_BLEND_OP_ADD;
```

结果：`C = a_s · C_src + (1-a_s) · C_dst`

**绘制顺序规则**：
1. 先绘制**不透明**物体
2. 将**半透明**物体按距离相机从远到近排序
3. 按**从后往前**顺序绘制半透明物体

原因是透明物体需要与其后方的场景像素混合，因此后方像素必须先写入后台缓冲区。

### 10.5.5 混合与深度缓冲区

对于加法/减法/乘法混合，物体之间不应互相遮挡（颜色是累加的）。解决方法是：在绘制混合物体集合时**禁用深度写入**，但保持深度测试和读取：

```cpp
D3D12_DEPTH_STENCIL_DESC dsDesc;
dsDesc.DepthEnable = true;
dsDesc.DepthWriteMask = D3D12_DEPTH_WRITE_MASK_ZERO;  // 禁用深度写入
dsDesc.DepthFunc = D3D12_COMPARISON_LESS;
```

这样混合物体不会遮挡同一集合中的其他物体，但仍会被不透明物体正确遮挡。

---

## 10.6 Alpha 通道

源 Alpha 来自漫反射材质。在本框架中，漫反射反照率的 Alpha 值从纹理采样得到：

```hlsl
float4 diffuseAlbedo = gDiffuseMap.Sample(gsamAnisotropicWrap, pin.TexC) * gDiffuseAlbedo;
litColor.a = diffuseAlbedo.a;
```

可在 Photoshop 等工具中为图像添加 Alpha 通道，然后保存为支持 Alpha 的 DDS 格式。

---

## 10.7 裁剪像素

**`clip(x)` 函数**：只能在像素着色器中调用，若 `x < 0` 则丢弃当前像素。

用途：渲染栅栏等 Alpha 测试（Alpha Test）对象——像素要么完全不透明，要么完全透明。

```hlsl
float4 PS(VertexOut pin) : SV_Target {
    float4 diffuseAlbedo = gDiffuseMap.Sample(gsamAnisotropicWrap, pin.TexC) * gDiffuseAlbedo;

#ifdef ALPHA_TEST
    // 尽早 clip，跳过剩余着色器代码
    clip(diffuseAlbedo.a - 0.1f);
#endif

    // ... 光照计算
    litColor.a = diffuseAlbedo.a;
    return litColor;
}
```

**Alpha Test vs Alpha Blend**：
- `clip` 比混合更高效：无需混合计算、无需排序、可提前退出着色器
- 但 `clip` 会禁用 early-z 优化（因为像素可能被丢弃）
- 由于过滤，Alpha 通道可能被轻微模糊，应留一定容差（如 clip alpha < 0.1 而非 alpha == 0）

对于 Alpha Tested 对象，通常禁用背面剔除（因为可以看到盒子内部）：

```cpp
D3D12_GRAPHICS_PIPELINE_STATE_DESC alphaTestedPsoDesc = opaquePsoDesc;
alphaTestedPsoDesc.RasterizerState.CullMode = D3D12_CULL_MODE_NONE;
```

---

## 10.8 雾效

雾效用于模拟天气状况，同时可掩盖远处的渲染瑕疵和物体弹出（popping）现象。

**线性雾模型**：

```
foggedColor = litColor + s · (fogColor - litColor)
            = (1-s) · litColor + s · fogColor
```

其中 `s` 是雾的权重，`0 ≤ s ≤ 1`：

```
s = saturate((dist(p, E) - fogStart) / fogRange)
```

| 距离条件 | 效果 |
|---------|------|
| `dist < fogStart` | `s = 0`，无雾 |
| `dist > fogStart + fogRange` | `s = 1`，完全被雾遮挡 |
| `fogStart < dist < fogEnd` | 线性过渡 |

**HLSL 实现**：

```hlsl
cbuffer cbPass : register(b1) {
    // ...
    float4 gFogColor;
    float  gFogStart;
    float  gFogRange;
};

float4 PS(VertexOut pin) : SV_Target {
    // ... 光照计算得到 litColor

    float3 toEyeW = gEyePosW - pin.PosW;
    float distToEye = length(toEyeW);

#ifdef FOG
    float fogAmount = saturate((distToEye - gFogStart) / gFogRange);
    litColor = lerp(litColor, gFogColor, fogAmount);
#endif

    litColor.a = diffuseAlbedo.a;
    return litColor;
}
```

注意 `toEyeW` 的长度和归一化可以共用：

```hlsl
// 高效写法：只计算一次长度
float3 toEyeW = gEyePosW - pin.PosW;
float distToEye = length(toEyeW);
toEyeW /= distToEye;  // 归一化

// 避免低效写法：
// float3 toEye = normalize(gEyePosW - pin.PosW);  // 内部计算了 length
// float distToEye = distance(gEyePosW, pin.PosW); // 又计算一次 length
```

雾效通过 `D3D_SHADER_MACRO` 条件编译控制：

```cpp
const D3D_SHADER_MACRO defines[] = {
    "FOG", "1",
    nullptr, nullptr
};
mShaders["opaquePS"] = d3dUtil::CompileShader(
    L"Shaders\\Default.hlsl", defines, "PS", "ps_5_0");
```
