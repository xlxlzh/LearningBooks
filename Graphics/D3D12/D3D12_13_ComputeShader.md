# 第 13 章 计算着色器

---

## 13.1 线程和线程组

计算着色器不直接属于渲染管线，而是独立于管线之外的并行计算阶段。它可以读取和写入 GPU 资源，实现数据并行算法而无需绘制任何内容。

GPU 将工作负载组织为**线程组（Thread Group）**的网格。每个线程组在单个多处理器上执行。为了充分利用 GPU，需要：
- 至少分配与多处理器数量相当的线程组
- 最好每个多处理器分配 2 个以上的线程组，以便在发生延迟时切换

每个线程组内的线程数量由 `[numthreads(X, Y, Z)]` 指定。NVIDIA 硬件使用 **warp（32 线程）**作为执行单位，ATI 使用 **wavefront（64 线程）**。建议线程组大小为 256（32 和 64 的公倍数）。

**`Dispatch` 方法：**

```cpp
void ID3D12GraphicsCommandList::Dispatch(
    UINT ThreadGroupCountX,
    UINT ThreadGroupCountY,
    UINT ThreadGroupCountZ
);
```

```
总线程数 = ThreadGroupCountX * ThreadGroupCountY * ThreadGroupCountZ * numthreads(X, Y, Z)
```

---

## 13.2 简单计算着色器

```hlsl
cbuffer cbSettings { };

Texture2D       gInputA;
Texture2D       gInputB;
RWTexture2D<float4> gOutput;

[numthreads(16, 16, 1)]
void CS(int3 dispatchThreadID : SV_DispatchThreadID) {
    gOutput[dispatchThreadID.xy] =
        gInputA[dispatchThreadID.xy] + gInputB[dispatchThreadID.xy];
}
```

| 属性/参数 | 说明 |
|----------|------|
| `[numthreads(x, y, z)]` | 定义线程组内线程的 3D 布局 |
| `SV_DispatchThreadID` | 线程在整个 Dispatch 网格中的全局 ID |

### 13.2.1 计算 PSO

```cpp
D3D12_COMPUTE_PIPELINE_STATE_DESC computePsoDesc = {};
computePsoDesc.pRootSignature = mComputeRootSignature.Get();
computePsoDesc.CS = {
    reinterpret_cast<BYTE*>(mShaders["wavesUpdateCS"]->GetBufferPointer()),
    mShaders["wavesUpdateCS"]->GetBufferSize()
};
computePsoDesc.Flags = D3D12_PIPELINE_STATE_FLAG_NONE;

ThrowIfFailed(md3dDevice->CreateComputePipelineState(
    &computePsoDesc, IID_PPV_ARGS(&mPSOs["wavesUpdate"])));
```

编译时使用 `cs_5_0` 目标：

```cpp
mShaders["wavesUpdateCS"] = d3dUtil::CompileShader(
    L"Shaders\WaveSim.hlsl", nullptr, "UpdateWavesCS", "cs_5_0");
```

---

## 13.3 数据输入和输出资源

### 13.3.1 纹理输入

通过 SRV 绑定纹理到计算着色器：

```hlsl
Texture2D gInputA;
Texture2D gInputB;
```

```cpp
cmdList->SetComputeRootDescriptorTable(1, mSrvA);
cmdList->SetComputeRootDescriptorTable(2, mSrvB);
```

### 13.3.2 纹理输出和无序访问视图（UAV）

输出资源使用 `RWTexture2D`（Read-Write Texture）：

```hlsl
RWTexture2D<float4> gOutput;
```

UAV 创建需要在资源创建时指定 `D3D12_RESOURCE_FLAG_ALLOW_UNORDERED_ACCESS`：

```cpp
D3D12_RESOURCE_DESC texDesc = {};
texDesc.Dimension = D3D12_RESOURCE_DIMENSION_TEXTURE2D;
texDesc.Width = mWidth;
texDesc.Height = mHeight;
texDesc.DepthOrArraySize = 1;
texDesc.MipLevels = 1;
texDesc.Format = DXGI_FORMAT_R8G8B8A8_UNORM;
texDesc.Flags = D3D12_RESOURCE_FLAG_ALLOW_UNORDERED_ACCESS;

ThrowIfFailed(md3dDevice->CreateCommittedResource(
    &CD3DX12_HEAP_PROPERTIES(D3D12_HEAP_TYPE_DEFAULT),
    D3D12_HEAP_FLAG_NONE, &texDesc,
    D3D12_RESOURCE_STATE_COMMON, nullptr,
    IID_PPV_ARGS(&mBlurMap0)));

// 创建 SRV 和 UAV
D3D12_SHADER_RESOURCE_VIEW_DESC srvDesc = {};
srvDesc.Format = mFormat;
srvDesc.ViewDimension = D3D12_SRV_DIMENSION_TEXTURE2D;
srvDesc.Texture2D.MipLevels = 1;

D3D12_UNORDERED_ACCESS_VIEW_DESC uavDesc = {};
uavDesc.Format = mFormat;
uavDesc.ViewDimension = D3D12_UAV_DIMENSION_TEXTURE2D;

md3dDevice->CreateShaderResourceView(mBlurMap0.Get(), &srvDesc, mBlur0CpuSrv);
md3dDevice->CreateUnorderedAccessView(mBlurMap0.Get(), nullptr, &uavDesc, mBlur0CpuUav);
```

**资源状态转换**：UAV 和 SRV 不能同时绑定到同一资源。使用资源屏障进行状态切换。

### 13.3.3 索引和采样纹理

计算着色器中可以使用整数索引或 `SampleLevel` 采样：

```hlsl
// 整数索引
float4 color = gInput[int2(x, y)];

// SampleLevel（需要显式指定 mipmap 级别）
float2 texCoord = float2(x, y) / 512.0f;
float4 color = gInput.SampleLevel(samPoint, texCoord, 0.0f);
```

越界读取返回 0，越界写入无操作。

### 13.3.4 结构化缓冲区

```hlsl
struct Data {
    float3 v1;
    float2 v2;
};

StructuredBuffer<Data>       gInputA : register(t0);
StructuredBuffer<Data>       gInputB : register(t1);
RWStructuredBuffer<Data>     gOutput : register(u0);
```

创建 UAV 结构化缓冲区：

```cpp
ThrowIfFailed(md3dDevice->CreateCommittedResource(
    &CD3DX12_HEAP_PROPERTIES(D3D12_HEAP_TYPE_DEFAULT),
    D3D12_HEAP_FLAG_NONE,
    &CD3DX12_RESOURCE_DESC::Buffer(byteSize,
        D3D12_RESOURCE_FLAG_ALLOW_UNORDERED_ACCESS),
    D3D12_RESOURCE_STATE_UNORDERED_ACCESS,
    nullptr, IID_PPV_ARGS(&mOutputBuffer)));
```

**根签名绑定（根描述符方式，无需描述符堆）**：

```cpp
CD3DX12_ROOT_PARAMETER slotRootParameter[3];
slotRootParameter[0].InitAsShaderResourceView(0);    // SRV to buffer
slotRootParameter[1].InitAsShaderResourceView(1);    // SRV to buffer
slotRootParameter[2].InitAsUnorderedAccessView(0);   // UAV to buffer

// 绑定
mCommandList->SetComputeRootShaderResourceView(0, mInputBufferA->GetGPUVirtualAddress());
mCommandList->SetComputeRootShaderResourceView(1, mInputBufferB->GetGPUVirtualAddress());
mCommandList->SetComputeRootUnorderedAccessView(2, mOutputBuffer->GetGPUVirtualAddress());
mCommandList->Dispatch(1, 1, 1);
```

### 13.3.5 将结果复制到系统内存

创建回读（readback）缓冲区，使用 `CopyResource`：

```cpp
// 创建回读缓冲区
ThrowIfFailed(md3dDevice->CreateCommittedResource(
    &CD3DX12_HEAP_PROPERTIES(D3D12_HEAP_TYPE_READBACK),
    D3D12_HEAP_FLAG_NONE,
    &CD3DX12_RESOURCE_DESC::Buffer(byteSize),
    D3D12_RESOURCE_STATE_COPY_DEST,
    nullptr, IID_PPV_ARGS(&mReadBackBuffer)));

// 复制 GPU 结果到 CPU 可访问缓冲区
mCommandList->ResourceBarrier(1, &CD3DX12_RESOURCE_BARRIER::Transition(
    mOutputBuffer.Get(), D3D12_RESOURCE_STATE_COMMON,
    D3D12_RESOURCE_STATE_COPY_SOURCE));
mCommandList->CopyResource(mReadBackBuffer.Get(), mOutputBuffer.Get());

// 等待 GPU 完成
FlushCommandQueue();

// Map 读取数据
Data* mappedData = nullptr;
mReadBackBuffer->Map(0, nullptr, reinterpret_cast<void**>(&mappedData));
// ... 使用数据
mReadBackBuffer->Unmap(0, nullptr);
```

---

## 13.4 线程标识系统值

| 系统值 | 语义 | 说明 |
|--------|------|------|
| `SV_GroupID` | 线程组 ID | 线程组在 Dispatch 网格中的位置 `(0..Gx-1, 0..Gy-1, 0..Gz-1)` |
| `SV_GroupThreadID` | 组内线程 ID | 线程在组内的位置 `(0..X-1, 0..Y-1, 0..Z-1)` |
| `SV_DispatchThreadID` | 调度线程 ID | 全局唯一 ID：`groupID * ThreadGroupSize + groupThreadID` |
| `SV_GroupIndex` | 组内线性索引 | `z*X*Y + y*X + x` |

---

## 13.5 Append 和 Consume 缓冲区

当不关心处理顺序时，可以使用更高效的 Append/Consume 缓冲区：

```hlsl
ConsumeStructuredBuffer<Particle> gInput;
AppendStructuredBuffer<Particle>  gOutput;

[numthreads(16, 16, 1)]
void CS() {
    Particle p = gInput.Consume();
    p.Velocity += p.Acceleration * TimeStep;
    p.Position += p.Velocity * TimeStep;
    gOutput.Append(p);
}
```

- `Consume()`：每个线程消耗一个元素，不会重复
- `Append()`：追加到输出缓冲区（缓冲区必须预先分配足够空间）

---

## 13.6 共享内存和同步

线程组内的线程可以访问**共享内存（Shared Memory）**，访问速度接近硬件缓存。

```hlsl
groupshared float4 gCache[256];
```

最大共享内存为 **32KB**。为获得最佳性能，线程组使用的共享内存不应超过 **16KB**，否则一个多处理器上无法同时容纳两个线程组。

**同步指令**：

```hlsl
GroupMemoryBarrierWithGroupSync();
```

所有线程必须先执行到同步点，然后才能继续。用于确保所有线程完成数据加载到共享内存后再开始读取。

**使用共享内存优化纹理读取的示例**：

```hlsl
groupshared float4 gCache[N + 2 * gMaxBlurRadius];

[numthreads(N, 1, 1)]
void HorzBlurCS(int3 groupThreadID : SV_GroupThreadID,
                int3 dispatchThreadID : SV_DispatchThreadID)
{
    // 每个线程加载一个纹理值到共享内存
    if (groupThreadID.x < gBlurRadius) {
        int x = max(dispatchThreadID.x - gBlurRadius, 0);
        gCache[groupThreadID.x] = gInput[int2(x, dispatchThreadID.y)];
    }
    if (groupThreadID.x >= N - gBlurRadius) {
        int x = min(dispatchThreadID.x + gBlurRadius, gInput.Length.x - 1);
        gCache[groupThreadID.x + 2 * gBlurRadius] = gInput[int2(x, dispatchThreadID.y)];
    }
    gCache[groupThreadID.x + gBlurRadius] = gInput[min(dispatchThreadID.xy, gInput.Length.xy - 1)];

    // 等待所有线程完成加载
    GroupMemoryBarrierWithGroupSync();

    // 现在可以安全地从共享内存读取
    float4 blurColor = float4(0, 0, 0, 0);
    for (int i = -gBlurRadius; i <= gBlurRadius; ++i) {
        int k = groupThreadID.x + gBlurRadius + i;
        blurColor += weights[i + gBlurRadius] * gCache[k];
    }
    gOutput[dispatchThreadID.xy] = blurColor;
}
```

---

## 13.7 模糊演示

### 13.7.1 模糊理论

模糊算法：对每个像素，计算以其为中心的 `m×n` 像素矩阵的加权平均值。

**高斯模糊**使用高斯函数生成权重。关键特性是**可分离**：一个 2D 模糊可以分解为两个 1D 模糊（先水平后垂直）。

对于 `9×9` 的 2D 模糊需要 81 次采样，而分离为两个 1D 模糊只需 `9 + 9 = 18` 次。

### 13.7.2 渲染到纹理

将场景渲染到离屏纹理，然后对纹理进行模糊处理，最后将模糊结果绘制到全屏四边形上。

或者直接渲染到后台缓冲区，然后使用 `CopyResource` 复制到模糊纹理：

```cpp
// 将后台缓冲区复制到 BlurMap0
cmdList->CopyResource(mBlurMap0.Get(), currentBackBuffer);
```

### 13.7.3 模糊实现

需要两个纹理 A 和 B，交替作为输入和输出：

```
Pass 1 (水平):  A(SRV) -> B(UAV)   // BlurH(A) = B
Pass 2 (垂直):  B(SRV) -> A(UAV)   // BlurV(B) = A (最终结果)
```

 Dispatch 数量计算：

```cpp
// 水平模糊：每行一个线程组，每组 256 线程
UINT numGroupsX = (UINT)ceilf(mWidth / 256.0f);
cmdList->Dispatch(numGroupsX, mHeight, 1);

// 垂直模糊：每列一个线程组
UINT numGroupsY = (UINT)ceilf(mHeight / 256.0f);
cmdList->Dispatch(mWidth, numGroupsY, 1);
```

每次 Dispatch 后需要资源屏障切换 UAV/SRV 状态。

### 13.7.4 完整模糊着色器

```hlsl
cbuffer cbSettings : register(b0) {
    int gBlurRadius;
    float w0, w1, w2, w3, w4, w5, w6, w7, w8, w9, w10;
};

Texture2D gInput : register(t0);
RWTexture2D<float4> gOutput : register(u0);

#define N 256
#define CacheSize (N + 2 * gMaxBlurRadius)
groupshared float4 gCache[CacheSize];

[numthreads(N, 1, 1)]
void HorzBlurCS(int3 groupThreadID : SV_GroupThreadID,
                int3 dispatchThreadID : SV_DispatchThreadID)
{
    float weights[11] = { w0, w1, w2, w3, w4, w5, w6, w7, w8, w9, w10 };

    // 加载到共享内存
    if (groupThreadID.x < gBlurRadius) {
        int x = max(dispatchThreadID.x - gBlurRadius, 0);
        gCache[groupThreadID.x] = gInput[int2(x, dispatchThreadID.y)];
    }
    if (groupThreadID.x >= N - gBlurRadius) {
        int x = min(dispatchThreadID.x + gBlurRadius, gInput.Length.x - 1);
        gCache[groupThreadID.x + 2 * gBlurRadius] = gInput[int2(x, dispatchThreadID.y)];
    }
    gCache[groupThreadID.x + gBlurRadius] = gInput[min(dispatchThreadID.xy, gInput.Length.xy - 1)];

    GroupMemoryBarrierWithGroupSync();

    // 模糊计算
    float4 blurColor = float4(0, 0, 0, 0);
    for (int i = -gBlurRadius; i <= gBlurRadius; ++i) {
        int k = groupThreadID.x + gBlurRadius + i;
        blurColor += weights[i + gBlurRadius] * gCache[k];
    }
    gOutput[dispatchThreadID.xy] = blurColor;
}

[numthreads(1, N, 1)]
void VertBlurCS(int3 groupThreadID : SV_GroupThreadID,
                int3 dispatchThreadID : SV_DispatchThreadID)
{
    // ... 类似水平模糊，但沿垂直方向
}
```
