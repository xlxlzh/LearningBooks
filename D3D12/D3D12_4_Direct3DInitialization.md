# 第 4 章 Direct3D 12 初始化

> **学习目标**：
> 1. 理解 Direct3D 12 在 3D 硬件编程中的角色。
> 2. 理解 COM 在 Direct3D 中的作用。
> 3. 学习基础图形概念：2D 图像存储方式、翻页、深度缓冲、多重采样、CPU/GPU 交互方式。
> 4. 学会使用性能计数器实现高精度计时。
> 5. 掌握 Direct3D 12 的完整初始化流程。
> 6. 熟悉本书所有示例共同使用的应用程序框架结构。

---

## 4.1 预备知识（Preliminaries）

在正式初始化 Direct3D 之前，我们需要先了解几个核心概念。这些概念不仅会在初始化过程中用到，还会贯穿整本书。

### 4.1.1 Direct3D 12 概览

Direct3D（通常简写为 D3D）是微软提供的一套**底层图形 API**，它允许开发者直接控制和编程 GPU，从而利用硬件加速来渲染虚拟 3D 世界。

打个比方，如果你想让 GPU 清除屏幕上的某个渲染目标（比如后台缓冲区），你会调用类似这样的 Direct3D 方法：

```cpp
mCommandList->ClearRenderTargetView(
    mBackBufferView,           // 要清除的渲染目标视图
    Colors::LightSteelBlue,    // 清除颜色（淡钢蓝）
    0, nullptr);               // 不限制清除区域
```

Direct3D 层和硬件驱动会把这些高层命令翻译成 GPU 能理解的底层机器指令。因此，只要 GPU 支持你所用的 Direct3D 版本，你就不必关心具体是 NVIDIA、AMD 还是 Intel 的显卡。NVIDIA、AMD、Intel 等厂商必须与 Direct3D 团队合作，提供兼容的驱动程序。

**Direct3D 12 相对于 D3D11 的核心改进**：
- **大幅降低 CPU 开销**：D3D12 重新设计以减少驱动层开销。
- **强化多线程支持**：允许多个 CPU 核心同时录制渲染命令。

代价是 D3D12 成为了一套**更低层、更暴露 GPU 架构细节的 API**。开发者需要做更多的"记账工作"（手动管理资源状态、同步、内存分配等），但换来的回报是显著的性能提升。

### 4.1.2 COM 技术

Direct3D 建立在 **COM（Component Object Model）** 技术之上。COM 让 DirectX 可以**跨编程语言**使用，并保持向后兼容性。

对我们来说，COM 对象可以简单地当成 C++ 类来使用，但有以下几点必须注意：

1. **不能直接用 `new` 创建 COM 对象**，而是通过特殊函数或其他 COM 接口的方法来获取指针。
2. **COM 对象使用引用计数**。用完之后要调用 `Release()` 方法，而不是 `delete`。当引用计数降到 0 时，COM 对象会自动释放内存。

为了简化 COM 对象的生命周期管理，Windows 提供了 `Microsoft::WRL::ComPtr` 智能指针（需要包含 `<wrl.h>`）。当 `ComPtr` 超出作用域时，它会自动调用 `Release()`。

本书中最常用的三个 `ComPtr` 方法：

| 方法 | 作用 | 典型用法 |
|------|------|---------|
| `Get()` | 返回底层 COM 接口指针 | 传给需要原始 COM 指针的函数参数 |
| `GetAddressOf()` | 返回 COM 接口指针的地址 | 用于接收函数输出的 COM 接口指针 |
| `Reset()` | 设为 `nullptr` 并递减引用计数 | 提前释放 COM 对象 |

示例：

```cpp
#include <wrl.h>
using Microsoft::WRL::ComPtr;

ComPtr<ID3D12RootSignature> mRootSignature;
// Get() 获取原始指针
mCommandList->SetGraphicsRootSignature(mRootSignature.Get());

ComPtr<ID3D12CommandAllocator> mDirectCmdListAlloc;
// GetAddressOf() 获取指针地址，用于输出参数
ThrowIfFailed(md3dDevice->CreateCommandAllocator(
    D3D12_COMMAND_LIST_TYPE_DIRECT,
    mDirectCmdListAlloc.GetAddressOf()));
```

> **命名约定**：所有 COM 接口名称都以大写字母 `I` 开头，例如 `ID3D12GraphicsCommandList`。

### 4.1.3 纹理格式（Texture Formats）

**纹理（Texture）** 本质上是一个数据元素阵列。最常见的用法是存储 2D 图像（每个元素存储一个像素颜色），但纹理的用途远不止于此——比如在法线贴图中，每个元素存储的是一个 3D 向量。

Direct3D 中纹理格式由 `DXGI_FORMAT` 枚举描述。以下是一些常见格式：

| 格式 | 含义 |
|------|------|
| `DXGI_FORMAT_R32G32B32_FLOAT` | 3 个 32 位浮点数（常用于存储位置/法线向量） |
| `DXGI_FORMAT_R16G16B16A16_UNORM` | 4 个 16 位无符号整数，归一化到 [0, 1] |
| `DXGI_FORMAT_R8G8B8A8_UNORM` | 4 个 8 位无符号整数，归一化到 [0, 1]（最常见的颜色格式） |
| `DXGI_FORMAT_R8G8B8A8_SNORM` | 4 个 8 位有符号整数，归一化到 [-1, 1] |
| `DXGI_FORMAT_R32G32_UINT` | 2 个 32 位无符号整数 |

R、G、B、A 分别代表红、绿、蓝和 Alpha（透明度）通道。但正如前面所说，即使格式名称带有颜色含义，你也可以用它来存储任意数据——例如 `DXGI_FORMAT_R32G32B32_FLOAT` 完全可以存储任意 3D 浮点向量。

此外还有**无类型格式（Typeless）**，例如 `DXGI_FORMAT_R16G16B16A16_TYPELESS`——它只预留内存，在后续绑定到管线时才指定具体的数据类型。

### 4.1.4 交换链与翻页（Swap Chain & Page Flipping）

为了避免画面撕裂（tearing），Direct3D 使用**双缓冲（Double Buffering）**机制：

1. **前台缓冲区（Front Buffer）**：当前正在显示在屏幕上的图像。
2. **后台缓冲区（Back Buffer）**：GPU 正在绘制的下一帧图像。

当 GPU 完成一帧的绘制后，两个缓冲区进行**交换（Swap/Flip）**：后台缓冲区变成前台缓冲区显示出来，而原来的前台缓冲区则变成下一帧的后台缓冲区。这个交换操作由 **交换链（Swap Chain）** 管理。

```
┌─────────────────┐         ┌─────────────────┐
│   Front Buffer  │◄───────►│   Back Buffer   │
│  (正在显示)     │  交换   │  (正在绘制)     │
└─────────────────┘         └─────────────────┘
        │                            │
        └────────── 显示器 ──────────┘
```

交换链通过 **IDXGISwapChain** 接口管理（注意：这是 DXGI API 的一部分，不是纯 D3D12 API）。

### 4.1.5 深度缓冲（Depth Buffering）

在 3D 场景中，多个物体会互相遮挡。为了正确绘制这种**遮挡关系**，Direct3D 使用**深度缓冲（Depth Buffer）**，也叫 Z 缓冲。

**工作原理**：
- 深度缓冲是一个与屏幕分辨率匹配的 2D 纹理，每个像素存储一个深度值（表示该像素对应的物体距离相机有多远）。
- 当 GPU 要绘制一个像素时，会先比较新像素的深度值和深度缓冲中已存储的深度值。
- 只有当新像素**更近**（深度值更小，假设使用标准深度范围）时，才会写入颜色和深度。

深度缓冲通常与模板缓冲（Stencil Buffer）共享同一个资源。常用格式为 `DXGI_FORMAT_D24_UNORM_S8_UINT`：24 位深度 + 8 位模板。

```
场景：一个红色方块在蓝色方块前面

深度测试前：        深度测试后：
┌───┬───┐          ┌───┬───┐
│红 │蓝 │   →→→    │红 │红 │  (红色更近，覆盖了蓝色)
└───┴───┘          └───┴───┘
```

### 4.1.6 资源与描述符（Resources & Descriptors）

在 Direct3D 12 中，**资源（Resource）** 不会直接绑定到渲染管线。相反，资源通过**描述符（Descriptor）**来引用。

**为什么需要描述符这层间接层？**

1. GPU 资源本质上是通用的内存块。同一块纹理数据，既可能作为**渲染目标**（写入），也可能作为**着色器资源**（读取）。资源本身不说明它的用途。
2. 你可能只想绑定资源的一部分子区域。
3. 如果资源创建时用了无类型格式，描述符需要明确指定类型。

**描述符 = 资源 + 用途说明**。它告诉 GPU：这是什么资源、怎么使用它、格式是什么、需要哪个子区域。

本书中使用的描述符类型：

| 描述符类型 | 用途 |
|-----------|------|
| **CBV/SRV/UAV** | 常量缓冲视图 / 着色器资源视图 / 无序访问视图 |
| **Sampler** | 采样器状态（纹理过滤、寻址模式等） |
| **RTV** | 渲染目标视图（后台缓冲、离屏渲染目标） |
| **DSV** | 深度/模板视图 |

> **术语**：描述符（Descriptor）和视图（View）是同义词。旧版 Direct3D 使用"View"，D3D12 文档中两者混用。

**描述符堆（Descriptor Heap）**：描述符的内存池。每种描述符类型需要独立的描述符堆。描述符应在**初始化时创建**，因为创建时会有类型检查和验证开销。

### 4.1.7 多重采样理论（Multisampling Theory）

显示器上的像素不是无限小的。当用像素矩阵来近似表示直线或三角形边缘时，会产生明显的**锯齿（Aliasing）**——阶梯状边缘。

**超级采样（Supersampling）**：把后台缓冲区和深度缓冲区做成屏幕分辨率的 4 倍大，渲染后再平均缩小。效果很好但性能开销极大（4 倍像素处理 + 4 倍显存）。

**多重采样（Multisampling, MSAA）**：折中方案。以 4× MSAA 为例：
- 每个像素分为 4 个子像素（subpixel）。
- 但**颜色只在像素中心计算一次**，然后根据深度/模板测试和几何覆盖情况把颜色共享给可见的子像素。
- 最终呈现时，4 个子像素的颜色取平均值。

关键区别：
- 超级采样 = **每个子像素独立计算颜色**（最准但最贵）。
- 多重采样 = **每个像素只算一次颜色**，共享给子像素（快得多，边缘效果足够好）。

4× MSAA 几乎所有 Direct3D 11 级别硬件都支持。

### 4.1.8 Direct3D 中的多重采样

多重采样设置通过 `DXGI_SAMPLE_DESC` 结构描述：

```cpp
typedef struct DXGI_SAMPLE_DESC {
    UINT Count;    // 每像素采样数（1 = 关闭 MSAA，4 = 4× MSAA）
    UINT Quality;  // 质量等级（0 ~ NumQualityLevels-1）
} DXGI_SAMPLE_DESC;
```

查询某格式和质量等级支持的代码：

```cpp
D3D12_FEATURE_DATA_MULTISAMPLE_QUALITY_LEVELS msQualityLevels;
msQualityLevels.Format = mBackBufferFormat;  // 如 DXGI_FORMAT_R8G8B8A8_UNORM
msQualityLevels.SampleCount = 4;
msQualityLevels.Flags = D3D12_MULTISAMPLE_QUALITY_LEVELS_FLAG_NONE;
msQualityLevels.NumQualityLevels = 0;  // 输入时设为 0

ThrowIfFailed(md3dDevice->CheckFeatureSupport(
    D3D12_FEATURE_MULTISAMPLE_QUALITY_LEVELS,
    &msQualityLevels,
    sizeof(msQualityLevels)));

// 输出：msQualityLevels.NumQualityLevels 表示支持的质量等级数量
```

> 后台缓冲区和深度缓冲区必须使用**相同的多重采样设置**。

### 4.1.9 特性等级（Feature Levels）

特性等级定义了 GPU 必须支持的一套完整功能集合。例如，支持 Feature Level 11_0 的 GPU 必须支持几乎所有 Direct3D 11 的功能。

```cpp
enum D3D_FEATURE_LEVEL {
    D3D_FEATURE_LEVEL_9_1  = 0x9100,
    D3D_FEATURE_LEVEL_9_2  = 0x9200,
    D3D_FEATURE_LEVEL_9_3  = 0x9300,
    D3D_FEATURE_LEVEL_10_0 = 0xa000,
    D3D_FEATURE_LEVEL_10_1 = 0xa100,
    D3D_FEATURE_LEVEL_11_0 = 0xb000,
    D3D_FEATURE_LEVEL_11_1 = 0xb100,
    D3D_FEATURE_LEVEL_12_0 = 0xc000,
    D3D_FEATURE_LEVEL_12_1 = 0xc100
};
```

实际应用中，你可以从最新特性等级开始向下探测，以支持更广泛的硬件。本书假设目标硬件至少支持 `D3D_FEATURE_LEVEL_11_0`。

### 4.1.10 DirectX 图形基础架构（DXGI）

**DXGI（DirectX Graphics Infrastructure）** 是与 Direct3D 配合使用的 API，负责处理多个图形 API 共有的任务：

- 创建和管理交换链
- 全屏模式切换
- 枚举显示适配器、显示器、显示模式
- 定义表面格式（`DXGI_FORMAT`）

**关键 DXGI 接口**：

| 接口 | 作用 |
|------|------|
| `IDXGIFactory` / `IDXGIFactory4` | 创建交换链、枚举显示适配器 |
| `IDXGIAdapter` | 代表一个显示适配器（物理显卡或软件模拟器） |
| `IDXGIOutput` | 代表一个显示输出（如显示器） |

枚举系统中所有适配器的示例：

```cpp
ComPtr<IDXGIFactory4> mdxgiFactory;
CreateDXGIFactory1(IID_PPV_ARGS(&mdxgiFactory));

UINT i = 0;
ComPtr<IDXGIAdapter> adapter;
while (mdxgiFactory->EnumAdapters(i, &adapter) != DXGI_ERROR_NOT_FOUND)
{
    DXGI_ADAPTER_DESC desc;
    adapter->GetDesc(&desc);
    // desc.Description 包含适配器名称，如 "NVIDIA GeForce GTX 760"
    ++i;
}
```

### 4.1.11 检查特性支持

`ID3D12Device::CheckFeatureSupport` 可以查询硬件支持的各项特性。

```cpp
HRESULT ID3D12Device::CheckFeatureSupport(
    D3D12_FEATURE Feature,        // [in] 要查询的特性类型
    void* pFeatureSupportData,    // [in/out] 特性数据结构（输入查询条件，输出结果）
    UINT FeatureSupportDataSize   // [in] 数据结构大小
);
```

**常用特性类型**：

| 特性类型 | 数据结构 | 查询内容 |
|---------|---------|---------|
| `D3D12_FEATURE_D3D12_OPTIONS` | `D3D12_FEATURE_DATA_D3D12_OPTIONS` | 各种 D3D12 可选功能（双精度着色器、UAV 绑定层数、VPAndRTArrayIndexFromAnyShaderFeedingRasterizer 等） |
| `D3D12_FEATURE_FEATURE_LEVELS` | `D3D12_FEATURE_DATA_FEATURE_LEVELS` | 支持的特性等级列表 |
| `D3D12_FEATURE_FORMAT_SUPPORT` | `D3D12_FEATURE_DATA_FORMAT_SUPPORT` | 特定格式支持的用法（能否作为渲染目标、深度缓冲、着色器资源等） |
| `D3D12_FEATURE_MULTISAMPLE_QUALITY_LEVELS` | `D3D12_FEATURE_DATA_MULTISAMPLE_QUALITY_LEVELS` | 多重采样质量等级 |
| `D3D12_FEATURE_ARCHITECTURE` | `D3D12_FEATURE_DATA_ARCHITECTURE` | GPU 架构信息（是否统一内存、是否缓存一致等） |

**示例：查询某格式是否支持作为渲染目标**：

```cpp
D3D12_FEATURE_DATA_FORMAT_SUPPORT formatSupport;
formatSupport.Format = DXGI_FORMAT_R8G8B8A8_UNORM;
formatSupport.Support1 = D3D12_FORMAT_SUPPORT1_NONE;
formatSupport.Support2 = D3D12_FORMAT_SUPPORT2_NONE;

md3dDevice->CheckFeatureSupport(
    D3D12_FEATURE_FORMAT_SUPPORT,
    &formatSupport, sizeof(formatSupport));

// 检查是否支持作为渲染目标
bool supportsRenderTarget = (formatSupport.Support1 &
    D3D12_FORMAT_SUPPORT1_RENDER_TARGET) != 0;
```

**示例：查询支持的特性等级**：

```cpp
D3D_FEATURE_LEVEL featureLevels[] = {
    D3D_FEATURE_LEVEL_12_1,
    D3D_FEATURE_LEVEL_12_0,
    D3D_FEATURE_LEVEL_11_1,
    D3D_FEATURE_LEVEL_11_0
};

D3D12_FEATURE_DATA_FEATURE_LEVELS levelData;
levelData.NumFeatureLevels = _countof(featureLevels);
levelData.pFeatureLevelsRequested = featureLevels;

md3dDevice->CheckFeatureSupport(
    D3D12_FEATURE_FEATURE_LEVELS,
    &levelData, sizeof(levelData));

// levelData.MaxSupportedFeatureLevel 返回实际支持的最高等级
```

### 4.1.12 资源驻留（Residency）

D3D12 允许开发者**显式管理资源在显存中的驻留状态**。当显存不足时，你可以将不常用的资源"驱逐"出显存，在需要时再加载回来。这对于管理大型开放世界游戏或复杂场景非常有用。

```cpp
// 使资源驻留（加载到显存）
md3dDevice->MakeResident(1, &mResource);

// 驱逐资源（释放显存）
md3dDevice->Evict(1, &mResource);
```

---

## 4.2 CPU/GPU 交互

图形编程中，**CPU 和 GPU 是并行工作的**。理想情况下，两者应该尽可能同时忙碌，只在必要时进行同步。

### 4.2.1 命令队列与命令列表（Command Queue & Command Lists）

GPU 有一个**命令队列（Command Queue）**。CPU 通过命令列表（Command List）把渲染命令提交到这个队列中：

```
CPU 侧：                          GPU 侧：
┌─────────────┐                  ┌─────────────┐
│ 录制命令到   │ ──提交──>        │  命令队列    │
│ 命令列表     │   ExecuteCommandLists  │  (FIFO)     │
└─────────────┘                  └──────┬──────┘
                                       │
                                       ▼
                                ┌─────────────┐
                                │  GPU 按顺序  │
                                │  执行命令    │
                                └─────────────┘
```

关键理解：**命令加入队列后不会立即执行**，GPU 会按 FIFO（先进先出）顺序处理队列中的命令。

**三个核心接口**：

| 接口 | 作用 |
|------|------|
| `ID3D12CommandQueue` | 命令队列，接收并调度命令列表 |
| `ID3D12CommandAllocator` | 命令分配器，为命令列表提供内存存储 |
| `ID3D12GraphicsCommandList` | 图形命令列表，录制具体的绘制命令 |

**创建命令队列**：

```cpp
D3D12_COMMAND_QUEUE_DESC queueDesc = {};
queueDesc.Type = D3D12_COMMAND_LIST_TYPE_DIRECT;  // 直接执行命令
queueDesc.Flags = D3D12_COMMAND_QUEUE_FLAG_NONE;

ThrowIfFailed(md3dDevice->CreateCommandQueue(
    &queueDesc, IID_PPV_ARGS(&mCommandQueue)));
```

#### `CreateCommandQueue` 与 `D3D12_COMMAND_QUEUE_DESC` 详解

```cpp
HRESULT ID3D12Device::CreateCommandQueue(
    const D3D12_COMMAND_QUEUE_DESC* pDesc,   // [in] 命令队列描述
    REFIID riid,                             // [in] 接口 ID
    void** ppCommandQueue                    // [out] 输出 ID3D12CommandQueue
);
```

**`D3D12_COMMAND_QUEUE_DESC` 结构体**：

```cpp
typedef struct D3D12_COMMAND_QUEUE_DESC {
    D3D12_COMMAND_LIST_TYPE Type;     // 命令队列类型
    INT Priority;                     // 优先级（0=Normal，非零值需要硬件支持）
    D3D12_COMMAND_QUEUE_FLAGS Flags;  // 队列标志
    UINT NodeMask;                    // 多 GPU 时的节点掩码（单 GPU 设为 0）
} D3D12_COMMAND_QUEUE_DESC;
```

| 字段 | 说明 |
|------|------|
| `Type` | 命令队列类型。本书主要使用 `D3D12_COMMAND_LIST_TYPE_DIRECT`（直接执行图形/计算命令）。其他类型包括 `COMPUTE`（计算队列，用于异步计算）、`COPY`（复制队列，专用于资源复制）。 |
| `Priority` | 队列执行优先级。`D3D12_COMMAND_QUEUE_PRIORITY_NORMAL` (0) 是默认值；`HIGH` 或 `GLOBAL_REALTIME` 可提升优先级，但 `GLOBAL_REALTIME` 需要管理员权限且可能阻塞系统。 |
| `Flags` | `D3D12_COMMAND_QUEUE_FLAG_NONE`（默认）或 `DISABLE_GPU_TIMEOUT`（禁用 GPU 超时检测，仅调试使用）。 |
| `NodeMask` | 对于单 GPU 系统设为 0。多 GPU（SLI/CrossFire）系统中，每个物理 GPU 对应一个节点，用位掩码指定队列关联到哪个节点。 |

**`ID3D12CommandQueue` 核心方法**：

| 方法 | 作用 |
|------|------|
| `ExecuteCommandLists(UINT Count, ID3D12CommandList* const* ppCommandLists)` | 将一组命令列表提交到 GPU 执行。命令列表按数组顺序执行。这是把 CPU 录制的命令真正交给 GPU 的入口。 |
| `Signal(ID3D12Fence* pFence, UINT64 Value)` | 在队列中插入一个"信号点"，当 GPU 执行到此点时，将围栏值设为指定值。用于 CPU/GPU 同步。 |
| `Wait(ID3D12Fence* pFence, UINT64 Value)` | 让 GPU 在队列中等待，直到围栏值达到或超过指定值。用于 GPU 与 GPU 之间的同步（如计算队列等待图形队列）。 |
| `GetTimestampFrequency(UINT64* pFrequency)` | 查询 GPU 时间戳频率，用于 GPU 计时。 |

**创建命令分配器和命令列表**：

```cpp
// 命令分配器
ThrowIfFailed(md3dDevice->CreateCommandAllocator(
    D3D12_COMMAND_LIST_TYPE_DIRECT,
    IID_PPV_ARGS(mDirectCmdListAlloc.GetAddressOf())));

// 命令列表（初始状态为"打开"）
ThrowIfFailed(md3dDevice->CreateCommandList(
    0,                                          // 单 GPU
    D3D12_COMMAND_LIST_TYPE_DIRECT,
    mDirectCmdListAlloc.Get(),                  // 关联的分配器
    nullptr,                                    // 初始 PSO（本章不需要）
    IID_PPV_ARGS(mCommandList.GetAddressOf())));

// 创建后先关闭，因为第一次使用时要 Reset
mCommandList->Close();
```

**命令列表的生命周期**：

1. **录制（Record）**：调用 `Reset()` 打开命令列表 → 添加各种命令 → 调用 `Close()` 关闭。
2. **提交（Submit）**：把关闭后的命令列表传给 `ExecuteCommandLists()`。
3. **重用（Reuse）**：等 GPU 执行完后，可以 `Reset()` 命令列表并重新录制。命令分配器也可以 `Reset()` 清空（但必须先确保 GPU 已完成其中的所有命令）。

> **重要规则**：一个分配器不能同时被多个打开的命令列表使用。所有关联的命令列表必须处于关闭状态，除了当前正在录制的那个。

### 4.2.2 CPU/GPU 同步

由于 CPU 和 GPU 并行运行，会产生同步问题。典型场景：

```
CPU 时间线：  更新资源 R 为 p1 ──> 提交绘制命令 C ──> 更新资源 R 为 p2
                                           ↑
                                           │ 错误！GPU 可能还没执行 C
GPU 时间线：                              执行 C（但 R 已经是 p2 了）
```

**围栏（Fence）** 是解决这个问题的机制。

#### Fence 详解

```cpp
// 1. 创建围栏
HRESULT ID3D12Device::CreateFence(
    UINT64 InitialValue,              // [in] 围栏初始值
    D3D12_FENCE_FLAGS Flags,          // [in] 围栏标志
    REFIID riid,                      // [in] 接口 ID
    void** ppFence                    // [out] 输出 ID3D12Fence
);
```

| 参数 | 说明 |
|------|------|
| `InitialValue` | 围栏的初始整数值。通常设为 0。 |
| `Flags` | `NONE`（默认）或 `SHARED`（允许跨进程/设备共享围栏）。 |
| `ppFence` | 输出 `ID3D12Fence` 接口。围栏内部维护一个 64 位无符号整数。 |

**Fence 核心方法**：

| 方法 | 作用 |
|------|------|
| `GetCompletedValue()` | 查询围栏当前已完成的值（GPU 已执行到的位置）。如果返回值小于你期望的值，说明 GPU 还未执行到对应位置。 |
| `SetEventOnCompletion(UINT64 Value, HANDLE hEvent)` | 当围栏值达到或超过指定值时，触发指定的事件对象。这是非阻塞的——调用后立即返回，事件会在未来某个时刻自动触发。 |
| `Signal(UINT64 Value)` | 在 CPU 侧直接将围栏值设为指定值。很少直接使用，通常用 `CommandQueue::Signal()` 让 GPU 来更新围栏。 |

**Flush 命令队列的完整实现**：

```cpp
UINT64 mCurrentFence = 0;

void FlushCommandQueue()
{
    // 推进围栏值，标记一个新的同步点
    mCurrentFence++;

    // 在命令队列末尾添加一个"信号"指令
    // GPU 按顺序执行命令，当它执行到这个 Signal 指令时，
    // 会将围栏值更新为 mCurrentFence
    ThrowIfFailed(mCommandQueue->Signal(mFence.Get(), mCurrentFence));

    // CPU 检查 GPU 是否已经达到这个围栏值
    if (mFence->GetCompletedValue() < mCurrentFence)
    {
        // 创建 Windows 事件对象
        HANDLE eventHandle = CreateEventEx(
            nullptr,           // 默认安全属性
            false,             // 手动重置 = false（自动重置）
            false,             // 初始状态 = 无信号
            EVENT_ALL_ACCESS); // 访问权限

        // 注册回调：当围栏值达到 mCurrentFence 时触发 eventHandle
        ThrowIfFailed(mFence->SetEventOnCompletion(mCurrentFence, eventHandle));

        // 阻塞等待事件触发（INFINITE = 无限等待）
        WaitForSingleObject(eventHandle, INFINITE);

        // 清理事件句柄
        CloseHandle(eventHandle);
    }
}
```

**`ID3D12CommandQueue::Signal` 与 `ID3D12Fence::Signal` 的区别**：

| 方法 | 执行者 | 时机 | 用途 |
|------|--------|------|------|
| `CommandQueue::Signal(fence, value)` | GPU | GPU 执行到队列中该位置时 | 标记 GPU 进度，用于 CPU 等待 GPU |
| `Fence::Signal(value)` | CPU | 调用后立即执行 | 强制设置围栏值，用于 GPU 等待 CPU |

**工作原理图解**：

```
CPU 时间线：
  Signal(fence, n+1) ──> Wait...
                          │
                          │  当 GPU 执行完 Signal 之前的所有命令后
                          │  fence 的值变为 n+1，事件触发，Wait 返回
                          ▼
                        继续执行

GPU 时间线（命令队列）：
  命令A ──> 命令B ──> ... ──> Signal(fence, n+1) ──> 命令C ──> ...
                               ↑
                               在此处 fence 值更新为 n+1
```

> **注意**：Flush 会让 CPU 等待 GPU，破坏了并行性。因此应尽量减少同步点。本书在前几章使用简单的 Flush 方案，第 7 章会引入更高效的帧资源（Frame Resources）方案来避免每帧都 Flush。

### 4.2.3 资源过渡（Resource Transitions）

GPU 资源有**状态（State）**。例如：纹理可能处于"渲染目标状态"（GPU 正在写入它），也可能处于"着色器资源状态"（GPU 正在读取它）。

如果 GPU 还没写完一个资源就开始读取它，会导致**资源冲突（Resource Hazard）**。Direct3D 12 要求开发者**显式声明资源状态转换**，这样 GPU 可以在必要时插入等待或缓存刷新操作。

**资源屏障（Resource Barrier）** 用于声明状态转换：

```cpp
// 示例：把后台缓冲区从"呈现状态"转换为"渲染目标状态"
mCommandList->ResourceBarrier(1,
    &CD3DX12_RESOURCE_BARRIER::Transition(
        mSwapChainBuffer[mCurrBackBuffer].Get(),
        D3D12_RESOURCE_STATE_PRESENT,           // 之前的状态
        D3D12_RESOURCE_STATE_RENDER_TARGET));   // 目标状态

// 绘制完成后，转回"呈现状态"
mCommandList->ResourceBarrier(1,
    &CD3DX12_RESOURCE_BARRIER::Transition(
        mSwapChainBuffer[mCurrBackBuffer].Get(),
        D3D12_RESOURCE_STATE_RENDER_TARGET,
        D3D12_RESOURCE_STATE_PRESENT));
```

> `CD3DX12_RESOURCE_BARRIER` 是 `d3dx12.h` 中定义的辅助结构体，极大简化了屏障的创建。建议总是使用它。

#### `ResourceBarrier` 详解

```cpp
void ID3D12GraphicsCommandList::ResourceBarrier(
    UINT NumBarriers,                              // [in] 屏障数量
    const D3D12_RESOURCE_BARRIER* pBarriers        // [in] 屏障描述数组
);
```

`D3D12_RESOURCE_BARRIER` 有三种类型：

**1. 转换屏障（Transition）**：最常用的类型，用于改变资源的状态。

```cpp
CD3DX12_RESOURCE_BARRIER::Transition(
    ID3D12Resource* pResource,           // 要转换的资源
    D3D12_RESOURCE_STATES stateBefore,   // 转换前的状态
    D3D12_RESOURCE_STATES stateAfter,    // 转换后的状态
    UINT subresource = D3D12_RESOURCE_BARRIER_ALL_SUBRESOURCES  // 子资源索引
);
```

- `stateBefore` 必须与资源当前实际状态一致，否则调试层会报错。
- `subresource` 默认 `ALL_SUBRESOURCES` 表示转换整个资源的所有子资源（所有 mipmap 层级和数组元素）。

**2. 别名屏障（Aliasing）**：用于在同一物理内存位置复用不同资源时，通知 GPU 清除缓存。

```cpp
CD3DX12_RESOURCE_BARRIER::Aliasing(
    ID3D12Resource* pResourceBefore,     // 之前占用的资源（可为 nullptr）
    ID3D12Resource* pResourceAfter       // 之后要使用的资源
);
```

**3. UAV 屏障（Unordered Access View）**：用于同步对同一 UAV 资源的多次无序访问，确保前一次写入完成后才开始下一次访问。

```cpp
CD3DX12_RESOURCE_BARRIER::UAV(ID3D12Resource* pResource);
```

- 当计算着色器先写入 UAV，随后另一个着色器阶段要读取同一 UAV 时，需要插入 UAV 屏障。
- 如果 `pResource` 为 `nullptr`，表示全局 UAV 栅栏，等待所有 UAV 操作完成。

**转换规则与常见路径**：

```
COMMON ──→ RENDER_TARGET ──→ PRESENT
   │            │
   └──→ COPY_SOURCE/DEST
   │
   └──→ DEPTH_WRITE ──→ PIXEL_SHADER_RESOURCE
```

> **注意**：某些状态转换是"单向"的或需要中间状态。调试层会验证转换路径的合法性。例如，不能直接从 `RENDER_TARGET` 转到 `COPY_SOURCE`（在某些驱动上需要先转到 `COMMON`）。

常见的资源状态：

| 状态 | 含义 |
|------|------|
| `D3D12_RESOURCE_STATE_COMMON` | 通用状态，可作为任何状态的初始状态 |
| `D3D12_RESOURCE_STATE_RENDER_TARGET` | 作为渲染目标被写入 |
| `D3D12_RESOURCE_STATE_DEPTH_WRITE` | 作为深度/模板缓冲区被写入 |
| `D3D12_RESOURCE_STATE_PIXEL_SHADER_RESOURCE` | 在像素着色器中被读取 |
| `D3D12_RESOURCE_STATE_NON_PIXEL_SHADER_RESOURCE` | 在非像素着色器阶段被读取 |
| `D3D12_RESOURCE_STATE_COPY_DEST` | 作为复制操作的目标 |
| `D3D12_RESOURCE_STATE_COPY_SOURCE` | 作为复制操作的源 |
| `D3D12_RESOURCE_STATE_PRESENT` | 用于呈现到屏幕 |

### 4.2.4 多线程命令录制

Direct3D 12 允许多个 CPU 线程**同时录制命令列表**。每个线程需要：
- 独立的 `ID3D12CommandAllocator`
- 独立的 `ID3D12GraphicsCommandList`

录制完成后，所有命令列表按需要的顺序提交到同一个命令队列。

```
线程 A：                    线程 B：
┌─────────────┐            ┌─────────────┐
│ 分配器 A    │            │ 分配器 B    │
│ 列表 A      │            │ 列表 B      │
└──────┬──────┘            └──────┬──────┘
       │                          │
       └──────────┬───────────────┘
                  ▼
           ExecuteCommandLists({A, B})
                  │
                  ▼
               GPU 执行
```

> 命令列表在 `ExecuteCommandLists` 数组中的顺序就是 GPU 的执行顺序。

---

## 4.3 初始化 Direct3D

Direct3D 的初始化过程虽然较长，但只需要执行一次。以下是完整的初始化步骤：

### 步骤 1：创建设备（Device）

```cpp
#if defined(DEBUG) || defined(_DEBUG)
{
    ComPtr<ID3D12Debug> debugController;
    ThrowIfFailed(D3D12GetDebugInterface(IID_PPV_ARGS(&debugController)));
    debugController->EnableDebugLayer();  // 启用调试层
}
#endif

// 创建 DXGI 工厂
ThrowIfFailed(CreateDXGIFactory1(IID_PPV_ARGS(&mdxgiFactory)));

#### `CreateDXGIFactory1` 详解

```cpp
HRESULT CreateDXGIFactory1(
    REFIID riid,       // [in] 接口 ID，通常为 __uuidof(IDXGIFactory4)
    void** ppFactory   // [out] 输出工厂接口指针
);
```

- `CreateDXGIFactory1` 创建 DXGI 1.1 工厂接口，支持枚举适配器、创建交换链等核心功能。
- 在 D3D12 中通常请求 `IDXGIFactory4` 接口（通过 `IID_PPV_ARGS` 自动推导），因为它支持额外的功能如 `EnumWarpAdapter`。
- 如果需要在 UWP（通用 Windows 平台）应用中使用，应改用 `CreateDXGIFactory2` 并传入 `DXGI_CREATE_FACTORY_DEBUG` 标志以启用 DXGI 调试。
- **必须在创建设备之前创建工厂**，因为 `D3D12CreateDevice` 的回退逻辑和交换链创建都需要用到它。

// 尝试创建硬件设备
HRESULT hardwareResult = D3D12CreateDevice(
    nullptr,                        // 使用主适配器
    D3D_FEATURE_LEVEL_11_0,         // 最低要求特性等级
    IID_PPV_ARGS(&md3dDevice));     // 输出设备指针

// 如果硬件设备创建失败，回退到 WARP（软件光栅化）
if (FAILED(hardwareResult))
{
    ComPtr<IDXGIAdapter> pWarpAdapter;
    ThrowIfFailed(mdxgiFactory->EnumWarpAdapter(IID_PPV_ARGS(&pWarpAdapter)));
    ThrowIfFailed(D3D12CreateDevice(
        pWarpAdapter.Get(),
        D3D_FEATURE_LEVEL_11_0,
        IID_PPV_ARGS(&md3dDevice)));
}
```

#### `D3D12CreateDevice` 详解

```cpp
HRESULT WINAPI D3D12CreateDevice(
    IUnknown* pAdapter,              // [in]  指定显示适配器，nullptr = 使用主适配器
    D3D_FEATURE_LEVEL MinimumFeatureLevel,  // [in]  应用所需的最低特性等级
    REFIID riid,                     // [in]  接口 ID，通常是 __uuidof(ID3D12Device)
    void** ppDevice                  // [out] 输出创建的 ID3D12Device 接口指针
);
```

| 参数 | 说明 |
|------|------|
| `pAdapter` | 指向 `IDXGIAdapter` 的指针，指定要为哪个显示适配器创建设备。传入 `nullptr` 表示使用系统主适配器（通常是性能最强的独立显卡）。如果需要指定特定显卡（如多 GPU 系统），可传入 `EnumAdapters` 枚举到的适配器指针。 |
| `MinimumFeatureLevel` | 应用所需的最低 D3D 特性等级。常见值：`D3D_FEATURE_LEVEL_11_0`、`D3D_FEATURE_LEVEL_12_0`、`D3D_FEATURE_LEVEL_12_1`。如果适配器不支持此等级，函数返回失败。 |
| `riid` | 要创建的 COM 接口 ID。使用 `IID_PPV_ARGS(&md3dDevice)` 宏可自动推导。 |
| `ppDevice` | 输出参数，函数成功时返回 `ID3D12Device` 接口指针。这是后续几乎所有 D3D12 对象的"工厂"——资源、命令列表、描述符堆等都通过它来创建。 |

**返回值**：`S_OK` 表示成功；`E_FAIL`、`DXGI_ERROR_UNSUPPORTED` 等表示失败（如 GPU 不支持所需特性等级）。

**调试层详解**：

```cpp
ComPtr<ID3D12Debug> debugController;
D3D12GetDebugInterface(IID_PPV_ARGS(&debugController));
debugController->EnableDebugLayer();
```

- `D3D12GetDebugInterface` 获取调试层接口。如果系统未安装 Graphics Tools（Windows 可选功能），此调用会失败。
- `EnableDebugLayer()` 激活后，D3D12 会在输出窗口报告以下类型的错误：
  - 命令列表未关闭就提交
  - 资源状态转换不匹配（如从 `RENDER_TARGET` 直接转到 `COPY_SOURCE` 而跳过中间状态）
  - 描述符堆类型与绑定目标不匹配
  - 绘制时缺少必需的描述符绑定
  - 内存越界访问

**WARP 设备**：当硬件设备创建失败时（如虚拟机中没有 GPU、驱动损坏），回退到 WARP（Windows Advanced Rasterization Platform）。WARP 是纯 CPU 实现的软件光栅化器，支持完整的 D3D11/D3D12 特性集，但性能远低于硬件 GPU，仅用于开发和测试。

```cpp
// 获取 WARP 适配器并创建设备
ComPtr<IDXGIAdapter> warpAdapter;
mdxgiFactory->EnumWarpAdapter(IID_PPV_ARGS(&warpAdapter));
D3D12CreateDevice(warpAdapter.Get(), D3D_FEATURE_LEVEL_11_0, IID_PPV_ARGS(&md3dDevice));
```

### 步骤 2：创建围栏并查询描述符大小

```cpp
// 创建围栏，用于 CPU/GPU 同步
ThrowIfFailed(md3dDevice->CreateFence(
    0, D3D12_FENCE_FLAG_NONE, IID_PPV_ARGS(&mFence)));

// 查询各种描述符的增量大小（不同 GPU 可能不同）
mRtvDescriptorSize = md3dDevice->GetDescriptorHandleIncrementSize(
    D3D12_DESCRIPTOR_HEAP_TYPE_RTV);
mDsvDescriptorSize = md3dDevice->GetDescriptorHandleIncrementSize(
    D3D12_DESCRIPTOR_HEAP_TYPE_DSV);
mCbvSrvUavDescriptorSize = md3dDevice->GetDescriptorHandleIncrementSize(
    D3D12_DESCRIPTOR_HEAP_TYPE_CBV_SRV_UAV);
```

#### `GetDescriptorHandleIncrementSize` 详解

```cpp
UINT ID3D12Device::GetDescriptorHandleIncrementSize(
    D3D12_DESCRIPTOR_HEAP_TYPE DescriptorHeapType   // [in] 描述符堆类型
);
```

- **返回值**：指定类型描述符在内存中的字节大小。
- **为什么需要查询**：不同 GPU 架构的描述符大小可能不同。例如，某 GPU 的 RTV 描述符可能是 32 字节，另一 GPU 可能是 64 字节。硬编码这个值会导致程序在不同硬件上崩溃。
- **典型值**：在大多数桌面 GPU 上，RTV/DSV 描述符大小为 32 字节，CBV/SRV/UAV 为 32 字节，Sampler 为 32 字节。但这些值**不能假设**，必须通过 API 查询。

**使用场景**：

```cpp
// 获取堆起始句柄
CD3DX12_CPU_DESCRIPTOR_HANDLE handle(
    mRtvHeap->GetCPUDescriptorHandleForHeapStart());

// 偏移到第 i 个描述符
handle.Offset(i, mRtvDescriptorSize);
// 等价于：handle.ptr += i * mRtvDescriptorSize;
```

### 步骤 3：检查 4× MSAA 质量支持

```cpp
D3D12_FEATURE_DATA_MULTISAMPLE_QUALITY_LEVELS msQualityLevels;
msQualityLevels.Format = mBackBufferFormat;  // DXGI_FORMAT_R8G8B8A8_UNORM
msQualityLevels.SampleCount = 4;
msQualityLevels.Flags = D3D12_MULTISAMPLE_QUALITY_LEVELS_FLAG_NONE;
msQualityLevels.NumQualityLevels = 0;

ThrowIfFailed(md3dDevice->CheckFeatureSupport(
    D3D12_FEATURE_MULTISAMPLE_QUALITY_LEVELS,
    &msQualityLevels, sizeof(msQualityLevels)));

m4xMsaaQuality = msQualityLevels.NumQualityLevels;
assert(m4xMsaaQuality > 0);  // D3D11 硬件一定支持
```

### 步骤 4：创建命令队列、命令分配器和命令列表

```cpp
D3D12_COMMAND_QUEUE_DESC queueDesc = {};
queueDesc.Type = D3D12_COMMAND_LIST_TYPE_DIRECT;
queueDesc.Flags = D3D12_COMMAND_QUEUE_FLAG_NONE;

ThrowIfFailed(md3dDevice->CreateCommandQueue(
    &queueDesc, IID_PPV_ARGS(&mCommandQueue)));

ThrowIfFailed(md3dDevice->CreateCommandAllocator(
    D3D12_COMMAND_LIST_TYPE_DIRECT,
    IID_PPV_ARGS(mDirectCmdListAlloc.GetAddressOf())));

ThrowIfFailed(md3dDevice->CreateCommandList(
    0,
    D3D12_COMMAND_LIST_TYPE_DIRECT,
    mDirectCmdListAlloc.Get(),
    nullptr,  // 初始 PSO，本章不需要
    IID_PPV_ARGS(mCommandList.GetAddressOf())));

mCommandList->Close();  // 初始状态设为关闭
```

#### `CreateCommandAllocator` 详解

```cpp
HRESULT ID3D12Device::CreateCommandAllocator(
    D3D12_COMMAND_LIST_TYPE type,   // [in] 分配器类型
    REFIID riid,                    // [in] 接口 ID
    void** ppCommandAllocator       // [out] 输出 ID3D12CommandAllocator
);
```

- **作用**：命令分配器是命令列表的"内存池"。向命令列表添加命令时，命令数据实际存储在关联的分配器中。
- **重要约束**：
  - 一个分配器不能同时被多个**打开（未关闭）**的命令列表使用。
  - 所有关联的命令列表必须处于关闭状态，除了当前正在录制的那个。
  - 调用 `Reset()` 前必须确保 GPU 已完成该分配器中所有命令的执行（通过 Fence 确认）。

#### `CreateCommandList` 详解

```cpp
HRESULT ID3D12Device::CreateCommandList(
    UINT nodeMask,                          // [in] 多 GPU 节点掩码（单 GPU 为 0）
    D3D12_COMMAND_LIST_TYPE type,           // [in] 命令列表类型
    ID3D12CommandAllocator* pCommandAllocator, // [in] 关联的命令分配器
    ID3D12PipelineState* pInitialState,     // [in] 初始管线状态对象（PSO）
    REFIID riid,                            // [in] 接口 ID
    void** ppCommandList                    // [out] 输出 ID3D12GraphicsCommandList
);
```

| 参数 | 说明 |
|------|------|
| `nodeMask` | 单 GPU 系统设为 0。多 GPU 时指定命令列表关联的物理 GPU 节点。 |
| `type` | 必须与关联的分配器类型一致：`DIRECT`（图形/通用）、`BUNDLE`（命令包，预录制优化）、`COMPUTE`、`COPY`。 |
| `pCommandAllocator` | 命令列表录制命令时使用的内存分配器。一个分配器可先后关联多个命令列表（但不能同时关联多个打开的列表）。 |
| `pInitialState` | 命令列表的初始 PSO。如果命令列表不包含绘制命令（如仅用于初始化或资源屏障），可设为 `nullptr`。对于包含绘制/计算的列表，首次 `Reset()` 时必须指定有效的 PSO。 |
| `ppCommandList` | 输出 `ID3D12GraphicsCommandList` 接口指针。 |

> **初始状态**：`CreateCommandList` 创建的列表默认处于**打开（Open）**状态。由于本书框架第一次使用时会调用 `Reset()`，而 `Reset()` 要求列表处于关闭状态，所以创建后立刻调用 `Close()`。

#### 命令列表生命周期方法详解

| 方法 | 调用时机 | 作用 |
|------|---------|------|
| `Reset(ID3D12CommandAllocator*, ID3D12PipelineState*)` | 重新录制命令前 | 将命令列表重置为初始状态，关联到指定的分配器和 PSO。分配器中的旧命令仍保留，直到 `Allocator::Reset()` 清空。 |
| `Close()` | 录制完成后 | 关闭命令列表，标记录制完成。关闭后才能提交给 `ExecuteCommandLists`。 |
| `ID3D12CommandAllocator::Reset()` | GPU 执行完该分配器中所有命令后 | 清空分配器中的命令数据（类似 `vector::clear`，保留容量）。必须在 GPU 完成后调用。 |

### 步骤 5：描述并创建交换链

```cpp
// 释放旧的交换链（如果有）
mSwapChain.Reset();

DXGI_SWAP_CHAIN_DESC sd;
sd.BufferDesc.Width = mClientWidth;       // 缓冲区宽度
sd.BufferDesc.Height = mClientHeight;     // 缓冲区高度
sd.BufferDesc.RefreshRate.Numerator = 60; // 刷新率
sd.BufferDesc.RefreshRate.Denominator = 1;
sd.BufferDesc.Format = mBackBufferFormat; // 颜色格式
sd.BufferDesc.ScanlineOrdering = DXGI_MODE_SCANLINE_ORDER_UNSPECIFIED;
sd.BufferDesc.Scaling = DXGI_MODE_SCALING_UNSPECIFIED;
sd.SampleDesc.Count = m4xMsaaState ? 4 : 1;
sd.SampleDesc.Quality = m4xMsaaState ? (m4xMsaaQuality - 1) : 0;
sd.BufferUsage = DXGI_USAGE_RENDER_TARGET_OUTPUT; // 用作渲染目标
sd.BufferCount = SwapChainBufferCount;    // 通常 2（双缓冲）
sd.OutputWindow = mhMainWnd;              // 窗口句柄
sd.Windowed = true;                       // 窗口模式
sd.SwapEffect = DXGI_SWAP_EFFECT_FLIP_DISCARD;  // D3D12 推荐
sd.Flags = DXGI_SWAP_CHAIN_FLAG_ALLOW_MODE_SWITCH;

// 通过 DXGI 工厂创建交换链
ThrowIfFailed(mdxgiFactory->CreateSwapChain(
    mCommandQueue.Get(),  // 交换链与命令队列关联
    &sd,
    mSwapChain.GetAddressOf()));
```

#### `DXGI_SWAP_CHAIN_DESC` 详解

```cpp
typedef struct DXGI_SWAP_CHAIN_DESC {
    DXGI_MODE_DESC BufferDesc;   // 缓冲区显示模式描述
    DXGI_SAMPLE_DESC SampleDesc; // 多重采样设置
    DXGI_USAGE BufferUsage;      // 缓冲区用途
    UINT BufferCount;            // 后台缓冲区数量
    HWND OutputWindow;           // 输出窗口句柄
    BOOL Windowed;               // 窗口模式还是全屏
    DXGI_SWAP_EFFECT SwapEffect; // 交换效果
    UINT Flags;                  // 附加标志
} DXGI_SWAP_CHAIN_DESC;
```

**`BufferDesc`（`DXGI_MODE_DESC`）**：

| 字段 | 说明 |
|------|------|
| `Width` / `Height` | 缓冲区分辨率。通常设为窗口客户区大小。全屏时可设为 0，让 DXGI 自动选择最佳模式。 |
| `RefreshRate` | 刷新率，用 `Numerator/Denominator` 分数表示（如 60/1 = 60Hz）。窗口模式下设为 0/0 表示不限制。 |
| `Format` | 缓冲区像素格式，如 `DXGI_FORMAT_R8G8B8A8_UNORM`。 |
| `ScanlineOrdering` | 扫描线顺序。`UNSPECIFIED`（默认）让驱动决定；`PROGRESSIVE` 逐行扫描；`UPPER_FIELD_FIRST`/`LOWER_FIELD_FIRST` 隔行扫描。 |
| `Scaling` | 缩放方式。`UNSPECIFIED` 让驱动决定；`CENTERED` 居中不缩放；`STRETCHED` 拉伸填充。 |

**`SampleDesc`**：多重采样描述。`Count=1, Quality=0` 表示关闭 MSAA。`Count=4` 开启 4× MSAA，`Quality` 设为支持的最大质量等级减 1。

**`BufferUsage`**：缓冲区用途标志。`DXGI_USAGE_RENDER_TARGET_OUTPUT` 表示缓冲区将用作渲染目标（绘制到屏幕上）。其他值如 `DXGI_USAGE_SHADER_INPUT` 表示缓冲区将用作着色器输入。

**`BufferCount`**：后台缓冲区数量。
- `1`：单缓冲（不推荐，会闪烁）。
- `2`：双缓冲（最常用，平衡性能和内存）。
- `3`：三缓冲（减少 CPU 等待，但多占一份显存）。

**`SwapEffect`**：交换效果，这是 D3D12 中最重要的设置之一：

| 值 | 说明 |
|----|------|
| `DXGI_SWAP_EFFECT_DISCARD` | 旧式位块传输（BitBlt）模式。每次 Present 后后台缓冲区内容被丢弃。D3D11 默认。 |
| `DXGI_SWAP_EFFECT_SEQUENTIAL` | 位块传输模式，保留后台缓冲区内容。 |
| `DXGI_SWAP_EFFECT_FLIP_SEQUENTIAL` | **翻页模式**，保留内容。比 BitBlt 更高效，支持独占全屏的低延迟呈现。 |
| `DXGI_SWAP_EFFECT_FLIP_DISCARD` | **翻页模式**，丢弃内容（D3D12 推荐）。效率最高，现代 Windows 默认。 |

> **D3D12 必须使用 `FLIP_DISCARD` 或 `FLIP_SEQUENTIAL`**。BitBlt 模式（`DISCARD`/`SEQUENTIAL`）在 D3D12 中已被废弃。翻页模式通过直接交换显存页表项而不是复制像素数据来实现，延迟更低、性能更好。

**`Flags`**：附加标志：
- `DXGI_SWAP_CHAIN_FLAG_ALLOW_MODE_SWITCH`：允许 Alt+Enter 切换全屏/窗口模式。
- `DXGI_SWAP_CHAIN_FLAG_FRAME_LATENCY_WAITABLE_OBJECT`：允许应用控制最大帧延迟（用于减少输入延迟）。
- `DXGI_SWAP_CHAIN_FLAG_ALLOW_TEARING`：允许在 VSync 关闭时撕裂呈现（用于最大化帧率）。

### 步骤 6：创建描述符堆

```cpp
// RTV 描述符堆（用于后台缓冲区的渲染目标视图）
D3D12_DESCRIPTOR_HEAP_DESC rtvHeapDesc;
rtvHeapDesc.NumDescriptors = SwapChainBufferCount;
rtvHeapDesc.Type = D3D12_DESCRIPTOR_HEAP_TYPE_RTV;
rtvHeapDesc.Flags = D3D12_DESCRIPTOR_HEAP_FLAG_NONE;
rtvHeapDesc.NodeMask = 0;
ThrowIfFailed(md3dDevice->CreateDescriptorHeap(
    &rtvHeapDesc, IID_PPV_ARGS(mRtvHeap.GetAddressOf())));

// DSV 描述符堆（用于深度/模板视图）
D3D12_DESCRIPTOR_HEAP_DESC dsvHeapDesc;
dsvHeapDesc.NumDescriptors = 1;
dsvHeapDesc.Type = D3D12_DESCRIPTOR_HEAP_TYPE_DSV;
dsvHeapDesc.Flags = D3D12_DESCRIPTOR_HEAP_FLAG_NONE;
dsvHeapDesc.NodeMask = 0;
ThrowIfFailed(md3dDevice->CreateDescriptorHeap(
    &dsvHeapDesc, IID_PPV_ARGS(mDsvHeap.GetAddressOf())));
```

#### `CreateDescriptorHeap` 与 `D3D12_DESCRIPTOR_HEAP_DESC` 详解

```cpp
HRESULT ID3D12Device::CreateDescriptorHeap(
    const D3D12_DESCRIPTOR_HEAP_DESC* pDescriptorHeapDesc,  // [in] 描述符堆描述
    REFIID riid,                                            // [in] 接口 ID
    void** ppvHeap                                          // [out] 输出 ID3D12DescriptorHeap
);
```

**`D3D12_DESCRIPTOR_HEAP_DESC` 字段**：

| 字段 | 说明 |
|------|------|
| `NumDescriptors` | 堆中描述符的数量。RTV 堆通常等于后台缓冲区数量（2 或 3）；DSV 堆通常只需 1 个；CBV/SRV/UAV 堆根据场景需求分配（可能几十到几千个）。 |
| `Type` | 描述符堆类型。不同类型不能混放在同一个堆中。 |
| `Flags` | `NONE`（默认，仅 CPU 访问）；`SHADER_VISIBLE`（GPU 可见，用于 CBV/SRV/UAV/Sampler 堆，着色器才能访问）。RTV 和 DSV 堆不能设 `SHADER_VISIBLE`。 |
| `NodeMask` | 多 GPU 节点掩码，单 GPU 为 0。 |

**四种描述符堆类型**：

| 类型 | 用途 | 能否 ShaderVisible |
|------|------|-------------------|
| `D3D12_DESCRIPTOR_HEAP_TYPE_CBV_SRV_UAV` | 常量缓冲/着色器资源/无序访问视图 | 可以 |
| `D3D12_DESCRIPTOR_HEAP_TYPE_SAMPLER` | 采样器状态 | 可以 |
| `D3D12_DESCRIPTOR_HEAP_TYPE_RTV` | 渲染目标视图 | 不可以 |
| `D3D12_DESCRIPTOR_HEAP_TYPE_DSV` | 深度/模板视图 | 不可以 |

#### 描述符句柄操作详解

描述符在堆中按固定步长排列。不同 GPU 的步长可能不同，必须通过 `GetDescriptorHandleIncrementSize` 查询。

```cpp
// 查询 RTV 描述符的增量大小
UINT rtvDescriptorSize = md3dDevice->GetDescriptorHandleIncrementSize(
    D3D12_DESCRIPTOR_HEAP_TYPE_RTV);
```

**获取描述符句柄的两种方式**：

```cpp
// 1. 获取堆的起始句柄
D3D12_CPU_DESCRIPTOR_HANDLE baseHandle =
    mRtvHeap->GetCPUDescriptorHandleForHeapStart();

// 2. 计算第 i 个描述符的句柄（两种方式等价）
// 方式 A：使用 CD3DX12 辅助类
CD3DX12_CPU_DESCRIPTOR_HANDLE handleA(baseHandle, i, rtvDescriptorSize);

// 方式 B：手动计算
D3D12_CPU_DESCRIPTOR_HANDLE handleB;
handleB.ptr = baseHandle.ptr + i * rtvDescriptorSize;
```

- `GetCPUDescriptorHandleForHeapStart()`：返回堆中第一个描述符的 CPU 句柄。
- `GetGPUDescriptorHandleForHeapStart()`：返回 GPU 可见堆中第一个描述符的 GPU 句柄（仅 `SHADER_VISIBLE` 堆可用）。
- 句柄本质上是一个指针（`ptr` 成员），指向描述符在内存中的位置。

### 步骤 7：创建渲染目标视图（RTV）

交换链创建的后台缓冲区是 `ID3D12Resource` 对象，需要为它们创建 RTV 才能作为渲染目标使用：

```cpp
// 获取描述符堆的起始句柄
CD3DX12_CPU_DESCRIPTOR_HANDLE rtvHeapHandle(
    mRtvHeap->GetCPUDescriptorHandleForHeapStart());

for (UINT i = 0; i < SwapChainBufferCount; ++i)
{
    // 获取交换链中的第 i 个缓冲区
    ThrowIfFailed(mSwapChain->GetBuffer(
        i, IID_PPV_ARGS(&mSwapChainBuffer[i])));

    // 创建 RTV
    md3dDevice->CreateRenderTargetView(
        mSwapChainBuffer[i].Get(), nullptr, rtvHeapHandle);

    // 偏移到下一个描述符位置
    rtvHeapHandle.Offset(1, mRtvDescriptorSize);
}
```

#### `IDXGISwapChain::GetBuffer` 详解

```cpp
HRESULT IDXGISwapChain::GetBuffer(
    UINT Buffer,       // [in] 缓冲区索引（0 ~ BufferCount-1）
    REFIID riid,       // [in] 接口 ID（通常是 __uuidof(ID3D12Resource)）
    void** ppSurface   // [out] 输出 ID3D12Resource 接口指针
);
```

- `Buffer`：后台缓冲区索引。双缓冲时，索引 0 和 1 分别对应两个后台缓冲区。交换链内部管理哪个索引是当前的后台缓冲区。
- `riid`：请求的 COM 接口 ID。交换链缓冲区总是以 `ID3D12Resource` 接口返回。
- `ppSurface`：输出参数，返回缓冲区的 `ID3D12Resource` 指针。

> **重要**：`GetBuffer` 会递增返回资源的引用计数。使用 `ComPtr` 接收时，引用计数会自动管理。返回的资源已经是 `D3D12_RESOURCE_STATE_COMMON` 状态，在作为渲染目标使用前需要通过 `ResourceBarrier` 转换到 `RENDER_TARGET` 状态。

#### `CreateRenderTargetView` 详解

```cpp
void ID3D12Device::CreateRenderTargetView(
    ID3D12Resource* pResource,                    // [in] 目标资源
    const D3D12_RENDER_TARGET_VIEW_DESC* pDesc,   // [in] RTV 描述（nullptr = 默认）
    D3D12_CPU_DESCRIPTOR_HANDLE DestDescriptor    // [in] 目标描述符位置
);
```

| 参数 | 说明 |
|------|------|
| `pResource` | 要创建 RTV 的资源。必须是带 `D3D12_RESOURCE_FLAG_ALLOW_RENDER_TARGET` 标志创建的资源。交换链的后台缓冲区默认允许作为渲染目标。 |
| `pDesc` | RTV 描述结构。`nullptr` 表示使用资源的默认格式和完整维度。需要指定 Mipmap 层级、数组切片、特定格式转换时提供非空描述。 |
| `DestDescriptor` | 目标 CPU 描述符句柄，指向描述符堆中的空闲槽位。 |

**`D3D12_RENDER_TARGET_VIEW_DESC` 结构**（仅在需要非默认视图时填写）：

```cpp
typedef struct D3D12_RENDER_TARGET_VIEW_DESC {
    DXGI_FORMAT Format;                    // 视图格式（可与资源格式不同）
    D3D12_RTV_DIMENSION ViewDimension;     // 视图维度
    union {
        D3D12_BUFFER_RTV Buffer;           // 缓冲区 RTV
        D3D12_TEX1D_RTV Texture1D;         // 1D 纹理
        D3D12_TEX2D_RTV Texture2D;         // 2D 纹理（常用）
        D3D12_TEX3D_RTV Texture3D;         // 3D 纹理
        D3D12_TEX1D_ARRAY_RTV Texture1DArray;
        D3D12_TEX2D_ARRAY_RTV Texture2DArray;
        D3D12_TEX2DMS_RTV Texture2DMS;     // 多重采样 2D
        D3D12_TEX2DMS_ARRAY_RTV Texture2DMSArray;
    };
} D3D12_RENDER_TARGET_VIEW_DESC;
```

对于普通的 2D 后台缓冲区，传入 `nullptr` 作为 `pDesc` 即可，驱动会自动推断格式和维度。

### 步骤 8：创建深度/模板缓冲区及视图

```cpp
// 描述深度/模板缓冲区资源
D3D12_RESOURCE_DESC depthStencilDesc;
depthStencilDesc.Dimension = D3D12_RESOURCE_DIMENSION_TEXTURE2D;
depthStencilDesc.Alignment = 0;
depthStencilDesc.Width = mClientWidth;
depthStencilDesc.Height = mClientHeight;
depthStencilDesc.DepthOrArraySize = 1;
depthStencilDesc.MipLevels = 1;
depthStencilDesc.Format = mDepthStencilFormat;  // DXGI_FORMAT_D24_UNORM_S8_UINT
depthStencilDesc.SampleDesc.Count = m4xMsaaState ? 4 : 1;
depthStencilDesc.SampleDesc.Quality = m4xMsaaState ? (m4xMsaaQuality - 1) : 0;
depthStencilDesc.Layout = D3D12_TEXTURE_LAYOUT_UNKNOWN;
depthStencilDesc.Flags = D3D12_RESOURCE_FLAG_ALLOW_DEPTH_STENCIL;

// 清除值（初始化时清除为深度 1.0，模板 0）
D3D12_CLEAR_VALUE optClear;
optClear.Format = mDepthStencilFormat;
optClear.DepthStencil.Depth = 1.0f;
optClear.DepthStencil.Stencil = 0;

#### `D3D12_CLEAR_VALUE` 详解

```cpp
typedef struct D3D12_CLEAR_VALUE {
    DXGI_FORMAT Format;         // 清除值的格式，必须与资源格式兼容
    union {
        FLOAT Color[4];         // 用于渲染目标的清除颜色（RGBA）
        D3D12_DEPTH_STENCIL_VALUE DepthStencil;  // 用于深度/模板缓冲的清除值
    };
} D3D12_CLEAR_VALUE;

typedef struct D3D12_DEPTH_STENCIL_VALUE {
    FLOAT Depth;     // 深度清除值，标准范围 [0.0, 1.0]
    UINT8 Stencil;   // 模板清除值，范围 [0, 255]
} D3D12_DEPTH_STENCIL_VALUE;
```

- `optClear` 不是强制参数（可传 `nullptr`），但**强烈建议**为会被频繁清除的资源提供它。
- 驱动可利用这个值优化快速清除操作（Fast Clear），在某些 GPU 架构上能显著提升性能。
- 颜色清除值必须与资源格式匹配。例如 `R8G8B8A8_UNORM` 对应 `{R, G, B, A}` 浮点值，范围 [0.0, 1.0]。
- 深度清除值 `1.0f` 表示"最远"，这是标准做法，因为深度测试默认保留更小（更近）的值。

#### `CD3DX12_HEAP_PROPERTIES` 详解

```cpp
CD3DX12_HEAP_PROPERTIES(D3D12_HEAP_TYPE_DEFAULT)
```

这是 `d3dx12.h` 提供的便捷类，构造一个 `D3D12_HEAP_PROPERTIES` 结构体：

```cpp
typedef struct D3D12_HEAP_PROPERTIES {
    D3D12_HEAP_TYPE Type;                  // 堆类型
    D3D12_CPU_PAGE_PROPERTY CPUPageProperty;  // CPU 页属性
    D3D12_MEMORY_POOL MemoryPoolPreference;   // 内存池偏好
    UINT CreationNodeMask;                 // 创建节点掩码
    UINT VisibleNodeMask;                  // 可见节点掩码
} D3D12_HEAP_PROPERTIES;
```

- `Type` 有三种主要值：`DEFAULT`（GPU 独占）、`UPLOAD`（CPU 上传）、`READBACK`（GPU 回读）。
- `CPUPageProperty` 和 `MemoryPoolPreference` 在 `Type` 为 `DEFAULT` 时通常设为 `UNKNOWN`/`UNKNOWN`，让驱动自动选择最优的内存位置（显存或系统内存）。
- `CD3DX12_HEAP_PROPERTIES` 构造函数自动把这些附加字段设为合理的默认值，只需传入堆类型即可。

// 创建深度/模板缓冲区资源
ThrowIfFailed(md3dDevice->CreateCommittedResource(
    &CD3DX12_HEAP_PROPERTIES(D3D12_HEAP_TYPE_DEFAULT),  // GPU 默认堆
    D3D12_HEAP_FLAG_NONE,
    &depthStencilDesc,
    D3D12_RESOURCE_STATE_COMMON,  // 初始状态
    &optClear,
    IID_PPV_ARGS(mDepthStencilBuffer.GetAddressOf())));

// 创建 DSV
md3dDevice->CreateDepthStencilView(
    mDepthStencilBuffer.Get(), nullptr,
    mDsvHeap->GetCPUDescriptorHandleForHeapStart());
```

#### `CreateCommittedResource` 详解

`CreateCommittedResource` 是 D3D12 中创建 GPU 资源最常用的高级方法。它一步完成三件事：分配堆内存、创建资源对象、提交资源到 GPU。

```cpp
HRESULT ID3D12Device::CreateCommittedResource(
    const D3D12_HEAP_PROPERTIES* pHeapProperties,  // [in] 堆属性
    D3D12_HEAP_FLAGS HeapFlags,                    // [in] 堆标志
    const D3D12_RESOURCE_DESC* pDesc,              // [in] 资源描述
    D3D12_RESOURCE_STATES InitialResourceState,    // [in] 初始资源状态
    const D3D12_CLEAR_VALUE* pOptimizedClearValue, // [in] 优化的清除值（可为 nullptr）
    REFIID riid,                                   // [in] 接口 ID
    void** ppvResource                             // [out] 输出 ID3D12Resource
);
```

| 参数 | 说明 |
|------|------|
| `pHeapProperties` | 堆的属性，包括堆类型（`DEFAULT`、`UPLOAD`、`READBACK`）和 CPU 访问属性。`CD3DX12_HEAP_PROPERTIES` 是便捷封装。 |
| `HeapFlags` | 堆附加标志。常用值见下方表格。 |
| `pDesc` | `D3D12_RESOURCE_DESC` 结构，描述资源的维度、大小、格式、mipmap 层级、采样设置等。 |
| `InitialResourceState` | 资源创建后的初始状态。必须与第一个使用它的命令中声明的"之前状态"一致。深度缓冲通常从 `COMMON` 开始，然后通过 ResourceBarrier 转换到 `DEPTH_WRITE`。 |
| `pOptimizedClearValue` | 如果资源会被频繁清除（如渲染目标、深度缓冲），提供此值可让驱动优化清除操作。对于纹理和静态缓冲可设为 `nullptr`。 |
| `ppvResource` | 输出创建的 `ID3D12Resource` 接口指针。 |

**`D3D12_HEAP_FLAGS` 常用值**：

| 标志 | 说明 |
|------|------|
| `D3D12_HEAP_FLAG_NONE` | 无特殊标志，最常用的默认值。 |
| `D3D12_HEAP_FLAG_SHARED` | 允许堆跨进程或跨设备共享。需要配合同步机制使用。 |
| `D3D12_HEAP_FLAG_DENY_BUFFERS` | 禁止在此堆中创建缓冲区资源（只能创建纹理）。可用于优化驱动的内存分配策略。 |
| `D3D12_HEAP_FLAG_ALLOW_DISPLAY` | 允许堆中的资源用于显示输出（如直接扫描输出）。 |
| `D3D12_HEAP_FLAG_SHARED_CROSS_ADAPTER` | 允许跨适配器共享（多 GPU 之间）。 |
| `D3D12_HEAP_FLAG_HARDWARE_PROTECTED` | 启用硬件保护（用于受 DRM 保护的内容）。 |
| `D3D12_HEAP_FLAG_ALLOW_WRITE_WATCH` | 允许对堆进行写监视（调试用途）。 |

> 绝大多数情况下使用 `NONE` 即可。只有在需要跨进程共享资源或特殊的内存优化时，才考虑其他标志。

**`D3D12_RESOURCE_DESC` 关键字段**：

| 字段 | 说明 |
|------|------|
| `Dimension` | 资源维度：`TEXTURE1D`、`TEXTURE2D`、`TEXTURE3D`、`BUFFER`。 |
| `Width` / `Height` / `DepthOrArraySize` | 资源尺寸。对于 2D 纹理，`DepthOrArraySize` 是数组大小（立方体贴图为 6）。 |
| `MipLevels` | Mipmap 层级数。1 表示无 mipmap；0 表示生成完整的 mipmap 链。 |
| `Format` | 像素格式，如 `DXGI_FORMAT_R8G8B8A8_UNORM`、`DXGI_FORMAT_D24_UNORM_S8_UINT`。 |
| `SampleDesc` | 多重采样描述。关闭 MSAA 时 `Count=1, Quality=0`。 |
| `Layout` | 纹理布局。`UNKNOWN` 让驱动选择最优布局；`ROW_MAJOR` 强制行主序（仅缓冲）。 |
| `Flags` | 资源标志，控制资源允许的用途。见下方表格。 |

**`D3D12_RESOURCE_FLAGS` 常用值**：

| 标志 | 说明 |
|------|------|
| `D3D12_RESOURCE_FLAG_NONE` | 无特殊标志。资源只能用于基本的读取操作。 |
| `D3D12_RESOURCE_FLAG_ALLOW_RENDER_TARGET` | 允许资源作为渲染目标（RTV）。创建时必须同时提供 `D3D12_CLEAR_VALUE`。 |
| `D3D12_RESOURCE_FLAG_ALLOW_DEPTH_STENCIL` | 允许资源作为深度/模板缓冲区（DSV）。创建时必须同时提供 `D3D12_CLEAR_VALUE`。 |
| `D3D12_RESOURCE_FLAG_ALLOW_UNORDERED_ACCESS` | 允许资源作为无序访问视图（UAV），即计算着色器的读写目标。 |
| `D3D12_RESOURCE_FLAG_DENY_SHADER_RESOURCE` | 禁止资源作为着色器资源（SRV）。可用于优化，防止意外读取。 |
| `D3D12_RESOURCE_FLAG_ALLOW_CROSS_ADAPTER` | 允许资源跨适配器使用（多 GPU 场景）。 |
| `D3D12_RESOURCE_FLAG_ALLOW_SIMULTANEOUS_ACCESS` | 允许资源被多个命令队列同时访问。 |

> 创建资源时，必须根据实际用途设置正确的标志。例如，如果需要用 `CreateRenderTargetView` 为资源创建 RTV，则创建资源时必须带 `ALLOW_RENDER_TARGET` 标志，否则调试层会报错。

**堆类型选择**：

| 堆类型 | 用途 | CPU 访问 | GPU 访问 |
|--------|------|---------|---------|
| `DEFAULT` | 最常用于 GPU 资源（纹理、顶点缓冲、深度缓冲等） | 不可直接访问 | 读写 |
| `UPLOAD` | 用于 CPU 向 GPU 上传数据的暂存缓冲 | 可写（通过 `Map`） | 只读 |
| `READBACK` | 用于 GPU 向 CPU 回传数据的缓冲 | 可读（通过 `Map`） | 只写 |

#### `CreateDepthStencilView` 详解

```cpp
void ID3D12Device::CreateDepthStencilView(
    ID3D12Resource* pResource,              // [in] 深度/模板资源
    const D3D12_DEPTH_STENCIL_VIEW_DESC* pDesc, // [in] DSV 描述（nullptr = 默认）
    D3D12_CPU_DESCRIPTOR_HANDLE DestDescriptor  // [in] 目标描述符句柄
);
```

- `pResource`：深度/模板缓冲区资源，必须带 `D3D12_RESOURCE_FLAG_ALLOW_DEPTH_STENCIL` 标志创建。
- `pDesc`：DSV 描述。`nullptr` 表示使用资源的默认格式和维度（最常用的简化写法）。需要指定子资源或特定格式时提供非空描述。
- `DestDescriptor`：目标 CPU 描述符句柄，指向描述符堆中的空闲位置。通过 `GetCPUDescriptorHandleForHeapStart()` 获取。

**`D3D12_DEPTH_STENCIL_VIEW_DESC` 结构**（需要非默认视图时填写）：

```cpp
typedef struct D3D12_DEPTH_STENCIL_VIEW_DESC {
    DXGI_FORMAT Format;                     // 视图格式
    D3D12_DSV_DIMENSION ViewDimension;      // 视图维度
    D3D12_DSV_FLAGS Flags;                  // DSV 标志
    union {
        D3D12_TEX1D_DSV Texture1D;
        D3D12_TEX1D_ARRAY_DSV Texture1DArray;
        D3D12_TEX2D_DSV Texture2D;          // 最常用的 2D DSV
        D3D12_TEX2D_ARRAY_DSV Texture2DArray;
        D3D12_TEX2DMS_DSV Texture2DMS;      // 多重采样 2D
        D3D12_TEX2DMS_ARRAY_DSV Texture2DMSArray;
    };
} D3D12_DEPTH_STENCIL_VIEW_DESC;
```

| 字段 | 说明 |
|------|------|
| `Format` | DSV 格式。必须与资源格式兼容，常见值：`DXGI_FORMAT_D24_UNORM_S8_UINT`、`DXGI_FORMAT_D32_FLOAT`、`DXGI_FORMAT_D16_UNORM`。 |
| `ViewDimension` | `D3D12_DSV_DIMENSION_TEXTURE2D`（普通 2D）、`TEXTURE2DMS`（多重采样）等。 |
| `Flags` | `D3D12_DSV_FLAG_NONE`（允许读写深度和模板）；`READ_ONLY_DEPTH`（只读深度，用于深度预 pass 后的只读深度测试）；`READ_ONLY_STENCIL`（只读模板）。 |

### 步骤 9：设置视口（Viewport）

视口定义了渲染输出在屏幕上的矩形区域：

```cpp
D3D12_VIEWPORT mScreenViewport;
mScreenViewport.TopLeftX = 0;
mScreenViewport.TopLeftY = 0;
mScreenViewport.Width = static_cast<float>(mClientWidth);
mScreenViewport.Height = static_cast<float>(mClientHeight);
mScreenViewport.MinDepth = 0.0f;
mScreenViewport.MaxDepth = 1.0f;

mCommandList->RSSetViewports(1, &mScreenViewport);
```

#### `D3D12_VIEWPORT` 与 `RSSetViewports` 详解

```cpp
typedef struct D3D12_VIEWPORT {
    FLOAT TopLeftX;    // 视口左上角 X 坐标（屏幕空间像素坐标）
    FLOAT TopLeftY;    // 视口左上角 Y 坐标
    FLOAT Width;       // 视口宽度（像素）
    FLOAT Height;      // 视口高度（像素）
    FLOAT MinDepth;    // 深度值映射的最小值（默认 0.0）
    FLOAT MaxDepth;    // 深度值映射的最大值（默认 1.0）
} D3D12_VIEWPORT;
```

**`MinDepth` / `MaxDepth` 的作用**：
- 光栅化后，顶点的 Z 值（在 [0, 1] 范围内）会被线性映射到 `[MinDepth, MaxDepth]`。
- 默认 `0.0 ~ 1.0` 表示正常使用完整深度范围。
- 例如设为 `0.0 ~ 0.5` 时，所有深度值会被压缩到前半范围，可用于特殊效果（如分屏渲染时每个视口使用不同深度段）。

```cpp
void ID3D12GraphicsCommandList::RSSetViewports(
    UINT NumViewports,                    // [in] 视口数量（D3D12 最多 16 个）
    const D3D12_VIEWPORT* pViewports      // [in] 视口数组指针
);
```

- 多个视口可用于**分屏渲染**、**立体渲染**（VR 左右眼）或**渲染到纹理数组的不同切片**。
- 如果 `NumViewports` 为 0，表示不绑定任何视口（绘制将被禁用）。
- 视口可以与后备缓冲区大小不同：大于后备缓冲区时，超出部分被裁剪；小于后备缓冲区时，只渲染到指定区域。

#### `RSSetScissorRects` 详解

```cpp
void ID3D12GraphicsCommandList::RSSetScissorRects(
    UINT NumRects,                        // [in] 裁剪矩形数量（最多 16 个）
    const D3D12_RECT* pRects              // [in] 裁剪矩形数组
);
```

```cpp
typedef struct D3D12_RECT {
    LONG left;      // 左边界（包含）
    LONG top;       // 上边界（包含）
    LONG right;     // 右边界（不包含）
    LONG bottom;    // 下边界（不包含）
} D3D12_RECT;
```

**裁剪矩形 vs 视口的区别**：

| 特性 | 视口（Viewport） | 裁剪矩形（Scissor Rect） |
|------|-----------------|------------------------|
| 作用阶段 | 光栅化阶段，决定顶点坐标到屏幕空间的映射 | 光栅化后的像素裁剪 |
| 对顶点着色器的影响 | 影响（顶点坐标按视口变换） | 不影响（所有顶点都执行） |
| 对像素着色器的影响 | 间接影响（视口外的像素不会生成） | 直接裁剪（scissor 外的像素被丢弃，PS 不执行） |
| 用途 | 定义渲染区域的位置和深度范围 | 精确控制哪些像素允许写入 |

- 裁剪矩形使用**半开区间** `[left, right)` / `[top, bottom)`，即左/上边界包含在内，右/下边界不包含。
- 如果 scissor rect 小于 viewport，viewport 内但 scissor 外的像素会被丢弃（像素着色器不会执行），可用于 UI 遮罩、分屏游戏的边界裁剪等。
- 默认状态必须至少设置一个 scissor rect，否则光栅化不会输出任何像素。

### 步骤 10：设置裁剪矩形（Scissor Rectangle）

裁剪矩形限制了光栅化阶段实际写入像素的区域：

```cpp
D3D12_RECT mScissorRect;
mScissorRect = { 0, 0, mClientWidth, mClientHeight };

mCommandList->RSSetScissorRects(1, &mScissorRect);
```

> 如果 scissor rect 小于 viewport，超出 scissor rect 的像素会被丢弃（但顶点着色器仍然执行）。

---

## 4.4 计时与动画

实时图形应用需要精确的时间测量来计算动画、物理模拟和帧率统计。

### 4.4.1 性能计数器

Windows 提供 `QueryPerformanceCounter` 获取高精度计时：

```cpp
// 获取计数器频率（每秒 tick 数）
__int64 cntsPerSec = 0;
QueryPerformanceFrequency((LARGE_INTEGER*)&cntsPerSec);
float secondsPerCount = 1.0f / (float)cntsPerSec;

// 获取当前计数器值
__int64 currTime = 0;
QueryPerformanceCounter((LARGE_INTEGER*)&currTime);
```

### 4.4.2 GameTimer 类

封装后的计时器类提供以下功能：

| 方法 | 返回值 | 用途 |
|------|--------|------|
| `GameTime()` | 总运行时间（秒） | 驱动与时间相关的动画 |
| `DeltaTime()` | 上一帧的间隔时间（秒） | 控制动画/物理更新速度 |
| `Reset()` | 无 | 在消息循环开始前调用，重置起始时间 |
| `Tick()` | 无 | 每帧调用，更新计时器状态 |

**Tick() 的典型用法**：

```cpp
// 在消息循环中：
while (msg.message != WM_QUIT)
{
    if (PeekMessage(&msg, 0, 0, 0, PM_REMOVE))
    {
        TranslateMessage(&msg);
        DispatchMessage(&msg);
    }
    else
    {
        mTimer.Tick();  // 更新计时

        if (!mAppPaused)
        {
            CalculateFrameStats();  // 计算 FPS
            Update(mTimer);         // 更新场景（传入计时器）
            Draw(mTimer);           // 绘制场景
        }
        else
        {
            Sleep(100);  // 暂停时降低 CPU 占用
        }
    }
}
```

### 4.4.3 帧间时间

`DeltaTime()` 是动画系统的关键。例如，要让一个物体以 10 单位/秒的速度移动：

```cpp
void Update(const GameTimer& gt)
{
    float dt = gt.DeltaTime();
    mObjectPosition += mVelocity * dt;  // velocity = 10 units/sec
}
```

这样无论帧率高低（30 FPS 还是 144 FPS），物体的移动速度都是一致的。

### 4.4.4 总时间

总时间用于周期性动画。例如，让一个物体上下浮动：

```cpp
float totalTime = gt.TotalTime();
mObjectHeight = baseHeight + amplitude * sin(totalTime * frequency);
```

---

## 4.5 演示应用程序框架

本书所有示例都基于 `D3DApp` 框架类，它统一处理了窗口创建、Direct3D 初始化、消息循环和帧统计。

### 4.5.1 D3DApp 类结构

```cpp
class D3DApp
{
public:
    D3DApp(HINSTANCE hInstance);   // 构造函数
    virtual ~D3DApp();

    int Run();                     // 消息循环入口

    // 可被子类重写的虚函数
    virtual bool Initialize();
    virtual void Update(const GameTimer& gt) = 0;
    virtual void Draw(const GameTimer& gt) = 0;
    virtual void OnResize();
    virtual void OnMouseDown(WPARAM btnState, int x, int y) { }
    virtual void OnMouseUp(WPARAM btnState, int x, int y) { }
    virtual void OnMouseMove(WPARAM btnState, int x, int y) { }

protected:
    // D3D 核心对象
    ComPtr<ID3D12Device> md3dDevice;
    ComPtr<IDXGISwapChain> mSwapChain;
    ComPtr<ID3D12CommandQueue> mCommandQueue;
    ComPtr<ID3D12CommandAllocator> mDirectCmdListAlloc;
    ComPtr<ID3D12GraphicsCommandList> mCommandList;
    ComPtr<ID3D12Fence> mFence;

    // 描述符堆
    ComPtr<ID3D12DescriptorHeap> mRtvHeap;
    ComPtr<ID3D12DescriptorHeap> mDsvHeap;

    // 资源
    ComPtr<ID3D12Resource> mSwapChainBuffer[SwapChainBufferCount];
    ComPtr<ID3D12Resource> mDepthStencilBuffer;

    // 视图句柄
    D3D12_CPU_DESCRIPTOR_HANDLE mBackBufferView;
    D3D12_CPU_DESCRIPTOR_HANDLE mDepthStencilView;

    // 视口和裁剪矩形
    D3D12_VIEWPORT mScreenViewport;
    D3D12_RECT mScissorRect;

    // 计时器
    GameTimer mTimer;

    // 窗口相关
    HWND mhMainWnd = nullptr;
    HINSTANCE mhAppInst = nullptr;
    bool mAppPaused = false;

    // 配置
    UINT mClientWidth = 1280;
    UINT mClientHeight = 720;
    bool m4xMsaaState = false;
    UINT m4xMsaaQuality = 0;

    // 常量
    static const int SwapChainBufferCount = 2;
    DXGI_FORMAT mBackBufferFormat = DXGI_FORMAT_R8G8B8A8_UNORM;
    DXGI_FORMAT mDepthStencilFormat = DXGI_FORMAT_D24_UNORM_S8_UINT;
};
```

### 4.5.2 非框架方法

这些方法在 `D3DApp` 中已实现，通常不需要子类重写：

| 方法 | 作用 |
|------|------|
| `Run()` | 主消息循环 |
| `Initialize()` | 初始化窗口和 Direct3D |
| `OnResize()` | 窗口大小改变时重新分配缓冲区 |
| `CalculateFrameStats()` | 计算并显示 FPS 和每帧毫秒数 |
| `FlushCommandQueue()` | 等待 GPU 完成所有命令 |

### 4.5.3 框架方法（子类必须/可以重写）

| 方法 | 必须重写？ | 作用 |
|------|-----------|------|
| `Update(const GameTimer&)` | 是 | 每帧更新场景逻辑（动画、输入处理等） |
| `Draw(const GameTimer&)` | 是 | 每帧录制并提交渲染命令 |
| `OnMouseDown/Up/Move` | 否 | 鼠标事件回调 |
| `OnResize` | 否 | 窗口大小变化时重新创建缓冲区（默认实现已足够） |

### 4.5.4 帧统计

`CalculateFrameStats()` 计算并输出 FPS：

```cpp
void D3DApp::CalculateFrameStats()
{
    static int frameCnt = 0;
    static float timeElapsed = 0.0f;

    frameCnt++;

    // 每秒统计一次
    if ((mTimer.TotalTime() - timeElapsed) >= 1.0f)
    {
        float fps = (float)frameCnt;  // 每秒帧数
        float mspf = 1000.0f / fps;    // 每帧毫秒数

        // 格式化标题栏
        std::wstring fpsStr = std::to_wstring(fps);
        std::wstring mspfStr = std::to_wstring(mspf);
        std::wstring windowText = mMainWndCaption +
            L"    fps: " + fpsStr +
            L"   mspf: " + mspfStr;

        SetWindowText(mhMainWnd, windowText.c_str());

        // 重置计数器
        frameCnt = 0;
        timeElapsed += 1.0f;
    }
}
```

### 4.5.5 消息处理器

`D3DApp::MsgProc` 处理常见的 Windows 消息：

| 消息 | 处理 |
|------|------|
| `WM_ACTIVATE` | 窗口激活/失活时暂停/恢复 |
| `WM_SIZE` | 窗口大小变化时调用 `OnResize()` |
| `WM_ENTERSIZEMOVE` | 开始拖动窗口时暂停计时 |
| `WM_EXITSIZEMOVE` | 结束拖动窗口时恢复计时并调用 `OnResize()` |
| `WM_DESTROY` | 窗口销毁时发送 `PostQuitMessage` |
| `WM_MENUCHAR` | 防止按 Alt 键时发出蜂鸣声 |
| `WM_GETMINMAXINFO` | 限制窗口最小尺寸 |
| `WM_LBUTTONDOWN/MOUSEMOVE/LBUTTONUP` | 转发给 `OnMouseDown/Move/Up` |

### 4.5.6 "Init Direct3D" 演示程序

本章的示例程序只是初始化 Direct3D，清屏为蓝色，然后呈现。没有绘制任何几何体。这是学习 D3D12 的"Hello World"。

**Draw() 的基本流程**：

```cpp
void InitDirect3DApp::Draw(const GameTimer& gt)
{
    // 1. 重置命令分配器和命令列表
    ThrowIfFailed(mDirectCmdListAlloc->Reset());
    ThrowIfFailed(mCommandList->Reset(mDirectCmdListAlloc.Get(), nullptr));

    // 2. 把后台缓冲区从 PRESENT 状态转为 RENDER_TARGET 状态
    mCommandList->ResourceBarrier(1,
        &CD3DX12_RESOURCE_BARRIER::Transition(
            CurrentBackBuffer(),
            D3D12_RESOURCE_STATE_PRESENT,
            D3D12_RESOURCE_STATE_RENDER_TARGET));

    // 3. 设置视口和裁剪矩形
    mCommandList->RSSetViewports(1, &mScreenViewport);
    mCommandList->RSSetScissorRects(1, &mScissorRect);

    // 4. 清除后台缓冲区和深度缓冲区
    mCommandList->ClearRenderTargetView(
        CurrentBackBufferView(), Colors::LightSteelBlue, 0, nullptr);
    mCommandList->ClearDepthStencilView(
        DepthStencilView(),
        D3D12_CLEAR_FLAG_DEPTH | D3D12_CLEAR_FLAG_STENCIL,
        1.0f, 0, 0, nullptr);

    // 5. 指定要渲染到的缓冲区和深度缓冲区
    mCommandList->OMSetRenderTargets(1, &CurrentBackBufferView(),
        true, &DepthStencilView());

    // 6. 把后台缓冲区从 RENDER_TARGET 状态转回 PRESENT 状态
    mCommandList->ResourceBarrier(1,
        &CD3DX12_RESOURCE_BARRIER::Transition(
            CurrentBackBuffer(),
            D3D12_RESOURCE_STATE_RENDER_TARGET,
            D3D12_RESOURCE_STATE_PRESENT));

    // 7. 关闭命令列表并提交到命令队列
    ThrowIfFailed(mCommandList->Close());
    ID3D12CommandList* cmdsLists[] = { mCommandList.Get() };
    mCommandQueue->ExecuteCommandLists(_countof(cmdsLists), cmdsLists);

    // 8. 呈现（交换前后缓冲区）
    ThrowIfFailed(mSwapChain->Present(0, 0));
    mCurrBackBuffer = (mCurrBackBuffer + 1) % SwapChainBufferCount;

    // 9. 等待 GPU 完成（简单但低效的做法）
    FlushCommandQueue();
}
```

#### `ClearRenderTargetView` 详解

```cpp
void ID3D12GraphicsCommandList::ClearRenderTargetView(
    D3D12_CPU_DESCRIPTOR_HANDLE RenderTargetView,  // [in] 要清除的 RTV
    const FLOAT ColorRGBA[4],                      // [in] 清除颜色 {R, G, B, A}
    UINT NumRects,                                 // [in] 矩形区域数量（0 = 全部）
    const D3D12_RECT* pRects                       // [in] 矩形区域数组（nullptr = 全部）
);
```

- `RenderTargetView`：要清除的渲染目标视图的 CPU 描述符句柄。
- `ColorRGBA`：清除后的统一颜色值，4 个 32 位浮点数。例如 `{0.0f, 0.2f, 0.4f, 1.0f}` 是深蓝色。
- `NumRects` / `pRects`：如果只想清除部分区域，可指定一个或多个矩形。设为 `0, nullptr` 表示清除整个渲染目标。

#### `ClearDepthStencilView` 详解

```cpp
void ID3D12GraphicsCommandList::ClearDepthStencilView(
    D3D12_CPU_DESCRIPTOR_HANDLE DepthStencilView,  // [in] 要清除的 DSV
    D3D12_CLEAR_FLAGS ClearFlags,                  // [in] 清除标志
    FLOAT Depth,                                   // [in] 深度清除值
    UINT8 Stencil,                                 // [in] 模板清除值
    UINT NumRects,                                 // [in] 矩形区域数量
    const D3D12_RECT* pRects                       // [in] 矩形区域数组
);
```

- `ClearFlags`：指定清除深度、模板或两者。
  - `D3D12_CLEAR_FLAG_DEPTH`：只清除深度
  - `D3D12_CLEAR_FLAG_STENCIL`：只清除模板
  - `D3D12_CLEAR_FLAG_DEPTH | D3D12_CLEAR_FLAG_STENCIL`：两者都清除
- `Depth`：深度清除值。标准深度范围是 [0, 1]，通常清除为 1.0（最远），这样任何新绘制的像素都更近。
- `Stencil`：模板清除值。通常设为 0。

#### `OMSetRenderTargets` 详解

```cpp
void ID3D12GraphicsCommandList::OMSetRenderTargets(
    UINT NumRenderTargetDescriptors,               // [in] RTV 数量
    const D3D12_CPU_DESCRIPTOR_HANDLE* pRenderTargetDescriptors, // [in] RTV 数组
    BOOL RTsSingleHandleToDescriptorRange,         // [in] 描述符是否连续
    const D3D12_CPU_DESCRIPTOR_HANDLE* pDepthStencilDescriptor   // [in] DSV（可为 nullptr）
);
```

| 参数 | 说明 |
|------|------|
| `NumRenderTargetDescriptors` | 同时绑定的渲染目标数量。D3D12 支持最多 8 个 MRT（Multiple Render Targets）。 |
| `pRenderTargetDescriptors` | RTV 描述符句柄数组。如果只有一个 RTV，传入其地址即可。 |
| `RTsSingleHandleToDescriptorRange` | `TRUE` 表示 `pRenderTargetDescriptors` 指向一个连续描述符范围的起始位置（第 0 个 RTV 的句柄），后续 RTV 在描述符堆中连续排列。`FALSE` 表示 `pRenderTargetDescriptors` 是一个指针数组，每个元素指向各自独立的描述符。对于单 RTV 情况，两种写法效果相同。 |
| `pDepthStencilDescriptor` | DSV 描述符句柄。如果不需要深度/模板测试，可设为 `nullptr`。 |

> **作用**：设置输出合并（Output Merger）阶段的目标。后续的绘制命令会把像素颜色和深度写入这里绑定的缓冲。

#### `ExecuteCommandLists` 详解

```cpp
void ID3D12CommandQueue::ExecuteCommandLists(
    UINT NumCommandLists,                          // [in] 命令列表数量
    ID3D12CommandList* const* ppCommandLists       // [in] 命令列表指针数组
);
```

- 命令列表按**数组顺序**在 GPU 上执行。
- 所有命令列表必须处于**关闭（Closed）**状态。
- 可以一次提交多个命令列表，例如多线程录制的结果：

```cpp
ID3D12CommandList* lists[] = { cmdListA.Get(), cmdListB.Get(), cmdListC.Get() };
mCommandQueue->ExecuteCommandLists(3, lists);
// GPU 保证按 A → B → C 的顺序执行
```

#### `Present` 详解

```cpp
HRESULT IDXGISwapChain::Present(
    UINT SyncInterval,     // [in] VSync 间隔
    UINT Flags             // [in] 呈现标志
);
```

| 参数 | 说明 |
|------|------|
| `SyncInterval` | 垂直同步间隔。`0` = 立即呈现（不等待 VSync，可能撕裂）；`1` = 等待 1 个 VSync（60Hz 显示器上限 60 FPS）；`2` = 等待 2 个 VSync（上限 30 FPS）。 |
| `Flags` | `0` = 无特殊标志；`DXGI_PRESENT_ALLOW_TEARING` = 允许撕裂（需交换链创建时带 `ALLOW_TEARING` 标志，且 `SyncInterval = 0`）。 |

**VSync 与撕裂**：
- 关闭 VSync（`SyncInterval = 0`）：GPU 完成一帧后立即呈现，最大化帧率。但可能在屏幕刷新中途交换缓冲区，导致**画面撕裂**（上半屏显示旧帧，下半屏显示新帧）。
- 开启 VSync（`SyncInterval = 1`）：Present 会等待显示器下一次垂直 blanking 期才交换，消除撕裂。代价是如果 GPU 渲染时间超过 16.67ms（60Hz），帧率会直接掉到 30 FPS（等待两个 VSync 周期）。

> **注意**：本章使用 `FlushCommandQueue()` 每帧都等待 GPU，这在实际应用中非常低效。第 7 章会介绍使用多组帧资源来避免这个问题。

---

## 4.6 调试 Direct3D 应用程序

### 启用调试层

调试构建时务必启用调试层：

```cpp
#if defined(DEBUG) || defined(_DEBUG)
{
    ComPtr<ID3D12Debug> debugController;
    D3D12GetDebugInterface(IID_PPV_ARGS(&debugController));
    debugController->EnableDebugLayer();
}
#endif
```

启用后，D3D12 会在 Visual Studio 输出窗口报告各种错误，例如：
- `D3D12 ERROR: ID3D12CommandList::Reset: Reset fails because the command list was not closed.`
- 资源状态转换不匹配
- 描述符使用错误

### 常用调试工具

| 工具 | 用途 |
|------|------|
| **Visual Studio Graphics Debugger** | 截取单帧，逐命令检查资源、像素历史 |
| **PIX** | 微软官方 GPU 性能分析工具，功能最全面 |
| **RenderDoc** | 开源帧捕获和分析工具，跨平台 |
| **GPU Validation** | D3D12 调试层的一个模式，检测更多 GPU 侧错误 |

### 启用 GPU 验证（Debug Layer 增强）

```cpp
ComPtr<ID3D12Debug1> debugController1;
debugController.As(&debugController1);
debugController1->SetEnableGPUBasedValidation(true);
```

这会启用基于 GPU 的验证，捕获更多错误（但会显著降低性能，仅限调试）。

---

## 本章小结

| 主题 | 核心要点 |
|------|---------|
| **D3D12 定位** | 底层 API，暴露更多 GPU 细节，换取更低 CPU 开销和更强多线程 |
| **COM 对象** | 引用计数，用 `ComPtr` 管理，接口名以 `I` 开头 |
| **资源与描述符** | 资源不直接绑定，通过描述符间接引用；描述符堆按类型分离 |
| **交换链** | 双缓冲/三缓冲 + 翻页，避免画面撕裂 |
| **深度缓冲** | 存储每个像素的深度值，解决遮挡关系 |
| **MSAA** | 4× 多重采样平衡了画质和性能 |
| **命令队列/列表/分配器** | CPU 录制命令 → 提交到队列 → GPU 异步执行 |
| **围栏同步** | `Signal()` + `GetCompletedValue()` + 事件等待 |
| **资源过渡** | 显式 `ResourceBarrier()` 声明状态转换，防止资源冲突 |
| **初始化十步** | 设备 → 围栏 → MSAA → 命令对象 → 交换链 → 描述符堆 → RTV → DSV → 视口 → 裁剪矩形 |
| **计时器** | `QueryPerformanceCounter` + `GameTimer`：总时间和帧间隔 |
| **D3DApp 框架** | `Update()` / `Draw()` 虚函数 + 消息处理 + 帧统计 |

---

## 课后练习思路

1. **修改清屏颜色**：在 `Draw()` 中改变 `ClearRenderTargetView` 的颜色参数，观察效果。
2. **切换 MSAA**：把 `m4xMsaaState` 设为 `true`，确认 4× MSAA 在硬件上正常工作。
3. **改变视口**：创建一个小于窗口大小的视口，观察渲染区域的变化。
4. **多适配器枚举**：实现适配器枚举代码，在调试输出中打印系统所有 GPU 的名称。
5. **测量 Flush 开销**：注释掉 `FlushCommandQueue()`，观察 FPS 变化（注意：这会引入撕裂，仅用于理解同步开销）。

---

> **下一步**：掌握初始化后，第 5 章将深入讲解**渲染管线**的各个阶段——从 3D 顶点数据如何一步步变成屏幕上的像素。
