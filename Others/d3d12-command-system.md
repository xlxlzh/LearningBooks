# D3D12 命令系统完全指南

在 Direct3D 12 中，CPU 与 GPU 之间的几乎所有工作都通过**命令系统**驱动。理解 **Command Queue（命令队列）**、**Command Allocator（命令分配器）**、**Command List（命令列表）** 以及它们之间的协作关系，是掌握 D3D12 的关键。本文从基础概念出发，逐步深入到多线程录制、Bundle、Fence 同步等高级主题，并配以完整代码示例。

---

## 目录

1. [整体架构概览](#1-整体架构概览)
2. [Command Queue](#2-command-queue)
3. [Command Allocator](#3-command-allocator)
4. [Command List](#4-command-list)
5. [命令录制与提交的生命周期](#5-命令录制与提交的生命周期)
6. [Fence 与 GPU/CPU 同步](#6-fence-与-gpucpu-同步)
7. [多线程命令录制](#7-多线程命令录制)
8. [Bundle](#8-bundle)
9. [完整示例：渲染循环](#9-完整示例渲染循环)
10. [最佳实践与常见陷阱](#10-最佳实践与常见陷阱)

---

## 1. 整体架构概览

D3D12 的命令系统可以用下面的流程概括：

```
CPU 端：
  Command Allocator → Command List → 录制命令（Draw、Copy、Barrier 等）
                                          ↓
  Command Queue ← ExecuteCommandLists([Command List(s)])
                                          ↓
GPU 端：
  按 FIFO 顺序执行 Command Queue 中的命令
```

核心对象：

| 对象 | 作用 | 典型数量 |
|------|------|----------|
| `ID3D12CommandQueue` | 向 GPU 提交命令队列，按 FIFO 执行 | 每类队列一个（Direct/Compute/Copy） |
| `ID3D12CommandAllocator` | 存储 Command List 的命令内存 | 每帧/每线程至少一个 |
| `ID3D12GraphicsCommandList` | 录制具体命令 | 可大量创建，轻量对象 |
| `ID3D12Fence` | GPU/CPU 同步 | 每队列一个即可 |

---

## 2. Command Queue

### 2.1 三种队列类型

D3D12 支持三种命令队列，对应 `D3D12_COMMAND_LIST_TYPE`：

| 类型 | 用途 | 可执行列表 |
|------|------|-----------|
| `D3D12_COMMAND_LIST_TYPE_DIRECT` | 渲染、通用 GPU 工作 | Direct/Bundle |
| `D3D12_COMMAND_LIST_TYPE_COMPUTE` | 计算着色器、UAV 工作 | Compute/Bundle |
| `D3D12_COMMAND_LIST_TYPE_COPY` | 纯复制工作 | Copy |

Direct Queue 可以执行所有类型的命令；Compute Queue 只能执行计算相关命令；Copy Queue 只能执行复制命令。

### 2.2 创建命令队列

```cpp
#include <d3d12.h>
#include <wrl/client.h>

using Microsoft::WRL::ComPtr;

ComPtr<ID3D12CommandQueue> CreateCommandQueue(
    ID3D12Device* device,
    D3D12_COMMAND_LIST_TYPE type)
{
    D3D12_COMMAND_QUEUE_DESC queueDesc = {};
    queueDesc.Type = type;
    queueDesc.Priority = D3D12_COMMAND_QUEUE_PRIORITY_NORMAL;
    queueDesc.Flags = D3D12_COMMAND_QUEUE_FLAG_NONE;
    queueDesc.NodeMask = 0;

    ComPtr<ID3D12CommandQueue> queue;
    device->CreateCommandQueue(
        &queueDesc,
        IID_PPV_ARGS(&queue)
    );

    return queue;
}
```

### 2.3 提交命令列表

```cpp
ID3D12CommandList* commandLists[] = { commandList.Get() };
commandQueue->ExecuteCommandLists(1, commandLists);
```

注意 `ExecuteCommandLists` 接受的是 `ID3D12CommandList*`，即使实际对象通常是 `ID3D12GraphicsCommandList`。

### 2.4 队列间的同步

不同队列之间的同步必须通过 `Fence` 完成。例如让 Copy Queue 上传完成后通知 Direct Queue：

```cpp
// Copy Queue 写入 fence
copyQueue->Signal(copyFence.Get(), copyValue);

// Direct Queue 等待 fence
directQueue->Wait(copyFence.Get(), copyValue);
```

---

## 3. Command Allocator

### 3.1 作用

`ID3D12CommandAllocator` 是命令的**内存池**。Command List 录制的所有命令实际上都存储在关联的 Allocator 中。

### 3.2 关键规则

1. **一个 Allocator 同一时刻只能被一个正在录制的 Command List 使用**。
2. **Allocator 中的命令被提交到 GPU 后，必须等待 GPU 执行完成才能 Reset**。
3. **一个 Allocator 可以反复 Reset/复用**，不需要频繁创建销毁。

### 3.3 创建 Allocator

```cpp
ComPtr<ID3D12CommandAllocator> CreateCommandAllocator(
    ID3D12Device* device,
    D3D12_COMMAND_LIST_TYPE type)
{
    ComPtr<ID3D12CommandAllocator> allocator;
    device->CreateCommandAllocator(
        type,
        IID_PPV_ARGS(&allocator)
    );
    return allocator;
}
```

### 3.4 Reset Allocator

```cpp
// 必须在 GPU 完成该 allocator 的所有命令后才能调用
allocator->Reset();
```

在多缓冲（multi-buffering）渲染中，通常每帧使用一个 Allocator，这样本帧录制时不会干扰上一帧 GPU 尚未完成的命令。

---

## 4. Command List

### 4.1 创建 Command List

```cpp
ComPtr<ID3D12GraphicsCommandList> CreateCommandList(
    ID3D12Device* device,
    ID3D12CommandAllocator* allocator,
    D3D12_COMMAND_LIST_TYPE type)
{
    ComPtr<ID3D12GraphicsCommandList> commandList;
    device->CreateCommandList(
        0,                          // node mask
        type,
        allocator,
        nullptr,                    // initial pipeline state
        IID_PPV_ARGS(&commandList)
    );

    return commandList;
}
```

创建后的 Command List 处于**录制状态（recording state）**。

### 4.2 常用录制命令

```cpp
// 设置渲染目标
cmdList->OMSetRenderTargets(1, &rtvHandle, FALSE, &dsvHandle);

// 清屏
cmdList->ClearRenderTargetView(rtvHandle, clearColor, 0, nullptr);

// 设置 viewport 和 scissor
D3D12_VIEWPORT viewport = { 0, 0, width, height, 0.0f, 1.0f };
D3D12_RECT scissor = { 0, 0, width, height };
cmdList->RSSetViewports(1, >viewport);
cmdList->RSSetScissorRects(1, &scissor);

// 设置 PSO、根签名
cmdList->SetPipelineState(pso);
cmdList->SetGraphicsRootSignature(rootSignature);

// 绘制
cmdList->DrawIndexedInstanced(indexCount, instanceCount, 0, 0, 0);
```

### 4.3 Close 与 Reset

```cpp
// 结束录制，准备提交
cmdList->Close();

// 复用前：先重置 allocator，再重置 command list
cmdList->Reset(allocator, nullptr);
```

`Reset` 不会清空 Command List 本身的内容引用（如 PSO、根签名），但会让 List 重新进入录制状态。实际命令内存来自 Allocator。

---

## 5. 命令录制与提交的生命周期

一个典型的渲染帧生命周期如下：

```cpp
void RenderFrame()
{
    // 1. 等待本帧的 fence
    WaitForFence(m_frameFences[m_frameIndex]);

    // 2. 重置 allocator 和 command list
    m_commandAllocators[m_frameIndex]->Reset();
    m_commandList->Reset(m_commandAllocators[m_frameIndex].Get(), m_pso.Get());

    // 3. 录制命令
    RecordCommands();

    // 4. 关闭 command list
    m_commandList->Close();

    // 5. 提交到队列
    ID3D12CommandList* lists[] = { m_commandList.Get() };
    m_commandQueue->ExecuteCommandLists(1, lists);

    // 6. 呈现
    m_swapChain->Present(1, 0);

    // 7. 设置 fence，标记 GPU 进度
    m_commandQueue->Signal(m_fence.Get(), m_currentFenceValue);
    m_frameFences[m_frameIndex] = m_currentFenceValue++;

    // 8. 切换到下一帧缓冲区
    m_frameIndex = (m_frameIndex + 1) % kFrameCount;
}
```

### 5.1 为什么每帧要一个 Allocator

如果所有帧共用一个 Allocator，那么你在第 N+1 帧调用 `Reset` 时，必须确保第 N 帧的命令已被 GPU 执行完。使用每帧独立的 Allocator，可以让 CPU 连续录制多帧而不阻塞等待 GPU。

---

## 6. Fence 与 GPU/CPU 同步

### 6.1 Fence 基础

`ID3D12Fence` 用于在 GPU 和 CPU 之间传递单调递增的整数值：

- `Signal(fence, value)`：队列执行到此处时，将 fence 设为 value。
- `Wait(fence, value)`：队列阻塞，直到 fence >= value。
- `SetEventOnCompletion(value, event)`：CPU 等待 fence 达到 value。

### 6.2 CPU 等待 GPU 完成

```cpp
class FenceHelper
{
public:
    void Initialize(ID3D12Device* device)
    {
        device->CreateFence(0, D3D12_FENCE_FLAG_NONE, IID_PPV_ARGS(&m_fence));
        m_event = CreateEvent(nullptr, FALSE, FALSE, nullptr);
    }

    void WaitForValue(uint64_t value)
    {
        if (m_fence->GetCompletedValue() < value)
        {
            m_fence->SetEventOnCompletion(value, m_event);
            WaitForSingleObject(m_event, INFINITE);
        }
    }

    void Cleanup()
    {
        CloseHandle(m_event);
    }

    ID3D12Fence* GetFence() const { return m_fence.Get(); }

private:
    ComPtr<ID3D12Fence> m_fence;
    HANDLE m_event = nullptr;
};
```

### 6.3 GPU 等待 GPU（队列间同步）

```cpp
// 让 directQueue 等待 copyQueue 的某个 fence 值
copyQueue->Signal(copyFence.Get(), 1);
directQueue->Wait(copyFence.Get(), 1);
```

### 6.4 多缓冲 fence 模式

```cpp
static constexpr uint32_t kFrameCount = 3;

uint64_t m_fenceValues[kFrameCount] = {};
uint32_t m_frameIndex = 0;

void MoveToNextFrame()
{
    const uint64_t currentFenceValue = m_fenceValue;
    m_commandQueue->Signal(m_fence.Get(), currentFenceValue);

    m_fenceValues[m_frameIndex] = currentFenceValue;
    m_frameIndex = (m_frameIndex + 1) % kFrameCount;

    if (m_fence->GetCompletedValue() < m_fenceValues[m_frameIndex])
    {
        m_fence->SetEventOnCompletion(m_fenceValues[m_frameIndex], m_fenceEvent);
        WaitForSingleObject(m_fenceEvent, INFINITE);
    }

    m_fenceValue++;
}
```

---

## 7. 多线程命令录制

### 7.1 设计原则

D3D12 允许从多个线程同时录制 Command List，核心规则：

1. **每个线程使用独立的 Command Allocator**。
2. **每个线程使用独立的 Command List**（或临时创建的 Bundle/List）。
3. **仅在主线程调用 `ExecuteCommandLists`**。

### 7.2 多线程录制示例

```cpp
#include <thread>
#include <vector>
#include <functional>

class ThreadedCommandRecorder
{
public:
    ThreadedCommandRecorder(ID3D12Device* device, uint32_t threadCount)
        : m_device(device)
        , m_threadCount(threadCount)
    {
        for (uint32_t i = 0; i < threadCount; ++i)
        {
            m_allocators.push_back(CreateCommandAllocator(device, D3D12_COMMAND_LIST_TYPE_DIRECT));

            ComPtr<ID3D12GraphicsCommandList> list;
            device->CreateCommandList(0, D3D12_COMMAND_LIST_TYPE_DIRECT,
                                       m_allocators.back().Get(), nullptr,
                                       IID_PPV_ARGS(&list));
            m_commandLists.push_back(list);
        }
    }

    void RecordParallel(const std::vector<std::function<void(ID3D12GraphicsCommandList*)>>& tasks)
    {
        std::vector<std::thread> threads;
        threads.reserve(tasks.size());

        for (size_t i = 0; i < tasks.size(); ++i)
        {
            threads.emplace_back([this, i, &tasks]() {
                auto* list = m_commandLists[i].Get();
                list->Reset(m_allocators[i].Get(), nullptr);
                tasks[i](list);
                list->Close();
            });
        }

        for (auto& t : threads)
            t.join();
    }

    void Submit(ID3D12CommandQueue* queue)
    {
        std::vector<ID3D12CommandList*> lists;
        lists.reserve(m_commandLists.size());

        for (auto& list : m_commandLists)
            lists.push_back(list.Get());

        queue->ExecuteCommandLists(static_cast<UINT>(lists.size()), lists.data());
    }

    void ResetAllocators()
    {
        for (auto& alloc : m_allocators)
            alloc->Reset();
    }

private:
    ComPtr<ID3D12Device> m_device;
    uint32_t m_threadCount = 0;
    std::vector<ComPtr<ID3D12CommandAllocator>> m_allocators;
    std::vector<ComPtr<ID3D12GraphicsCommandList>> m_commandLists;
};
```

### 7.3 二级命令列表

D3D12 提供了**Bundle**，它是一种可以在多个 Direct/Compute Command List 中复用的预录制命令序列。Bundle 的限制较多：

- 不能修改渲染目标、viewport、scissor（除非 PSO 中设为 dynamic）。
- 不能执行 `Copy*` 命令。
- 适合录制静态的 Draw Call 序列。

---

## 8. Bundle

### 8.1 创建 Bundle

```cpp
ComPtr<ID3D12GraphicsCommandList> CreateBundle(
    ID3D12Device* device,
    ID3D12CommandAllocator* bundleAllocator,
    ID3D12PipelineState* pso)
{
    ComPtr<ID3D12GraphicsCommandList> bundle;
    device->CreateCommandList(
        0,
        D3D12_COMMAND_LIST_TYPE_BUNDLE,
        bundleAllocator,
        pso,
        IID_PPV_ARGS(&bundle)
    );
    return bundle;
}
```

### 8.2 在 Direct Command List 中执行 Bundle

```cpp
bundle->Close();
directCommandList->ExecuteBundle(bundle.Get());
```

### 8.3 Bundle 的适用场景

- 静态 UI 渲染。
- 固定网格的多次绘制。
- 需要跨多个 Parent List 复用命令序列时。

现代 CPU 端通常用多线程 Command List 替代 Bundle，但在命令高度重复的场景 Bundle 仍然有用。

---

## 9. 完整示例：渲染循环

下面是一个完整的渲染循环示例，展示了 Queue、Allocator、Command List、Fence、SwapChain 的协同工作：

```cpp
#include <d3d12.h>
#include <dxgi1_6.h>
#include <wrl/client.h>
#include <windows.h>
#include <cstdint>

using Microsoft::WRL::ComPtr;

class D3D12Renderer
{
public:
    static constexpr uint32_t kFrameCount = 3;

    bool Initialize(HWND hwnd, uint32_t width, uint32_t height)
    {
        // 创建设备（省略 adapter 枚举）
        D3D12CreateDevice(nullptr, D3D_FEATURE_LEVEL_12_0, IID_PPV_ARGS(&m_device));

        // 创建命令队列
        m_commandQueue = CreateCommandQueue(m_device.Get(), D3D12_COMMAND_LIST_TYPE_DIRECT);

        // 创建交换链（省略工厂创建细节）
        // ...

        // 创建 RTV 描述符堆
        D3D12_DESCRIPTOR_HEAP_DESC rtvHeapDesc = {};
        rtvHeapDesc.Type = D3D12_DESCRIPTOR_HEAP_TYPE_RTV;
        rtvHeapDesc.NumDescriptors = kFrameCount;
        m_device->CreateDescriptorHeap(&rtvHeapDesc, IID_PPV_ARGS(&m_rtvHeap));
        m_rtvDescriptorSize = m_device->GetDescriptorHandleIncrementSize(D3D12_DESCRIPTOR_HEAP_TYPE_RTV);

        // 创建渲染目标视图
        CD3DX12_CPU_DESCRIPTOR_HANDLE rtvHandle(m_rtvHeap->GetCPUDescriptorHandleForHeapStart());
        for (uint32_t i = 0; i < kFrameCount; ++i)
        {
            m_swapChain->GetBuffer(i, IID_PPV_ARGS(&m_renderTargets[i]));
            m_device->CreateRenderTargetView(m_renderTargets[i].Get(), nullptr, rtvHandle);
            rtvHandle.Offset(1, m_rtvDescriptorSize);
        }

        // 创建每帧的 allocator 和一个 command list
        for (uint32_t i = 0; i < kFrameCount; ++i)
        {
            m_commandAllocators[i] = CreateCommandAllocator(
                m_device.Get(), D3D12_COMMAND_LIST_TYPE_DIRECT);
        }

        m_device->CreateCommandList(
            0,
            D3D12_COMMAND_LIST_TYPE_DIRECT,
            m_commandAllocators[0].Get(),
            nullptr,
            IID_PPV_ARGS(&m_commandList));
        m_commandList->Close();

        // 创建 fence
        m_device->CreateFence(0, D3D12_FENCE_FLAG_NONE, IID_PPV_ARGS(&m_fence));
        m_fenceEvent = CreateEvent(nullptr, FALSE, FALSE, nullptr);
        m_fenceValues[m_frameIndex]++;

        return true;
    }

    void Render()
    {
        // 1. 等待本帧渲染目标可用
        WaitForGpu(m_frameIndex);

        // 2. 重置 allocator 和 command list
        m_commandAllocators[m_frameIndex]->Reset();
        m_commandList->Reset(m_commandAllocators[m_frameIndex].Get(), m_pipelineState.Get());

        // 3. 设置渲染目标
        CD3DX12_CPU_DESCRIPTOR_HANDLE rtvHandle(
            m_rtvHeap->GetCPUDescriptorHandleForHeapStart(),
            m_frameIndex,
            m_rtvDescriptorSize);

        // 资源状态转换：呈现 → 渲染目标
        TransitionResource(m_commandList.Get(),
                           m_renderTargets[m_frameIndex].Get(),
                           D3D12_RESOURCE_STATE_PRESENT,
                           D3D12_RESOURCE_STATE_RENDER_TARGET);

        // 清屏
        const float clearColor[] = { 0.0f, 0.2f, 0.4f, 1.0f };
        m_commandList->ClearRenderTargetView(rtvHandle, clearColor, 0, nullptr);

        // 设置 viewport / scissor / PSO / 根签名 / 网格
        // ...

        // 绘制
        // m_commandList->DrawIndexedInstanced(...);

        // 资源状态转换：渲染目标 → 呈现
        TransitionResource(m_commandList.Get(),
                           m_renderTargets[m_frameIndex].Get(),
                           D3D12_RESOURCE_STATE_RENDER_TARGET,
                           D3D12_RESOURCE_STATE_PRESENT);

        // 4. 关闭 command list
        m_commandList->Close();

        // 5. 提交
        ID3D12CommandList* lists[] = { m_commandList.Get() };
        m_commandQueue->ExecuteCommandLists(1, lists);

        // 6. 呈现
        m_swapChain->Present(1, 0);

        // 7. 记录 fence
        MoveToNextFrame();
    }

    void Shutdown()
    {
        WaitForGpu(m_frameIndex);
        CloseHandle(m_fenceEvent);
    }

private:
    void TransitionResource(ID3D12GraphicsCommandList* cmdList,
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

    void WaitForGpu(uint32_t frameIndex)
    {
        if (m_fence->GetCompletedValue() < m_fenceValues[frameIndex])
        {
            m_fence->SetEventOnCompletion(m_fenceValues[frameIndex], m_fenceEvent);
            WaitForSingleObject(m_fenceEvent, INFINITE);
        }
    }

    void MoveToNextFrame()
    {
        const uint64_t currentFenceValue = m_fenceValue;
        m_commandQueue->Signal(m_fence.Get(), currentFenceValue);
        m_fenceValues[m_frameIndex] = currentFenceValue;

        m_frameIndex = (m_frameIndex + 1) % kFrameCount;
        m_fenceValue++;
    }

    ComPtr<ID3D12Device> m_device;
    ComPtr<ID3D12CommandQueue> m_commandQueue;
    ComPtr<IDXGISwapChain3> m_swapChain;
    ComPtr<ID3D12DescriptorHeap> m_rtvHeap;
    uint32_t m_rtvDescriptorSize = 0;

    ComPtr<ID3D12CommandAllocator> m_commandAllocators[kFrameCount];
    ComPtr<ID3D12Resource> m_renderTargets[kFrameCount];
    ComPtr<ID3D12GraphicsCommandList> m_commandList;
    ComPtr<ID3D12PipelineState> m_pipelineState;

    ComPtr<ID3D12Fence> m_fence;
    HANDLE m_fenceEvent = nullptr;
    uint64_t m_fenceValue = 1;
    uint64_t m_fenceValues[kFrameCount] = {};
    uint32_t m_frameIndex = 0;
};
```

---

## 10. 最佳实践与常见陷阱

### 10.1 推荐做法

1. **每帧一个 Command Allocator**：避免 CPU 等待 GPU 完成上一帧。
2. **重用一个 Command List**：不需要为每帧创建新的 List，`Reset` 即可复用。
3. **合并 `ExecuteCommandLists`**：一次提交多个 List 比多次提交单个 List 更高效。
4. **使用 Fence 而非 `Flush`**：只在真正需要同步时等待，不要每帧都阻塞。
5. **多线程录制时保持 Allocator 隔离**：每个线程独立 Allocator，提交前合并。
6. **合理划分 Queue**：上传工作交给 Copy Queue，计算交给 Compute Queue，与 Direct Queue 并行。
7. **记录 Bundle 用于重复命令**：静态 UI、固定网格适合 Bundle。

### 10.2 常见错误

| 错误 | 后果 |
|------|------|
| 在 GPU 未完成时 Reset Allocator | 命令被覆盖，崩溃或渲染错误。 |
| 多个 List 同时使用同一个 Allocator | 验证层报错，命令损坏。 |
| 忘记 `Close` 就 Execute | 运行时错误。 |
| 忘记 Present 前转换 SwapChain Buffer 到 `PRESENT` 状态 | Present 失败或验证层报错。 |
| 每帧创建/销毁 Command List | 不必要的 CPU 开销。 |
| 跨队列访问资源无 Fence 同步 | 数据竞争，时好时坏。 |
| 在 Bundle 中修改非 dynamic 状态 | Bundle 录制失败或行为未定义。 |

### 10.3 调试建议

- 开启调试层：

```cpp
ComPtr<ID3D12Debug> debugInterface;
D3D12GetDebugInterface(IID_PPV_ARGS(&debugInterface));
debugInterface->EnableDebugLayer();
```

- 启用 GPU-based validation 捕获多线程/同步问题。
- 使用 PIX 捕获帧，检查 Command List 的提交顺序和状态。
- 对 Device Removed 问题启用 DRED：

```cpp
ComPtr<ID3D12DeviceRemovedExtendedDataSettings> dredSettings;
D3D12GetDebugInterface(IID_PPV_ARGS(&dredSettings));
dredSettings->SetAutoBreadcrumbsEnablement(D3D12_DRED_ENABLEMENT_FORCED_ON);
dredSettings->SetPageFaultEnablement(D3D12_DRED_ENABLEMENT_FORCED_ON);
```

---

## 结语

D3D12 的命令系统虽然显式、繁琐，但一旦掌握，就能精确控制 CPU 与 GPU 的协作方式。关键在于：

- **Queue 负责任务提交与顺序**
- **Allocator 负责命令内存**
- **Command List 负责录制具体命令**
- **Fence 负责跨帧、跨队列、跨 CPU/GPU 同步**

合理利用多线程录制、多队列并行和对象复用，可以极大提升渲染器的 CPU 效率和整体吞吐量。
