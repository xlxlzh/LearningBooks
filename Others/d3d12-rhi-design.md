# 跨平台多 API 引擎的 RHI 设计指南

本文面向需要同时支持 D3D12、Vulkan、Metal、D3D11 等多个图形 API 的游戏引擎，系统性地介绍 RHI（Render Hardware Interface）的架构设计方法，重点覆盖资源抽象、命令系统抽象、三种 Command List 的分层设计，以及 D3D11 这种立即模式 API 的兼容策略。

---

## 目录

1. [RHI 的整体分层](#1-rhi-的整体分层)
2. [抽象粒度与核心原则](#2-抽象粒度与核心原则)
3. [核心对象抽象](#3-核心对象抽象)
4. [Command List 的分层设计](#4-command-list-的分层设计)
5. [绑定模型抽象](#5-绑定模型抽象)
6. [资源管理与内存抽象](#6-资源管理与内存抽象)
7. [Shader 与 Pipeline 抽象](#7-shader-与-pipeline-抽象)
8. [多线程设计](#8-多线程设计)
9. [D3D11 兼容设计](#9-d3d11-兼容设计)
10. [Capabilities 设计](#10-capabilities-设计)
11. [最小化 RHI 接口示例](#11-最小化-rhi-接口示例)
12. [最佳实践与常见误区](#12-最佳实践与常见误区)

---

## 1. RHI 的整体分层

```
┌─────────────────────────────────────┐
│  上层 Renderer / Render Graph       │  上层只认识 RHI 类型
├─────────────────────────────────────┤
│  RHI 抽象层（平台无关接口）          │  IRHICommandList, IRHITexture...
├─────────────────────────────────────┤
│  RHI 平台层                          │  D3D12 / Vulkan / Metal / D3D11
├─────────────────────────────────────┤
│  底层 Graphics API                   │  D3D12, Vulkan, Metal, D3D11...
└─────────────────────────────────────┘
```

核心设计原则：

- **上层不直接包含任何平台头文件**。
- **RHI 接口以现代 GPU API（D3D12/Vulkan/Metal）为基准设计**。
- **允许平台特化**：通过 Capabilities 和可选扩展表达差异。

---

## 2. 抽象粒度与核心原则

| 粒度 | 优点 | 缺点 | 适合 |
|------|------|------|------|
| 细粒度 | 性能高、可控 | 抽象复杂 | 自研引擎、UE5 |
| 粗粒度 | 上层简单、易移植 | 高级特性难表达 | 小型引擎、工具 |
| 中等粒度 | 平衡 | 需要精心设计 | 大多数商业引擎 |

**推荐：中等粒度，命令列表式**。既保留现代 GPU 的显式控制，又不过度暴露底层细节。

---

## 3. 核心对象抽象

### 3.1 Device

```cpp
class IRHIDevice
{
public:
    virtual ~IRHIDevice() = default;

    virtual IRHICommandQueue* CreateCommandQueue(RHICommandQueueType type) = 0;
    virtual IRHICommandAllocator* CreateCommandAllocator(RHICommandListType type) = 0;

    virtual IRHIDirectCommandList* CreateDirectCommandList(IRHICommandAllocator* allocator) = 0;
    virtual IRHIComputeCommandList* CreateComputeCommandList(IRHICommandAllocator* allocator) = 0;
    virtual IRHICopyCommandList* CreateCopyCommandList(IRHICommandAllocator* allocator) = 0;

    virtual IRHIFence* CreateFence(uint64_t initialValue) = 0;
    virtual IRHIDescriptorHeap* CreateDescriptorHeap(const RHIDescriptorHeapDesc& desc) = 0;

    virtual IRHIBuffer* CreateBuffer(const RHIBufferDesc& desc, RHIResourceState initialState) = 0;
    virtual IRHITexture* CreateTexture(const RHITextureDesc& desc, RHIResourceState initialState) = 0;

    virtual IRHIGraphicsPipelineState* CreateGraphicsPipeline(const RHIGraphicsPipelineDesc& desc) = 0;
    virtual IRHIComputePipelineState* CreateComputePipeline(const RHIComputePipelineDesc& desc) = 0;

    virtual IRHIShader* CreateShader(const RHIShaderBytecode& bytecode) = 0;

    virtual const RHICapabilities& GetCapabilities() const = 0;
};
```

### 3.2 Command Queue

```cpp
enum class RHICommandQueueType
{
    Graphics,   // Direct
    Compute,    // Async Compute
    Copy,       // Transfer
};

class IRHICommandQueue
{
public:
    virtual ~IRHICommandQueue() = default;

    virtual void ExecuteCommandLists(Span<IRHICommandList*> lists) = 0;
    virtual void Signal(IRHIFence* fence, uint64_t value) = 0;
    virtual void Wait(IRHIFence* fence, uint64_t value) = 0;
};
```

### 3.3 Command Allocator

```cpp
class IRHICommandAllocator
{
public:
    virtual ~IRHICommandAllocator() = default;
    virtual void Reset() = 0;
};
```

---

## 4. Command List 的分层设计

### 4.1 为什么要分开实现？

Direct、Compute、Copy 三种 Command List 的命令集合、状态机和后端对象类型都有明显差异：

| 能力 | Direct List | Compute List | Copy List |
|------|-------------|--------------|-----------|
| Draw 命令 | ✅ | ❌ | ❌ |
| Dispatch | ✅ | ✅ | ❌ |
| Copy 命令 | ✅ | ✅ | ✅ |
| RenderPass | ✅ | ❌ | ❌ |
| Viewport/Scissor | ✅ | ❌ | ❌ |
| Graphics PSO | ✅ | ❌ | ❌ |
| Compute PSO | ✅ | ✅ | ❌ |
| 顶点/索引缓冲 | ✅ | ❌ | ❌ |

如果把三种 List 合并成一个类，每个 Draw/RenderPass 接口在 Compute 和 Copy 中只能空实现或抛异常，接口语义会变得非常混乱。

### 4.2 推荐接口分层

```cpp
// 公共接口
class IRHICommandList
{
public:
    virtual ~IRHICommandList() = default;

    virtual void Reset(IRHICommandAllocator* allocator, IRHIPipelineState* initialPso = nullptr) = 0;
    virtual void Close() = 0;
    virtual RHICommandListType GetType() const = 0;

    virtual void ResourceBarrier(Span<RHIResourceBarrier> barriers) = 0;
    virtual void CopyBuffer(IRHIBuffer* dst, uint64_t dstOffset, IRHIBuffer* src, uint64_t srcOffset, uint64_t size) = 0;
    virtual void CopyTexture(const RHITextureCopyLocation& dst, const RHITextureCopyLocation& src) = 0;

    virtual void BeginEvent(const char* name) = 0;
    virtual void EndEvent() = 0;
    virtual void SetMarker(const char* name) = 0;
};

// Direct List：Graphics + Compute + Copy
class IRHIDirectCommandList : public IRHICommandList
{
public:
    virtual ~IRHIDirectCommandList() = default;

    virtual void SetViewport(const RHIViewport& viewport) = 0;
    virtual void SetScissor(const RHIScissorRect& scissor) = 0;

    virtual void BeginRenderPass(const RHIRenderPassInfo& info) = 0;
    virtual void EndRenderPass() = 0;

    virtual void SetGraphicsPipelineState(IRHIGraphicsPipelineState* pso) = 0;
    virtual void SetGraphicsRootSignature(IRHIRootSignature* rootSig) = 0;
    virtual void SetGraphicsRootDescriptorTable(uint32_t slot, RHIDescriptorTableHandle table) = 0;
    virtual void SetGraphicsRootConstantBufferView(uint32_t slot, RHIGPUAddress address) = 0;
    virtual void SetGraphicsRoot32BitConstants(uint32_t slot, uint32_t count, const void* data, uint32_t offset) = 0;

    virtual void SetVertexBuffer(uint32_t slot, IRHIBuffer* buffer, uint64_t offset) = 0;
    virtual void SetIndexBuffer(IRHIBuffer* buffer, uint64_t offset, RHIIndexFormat format) = 0;
    virtual void SetPrimitiveTopology(RHIPrimitiveTopology topology) = 0;

    virtual void DrawInstanced(uint32_t vertexCount, uint32_t instanceCount, uint32_t startVertex, uint32_t startInstance) = 0;
    virtual void DrawIndexedInstanced(uint32_t indexCount, uint32_t instanceCount, uint32_t startIndex, int32_t baseVertex, uint32_t startInstance) = 0;
    virtual void DrawIndirect(IRHIBuffer* argsBuffer, uint64_t offset) = 0;

    // Direct List 也可以做 Compute
    virtual void SetComputePipelineState(IRHIComputePipelineState* pso) = 0;
    virtual void SetComputeRootSignature(IRHIRootSignature* rootSig) = 0;
    virtual void SetComputeRootDescriptorTable(uint32_t slot, RHIDescriptorTableHandle table) = 0;
    virtual void Dispatch(uint32_t x, uint32_t y, uint32_t z) = 0;
    virtual void DispatchIndirect(IRHIBuffer* argsBuffer, uint64_t offset) = 0;
};

// Compute List
class IRHIComputeCommandList : public IRHICommandList
{
public:
    virtual ~IRHIComputeCommandList() = default;

    virtual void SetComputePipelineState(IRHIComputePipelineState* pso) = 0;
    virtual void SetComputeRootSignature(IRHIRootSignature* rootSig) = 0;
    virtual void SetComputeRootDescriptorTable(uint32_t slot, RHIDescriptorTableHandle table) = 0;
    virtual void SetComputeRootConstantBufferView(uint32_t slot, RHIGPUAddress address) = 0;
    virtual void Dispatch(uint32_t x, uint32_t y, uint32_t z) = 0;
    virtual void DispatchIndirect(IRHIBuffer* argsBuffer, uint64_t offset) = 0;
};

// Copy List
class IRHICopyCommandList : public IRHICommandList
{
public:
    virtual ~IRHICopyCommandList() = default;

    virtual void CopyBufferRegion(IRHIBuffer* dst, uint64_t dstOffset,
                                   IRHIBuffer* src, uint64_t srcOffset,
                                   uint64_t size) = 0;
    virtual void CopyTextureRegion(IRHITexture* dst, IRHITexture* src,
                                    const RHITextureCopyRegion& region) = 0;
    virtual void CopyBufferToTexture(IRHITexture* dst, IRHIBuffer* src,
                                      const RHITextureSubresourceLayers& subresource) = 0;
};
```

### 4.3 后端实现组织

通过基类复用公共逻辑，再派生三个具体类：

```cpp
class FD3D12CommandListBase : public IRHICommandList
{
public:
    FD3D12CommandListBase(ID3D12GraphicsCommandList* cmdList, RHICommandListType type)
        : m_d3d12List(cmdList)
        , m_type(type)
    {}

    void Reset(IRHICommandAllocator* allocator, IRHIPipelineState* initialPso) override;
    void Close() override;
    RHICommandListType GetType() const override { return m_type; }

    void ResourceBarrier(Span<RHIResourceBarrier> barriers) override;
    void CopyBuffer(IRHIBuffer* dst, uint64_t dstOffset, IRHIBuffer* src, uint64_t srcOffset, uint64_t size) override;
    void CopyTexture(const RHITextureCopyLocation& dst, const RHITextureCopyLocation& src) override;

    void BeginEvent(const char* name) override;
    void EndEvent() override;
    void SetMarker(const char* name) override;

protected:
    ComPtr<ID3D12GraphicsCommandList> m_d3d12List;
    RHICommandListType m_type;
};

class FD3D12DirectCommandList : public FD3D12CommandListBase, public IRHIDirectCommandList
{
public:
    void SetViewport(const RHIViewport& viewport) override;
    void SetScissor(const RHIScissorRect& scissor) override;
    void BeginRenderPass(const RHIRenderPassInfo& info) override;
    void EndRenderPass() override;
    void SetGraphicsPipelineState(IRHIGraphicsPipelineState* pso) override;
    void DrawIndexedInstanced(uint32_t indexCount, uint32_t instanceCount, uint32_t startIndex, int32_t baseVertex, uint32_t startInstance) override;
    void Dispatch(uint32_t x, uint32_t y, uint32_t z) override;
    // ...
};

class FD3D12ComputeCommandList : public FD3D12CommandListBase, public IRHIComputeCommandList
{
public:
    void SetComputePipelineState(IRHIComputePipelineState* pso) override;
    void Dispatch(uint32_t x, uint32_t y, uint32_t z) override;
};

class FD3D12CopyCommandList : public FD3D12CommandListBase, public IRHICopyCommandList
{
public:
    void CopyBufferRegion(IRHIBuffer* dst, uint64_t dstOffset, IRHIBuffer* src, uint64_t srcOffset, uint64_t size) override;
    void CopyTextureRegion(IRHITexture* dst, IRHITexture* src, const RHITextureCopyRegion& region) override;
};
```

### 4.4 分开实现 vs 合并实现

| 维度 | 分开实现 | 合并实现 |
|------|---------|---------|
| 类型安全 | 编译期阻止非法调用 | 运行时检查/空实现 |
| 接口清晰度 | 每个类职责单一 | 巨大类，很多方法在某些类型下无效 |
| 后端实现 | 贴合 D3D12/Vulkan 原生模型 | 内部用 if-else 分发 |
| 性能 | 无运行时类型分发 | 轻微运行时判断开销 |
| 扩展性 | 新增类型不影响其他类 | 类会越来越大 |

---

## 5. 绑定模型抽象

推荐以 **Bind Group / Descriptor Set** 为统一抽象：

```cpp
enum class RHIDescriptorType
{
    ConstantBuffer,
    ShaderResource,
    UnorderedAccess,
    Sampler,
};

struct RHIBindGroupLayoutEntry
{
    uint32_t binding;
    RHIDescriptorType type;
    uint32_t count;
    RHIShaderStageFlags stageFlags;
};

class IRHIBindGroupLayout
{
public:
    virtual ~IRHIBindGroupLayout() = default;
};

class IRHIBindGroup
{
public:
    virtual ~IRHIBindGroup() = default;
    virtual void Update(uint32_t binding, IRHIResourceView* view) = 0;
};
```

后端映射：

| RHI 抽象 | D3D12 | Vulkan | Metal |
|----------|-------|--------|-------|
| Bind Group Layout | Root Signature 参数段 | VkDescriptorSetLayout | Argument Encoder |
| Bind Group | Descriptor Table | VkDescriptorSet | Argument Buffer |

---

## 6. 资源管理与内存抽象

### 6.1 资源接口

```cpp
class IRHIResource
{
public:
    virtual ~IRHIResource() = default;
    virtual RHIResourceType GetType() const = 0;
    virtual const RHIResourceDesc& GetDesc() const = 0;
    virtual void* Map(uint32_t subresource, const RHIRange* range) = 0;
    virtual void Unmap(uint32_t subresource, const RHIRange* range) = 0;
};

class IRHIBuffer : public IRHIResource {};
class IRHITexture : public IRHIResource {};
```

### 6.2 内存用途分级

```cpp
enum class RHIMemoryUsage
{
    GpuOnly,      // DEFAULT
    Upload,       // UPLOAD
    Readback,     // READBACK
};
```

由后端根据 `MemoryUsage` 决定实际堆类型，而不是强制上层理解 D3D12 Heap。

### 6.3 资源状态

现代后端需要显式屏障，D3D11 后端可忽略或模拟：

```cpp
struct RHIResourceBarrier
{
    IRHIResource* resource;
    RHIResourceState stateBefore;
    RHIResourceState stateAfter;
    uint32_t subresource;
};
```

---

## 7. Shader 与 Pipeline 抽象

### 7.1 Shader

```cpp
enum class RHIShaderLanguage
{
    DXIL,
    SPIRV,
    MSL,
    DXBC,    // D3D11
    GLSL,
};

struct RHIShaderBytecode
{
    RHIShaderLanguage language;
    const void* data;
    size_t size;
};

class IRHIShader
{
public:
    virtual ~IRHIShader() = default;
    virtual RHIShaderStage GetStage() const = 0;
};
```

推荐统一使用 **Slang** 或 **DXC** 编译：

- Slang → DXIL / SPIR-V / MSL / GLSL
- DXC → DXIL / SPIR-V

### 7.2 Pipeline State

```cpp
struct RHIGraphicsPipelineDesc
{
    IRHIRootSignature* rootSignature = nullptr;
    IRHIShader* vertexShader = nullptr;
    IRHIShader* pixelShader = nullptr;
    IRHIShader* domainShader = nullptr;
    IRHIShader* hullShader = nullptr;
    IRHIShader* geometryShader = nullptr;

    RHIBlendState blendState;
    RHIRasterizerState rasterizerState;
    RHIDepthStencilState depthStencilState;

    RHIVertexInputLayout inputLayout;
    RHIPrimitiveTopology primitiveTopology;

    RHIRenderTargetFormats renderTargetFormats;
    RHIDepthStencilFormat depthStencilFormat;
};

class IRHIGraphicsPipelineState
{
public:
    virtual ~IRHIGraphicsPipelineState() = default;
};

class IRHIComputePipelineState
{
public:
    virtual ~IRHIComputePipelineState() = default;
};
```

---

## 8. 多线程设计

### 8.1 推荐模型

- 每个线程独立录制 Command List。
- 主线程负责收集并 `ExecuteCommandLists`。
- RHI 提供线程安全的 Command List 池。

```cpp
class IRHICommandListPool
{
public:
    virtual IRHICommandList* Allocate(RHICommandListType type) = 0;
    virtual void Free(IRHICommandList* list) = 0;
    virtual void Reset() = 0;
};
```

### 8.2 注意 API 差异

| API | 线程隔离对象 |
|-----|-------------|
| D3D12 | Command Allocator |
| Vulkan | Command Pool |
| Metal | Command Buffer / Encoder |
| D3D11 | Immediate Context（单线程） |

RHI 应隐藏这些差异。

---

## 9. D3D11 兼容设计

### 9.1 D3D11 与现代 API 的核心差异

| 特性 | D3D11 | D3D12/Vulkan/Metal |
|------|-------|---------------------|
| 执行模型 | 立即模式 | 录制模式 |
| Command List | Deferred Context 受限 | 核心特性 |
| Command Queue | 无显式 Queue | 必须 |
| Fence | `ID3D11Fence`（仅 11.3）或 Query | 原生 |
| 资源状态 | 驱动隐式管理 | 应用显式管理 |
| 多线程 | 有限 | 完整 |

### 9.2 推荐策略

**RHI 以现代 API 为基准，D3D11 作为兼容降级层**：

- D3D11 后端不保证多线程录制性能。
- `ResourceBarrier` 在 D3D11 后端为空操作或简单校验。
- Queue 在 D3D11 后端是虚拟的，Execute 直接映射到 Immediate Context。
- Fence 用 `ID3D11Fence` 或 Query 模拟。

### 9.3 D3D11 Command List 后端示例

```cpp
class FD3D11CommandListBase : public IRHICommandList
{
public:
    FD3D11CommandListBase(ID3D11DeviceContext* context)
        : m_immediateContext(context)
    {}

    void Reset(IRHICommandAllocator* allocator, IRHIPipelineState* initialPso) override
    {
        // D3D11 是立即模式，无需真正 Reset
    }

    void Close() override
    {
        // D3D11 没有 Close 概念
    }

    void ResourceBarrier(Span<RHIResourceBarrier> barriers) override
    {
        // D3D11 驱动自动管理资源状态
    }

    void CopyBuffer(IRHIBuffer* dst, uint64_t dstOffset,
                    IRHIBuffer* src, uint64_t srcOffset,
                    uint64_t size) override
    {
        auto* dstBuf = static_cast<FD3D11Buffer*>(dst)->GetHandle();
        auto* srcBuf = static_cast<FD3D11Buffer*>(src)->GetHandle();

        D3D11_BOX box = {};
        box.left = static_cast<uint32_t>(srcOffset);
        box.right = static_cast<uint32_t>(srcOffset + size);
        box.top = 0; box.bottom = 1;
        box.front = 0; box.back = 1;

        m_immediateContext->CopySubresourceRegion(
            dstBuf, 0, static_cast<uint32_t>(dstOffset), 0, 0,
            srcBuf, 0, &box);
    }

protected:
    ID3D11DeviceContext* m_immediateContext = nullptr;
};

class FD3D11DirectCommandList : public FD3D11CommandListBase, public IRHIDirectCommandList
{
public:
    using FD3D11CommandListBase::FD3D11CommandListBase;

    void SetViewport(const RHIViewport& viewport) override
    {
        D3D11_VIEWPORT vp = Convert(viewport);
        m_immediateContext->RSSetViewports(1, &vp);
    }

    void SetScissor(const RHIScissorRect& scissor) override
    {
        D3D11_RECT rect = Convert(scissor);
        m_immediateContext->RSSetScissorRects(1, &rect);
    }

    void SetGraphicsPipelineState(IRHIGraphicsPipelineState* pso) override
    {
        static_cast<FD3D11GraphicsPipelineState*>(pso)->Apply(m_immediateContext);
    }

    void DrawIndexedInstanced(uint32_t indexCount, uint32_t instanceCount,
                              uint32_t startIndex, int32_t baseVertex,
                              uint32_t startInstance) override
    {
        m_immediateContext->DrawIndexedInstanced(
            indexCount, instanceCount, startIndex, baseVertex, startInstance);
    }

    void Dispatch(uint32_t x, uint32_t y, uint32_t z) override
    {
        m_immediateContext->Dispatch(x, y, z);
    }

    // ...
};
```

### 9.4 D3D11 Fence 实现

```cpp
class FD3D11Fence : public IRHIFence
{
public:
    void Initialize(ID3D11Device* device, bool useD3D11_3)
    {
        if (useD3D11_3)
        {
            device->CreateFence(0, D3D11_FENCE_FLAG_NONE, IID_PPV_ARGS(&m_fence));
        }
    }

    void Signal(ID3D11DeviceContext* context, uint64_t value)
    {
        if (m_fence)
        {
            context->Signal(m_fence.Get(), value);
        }
        else
        {
            // 老版本 D3D11：用 Query
            ComPtr<ID3D11Query> query;
            D3D11_QUERY_DESC desc = { D3D11_QUERY_EVENT, 0 };
            // 创建并 End query...
            m_queries[value] = query;
        }
    }

    void Wait(uint64_t value)
    {
        if (m_fence)
        {
            HANDLE event = CreateEvent(nullptr, FALSE, FALSE, nullptr);
            m_fence->SetEventOnCompletion(value, event);
            WaitForSingleObject(event, INFINITE);
            CloseHandle(event);
        }
        else
        {
            // 轮询 Query
        }
    }

private:
    ComPtr<ID3D11Fence> m_fence;
    std::map<uint64_t, ComPtr<ID3D11Query>> m_queries;
};
```

---

## 10. Capabilities 设计

```cpp
struct RHICapabilities
{
    bool multiThreadCommandRecording = true;
    bool multiThreadResourceCreation = true;

    bool asyncCompute = true;
    bool asyncCopy = true;

    bool explicitResourceBarriers = true;
    bool placedResources = true;
    bool reservedResources = false;

    bool bindlessDescriptors = false;
    bool descriptorIndexing = false;

    bool rayTracing = false;
    bool meshShaders = false;
    bool variableRateShading = false;
    bool conservativeRasterization = false;

    uint32_t maxTextureSize = 16384;
    uint32_t maxComputeGroupSize = 1024;
    RHIShaderModel maxShaderModel = RHIShaderModel::SM6_0;
};
```

D3D11 后端的典型值：

```cpp
RHICapabilities caps;
caps.multiThreadCommandRecording = false;
caps.asyncCompute = false;
caps.explicitResourceBarriers = false;
caps.placedResources = false;
caps.bindlessDescriptors = false;
caps.rayTracing = false;
caps.meshShaders = false;
```

---

## 11. 最小化 RHI 接口示例

```cpp
class IRHIFactory
{
public:
    static IRHIDevice* CreateDevice(RHIApiType api, const RHIDeviceDesc& desc);
};

enum class RHIApiType
{
    D3D12,
    Vulkan,
    Metal,
    D3D11,
    WebGPU,
};

class IRHIDevice
{
public:
    virtual ~IRHIDevice() = default;

    // Queue
    virtual IRHICommandQueue* CreateCommandQueue(RHICommandQueueType type) = 0;

    // Allocator
    virtual IRHICommandAllocator* CreateCommandAllocator(RHICommandListType type) = 0;

    // Command List（按类型分开）
    virtual IRHIDirectCommandList* CreateDirectCommandList(IRHICommandAllocator* allocator) = 0;
    virtual IRHIComputeCommandList* CreateComputeCommandList(IRHICommandAllocator* allocator) = 0;
    virtual IRHICopyCommandList* CreateCopyCommandList(IRHICommandAllocator* allocator) = 0;

    // Synchronization
    virtual IRHIFence* CreateFence(uint64_t initialValue) = 0;

    // Resources
    virtual IRHIBuffer* CreateBuffer(const RHIBufferDesc& desc, RHIResourceState initialState) = 0;
    virtual IRHITexture* CreateTexture(const RHITextureDesc& desc, RHIResourceState initialState) = 0;

    // Pipeline
    virtual IRHIGraphicsPipelineState* CreateGraphicsPipeline(const RHIGraphicsPipelineDesc& desc) = 0;
    virtual IRHIComputePipelineState* CreateComputePipeline(const RHIComputePipelineDesc& desc) = 0;

    // Shader
    virtual IRHIShader* CreateShader(const RHIShaderBytecode& bytecode) = 0;

    // Binding
    virtual IRHIBindGroupLayout* CreateBindGroupLayout(const std::vector<RHIBindGroupLayoutEntry>& entries) = 0;
    virtual IRHIBindGroup* CreateBindGroup(IRHIBindGroupLayout* layout) = 0;

    // Capabilities
    virtual const RHICapabilities& GetCapabilities() const = 0;
};
```

---

## 12. 最佳实践与常见误区

### 12.1 推荐做法

1. **RHI 以现代 API 为基准**，不被 D3D11 拖累抽象水平。
2. **三种 Command List 接口分开**，Direct List 同时包含 Compute 能力。
3. **用 Capabilities 表达平台差异**，上层根据能力选择代码路径。
4. **在 RHI 之上构建 Render Graph**，不要把渲染调度逻辑塞进 RHI。
5. **Shader 统一用 Slang/DXC 编译**，输出各后端所需格式。
6. **资源管理提供分级接口**，不强制暴露 D3D12 Heap 细节。
7. **多线程录制时隔离 Allocator/Pool**，隐藏 API 差异。
8. **D3D11 作为降级后端**，明确标记不支持的能力。

### 12.2 常见误区

| 误区 | 问题 | 建议 |
|------|------|------|
| 以 OpenGL/D3D11 为基准设计 RHI | 现代 API 特性难以表达 | 以 D3D12/Vulkan/Metal 为基准 |
| 三种 Command List 合并实现 | 接口污染、类型不安全 | 接口分开，实现用基类复用 |
| 过度抽象所有 API 差异 | 性能损失、特性受限 | Capabilities + 可选扩展 |
| 在 RHI 层做渲染逻辑 | RHI 膨胀、难以维护 | RHI 只做 GPU 抽象 |
| 每个 API 单独写 Shader | 维护成本高 | Slang/DXC 统一编译 |
| 忽视资源状态管理 | 跨 API 行为不一致 | 强制上层或 Render Graph 管理状态 |
| D3D11 后端强行多线程 | Deferred Context 功能受限 | 标记为不支持多线程录制 |

---

## 结语

跨平台 RHI 的设计没有银弹，关键在于**找到合适的抽象层**。以现代 API 为基准，让 Direct/Compute/Copy 三种 Command List 在接口层面清晰分离，同时通过基类复用公共实现；对于 D3D11 这样的 legacy API，以兼容降级层的思路接入，用 Capabilities 明确能力边界。这样可以在保持上层代码统一的同时，让每个后端都有实现自己最佳路径的空间。
