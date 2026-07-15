# D3D12 资源管理完全指南

Direct3D 12（D3D12）将大量原本由驱动程序隐式管理的职责上移到应用程序层，其中**资源管理**是最核心、最复杂也最容易影响性能的部分。本文从最基本的 `ID3D12Resource` 讲起，逐步深入到描述符管理、内存分配策略、对象池、环形缓冲区等高级应用，并配以可直接运行的代码片段。

---

## 目录

1. [基础概念](#1-基础概念)
2. [ID3D12Resource 与 D3D12_HEAP_PROPERTIES](#2-id3d12resource-与-d3d12_heap_properties)
3. [资源状态与屏障](#3-资源状态与屏障)
4. [上传与读回](#4-上传与读回)
5. [描述符与描述符堆](#5-描述符与描述符堆)
6. [内存分配策略](#6-内存分配策略)
7. [高级应用：对象池与内存池](#7-高级应用对象池与内存池)
8. [完整示例：一个最小化的资源管理器](#8-完整示例一个最小化的资源管理器)
9. [最佳实践与常见陷阱](#9-最佳实践与常见陷阱)

---

## 1. 基础概念

### 1.1 资源（Resource）

在 D3D12 中，几乎所有 GPU 可读写的显式对象都称为**资源**，包括：

- **缓冲区（Buffer）**：顶点缓冲、索引缓冲、常量缓冲、结构化缓冲、UAV 缓冲等。
- **纹理（Texture）**：1D/2D/3D 纹理、Depth/Stencil、Render Target 等。

资源由 `ID3D12Resource` 接口表示，创建后本身不直接记录“当前状态”——状态跟踪由应用程序负责。

### 1.2 堆（Heap）

D3D12 允许将资源创建在独立的 GPU 内存堆中，或者由运行时隐式分配。堆由 `ID3D12Heap` 表示，类型包括：

| 堆类型 | 说明 |
|--------|------|
| `D3D12_HEAP_TYPE_DEFAULT` | GPU 本地内存（显存），CPU 不可直接访问，读写需通过上传/读回缓冲。 |
| `D3D12_HEAP_TYPE_UPLOAD` | CPU 可写、GPU 可读的共享内存，通常用于上传数据。 |
| `D3D12_HEAP_TYPE_READBACK` | GPU 可写、CPU 可读的共享内存，用于读回 GPU 结果。 |
| `D3D12_HEAP_TYPE_CUSTOM` | 根据适配器属性自定义内存池和 CPU 缓存属性。 |

### 1.3 放置资源 vs 提交资源

- **提交资源（Committed Resource）**：`CreateCommittedResource` 一次性分配内存并创建资源，最简单，适合一次性或长期存在的资源。
- **放置资源（Placed Resource）**：`CreatePlacedResource` 在已有的 `ID3D12Heap` 中指定偏移创建资源，可以复用内存、控制别名（aliasing），适合资源池。
- **保留资源（Reserved Resource）**：`CreateReservedResource` 创建逻辑资源，物理页通过 `UpdateTileMappings` 动态映射，用于稀疏纹理/虚拟纹理。

---

## 2. ID3D12Resource 与 D3D12_HEAP_PROPERTIES

### 2.1 提交资源的创建

下面创建一个简单的 64KB 上传缓冲区，用于存储常量数据：

```cpp
#include <d3d12.h>
#include <wrl/client.h>
#include <cstdint>

using Microsoft::WRL::ComPtr;

ComPtr<ID3D12Resource> CreateUploadBuffer(ID3D12Device* device, uint64_t size)
{
    D3D12_HEAP_PROPERTIES heapProps = {};
    heapProps.Type = D3D12_HEAP_TYPE_UPLOAD;

    D3D12_RESOURCE_DESC desc = {};
    desc.Dimension = D3D12_RESOURCE_DIMENSION_BUFFER;
    desc.Width = size;
    desc.Height = 1;
    desc.DepthOrArraySize = 1;
    desc.MipLevels = 1;
    desc.Format = DXGI_FORMAT_UNKNOWN;
    desc.SampleDesc.Count = 1;
    desc.Layout = D3D12_TEXTURE_LAYOUT_ROW_MAJOR;
    desc.Flags = D3D12_RESOURCE_FLAG_NONE;

    ComPtr<ID3D12Resource> resource;
    HRESULT hr = device->CreateCommittedResource(
        &heapProps,
        D3D12_HEAP_FLAG_NONE,
        &desc,
        D3D12_RESOURCE_STATE_GENERIC_READ,
        nullptr,
        IID_PPV_ARGS(&resource)
    );

    if (FAILED(hr))
        return nullptr;

    return resource;
}
```

### 2.2 创建默认堆纹理

```cpp
ComPtr<ID3D12Resource> CreateDefaultTexture2D(
    ID3D12Device* device,
    uint32_t width,
    uint32_t height,
    DXGI_FORMAT format,
    D3D12_RESOURCE_FLAGS flags = D3D12_RESOURCE_FLAG_NONE)
{
    D3D12_HEAP_PROPERTIES heapProps = {};
    heapProps.Type = D3D12_HEAP_TYPE_DEFAULT;

    D3D12_RESOURCE_DESC desc = {};
    desc.Dimension = D3D12_RESOURCE_DIMENSION_TEXTURE2D;
    desc.Width = width;
    desc.Height = height;
    desc.DepthOrArraySize = 1;
    desc.MipLevels = 1;
    desc.Format = format;
    desc.SampleDesc.Count = 1;
    desc.Layout = D3D12_TEXTURE_LAYOUT_UNKNOWN;
    desc.Flags = flags;

    ComPtr<ID3D12Resource> resource;
    HRESULT hr = device->CreateCommittedResource(
        &heapProps,
        D3D12_HEAP_FLAG_NONE,
        &desc,
        D3D12_RESOURCE_STATE_COMMON,
        nullptr,
        IID_PPV_ARGS(&resource)
    );

    return resource;
}
```

### 2.3 资源的 CPU 映射

对于 `UPLOAD` 和 `READBACK` 堆上的缓冲区，可以调用 `Map` 获取 CPU 指针。注意上传缓冲区在 GPU 使用前不能被 CPU 覆盖，必须处理好同步。

```cpp
void* mappedData = nullptr;
resource->Map(0, nullptr, &mappedData);
std::memcpy(mappedData, sourceData, dataSize);
resource->Unmap(0, nullptr);
```

---

## 3. 资源状态与屏障

D3D12 要求应用程序显式声明资源状态转换，使用**资源屏障（Resource Barrier）**。

### 3.1 常见资源状态

| 状态 | 用途 |
|------|------|
| `D3D12_RESOURCE_STATE_COMMON` | 通用状态，可作为多个状态的入口/出口。 |
| `D3D12_RESOURCE_STATE_VERTEX_AND_CONSTANT_BUFFER` | 作为顶点/常量缓冲区读取。 |
| `D3D12_RESOURCE_STATE_INDEX_BUFFER` | 作为索引缓冲区读取。 |
| `D3D12_RESOURCE_STATE_RENDER_TARGET` | 作为渲染目标写入。 |
| `D3D12_RESOURCE_STATE_DEPTH_WRITE` / `DEPTH_READ` | 深度/模板写入或只读。 |
| `D3D12_RESOURCE_STATE_COPY_DEST` | 复制目标。 |
| `D3D12_RESOURCE_STATE_COPY_SOURCE` | 复制源。 |
| `D3D12_RESOURCE_STATE_GENERIC_READ` | 上传缓冲区的稳定状态。 |
| `D3D12_RESOURCE_STATE_UNORDERED_ACCESS` | UAV 读写，通常配合 `UAV Barrier` 使用。 |

### 3.2 状态转换示例

```cpp
void TransitionResource(
    ID3D12GraphicsCommandList* cmdList,
    ID3D12Resource* resource,
    D3D12_RESOURCE_STATES before,
    D3D12_RESOURCE_STATES after)
{
    D3D12_RESOURCE_BARRIER barrier = {};
    barrier.Type = D3D12_RESOURCE_BARRIER_TYPE_TRANSITION;
    barrier.Transition.pResource = resource;
    barrier.Transition.StateBefore = before;
    barrier.Transition.StateAfter = after;
    barrier.Transition.Subresource = D3D12_RESOURCE_BARRIER_ALL_SUBRESOURCES;

    cmdList->ResourceBarrier(1, &barrier);
}
```

### 3.3 合并屏障（Grouped Barriers）

现代 D3D12 推荐尽量合并屏障，一次性提交多个转换：

```cpp
void TransitionResources(
    ID3D12GraphicsCommandList* cmdList,
    const std::vector<std::tuple<ID3D12Resource*, D3D12_RESOURCE_STATES, D3D12_RESOURCE_STATES>>& transitions)
{
    std::vector<D3D12_RESOURCE_BARRIER> barriers;
    barriers.reserve(transitions.size());

    for (auto& [res, before, after] : transitions)
    {
        D3D12_RESOURCE_BARRIER b = {};
        b.Type = D3D12_RESOURCE_BARRIER_TYPE_TRANSITION;
        b.Transition.pResource = res;
        b.Transition.StateBefore = before;
        b.Transition.StateAfter = after;
        b.Transition.Subresource = D3D12_RESOURCE_BARRIER_ALL_SUBRESOURCES;
        barriers.push_back(b);
    }

    if (!barriers.empty())
        cmdList->ResourceBarrier(static_cast<UINT>(barriers.size()), barriers.data());
}
```

### 3.4 UAV 屏障与别名屏障

- **UAV Barrier**：确保 UAV 读写有序，常用于计算着色器 dispatch 之间。
- **Aliasing Barrier**：同一堆内两个资源共享物理内存时，切换活动资源前必须使用别名屏障。

```cpp
void UAVBarrier(ID3D12GraphicsCommandList* cmdList, ID3D12Resource* resource)
{
    D3D12_RESOURCE_BARRIER barrier = {};
    barrier.Type = D3D12_RESOURCE_BARRIER_TYPE_UAV;
    barrier.UAV.pResource = resource;
    cmdList->ResourceBarrier(1, &barrier);
}
```

---

## 4. 上传与读回

### 4.1 上传缓冲区策略

最常见的上传模式：

1. 创建较大的环形上传缓冲区（Ring Buffer）。
2. 每帧在 ring buffer 中分配一段空间。
3. 用 `Map`/`Unmap` 写入数据。
4. 通过 `CopyBufferRegion` 或 `CopyTextureRegion` 复制到默认堆资源。

### 4.2 最小上传辅助类

```cpp
class UploadBuffer
{
public:
    UploadBuffer(ID3D12Device* device, uint64_t pageSize = 64 * 1024 * 1024)
        : m_pageSize(pageSize)
    {
        m_currentPage = AllocatePage(device);
    }

    uint8_t* Allocate(uint64_t size, uint64_t alignment, uint64_t& outOffset)
    {
        uint64_t alignedOffset = AlignUp(m_currentOffset, alignment);
        if (alignedOffset + size > m_pageSize)
        {
            // 需要新页（假设已有 device 指针，实际项目中应传入）
            return nullptr;
        }

        m_currentOffset = alignedOffset + size;
        outOffset = alignedOffset;

        if (!m_mappedPtr)
            m_currentPage->Map(0, nullptr, reinterpret_cast<void**>(&m_mappedPtr));

        return m_mappedPtr + alignedOffset;
    }

    void Reset()
    {
        m_currentOffset = 0;
        m_mappedPtr = nullptr;
        // 实际应回收页或等待 GPU 完成
    }

    ID3D12Resource* GetResource() const { return m_currentPage.Get(); }

private:
    ComPtr<ID3D12Resource> AllocatePage(ID3D12Device* device)
    {
        return CreateUploadBuffer(device, m_pageSize);
    }

    static uint64_t AlignUp(uint64_t value, uint64_t alignment)
    {
        return (value + alignment - 1) & ~(alignment - 1);
    }

    ComPtr<ID3D12Resource> m_currentPage;
    uint8_t* m_mappedPtr = nullptr;
    uint64_t m_currentOffset = 0;
    uint64_t m_pageSize = 0;
};
```

### 4.3 读回数据

```cpp
void ReadbackBuffer(ID3D12GraphicsCommandList* cmdList,
                    ID3D12Resource* readbackBuffer,
                    ID3D12Resource* sourceBuffer,
                    uint64_t size)
{
    TransitionResource(cmdList, sourceBuffer,
                       D3D12_RESOURCE_STATE_COMMON,
                       D3D12_RESOURCE_STATE_COPY_SOURCE);

    cmdList->CopyBufferRegion(readbackBuffer, 0, sourceBuffer, 0, size);

    TransitionResource(cmdList, sourceBuffer,
                       D3D12_RESOURCE_STATE_COPY_SOURCE,
                       D3D12_RESOURCE_STATE_COMMON);
}
```

---

## 5. 描述符与描述符堆

### 5.1 描述符类型

- **CBV / SRV / UAV**：常量缓冲、着色器资源、无序访问视图。
- **RTV / DSV**：渲染目标和深度模板视图。
- **Sampler**：采样器。

### 5.2 描述符堆

描述符必须存放在描述符堆（`ID3D12DescriptorHeap`）中：

```cpp
ComPtr<ID3D12DescriptorHeap> CreateCBVSRVUAVHeap(ID3D12Device* device, uint32_t numDescriptors)
{
    D3D12_DESCRIPTOR_HEAP_DESC desc = {};
    desc.Type = D3D12_DESCRIPTOR_HEAP_TYPE_CBV_SRV_UAV;
    desc.NumDescriptors = numDescriptors;
    desc.Flags = D3D12_DESCRIPTOR_HEAP_FLAG_SHADER_VISIBLE;

    ComPtr<ID3D12DescriptorHeap> heap;
    device->CreateDescriptorHeap(&desc, IID_PPV_ARGS(&heap));
    return heap;
}
```

### 5.3 创建 SRV

```cpp
void CreateBufferSRV(ID3D12Device* device,
                     ID3D12Resource* resource,
                     D3D12_CPU_DESCRIPTOR_HANDLE cpuHandle,
                     uint32_t numElements,
                     uint32_t stride)
{
    D3D12_SHADER_RESOURCE_VIEW_DESC srvDesc = {};
    srvDesc.Format = DXGI_FORMAT_UNKNOWN;
    srvDesc.ViewDimension = D3D12_SRV_DIMENSION_BUFFER;
    srvDesc.Shader4ComponentMapping = D3D12_DEFAULT_SHADER_4_COMPONENT_MAPPING;
    srvDesc.Buffer.FirstElement = 0;
    srvDesc.Buffer.NumElements = numElements;
    srvDesc.Buffer.StructureByteStride = stride;

    device->CreateShaderResourceView(resource, &srvDesc, cpuHandle);
}
```

---

## 6. 内存分配策略

### 6.1 按用途选择堆类型

| 用途 | 推荐堆类型 |
|------|-----------|
| 长期静态几何/纹理 | `DEFAULT` 提交资源 |
| 每帧动态上传数据 | `UPLOAD` 环形缓冲区 |
| 临时渲染目标/G-Buffer | `DEFAULT` 放置资源 + 池复用 |
| GPU 计算结果回读 | `READBACK` 缓冲区 |

### 6.2 对齐与偏移

D3D12 对资源有严格的对齐要求：

- 普通缓冲区：64 KB（`D3D12_DEFAULT_RESOURCE_PLACEMENT_ALIGNMENT`）。
- MSAA 纹理：4 MB（`D3D12_DEFAULT_MSAA_RESOURCE_PLACEMENT_ALIGNMENT`）。
- 创建放置资源时，偏移必须对齐到 `D3D12_RESOURCE_ALLOCATION_INFO` 返回的 `Alignment`。

### 6.3 资源别名

在显存受限时，可以在同一个堆中放置多个互斥使用的资源，通过别名屏障切换：

```cpp
D3D12_RESOURCE_BARRIER aliasBarrier = {};
aliasBarrier.Type = D3D12_RESOURCE_BARRIER_TYPE_ALIASING;
aliasBarrier.Aliasing.pResourceBefore = previousResource;
aliasBarrier.Aliasing.pResourceAfter = nextResource;
cmdList->ResourceBarrier(1, &aliasBarrier);
```

---

## 7. 高级应用：对象池与内存池

### 7.1 为什么需要对象池

D3D12 中创建/销毁 `ID3D12Resource` 和 `ID3D12Heap` 开销较大，且频繁分配会导致显存碎片化。对象池可以：

- 复用已销毁的资源，避免反复调用 `CreateCommittedResource`。
- 控制显存预算，避免驱动程序隐式分配失控。
- 实现帧分配器，按帧回收临时资源。

### 7.2 资源对象池设计

基本思路：以资源的 `D3D12_RESOURCE_DESC` 为 key，缓存相同描述符的资源。需要使用时从池中取出，不需要时归还（标记为可复用）。

```cpp
#include <unordered_map>
#include <vector>
#include <mutex>

class ResourcePool
{
public:
    struct DescHash
    {
        size_t operator()(const D3D12_RESOURCE_DESC& desc) const
        {
            size_t h = 0;
            HashCombine(h, desc.Dimension, desc.Width, desc.Height,
                        desc.DepthOrArraySize, desc.MipLevels,
                        static_cast<uint32_t>(desc.Format),
                        desc.SampleDesc.Count, desc.SampleDesc.Quality,
                        static_cast<uint32_t>(desc.Layout),
                        static_cast<uint32_t>(desc.Flags));
            return h;
        }

    private:
        template<typename... Args>
        void HashCombine(size_t& seed, Args... args) const
        {
            ((seed ^= std::hash<uint64_t>{}(args) + 0x9e3779b9 + (seed << 6) + (seed >> 2)), ...);
        }
    };

    struct DescEqual
    {
        bool operator()(const D3D12_RESOURCE_DESC& a, const D3D12_RESOURCE_DESC& b) const
        {
            return std::memcmp(&a, &b, sizeof(D3D12_RESOURCE_DESC)) == 0;
        }
    };

    ComPtr<ID3D12Resource> Acquire(ID3D12Device* device,
                                   const D3D12_RESOURCE_DESC& desc,
                                   D3D12_RESOURCE_STATES initialState,
                                   const D3D12_CLEAR_VALUE* clearValue = nullptr)
    {
        std::lock_guard<std::mutex> lock(m_mutex);

        auto& pool = m_pools[desc];
        for (auto it = pool.begin(); it != pool.end(); ++it)
        {
            if (IsResourceCompatible(*it, desc, clearValue))
            {
                ComPtr<ID3D12Resource> res = *it;
                pool.erase(it);
                return res;
            }
        }

        D3D12_HEAP_PROPERTIES heapProps = {};
        heapProps.Type = D3D12_HEAP_TYPE_DEFAULT;

        ComPtr<ID3D12Resource> resource;
        device->CreateCommittedResource(
            &heapProps,
            D3D12_HEAP_FLAG_NONE,
            &desc,
            initialState,
            clearValue,
            IID_PPV_ARGS(&resource)
        );

        return resource;
    }

    void Release(ID3D12Resource* resource)
    {
        if (!resource) return;

        D3D12_RESOURCE_DESC desc = resource->GetDesc();

        std::lock_guard<std::mutex> lock(m_mutex);
        m_pools[desc].push_back(resource);
    }

    void Clear()
    {
        std::lock_guard<std::mutex> lock(m_mutex);
        m_pools.clear();
    }

private:
    bool IsResourceCompatible(ID3D12Resource* resource,
                              const D3D12_RESOURCE_DESC& desc,
                              const D3D12_CLEAR_VALUE* clearValue)
    {
        // 简单实现：DESC 相同即兼容。严格场景应进一步检查 ClearValue。
        return true;
    }

    std::unordered_map<D3D12_RESOURCE_DESC, std::vector<ComPtr<ID3D12Resource>>, DescHash, DescEqual> m_pools;
    std::mutex m_mutex;
};
```

### 7.3 帧资源分配器（Frame Allocator）

渲染中很多临时资源只需存在一帧，可以按帧号分配，在 GPU 完成该帧后统一回收：

```cpp
class FrameResourcePool
{
public:
    FrameResourcePool(ID3D12Device* device, uint32_t frameCount)
        : m_device(device), m_frameCount(frameCount)
    {
        m_frameBins.resize(frameCount);
    }

    // 在当前帧借用一个资源
    ComPtr<ID3D12Resource> Allocate(const D3D12_RESOURCE_DESC& desc,
                                    D3D12_RESOURCE_STATES initialState)
    {
        return m_pool.Acquire(m_device.Get(), desc, initialState);
    }

    // 归还到指定帧的回收箱
    void Release(uint32_t frameIndex, ID3D12Resource* resource)
    {
        m_frameBins[frameIndex % m_frameCount].push_back(resource);
    }

    // 当 GPU 完成某帧后调用
    void RecycleCompletedFrame(uint32_t frameIndex)
    {
        auto& bin = m_frameBins[frameIndex % m_frameCount];
        for (auto& res : bin)
            m_pool.Release(res.Get());
        bin.clear();
    }

private:
    ComPtr<ID3D12Device> m_device;
    ResourcePool m_pool;
    uint32_t m_frameCount = 0;
    std::vector<std::vector<ComPtr<ID3D12Resource>>> m_frameBins;
};
```

### 7.4 堆池（Heap Pool）

对于大量放置资源，可以先创建一堆固定大小的堆，再在堆中分配偏移。堆池示例：

```cpp
class HeapAllocator
{
public:
    HeapAllocator(ID3D12Device* device, uint64_t heapSize)
        : m_device(device), m_heapSize(heapSize)
    {
        AllocateNewHeap();
    }

    bool Allocate(const D3D12_RESOURCE_ALLOCATION_INFO& allocInfo,
                  ID3D12Heap** outHeap,
                  uint64_t& outOffset)
    {
        uint64_t alignedOffset = AlignUp(m_currentOffset, allocInfo.Alignment);
        if (alignedOffset + allocInfo.SizeInBytes > m_heapSize)
        {
            if (!AllocateNewHeap())
                return false;
            alignedOffset = 0;
        }

        *outHeap = m_heaps.back().Get();
        outOffset = alignedOffset;
        m_currentOffset = alignedOffset + allocInfo.SizeInBytes;
        return true;
    }

private:
    bool AllocateNewHeap()
    {
        D3D12_HEAP_DESC heapDesc = {};
        heapDesc.SizeInBytes = m_heapSize;
        heapDesc.Properties.Type = D3D12_HEAP_TYPE_DEFAULT;
        heapDesc.Alignment = 0;
        heapDesc.Flags = D3D12_HEAP_FLAG_ALLOW_ONLY_NON_RT_DS_TEXTURES;

        ComPtr<ID3D12Heap> heap;
        HRESULT hr = m_device->CreateHeap(&heapDesc, IID_PPV_ARGS(&heap));
        if (FAILED(hr))
            return false;

        m_heaps.push_back(heap);
        m_currentOffset = 0;
        return true;
    }

    static uint64_t AlignUp(uint64_t value, uint64_t alignment)
    {
        return (value + alignment - 1) & ~(alignment - 1);
    }

    ComPtr<ID3D12Device> m_device;
    uint64_t m_heapSize = 0;
    uint64_t m_currentOffset = 0;
    std::vector<ComPtr<ID3D12Heap>> m_heaps;
};
```

---

## 8. 完整示例：一个最小化的资源管理器

下面是一个可用于小型 D3D12 程序的简化资源管理器，整合了上传缓冲、描述符分配和临时资源池：

```cpp
#include <d3d12.h>
#include <wrl/client.h>
#include <vector>
#include <memory>
#include <cstdint>

using Microsoft::WRL::ComPtr;

class SimpleResourceManager
{
public:
    SimpleResourceManager(ID3D12Device* device, uint32_t frameCount)
        : m_device(device)
        , m_uploadBuffer(device, 16 * 1024 * 1024)
        , m_framePool(device, frameCount)
    {
        // 创建全局 CBV/SRV/UAV 描述符堆
        D3D12_DESCRIPTOR_HEAP_DESC heapDesc = {};
        heapDesc.Type = D3D12_DESCRIPTOR_HEAP_TYPE_CBV_SRV_UAV;
        heapDesc.NumDescriptors = 1024;
        heapDesc.Flags = D3D12_DESCRIPTOR_HEAP_FLAG_SHADER_VISIBLE;
        device->CreateDescriptorHeap(&heapDesc, IID_PPV_ARGS(&m_descriptorHeap));

        m_descriptorSize = device->GetDescriptorHandleIncrementSize(heapDesc.Type);
        m_nextDescriptorIndex = 0;
    }

    // 上传数据到默认堆缓冲区
    void UploadData(ID3D12GraphicsCommandList* cmdList,
                    ID3D12Resource* dstResource,
                    const void* data,
                    uint64_t size)
    {
        uint64_t offset = 0;
        uint8_t* uploadPtr = m_uploadBuffer.Allocate(size, 256, offset);
        std::memcpy(uploadPtr, data, size);

        TransitionResource(cmdList, dstResource,
                           D3D12_RESOURCE_STATE_COMMON,
                           D3D12_RESOURCE_STATE_COPY_DEST);

        cmdList->CopyBufferRegion(dstResource, 0, m_uploadBuffer.GetResource(), offset, size);

        TransitionResource(cmdList, dstResource,
                           D3D12_RESOURCE_STATE_COPY_DEST,
                           D3D12_RESOURCE_STATE_VERTEX_AND_CONSTANT_BUFFER);
    }

    // 创建一个临时渲染目标
    ComPtr<ID3D12Resource> CreateTempRenderTarget(uint32_t width,
                                                   uint32_t height,
                                                   DXGI_FORMAT format,
                                                   uint32_t frameIndex)
    {
        D3D12_RESOURCE_DESC desc = {};
        desc.Dimension = D3D12_RESOURCE_DIMENSION_TEXTURE2D;
        desc.Width = width;
        desc.Height = height;
        desc.DepthOrArraySize = 1;
        desc.MipLevels = 1;
        desc.Format = format;
        desc.SampleDesc.Count = 1;
        desc.Layout = D3D12_TEXTURE_LAYOUT_UNKNOWN;
        desc.Flags = D3D12_RESOURCE_FLAG_ALLOW_RENDER_TARGET;

        return m_framePool.Allocate(desc, D3D12_RESOURCE_STATE_RENDER_TARGET);
    }

    D3D12_GPU_DESCRIPTOR_HANDLE AllocateSRV(ID3D12Resource* resource,
                                            uint32_t numElements,
                                            uint32_t stride)
    {
        uint32_t index = m_nextDescriptorIndex++;

        CD3DX12_CPU_DESCRIPTOR_HANDLE cpuHandle(
            m_descriptorHeap->GetCPUDescriptorHandleForHeapStart(),
            index,
            m_descriptorSize);

        CD3DX12_GPU_DESCRIPTOR_HANDLE gpuHandle(
            m_descriptorHeap->GetGPUDescriptorHandleForHeapStart(),
            index,
            m_descriptorSize);

        D3D12_SHADER_RESOURCE_VIEW_DESC srvDesc = {};
        srvDesc.Format = DXGI_FORMAT_UNKNOWN;
        srvDesc.ViewDimension = D3D12_SRV_DIMENSION_BUFFER;
        srvDesc.Shader4ComponentMapping = D3D12_DEFAULT_SHADER_4_COMPONENT_MAPPING;
        srvDesc.Buffer.NumElements = numElements;
        srvDesc.Buffer.StructureByteStride = stride;

        m_device->CreateShaderResourceView(resource, &srvDesc, cpuHandle);
        return gpuHandle;
    }

    void EndFrame(uint32_t frameIndex)
    {
        m_framePool.RecycleCompletedFrame(frameIndex);
        m_uploadBuffer.Reset();
        m_nextDescriptorIndex = 0;
    }

private:
    ComPtr<ID3D12Device> m_device;
    UploadBuffer m_uploadBuffer;
    FrameResourcePool m_framePool;
    ComPtr<ID3D12DescriptorHeap> m_descriptorHeap;
    uint32_t m_descriptorSize = 0;
    uint32_t m_nextDescriptorIndex = 0;
};
```

> 注：上面的示例为了篇幅省略了 fences/等待同步代码。实际项目中，`UploadBuffer::Reset` 和 `RecycleCompletedFrame` 应在确认 GPU 已完成对应帧后执行。

---

## 9. 最佳实践与常见陷阱

### 9.1 推荐做法

1. **合并资源屏障**：尽量一次性提交多个转换，减少命令列表拆分。
2. **使用环形上传缓冲区**：避免为每个小资源单独创建上传缓冲区。
3. **显式管理资源状态**：不要依赖 `D3D12_RESOURCE_STATE_COMMON` 自动提升/衰减，性能不可控。
4. **复用渲染目标和临时资源**：使用对象池或帧分配器减少分配开销。
5. **监控显存预算**：使用 `IDXGIAdapter3::QueryVideoMemoryInfo` 跟踪本地/非本地显存使用。
6. **使用放置资源控制别名**：在显存紧张的场景下，通过别名屏障让多个资源共用物理内存。

### 9.2 常见错误

| 错误 | 后果 |
|------|------|
| 写入 `UPLOAD` 堆后未等待 GPU 读取完成 | 数据竞争、画面闪烁或崩溃。 |
| 忘记状态转换直接使用资源 | 验证层报错，渲染结果错误。 |
| 过度细分资源，每个资源一个堆 | 显存碎片化，分配失败。 |
| 在渲染中频繁创建/销毁资源 | CPU 卡顿，驱动分配开销大。 |
| 未对齐放置资源偏移 | `CreatePlacedResource` 失败。 |
| 忽略 UAV Barrier | 计算结果依赖顺序出错。 |

### 9.3 调试建议

- 开启 D3D12 调试层：

```cpp
ComPtr<ID3D12Debug> debugInterface;
D3D12GetDebugInterface(IID_PPV_ARGS(&debugInterface));
debugInterface->EnableDebugLayer();
```

- 使用 GPU 验证（GPU-based validation）和 DRED（Device Removed Extended Data）定位设备移除问题。
- 使用 PIX 或 RenderDoc 捕获帧，检查资源状态转换和显存使用。

---

## 结语

D3D12 的资源管理将显式控制交给了开发者，既带来了极高的性能潜力，也带来了更高的复杂度。理解 `Committed` / `Placed` / `Reserved` 三种资源、合理设计上传/读回路径、使用对象池和帧分配器复用资源，是构建高性能 D3D12 渲染器的关键。

随着项目规模增长，建议逐步引入专门的内存分配器（如开源的 [D3D12MemoryAllocator](https://github.com/GPUOpen-LibrariesAndSDKs/D3D12MemoryAllocator)），以获得更完善的堆管理、预算控制和碎片化治理。
