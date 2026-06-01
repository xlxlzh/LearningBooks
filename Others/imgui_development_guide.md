# Dear ImGui 开发完全指南

> Dear ImGui（简称 ImGui）是由 Omar Cornut 开发的即时模式图形用户界面库，专为游戏引擎、3D 应用和工具开发而设计。它以极简的 API、出色的性能和极高的可移植性著称。

---

## 目录

1. [ImGui 简介](#1-imgui-简介)
2. [核心概念：即时模式 vs 保留模式](#2-核心概念即时模式-vs-保留模式)
3. [环境搭建与集成](#3-环境搭建与集成)
4. [基本程序框架](#4-基本程序框架)
5. [窗口与布局](#5-窗口与布局)
6. [基础控件详解](#6-基础控件详解)
7. [高级控件](#7-高级控件)
8. [绘图与自定义渲染](#8-绘图与自定义渲染)
9. [样式与主题](#9-样式与主题)
10. [高级功能](#10-高级功能)
11. [完整示例程序](#11-完整示例程序)
12. [最佳实践与性能优化](#12-最佳实践与性能优化)

---

## 1. ImGui 简介

### 1.1 什么是 ImGui

Dear ImGui 是一个**即时模式（Immediate Mode）**的 GUI 库。与传统 GUI 框架（如 Qt、WPF、WinForms）不同，ImGui 不存在持久的 UI 对象树。每一帧，你描述你想显示什么，ImGui 立即处理输入并渲染结果。

这个名字中的 "Dear" 并非缩写，而是作者 Omar Cornut 为表达对用户的尊重而添加的前缀。在很长一段时间里，人们简单地称它为 "ImGui"，但为了避免与 "Immediate Mode GUI" 这一通用概念混淆，作者建议在使用这个具体库时使用 "Dear ImGui" 或 "ImGui" 的称呼。

### 1.2 主要特性

| 特性 | 说明 |
|------|------|
| **即时模式** | 无持久对象，代码即 UI 描述 |
| **极简 API** | 仅需包含头文件即可使用 |
| **零依赖** | 核心库不依赖任何外部库 |
| **可移植** | 支持 Windows、macOS、Linux、移动端等 |
| **多后端** | 支持 DirectX、OpenGL、Vulkan、Metal 等 |
| **自包含** | 核心约 20,000 行代码，单一编译单元 |
| **Docking** | 支持窗口停靠（ docking 分支） |
| **多视口** | 窗口可脱离主窗口（ docking 分支） |

### 1.3 适用场景

- 游戏内调试工具和编辑器
- 3D 建模和渲染工具的 UI
- 实时可视化面板
- 原型开发工具
- 引擎和中间件的配置界面

### 1.4 不适用场景

- 需要复杂数据绑定的业务应用
- 需要持久化 UI 状态的复杂界面
- 需要原生操作系统外观的应用

---

## 2. 核心概念：即时模式 vs 保留模式

要真正理解 ImGui，必须先理解它与传统 GUI 框架在设计哲学上的根本差异。这个差异不是实现细节的不同，而是整个架构范式的不同。

### 2.1 保留模式（Retained Mode）

传统 GUI 框架（Qt、MFC、WPF）使用保留模式。这种模式的核心理念是：**UI 是一棵持久存在的对象树**。你的代码创建按钮、标签、输入框等对象，将它们组织成层次结构，然后这些对象一直存在于内存中，直到你显式删除它们。

在保留模式下，界面状态的管理是双向的：当你修改数据时，需要通知 UI 对象更新显示；当用户在界面上操作时，UI 对象通过回调函数通知你的代码。这意味着你必须维护两套状态——应用的数据状态和 UI 的显示状态——并保持它们同步。

```cpp
// 保留模式：先创建对象，再修改属性
QPushButton* button = new QPushButton("Click Me", parent);
button->setEnabled(true);
button->connect(clicked, &onClick);  // 注册回调

// 对象一直存在，直到显式删除
```

**保留模式的特点**：
- UI 作为对象树持久存在
- 通过回调函数响应事件
- 需要管理对象生命周期
- 数据绑定通常通过观察者模式

保留模式的优势在于它可以实现非常复杂的交互和自动化的状态管理。但它的代价也同样明显：你需要学习一整套对象模型、信号槽机制或数据绑定框架，调试时需要同时追踪应用状态和 UI 状态两套逻辑。

### 2.2 即时模式（Immediate Mode）

ImGui 使用即时模式。这种模式的核心理念是：**没有持久的 UI 对象，你的代码本身就是 UI 的描述**。每一帧，你调用 ImGui 的 API 来描述你希望这一帧显示什么。ImGui 不会创建持久的按钮对象或标签对象，它只是根据你的描述，处理输入、计算布局、生成绘制命令。

这意味着界面的状态完全由你的应用数据决定。如果你希望按钮显示为禁用状态，你只需要在调用 `Button()` 之前检查条件，而不是去修改一个按钮对象的 `enabled` 属性。如果你希望文本显示某个数值，你只需要把变量传给 `Text()`，而不需要维护一个标签对象并在变量变化时更新它。

```cpp
// 即时模式：每帧重新描述 UI
if (ImGui::Button("Click Me")) {
    // 按钮被点击，立即处理
    DoSomething();
}

// 没有持久的 button 对象
// 下一帧需要再次调用 Button() 来显示按钮
```

**即时模式的特点**：
- 每帧重新构建 UI 描述
- 无持久对象，无生命周期管理
- 返回值直接表示交互结果
- 应用代码直接控制所有状态

即时模式最大的优势在于简洁和透明。界面的状态就是你的数据的状态，不存在两套状态不同步的问题。交互的处理是直接的、同步的——按钮被点击了，它的返回值告诉你了，你立即处理，不需要注册回调函数。

但即时模式也有其局限性。因为它每帧都重新构建整个 UI 描述，对于极其复杂的界面，这可能会带来性能开销（尽管 ImGui 的实现非常高效，通常这不是问题）。此外，即时模式不适合需要操作系统原生外观的应用，也不适合需要复杂数据绑定的场景。

### 2.3 核心差异对比

让我们通过一个具体的例子来感受这两种模式的差异。假设我们需要一个复选框来控制某个功能的启用状态：

```cpp
// ========== 保留模式：需要维护状态对象 ==========
class MyPanel {
    bool checkboxState = false;  // 数据
    QCheckBox* checkbox;          // UI 对象

public:
    void init() {
        checkbox = new QCheckBox("Enable");
        // 建立数据绑定
        connect(checkbox, &QCheckBox::toggled, [this](bool checked) {
            checkboxState = checked;  // 同步数据
        });
    }

    void setChecked(bool value) {
        checkboxState = value;
        checkbox->setChecked(value);  // 同步 UI
    }
};

// ========== 即时模式：数据即一切 ==========
bool checkboxState = false;  // 只有数据，没有 UI 对象

void DrawUI() {
    // 每帧直接读取和写入数据
    ImGui::Checkbox("Enable", &checkboxState);
    // 如果用户点击了 checkbox，checkboxState 已经被修改
}
```

在保留模式的例子中，我们需要维护 `checkboxState` 和 `checkbox` 两个实体，并在它们之间建立同步机制。初始化时需要创建对象并绑定信号，修改状态时需要同时更新数据和 UI。如果项目中有多处需要修改这个状态，每处都要确保同步。

在即时模式的例子中，我们只维护 `checkboxState` 这一个实体。UI 只是这个数据的可视化呈现——每一帧，`Checkbox()` 函数读取 `checkboxState` 的值来决定复选框的显示状态；如果用户点击了复选框，函数直接修改 `checkboxState` 的值。没有同步问题，因为根本不存在两套状态。

### 2.4 即时模式的工作流程

理解 ImGui 的帧工作流程对于正确使用它至关重要。以下是每帧执行的完整流程：

```
┌─────────────────┐
│   帧开始 (NewFrame)  │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ 处理输入事件    │  ← 更新鼠标、键盘、手柄状态
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ 构建 UI 描述    │  ← 你的代码调用 ImGui::Button() 等
│ (Begin/End 块)  │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ 结束帧 (Render) │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ 生成绘制命令    │  ← ImDrawList / ImDrawCmd
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ 后端执行绘制    │  ← 你的渲染代码执行实际的 GPU 绘制
└─────────────────┘
```

**关键理解**：ImGui 本身不绘制任何内容，它只生成**绘制命令列表（Draw Commands）**。你需要提供一个渲染后端来执行这些命令。

这个设计是 ImGui 高度可移植的关键。ImGui 核心库只关心如何将你的 UI 描述转换为几何数据（顶点、索引、绘制命令），而实际的 GPU 渲染由你提供的后端代码完成。这就是为什么同一个 ImGui 代码可以同时工作在 DirectX、OpenGL、Vulkan 等完全不同的渲染 API 上——ImGui 不直接调用任何图形 API。

---

## 3. 环境搭建与集成

### 3.1 获取 ImGui

ImGui 的源代码可以直接从 GitHub 获取。由于 ImGui 是一个**头文件加实现文件**形式的库（不是预编译的静态库或动态库），你需要将源文件直接添加到你的项目中编译。

```bash
# 克隆官方仓库
git clone https://github.com/ocornut/imgui.git

# 或者仅下载核心文件（只需要这些文件即可编译）
```

这里需要强调的是，ImGui 的 "安装" 过程就是将源代码文件复制到你的项目中。没有 `make install`，没有包管理器，没有复杂的依赖解析。这种极简的分发方式是 ImGui 设计哲学的体现。

### 3.2 核心文件

将以下文件加入你的项目：

```
imgui/
├── imgui.h              # 主头文件
├── imgui.cpp            # 核心实现
├── imgui_demo.cpp       # 演示窗口（强烈推荐包含）
├── imgui_draw.cpp       # 绘制相关
├── imgui_tables.cpp     # 表格支持
├── imgui_widgets.cpp    # 控件实现
├── imconfig.h           # 配置文件（可选）
│
├── backends/            # 平台/渲染后端
│   ├── imgui_impl_win32.cpp      # Windows 平台
│   ├── imgui_impl_win32.h
│   ├── imgui_impl_dx11.cpp       # DirectX 11
│   ├── imgui_impl_dx11.h
│   ├── imgui_impl_dx12.cpp       # DirectX 12
│   ├── imgui_impl_dx12.h
│   ├── imgui_impl_opengl3.cpp    # OpenGL 3
│   ├── imgui_impl_opengl3.h
│   ├── imgui_impl_glfw.cpp       # GLFW 平台
│   ├── imgui_impl_glfw.h
│   └── ...
│
└── misc/                # 杂项工具
    ├── cpp/
    │   └── imgui_stdlib.h      # std::string 支持
    └── fonts/
        └── ...
```

**关于后端文件的说明**：ImGui 将平台相关的代码（窗口创建、消息处理）和渲染相关的代码（调用 DirectX/OpenGL API 进行绘制）分离为独立的 "后端" 文件。这种分离使得你可以混合搭配——例如，使用 GLFW 处理窗口和输入，同时使用 DirectX 11 进行渲染。

`imgui_demo.cpp` 虽然看起来只是一个演示文件，但它实际上是最好的学习资源。这个文件包含了几乎所有 ImGui 特性的使用示例，而且是实际可运行的代码。强烈建议将其包含在你的项目中，这样你就可以通过 `ImGui::ShowDemoWindow()` 随时查看这些示例。

### 3.3 CMake 集成示例

```cmake
# CMakeLists.txt

# ImGui 核心库
set(IMGUI_DIR ${CMAKE_SOURCE_DIR}/third_party/imgui)

add_library(imgui STATIC
    ${IMGUI_DIR}/imgui.cpp
    ${IMGUI_DIR}/imgui_demo.cpp
    ${IMGUI_DIR}/imgui_draw.cpp
    ${IMGUI_DIR}/imgui_tables.cpp
    ${IMGUI_DIR}/imgui_widgets.cpp
)

target_include_directories(imgui PUBLIC ${IMGUI_DIR})

# 平台后端（以 Win32 + DX11 为例）
add_library(imgui_backend STATIC
    ${IMGUI_DIR}/backends/imgui_impl_win32.cpp
    ${IMGUI_DIR}/backends/imgui_impl_dx11.cpp
)

target_link_libraries(imgui_backend PUBLIC imgui)

# 你的主程序
add_executable(MyApp main.cpp)
target_link_libraries(MyApp PRIVATE imgui imgui_backend d3d11.lib)
```

注意这里我们把 ImGui 核心库和后端代码分别编译为两个静态库。这种分离是可选的，你可以把所有源文件直接添加到主目标中。分离的好处是如果你的项目中有多个可执行文件都需要使用 ImGui，它们可以共享同一个 ImGui 库。

### 3.4 常用后端组合

| 平台 | 窗口系统 | 渲染 API | 后端文件 |
|------|----------|----------|----------|
| Windows | Win32 API | DirectX 11 | `imgui_impl_win32.cpp` + `imgui_impl_dx11.cpp` |
| Windows | Win32 API | DirectX 12 | `imgui_impl_win32.cpp` + `imgui_impl_dx12.cpp` |
| 跨平台 | GLFW | OpenGL 3 | `imgui_impl_glfw.cpp` + `imgui_impl_opengl3.cpp` |
| 跨平台 | GLFW | Vulkan | `imgui_impl_glfw.cpp` + `imgui_impl_vulkan.cpp` |
| 跨平台 | SDL2 | OpenGL 3 | `imgui_impl_sdl2.cpp` + `imgui_impl_opengl3.cpp` |

选择后端组合时需要考虑两个因素：**平台后端**负责窗口创建和输入处理（通常选用你项目已经在使用的窗口库），**渲染后端**负责实际的 GPU 绘制（通常选用你项目已经在使用的图形 API）。理想情况下，你不需要为了 ImGui 引入额外的依赖——直接使用你项目已经在使用的平台和渲染 API 即可。

---

## 4. 基本程序框架

### 4.1 最小完整示例（DirectX 11 + Win32）

以下是一个可以直接编译运行的完整示例。虽然代码看起来很长，但实际上可以清晰地分为几个部分：窗口创建、ImGui 初始化、主循环、清理。理解了这个结构，你就可以把它适配到任何平台和渲染 API 上。

```cpp
#include <windows.h>
#include <d3d11.h>
#include <tchar.h>

#include "imgui.h"
#include "imgui_impl_win32.h"
#include "imgui_impl_dx11.h"

// 链接必要的库
#pragma comment(lib, "d3d11.lib")

// DirectX 全局对象
ID3D11Device* g_pd3dDevice = nullptr;
ID3D11DeviceContext* g_pd3dDeviceContext = nullptr;
IDXGISwapChain* g_pSwapChain = nullptr;
ID3D11RenderTargetView* g_mainRenderTargetView = nullptr;

// 前向声明
bool CreateDeviceD3D(HWND hWnd);
void CleanupDeviceD3D();
void CreateRenderTarget();
void CleanupRenderTarget();
LRESULT WINAPI WndProc(HWND hWnd, UINT msg, WPARAM wParam, LPARAM lParam);

int WINAPI WinMain(HINSTANCE hInstance, HINSTANCE, LPSTR, int nCmdShow)
{
    // ========== 1. 创建 Windows 窗口 ==========
    WNDCLASSEX wc = {};
    wc.cbSize = sizeof(WNDCLASSEX);
    wc.style = CS_CLASSDC;
    wc.lpfnWndProc = WndProc;
    wc.hInstance = hInstance;
    wc.lpszClassName = _T("ImGui Example");
    RegisterClassEx(&wc);

    HWND hwnd = CreateWindow(
        wc.lpszClassName,
        _T("Dear ImGui DirectX11 Example"),
        WS_OVERLAPPEDWINDOW,
        100, 100, 1280, 800,
        nullptr, nullptr, wc.hInstance, nullptr
    );

    if (!CreateDeviceD3D(hwnd)) {
        CleanupDeviceD3D();
        UnregisterClass(wc.lpszClassName, wc.hInstance);
        return 1;
    }

    ShowWindow(hwnd, nCmdShow);
    UpdateWindow(hwnd);

    // ========== 2. 初始化 ImGui ==========
    IMGUI_CHECKVERSION();
    ImGui::CreateContext();
    ImGuiIO& io = ImGui::GetIO();
    io.ConfigFlags |= ImGuiConfigFlags_NavEnableKeyboard;  // 启用键盘导航

    // 设置 ImGui 样式
    ImGui::StyleColorsDark();  // 使用暗色主题

    // 初始化平台后端和渲染后端
    ImGui_ImplWin32_Init(hwnd);
    ImGui_ImplDX11_Init(g_pd3dDevice, g_pd3dDeviceContext);

    // ========== 3. 主循环 ==========
    bool done = false;
    while (!done)
    {
        // 处理 Windows 消息
        MSG msg;
        while (PeekMessage(&msg, nullptr, 0U, 0U, PM_REMOVE))
        {
            TranslateMessage(&msg);
            DispatchMessage(&msg);
            if (msg.message == WM_QUIT)
                done = true;
        }
        if (done) break;

        // ========== 4. 开始 ImGui 帧 ==========
        ImGui_ImplDX11_NewFrame();
        ImGui_ImplWin32_NewFrame();
        ImGui::NewFrame();

        // ========== 5. 在这里构建你的 UI ==========
        {
            ImGui::Begin("Hello, ImGui!");  // 创建一个窗口
            ImGui::Text("This is some useful text.");
            ImGui::Text("Application average %.3f ms/frame (%.1f FPS)",
                        1000.0f / io.Framerate, io.Framerate);
            ImGui::End();
        }

        // ========== 6. 渲染 ==========
        ImGui::Render();

        // 清除后台缓冲区
        const float clear_color[4] = { 0.45f, 0.55f, 0.60f, 1.00f };
        g_pd3dDeviceContext->ClearRenderTargetView(
            g_mainRenderTargetView, clear_color
        );

        // 执行 ImGui 绘制命令
        ImGui_ImplDX11_RenderDrawData(ImGui::GetDrawData());

        // 呈现到屏幕
        g_pSwapChain->Present(1, 0);  // 1 = 启用垂直同步
    }

    // ========== 7. 清理 ==========
    ImGui_ImplDX11_Shutdown();
    ImGui_ImplWin32_Shutdown();
    ImGui::DestroyContext();

    CleanupDeviceD3D();
    DestroyWindow(hwnd);
    UnregisterClass(wc.lpszClassName, wc.hInstance);

    return 0;
}

// ========== DirectX 辅助函数 ==========
bool CreateDeviceD3D(HWND hWnd)
{
    DXGI_SWAP_CHAIN_DESC sd = {};
    sd.BufferCount = 2;
    sd.BufferDesc.Width = 0;
    sd.BufferDesc.Height = 0;
    sd.BufferDesc.Format = DXGI_FORMAT_R8G8B8A8_UNORM;
    sd.BufferDesc.RefreshRate.Numerator = 60;
    sd.BufferDesc.RefreshRate.Denominator = 1;
    sd.Flags = DXGI_SWAP_CHAIN_FLAG_FRAME_LATENCY_WAITABLE_OBJECT;
    sd.BufferUsage = DXGI_USAGE_RENDER_TARGET_OUTPUT;
    sd.OutputWindow = hWnd;
    sd.SampleDesc.Count = 1;
    sd.SampleDesc.Quality = 0;
    sd.Windowed = TRUE;
    sd.SwapEffect = DXGI_SWAP_EFFECT_FLIP_DISCARD;

    UINT createDeviceFlags = 0;
    D3D_FEATURE_LEVEL featureLevel;
    const D3D_FEATURE_LEVEL featureLevelArray[2] = {
        D3D_FEATURE_LEVEL_11_0, D3D_FEATURE_LEVEL_10_0
    };

    HRESULT res = D3D11CreateDeviceAndSwapChain(
        nullptr, D3D_DRIVER_TYPE_HARDWARE, nullptr,
        createDeviceFlags, featureLevelArray, 2,
        D3D11_SDK_VERSION, &sd, &g_pSwapChain,
        &g_pd3dDevice, &featureLevel, &g_pd3dDeviceContext
    );

    if (res == DXGI_ERROR_UNSUPPORTED)
        res = D3D11CreateDeviceAndSwapChain(
            nullptr, D3D_DRIVER_TYPE_WARP, nullptr,
            createDeviceFlags, featureLevelArray, 2,
            D3D11_SDK_VERSION, &sd, &g_pSwapChain,
            &g_pd3dDevice, &featureLevel, &g_pd3dDeviceContext
        );

    if (res != S_OK) return false;

    CreateRenderTarget();
    return true;
}

void CleanupDeviceD3D()
{
    CleanupRenderTarget();
    if (g_pSwapChain) { g_pSwapChain->Release(); g_pSwapChain = nullptr; }
    if (g_pd3dDeviceContext) { g_pd3dDeviceContext->Release(); g_pd3dDeviceContext = nullptr; }
    if (g_pd3dDevice) { g_pd3dDevice->Release(); g_pd3dDevice = nullptr; }
}

void CreateRenderTarget()
{
    ID3D11Texture2D* pBackBuffer;
    g_pSwapChain->GetBuffer(0, IID_PPV_ARGS(&pBackBuffer));
    g_pd3dDevice->CreateRenderTargetView(pBackBuffer, nullptr, &g_mainRenderTargetView);
    pBackBuffer->Release();
}

void CleanupRenderTarget()
{
    if (g_mainRenderTargetView) {
        g_mainRenderTargetView->Release();
        g_mainRenderTargetView = nullptr;
    }
}

LRESULT WINAPI WndProc(HWND hWnd, UINT msg, WPARAM wParam, LPARAM lParam)
{
    // 将消息传递给 ImGui
    if (ImGui_ImplWin32_WndProcHandler(hWnd, msg, wParam, lParam))
        return true;

    switch (msg)
    {
    case WM_SIZE:
        if (g_pd3dDevice != nullptr && wParam != SIZE_MINIMIZED)
        {
            CleanupRenderTarget();
            g_pSwapChain->ResizeBuffers(0, LOWORD(lParam), HIWORD(lParam),
                                        DXGI_FORMAT_UNKNOWN, 0);
            CreateRenderTarget();
        }
        return 0;
    case WM_DESTROY:
        PostQuitMessage(0);
        return 0;
    }
    return DefWindowProc(hWnd, msg, wParam, lParam);
}
```

### 4.2 初始化流程详解

ImGui 的初始化是一个分步骤的过程，每个步骤都有明确的目的。理解这些步骤有助于你在遇到问题时知道从哪里排查。

```cpp
// 步骤 1：检查版本兼容性
IMGUI_CHECKVERSION();
```

这个宏检查编译时和运行时的 ImGui 版本是否一致。如果你不小心混合了不同版本的 imgui.h 和 imgui.cpp，这里会触发断言。这是预防版本不匹配导致奇怪 Bug 的第一道防线。

```cpp
// 步骤 2：创建 ImGui 上下文（全局状态）
ImGui::CreateContext();
```

ImGui 使用一个全局的 "上下文"（Context）来存储所有内部状态。这个上下文包含字体数据、样式设置、窗口状态等所有运行时信息。在大多数应用中，你只需要一个上下文。但在某些高级场景下（比如你想在同一个进程中运行两个完全独立的 ImGui 实例），你可以创建多个上下文并用 `ImGui::SetCurrentContext()` 切换。

```cpp
// 步骤 3：获取 IO 对象并配置
ImGuiIO& io = ImGui::GetIO();
io.ConfigFlags |= ImGuiConfigFlags_NavEnableKeyboard;
// 可选配置：
// io.ConfigFlags |= ImGuiConfigFlags_DockingEnable;      // 启用停靠
// io.ConfigFlags |= ImGuiConfigFlags_ViewportsEnable;    // 启用多视口
```

`ImGuiIO` 是 ImGui 与应用之间的主要接口。它包含输入状态（鼠标位置、按键状态）、输出状态（绘制数据）、以及各种配置选项。`NavEnableKeyboard` 允许用户使用键盘（方向键、Tab 键等）在界面上导航，这对于无鼠标环境或无障碍访问很重要。

```cpp
// 步骤 4：设置样式
ImGui::StyleColorsDark();   // 暗色主题
// ImGui::StyleColorsLight(); // 亮色主题
// ImGui::StyleColorsClassic(); // 经典主题
```

样式系统定义了所有视觉参数——颜色、间距、圆角大小等。`StyleColorsDark()` 等函数会一次性设置好所有默认值。你也可以在初始化后手动修改 `ImGui::GetStyle()` 中的具体参数来进一步定制外观。

```cpp
// 步骤 5：初始化后端
ImGui_ImplWin32_Init(hwnd);                  // 平台后端
ImGui_ImplDX11_Init(device, deviceContext);  // 渲染后端
```

后端初始化将 ImGui 与你的平台（这里是 Windows）和渲染 API（这里是 DirectX 11）连接起来。平台后端负责告诉 ImGui 当前的鼠标位置、键盘状态等输入信息；渲染后端负责将 ImGui 生成的绘制数据转换为实际的 GPU 绘制调用。

```cpp
// 步骤 6：加载字体（可选，默认已有内建字体）
// io.Fonts->AddFontFromFileTTF("arial.ttf", 16.0f);
```

如果不加载自定义字体，ImGui 会使用内建的 Proggy Clean 字体（13px，位图字体）。这个字体只包含 ASCII 字符，如果你需要显示中文或其他非拉丁字符，必须加载支持这些字符的字体。

### 4.3 每帧执行流程

主循环中的 ImGui 相关代码遵循严格的顺序，这个顺序不能改变：

```cpp
// 1. 通知后端新帧开始
ImGui_ImplDX11_NewFrame();   // 渲染后端
ImGui_ImplWin32_NewFrame();  // 平台后端
```

`NewFrame()` 调用通知后端准备新的一帧。平台后端在这里会更新输入状态（读取当前鼠标位置、按键状态等），渲染后端会准备绘制资源。

```cpp
// 2. ImGui 内部初始化新帧
ImGui::NewFrame();
```

这个调用初始化 ImGui 内部的帧状态。它会重置绘制列表、更新动画状态（如按钮的高亮渐变）、处理导航状态等。在这之后，直到 `Render()` 之前，你可以随意调用各种 ImGui 控件函数。

```cpp
// 3. === 在这里构建你的 UI ===
//    所有 ImGui::Begin(), ImGui::Button() 等调用都放在这里
```

这是你的代码发挥作用的地方。所有 UI 构建代码都放在 `NewFrame()` 和 `Render()` 之间。你可以在这里创建窗口、绘制控件、处理交互。

```cpp
// 4. 完成帧构建，生成绘制数据
ImGui::Render();
```

`Render()` 处理所有的 UI 描述，计算最终布局，生成顶点缓冲区和绘制命令。在这之后，所有控件函数都不应该再被调用。

```cpp
// 5. 获取绘制数据并交给后端渲染
ImDrawData* draw_data = ImGui::GetDrawData();
ImGui_ImplDX11_RenderDrawData(draw_data);
```

`GetDrawData()` 返回一个 `ImDrawData` 结构，包含这一帧所有需要绘制的顶点、索引和命令。渲染后端接收这个数据并执行实际的 GPU 绘制。

### 4.4 清理流程

清理顺序与初始化顺序相反——先关闭依赖项，再关闭被依赖项：

```cpp
// 按相反顺序清理
ImGui_ImplDX11_Shutdown();   // 先关闭渲染后端
ImGui_ImplWin32_Shutdown();  // 再关闭平台后端
ImGui::DestroyContext();     // 最后销毁上下文
```

为什么是这个顺序？因为上下文持有对字体纹理等 GPU 资源的引用，而渲染后端负责释放这些 GPU 资源。如果先销毁上下文再关闭后端，可能会导致资源泄漏或访问已释放的内存。

---

## 5. 窗口与布局

窗口系统是 ImGui 中最基础也最核心的概念之一。理解窗口的工作方式、坐标系和 ID 机制，对于构建复杂的用户界面至关重要。ImGui 的窗口系统既是 UI 容器，也是状态管理单元——每个窗口都有自己的位置、大小、滚动位置和折叠状态，这些状态会自动持久化到 `.ini` 文件中。

本章将详细介绍窗口的创建与配置、布局系统的使用方式、以及 ImGui 中独特的 ID 系统。掌握这些内容后，你就可以构建从简单的浮动面板到复杂的多面板编辑器界面。

### 5.1 基础窗口

ImGui 中几乎所有的 UI 元素都需要在窗口内绘制。窗口是 ImGui 中最高级别的容器，提供独立的坐标系、裁剪区域和滚动功能。

```cpp
// 最基本的窗口
ImGui::Begin("Window Title");  // 开始一个窗口
ImGui::Text("Hello inside a window!");
ImGui::End();                  // 必须配对调用 End()
```

**关于 Begin() 的返回值**：`Begin()` 返回一个 `bool`，表示窗口当前是否可见（未被折叠、未关闭）。如果窗口被折叠为标题栏，或者在 `p_open` 模式下被用户关闭，返回值为 `false`。虽然即使返回 `false` 你仍然可以调用 `End()`（这是合法的），但通常你应该在返回 `true` 时才绘制窗口内容，以节省一点 CPU 时间。

**窗口标题作为唯一标识符**：`Begin()` 的第一个参数 `"Window Title"` 有两个作用：它既是窗口的显示标题，也是窗口的唯一标识符。ImGui 内部使用这个字符串来区分不同的窗口，保存它们的位置、大小等状态到 `.ini` 文件。如果你需要在不同地方创建标题相同但独立的窗口，可以使用 `##` 语法添加隐藏的标识部分：

```cpp
// 带关闭按钮的窗口
bool show_window = true;
if (ImGui::Begin("My Window", &show_window)) {
    ImGui::Text("Window content...");
}
ImGui::End();

// 如果用户点击了关闭按钮，show_window 会被设为 false
if (!show_window) {
    // 窗口被关闭了，可以在这里处理
}
```

**窗口 ID 系统的工作原理**：ImGui 内部使用一个 ID 栈来唯一标识每个控件。当你调用 `Begin("Window Title")` 时，这个标题字符串就被哈希为一个 32 位的 ID。这个 ID 不仅用于区分不同的窗口，还用于保存和恢复窗口的状态。这意味着如果你把窗口标题从 `"My Window"` 改为 `"My Window "`（多了一个空格），ImGui 会把它当作一个全新的窗口，丢失之前保存的位置和大小。

如果你需要在运行时动态改变窗口标题，同时保持其状态，可以使用 `##` 语法将显示标题和 ID 分开：

```cpp
// "My Window" 是显示给用户看的，"PersistentID" 是内部使用的 ID
ImGui::Begin("My Window##PersistentID");
```

这在需要根据状态改变标题时特别有用，例如 `"Scene View [Playing]##SceneView"`。

### 5.2 窗口标志（Window Flags）

窗口标志让你可以精确控制窗口的行为。这些标志通过位或（`|`）操作组合。合理使用标志可以创建从完全可交互的浮动面板到固定不动的 HUD 元素等各种界面。

```cpp
ImGuiWindowFlags flags = 0;

// 标题栏相关
flags |= ImGuiWindowFlags_NoTitleBar;       // 不显示标题栏
flags |= ImGuiWindowFlags_NoCollapse;       // 禁用折叠按钮

// 大小和位置
flags |= ImGuiWindowFlags_NoResize;         // 禁止调整大小
flags |= ImGuiWindowFlags_NoMove;           // 禁止移动
flags |= ImGuiWindowFlags_AlwaysAutoResize; // 自动根据内容调整大小
flags |= ImGuiWindowFlags_NoSavedSettings;  // 不保存到 .ini 文件

// 输入和焦点
flags |= ImGuiWindowFlags_NoMouseInputs;    // 忽略鼠标输入
flags |= ImGuiWindowFlags_NoNavInputs;      // 忽略导航输入
flags |= ImGuiWindowFlags_NoNavFocus;       // 不参与导航聚焦
flags |= ImGuiWindowFlags_NoBringToFrontOnFocus; // 聚焦时不移到最前

// 背景和边框
flags |= ImGuiWindowFlags_NoBackground;     // 不显示背景
flags |= ImGuiWindowFlags_NoBorder;         // 不显示边框

// 子窗口相关
flags |= ImGuiWindowFlags_HorizontalScrollbar; // 启用水平滚动条
flags |= ImGuiWindowFlags_AlwaysVerticalScrollbar; // 总是显示垂直滚动条
flags |= ImGuiWindowFlags_AlwaysHorizontalScrollbar; // 总是显示水平滚动条

// 菜单栏
flags |= ImGuiWindowFlags_MenuBar;          // 显示菜单栏

// 组合示例：固定位置和大小的工具窗口
ImGui::Begin("Tool Panel", nullptr,
    ImGuiWindowFlags_NoResize |
    ImGuiWindowFlags_NoMove |
    ImGuiWindowFlags_NoCollapse |
    ImGuiWindowFlags_NoTitleBar
);
```

**常用组合模式**：

- **浮动调试面板**：`NoCollapse | NoSavedSettings` ——用户可以调整大小和位置，但不能折叠，重启后恢复默认位置。
- **固定 HUD 元素**：`NoTitleBar | NoResize | NoMove | NoBackground | NoInputs` ——完全固定的覆盖层，只显示内容不接收输入。
- **自动大小弹窗**：`NoResize | AlwaysAutoResize | NoSavedSettings` ——根据内容自动调整大小，适合临时的信息展示。

### 5.3 窗口位置和大小

ImGui 会自动记住用户调整的窗口位置和大小，并保存到 `.ini` 文件中（除非指定了 `NoSavedSettings`）。但很多时候你需要为窗口设置初始位置和大小：

```cpp
// 设置初始位置和大小（只在窗口第一次出现时生效）
ImGui::SetNextWindowPos(ImVec2(100, 100), ImGuiCond_FirstUseEver);
ImGui::SetNextWindowSize(ImVec2(400, 300), ImGuiCond_FirstUseEver);
ImGui::Begin("Positioned Window");
// ...
ImGui::End();

// 条件参数说明：
// ImGuiCond_Always       - 每次都设置（覆盖用户操作）
// ImGuiCond_FirstUseEver - 仅在第一次使用时设置（默认值）
// ImGuiCond_Once         - 只设置一次
// ImGuiCond_Appearing    - 窗口正在出现时设置
```

**条件参数的含义**：这是 ImGui 中一个重要的概念。因为 ImGui 每帧都重新构建 UI，如果你每帧都调用 `SetNextWindowPos(pos, Always)`，用户就永远无法拖动窗口——你每帧都在覆盖他们的操作。使用 `FirstUseEver` 可以在窗口首次出现时设置初始值，之后让用户自由调整。使用 `Once` 则只在程序启动后的第一帧设置，适合某些需要动态计算初始位置的场景。

```cpp
// 设置窗口大小约束
ImGui::SetNextWindowSizeConstraints(
    ImVec2(200, 100),   // 最小大小
    ImVec2(800, 600)    // 最大大小
);

// 设置窗口背景透明度
ImGui::SetNextWindowBgAlpha(0.5f);  // 半透明背景
```

### 5.4 子窗口和滚动区域

当窗口内容超出其可见区域时，ImGui 会自动显示滚动条。但有时你需要更精细的控制——比如在窗口内创建一个独立的可滚动区域。

```cpp
ImGui::Begin("Main Window");

// 创建可滚动的子区域
ImGui::BeginChild("ScrollingRegion", ImVec2(0, 300), true);
// 最后一个参数 true 表示带边框

for (int i = 0; i < 100; i++) {
    ImGui::Text("Line %d: Some content here...", i);
}

ImGui::EndChild();

ImGui::End();
```

`BeginChild()` 创建一个逻辑上的子窗口。它有自己的滚动位置、裁剪区域和滚动条。参数中的 `"ScrollingRegion"` 是子窗口的 ID，用于区分不同的子窗口并保存它们各自的状态（如滚动位置）。第二个参数 `ImVec2(0, 300)` 指定大小——宽度为 0 表示使用父窗口的可用宽度，高度为 300 像素。

### 5.5 分组（Group）

分组（Group）是一个轻量级的布局工具，它将多个控件视为一个整体。这在某些需要获取一组控件的整体尺寸，或者需要对一组控件进行统一样式设置时非常有用。

```cpp
// 将多个控件视为一个整体
ImGui::BeginGroup();
ImGui::Text("Group Label");
ImGui::Button("Button 1");
ImGui::SameLine();
ImGui::Button("Button 2");
ImGui::EndGroup();

// 获取组的大小
ImVec2 group_size = ImGui::GetItemRectSize();
```

### 5.6 水平布局（SameLine）

默认情况下，ImGui 的控件是垂直排列的。`SameLine()` 允许你将下一个控件放在当前控件的右侧，实现水平布局。

```cpp
ImGui::Button("Button 1");

// 下一项与当前项在同一行
ImGui::SameLine();
ImGui::Button("Button 2");

// 指定偏移量
ImGui::SameLine(200);  // 从窗口左边缘偏移 200 像素
ImGui::Button("Button 3");

// 指定间距
ImGui::SameLine(0, 50);  // 默认偏移，间距 50 像素
ImGui::Button("Button 4");

// 对齐到右侧
float windowWidth = ImGui::GetWindowWidth();
float buttonWidth = 100.0f;
ImGui::SameLine(windowWidth - buttonWidth - ImGui::GetStyle().WindowPadding.x);
ImGui::Button("Right Aligned", ImVec2(buttonWidth, 0));
```

### 5.7 分隔线

分隔线（Separator）用于在视觉上将内容分组。它会在当前位置绘制一条水平线，横跨整个内容区域。

```cpp
ImGui::Button("Above");
ImGui::Separator();  // 水平分隔线
ImGui::Button("Below");

// 带文本的分隔线
ImGui::SeparatorText("Section Header");
```

### 5.8 间距和缩进

```cpp
// 垂直间距
ImGui::Spacing();        // 空一行
ImGui::Dummy(ImVec2(0, 20));  // 指定高度的空行

// 水平缩进
ImGui::Indent(20.0f);    // 增加缩进
ImGui::Text("Indented text");
ImGui::Unindent(20.0f);  // 恢复缩进

// 临时改变间距
ImGui::PushStyleVar(ImGuiStyleVar_ItemSpacing, ImVec2(10, 20));
// ... 绘制控件
ImGui::PopStyleVar();
```

---

## 6. 基础控件详解

ImGui 的控件 API 遵循一套统一的设计模式。理解这些通用模式后，学习具体控件就会事半功倍：

**返回值模式**：大多数交互控件返回 `bool` 表示是否发生了用户交互。例如 `Button()` 返回 `true` 表示被点击，`Checkbox()` 返回 `true` 表示值被修改，`InputText()` 在按 Enter 时返回 `true`。这个设计使得你可以把交互处理直接写在 `if` 语句中。

**直接修改模式**：编辑类控件（如 `SliderFloat`、`InputText`、`Checkbox`）直接修改传入的变量。你传入的是指针，用户操作后变量的值就已经变了。这与保留模式 GUI 中需要监听回调再手动更新的方式截然不同。

**标签与 ID**：几乎每个控件的第一个参数都是标签字符串。这个标签有两个作用：显示给用户看，以及作为控件的 ID。如果在同一窗口的同一层次放置两个标签相同的控件，它们的 ID 会冲突，导致交互异常。使用 `PushID`/`PopID` 或 `##` 语法可以解决。

**静态变量模式**：示例代码中大量使用 `static` 变量来保存控件状态。这是因为 ImGui 本身不保存你的数据——它只读取和修改你提供的变量。这些 `static` 变量在示例中代表你的应用数据。在实际项目中，你通常会传入类的成员变量或引擎的属性值。

### 6.1 文本显示

文本显示是 UI 中最基本的元素。ImGui 提供了多种文本显示函数，适用于不同的场景。

```cpp
// 普通文本
ImGui::Text("Hello, World!");

// 格式化文本（类似 printf）
int count = 42;
float value = 3.14159f;
ImGui::Text("Count: %d, Value: %.2f", count, value);

// 彩色文本
ImGui::TextColored(ImVec4(1.0f, 0.0f, 0.0f, 1.0f), "Red text");
ImGui::TextColored(ImVec4(0.0f, 1.0f, 0.0f, 1.0f), "Green text");
ImGui::TextColored(ImVec4(0.0f, 0.5f, 1.0f, 1.0f), "Blue text");

// 带失焦效果的文本
ImGui::TextDisabled("Disabled text (grayed out)");

// 带标签的文本（左侧标签，右侧值）
ImGui::LabelText("label", "Value is %d", 100);

// 多行文本（自动换行）
ImGui::TextWrapped("This is a very long text that will automatically wrap to the next line when it exceeds the available width of the window.");

// 项目标签（类似表单的左对齐标签）
ImGui::TextUnformatted("Bullet text");
ImGui::Bullet();  // 项目符号
ImGui::Text("Item with bullet");

// 超链接样式的文本
ImGui::TextLink("Click here");  // ImGui 1.90+

// 带颜色标记的文本
ImGui::Text("Normal text [");
ImGui::SameLine(0, 0);
ImGui::TextColored(ImVec4(1, 1, 0, 1), "yellow");
ImGui::SameLine(0, 0);
ImGui::Text("] inline");
```

**文本显示函数的选择指南**：
- `Text()` 是最常用的，支持 printf 格式化，适合显示动态数据。但要注意：每帧都格式化字符串有一定开销，对于性能敏感的代码，考虑缓存格式化结果或使用 `TextUnformatted()`。
- `TextColored()` 适合高亮关键信息，如错误消息、警告、重要数值。
- `TextDisabled()` 适合显示当前不可用的选项或次级信息。
- `TextWrapped()` 适合长段落说明文字，会自动适应窗口宽度换行。
- `LabelText()` 适合表单式的 "标签: 值" 布局，它会在左侧显示标签，右侧显示值，且自动对齐。
- `TextUnformatted()` 不执行格式化解析，比 `Text()` 略快，适合显示已知不含格式说明符的字符串。

### 6.2 按钮

按钮是交互式 UI 中最基础的控件。在 ImGui 中，按钮的使用极其直观：调用 `Button()`，如果返回 `true`，说明用户在这一帧点击了按钮。

```cpp
// 普通按钮
if (ImGui::Button("Click Me")) {
    // 按钮被点击时执行
    printf("Button clicked!\n");
}

// 指定大小的按钮
if (ImGui::Button("Big Button", ImVec2(200, 50))) {
    // ...
}

// 小型按钮
if (ImGui::SmallButton("Small")) {
    // ...
}

// 可复选按钮（Toggle 效果）
static bool toggle = false;
if (ImGui::Button("Toggle", ImVec2(0, 0))) {
    toggle = !toggle;
}
if (toggle) {
    ImGui::SameLine();
    ImGui::Text("ON");
}

// 箭头按钮
if (ImGui::ArrowButton("left", ImGuiDir_Left)) { /* ... */ }
ImGui::SameLine();
if (ImGui::ArrowButton("right", ImGuiDir_Right)) { /* ... */ }

// 重复按钮（按住时持续触发）
ImGui::PushButtonRepeat(true);
static int counter = 0;
if (ImGui::ArrowButton("up", ImGuiDir_Up)) {
    counter++;
}
ImGui::SameLine();
if (ImGui::ArrowButton("down", ImGuiDir_Down)) {
    counter--;
}
ImGui::PopButtonRepeat();
ImGui::SameLine();
ImGui::Text("Counter: %d", counter);

// 禁用按钮
bool can_click = false;
if (!can_click) ImGui::BeginDisabled();
if (ImGui::Button("Disabled Button")) {
    // 不会触发，因为按钮被禁用
}
if (!can_click) ImGui::EndDisabled();

// 不可见按钮（用于占位或交互区域）
if (ImGui::InvisibleButton("invisible", ImVec2(100, 100))) {
    // 点击了不可见按钮区域
}
```

**按钮交互模型详解**：`Button()` 的返回值遵循一个重要的规则：它只在鼠标**按下并释放**的同一帧返回 `true`。这意味着你不能用 `Button()` 来检测 "鼠标正按住按钮" 的状态——如果需要这种行为，应该使用 `InvisibleButton()` 配合 `IsItemActive()`。

`BeginDisabled()`/`EndDisabled()` 是一对非常有用的函数。它们不仅会让按钮变灰，还会阻止所有交互事件。这比简单地让按钮返回 `false` 更彻底——禁用的控件甚至不会接收焦点。注意这对函数会影响它们之间**所有**的控件，所以一定要确保 `Begin` 和 `End` 配对正确。

`InvisibleButton()` 是构建自定义交互组件的基础。它创建一个指定大小的交互区域，但不绘制任何可见内容。配合 `IsItemHovered()`、`IsItemActive()` 和 `GetWindowDrawList()`，你可以构建任何自定义的可交互可视化组件。

### 6.3 复选框

复选框（Checkbox）是最简单的布尔值编辑器。它的工作方式非常直接：传入一个 `bool*` 指针，函数会根据当前值显示勾选或未勾选状态；如果用户点击了复选框，函数会直接修改这个 `bool` 变量的值。

```cpp
static bool checked = false;
if (ImGui::Checkbox("Enable Feature", &checked)) {
    // checked 的值已经被修改
    // 可以在这里响应状态变化
    if (checked) {
        printf("Feature enabled!\n");
    }
}

// 多个复选框
static bool flags[4] = { false, true, false, false };
ImGui::CheckboxFlags("Flag A", &flags, 1 << 0);
ImGui::CheckboxFlags("Flag B", &flags, 1 << 1);
ImGui::CheckboxFlags("Flag C", &flags, 1 << 2);
```

### 6.4 单选按钮

单选按钮（RadioButton）用于在一组互斥选项中选择一个。它的 API 设计稍微有些不同：`RadioButton(label, active)` 形式中，`active` 是一个布尔值，表示当前选项是否被选中。如果用户点击了这个单选按钮，你应该将对应的变量设置为这个选项的值。

```cpp
static int selected = 0;

// 单选按钮组
if (ImGui::RadioButton("Option A", selected == 0)) selected = 0;
if (ImGui::RadioButton("Option B", selected == 1)) selected = 1;
if (ImGui::RadioButton("Option C", selected == 2)) selected = 2;

// 更简洁的方式
const char* options[] = { "Option A", "Option B", "Option C" };
for (int i = 0; i < IM_ARRAYSIZE(options); i++) {
    if (ImGui::RadioButton(options[i], selected == i))
        selected = i;
}

// 内联显示
ImGui::RadioButton("A", &selected, 0); ImGui::SameLine();
ImGui::RadioButton("B", &selected, 1); ImGui::SameLine();
ImGui::RadioButton("C", &selected, 2);
```

### 6.5 滑块（Slider）

滑块（Slider）适合在已知范围内选择一个数值。用户可以通过拖动滑块柄来快速浏览整个范围，也可以点击滑块条来跳转到特定位置。滑块的精度受限于像素分辨率，所以如果需要精确到小数点后多位的数值，可能更适合使用 Drag 控件。

```cpp
// 整数滑块
static int int_value = 50;
ImGui::SliderInt("Integer", &int_value, 0, 100);
ImGui::SliderInt("Integer (with format)", &int_value, 0, 100, "%d%%");

// 浮点数滑块
static float float_value = 0.5f;
ImGui::SliderFloat("Float", &float_value, 0.0f, 1.0f);
ImGui::SliderFloat("Float (2 decimals)", &float_value, 0.0f, 1.0f, "%.2f");

// 对数滑块（适合大范围的值）
static float log_value = 0.0f;
ImGui::SliderFloat("Logarithmic", &log_value, -10.0f, 10.0f,
                    "%.4f", ImGuiSliderFlags_Logarithmic);

// 非线性滑块（需要更多精度时）
static float nonlinear = 0.5f;
ImGui::SliderFloat("Non-linear", &nonlinear, 0.0f, 1.0f,
                    "%.3f", ImGuiSliderFlags_None);

// 角度滑块
static float angle = 0.0f;
ImGui::SliderAngle("Angle", &angle);  // 以度为单位，内部存储为弧度

// 2D 滑块
static ImVec2 slider2d = ImVec2(0.5f, 0.5f);
ImGui::SliderFloat2("Position", &slider2d.x, 0.0f, 1.0f);

// 3D 滑块
static float vec3[3] = { 0.0f, 0.0f, 0.0f };
ImGui::SliderFloat3("Vector3", vec3, -1.0f, 1.0f);

// 4D 滑块（如颜色）
static float vec4[4] = { 1.0f, 1.0f, 1.0f, 1.0f };
ImGui::SliderFloat4("Vector4", vec4, 0.0f, 1.0f);

// 垂直滑块
static float vslider_value = 0.5f;
ImGui::VSliderFloat("Vertical", ImVec2(30, 100), &vslider_value, 0.0f, 1.0f);

// 带范围的滑块（双滑块）
static float range_min = 0.2f, range_max = 0.8f;
ImGui::DragFloatRange2("Range", &range_min, &range_max, 0.01f, 0.0f, 1.0f);
```

**Slider vs Drag：如何选择？**

这是 ImGui 新手最常遇到的问题之一。两者都可以编辑数值，但适用场景不同：

- **Slider** 适合**已知范围**的场景。用户一眼就能看到当前值在范围中的位置，适合亮度、音量、透明度等设置。Slider 的缺点是精度受限于像素——一个 200 像素宽的滑块，范围 0-1，理论精度只有 0.005。

- **Drag** 适合**需要精确控制**或**范围不确定**的场景。用户可以通过拖拽速度来控制精度——慢速拖拽是小步调整，快速拖拽是大步调整。Drag 还允许用户直接输入数值（双击即可编辑）。对于位置坐标、旋转角度等需要精确到小数点后多位的值，Drag 是更好的选择。

- **Input**（`InputFloat` 等）适合**需要精确输入特定值**的场景。用户可以直接键入数值，不受拖拽精度的限制。但它没有 Slider 那样的范围直观性。

在实际项目中，一个常见的模式是并排使用 Slider 和 Input：Slider 用于快速浏览，Input 用于精确输入。

### 6.6 拖拽输入（Drag）

Drag 控件提供比 Slider 更精确的数值输入方式。用户可以通过水平拖动来微调数值，也可以直接输入。`speed` 参数控制拖动的灵敏度——值越大，同样的鼠标移动会产生越大的数值变化。

```cpp
// Drag 控件比 Slider 更精确，适合需要精确输入的数值

static float drag_value = 0.0f;
ImGui::DragFloat("Drag Float", &drag_value);
ImGui::DragFloat("Drag (0-1)", &drag_value, 0.01f, 0.0f, 1.0f);
ImGui::DragFloat("Drag (slow)", &drag_value, 0.001f);  // 更慢的拖拽速度

// 整数拖拽
static int drag_int = 0;
ImGui::DragInt("Drag Int", &drag_int);

// 2D 拖拽
static ImVec2 drag2(0, 0);
ImGui::DragFloat2("Drag 2D", &drag2.x, 0.1f);

// 带速率和范围的拖拽
static float speed = 1.0f;
ImGui::DragFloat("Speed", &speed, 0.1f, 0.0f, 100.0f, "%.1f m/s");

// 幂函数拖拽（在低端更精确）
static float power_val = 0.5f;
ImGui::DragFloat("Power", &power_val, 0.01f, 0.0f, 1.0f,
                 "%.3f", ImGuiSliderFlags_None);
```

### 6.7 文本输入

文本输入是交互式 UI 中最复杂的控件之一。ImGui 的 `InputText` 提供了丰富的功能，包括多行输入、密码模式、输入验证、自动补全等。

```cpp
// 基础文本输入
static char text[128] = "Hello";
ImGui::InputText("Label", text, IM_ARRAYSIZE(text));

// 带标志的文本输入
static char password[64] = "";
ImGui::InputText("Password", password, IM_ARRAYSIZE(password),
    ImGuiInputTextFlags_Password);

// 只读文本
static char readonly[64] = "Cannot edit this";
ImGui::InputText("Read Only", readonly, IM_ARRAYSIZE(readonly),
    ImGuiInputTextFlags_ReadOnly);

// 自动选中全部
static char autoselect[64] = "Auto selected";
ImGui::InputText("Auto Select", autoselect, IM_ARRAYSIZE(autoselect),
    ImGuiInputTextFlags_AutoSelectAll);

// 按 Enter 确认
static char enter_text[64] = "";
if (ImGui::InputText("Press Enter", enter_text, IM_ARRAYSIZE(enter_text),
    ImGuiInputTextFlags_EnterReturnsTrue)) {
    printf("User pressed Enter: %s\n", enter_text);
}

// 多行文本输入
static char multiline[1024] = "Line 1\nLine 2\nLine 3";
ImGui::InputTextMultiline("Multiline", multiline, IM_ARRAYSIZE(multiline),
    ImVec2(-1, ImGui::GetTextLineHeight() * 8));  // 宽度-1 = 填充可用空间

// 带回调的文本输入（用于输入验证等）
static char validated[64] = "";
ImGui::InputText("Validated", validated, IM_ARRAYSIZE(validated),
    ImGuiInputTextFlags_CallbackCharFilter,
    [](ImGuiInputTextCallbackData* data) -> int {
        // 只允许数字输入
        if (data->EventChar < '0' || data->EventChar > '9')
            return 1;  // 拒绝这个字符
        return 0;
    });

// 使用 std::string（需要 imgui_stdlib.h）
#include "misc/cpp/imgui_stdlib.h"
static std::string str_value = "std::string";
ImGui::InputText("std::string", &str_value);
```

**回调机制详解**：`InputText` 的回调系统是其最强大的功能之一。回调函数在特定事件发生时由 ImGui 调用，允许你对输入过程进行精细控制。回调函数接收一个 `ImGuiInputTextCallbackData*` 参数，通过这个结构你可以：

- `data->Buf` / `data->BufTextLen`：访问和修改当前缓冲区内容
- `data->CursorPos`：获取或设置光标位置
- `data->SelectionStart` / `data->SelectionEnd`：获取或设置选区
- `data->EventChar`：获取触发回调的字符（用于 `CallbackCharFilter`）
- `data->EventKey`：获取触发回调的按键（用于 `CallbackHistory`、`CallbackCompletion`）
- `data->InsertChars()` / `data->DeleteChars()`：安全地修改缓冲区

常见的回调使用场景包括：自动补全（Tab 键）、命令历史（上下箭头）、实时搜索建议、输入验证、模板插入等。`imgui_demo.cpp` 中有一个完整的自动补全示例值得参考。

**关于 `std::string` 支持**：ImGui 核心库只接受 `char*` 缓冲区，这是为了兼容 C 和避免对 C++ 标准库的依赖。如果你需要使用 `std::string`，可以包含 `misc/cpp/imgui_stdlib.h`，它提供了重载版本。在内部，这些重载会在必要时自动调整字符串大小以适应输入内容。

**InputText 常用标志**：

```cpp
ImGuiInputTextFlags_None                  // 默认
ImGuiInputTextFlags_CharsDecimal          // 只允许十进制字符
ImGuiInputTextFlags_CharsHexadecimal      // 只允许十六进制字符
ImGuiInputTextFlags_CharsUppercase        // 转为大写
ImGuiInputTextFlags_CharsNoBlank          // 不允许空格
ImGuiInputTextFlags_AutoSelectAll         // 激活时自动全选
ImGuiInputTextFlags_EnterReturnsTrue      // 按 Enter 返回 true
ImGuiInputTextFlags_CallbackCompletion    // 按 Tab 触发回调
ImGuiInputTextFlags_CallbackHistory       // 按上下箭头触发回调
ImGuiInputTextFlags_CallbackAlways        // 每帧都调用回调
ImGuiInputTextFlags_CallbackCharFilter    // 字符过滤回调
ImGuiInputTextFlags_AllowTabInput         // 允许 Tab 输入（多行）
ImGuiInputTextFlags_CtrlEnterForNewLine   // Ctrl+Enter 换行
ImGuiInputTextFlags_NoHorizontalScroll    // 禁用水平滚动
ImGuiInputTextFlags_AlwaysOverwrite       // 覆盖模式
ImGuiInputTextFlags_ReadOnly              // 只读
ImGuiInputTextFlags_Password              // 密码模式
ImGuiInputTextFlags_NoUndoRedo            // 禁用撤销/重做
```

### 6.8 数字输入

```cpp
// 整数输入
static int int_val = 0;
ImGui::InputInt("Integer", &int_val);
ImGui::InputInt("Integer (step=5)", &int_val, 5);  // 步长 5

// 浮点数输入
static float float_val = 0.0f;
ImGui::InputFloat("Float", &float_val);
ImGui::InputFloat("Float (step=0.1)", &float_val, 0.1f);
ImGui::InputFloat("Float (step=0.1, fast=1.0)", &float_val, 0.1f, 1.0f);

// 2D 输入
static ImVec2 input2(0, 0);
ImGui::InputFloat2("Input 2D", &input2.x);

// 3D 输入
static float input3[3] = {0, 0, 0};
ImGui::InputFloat3("Input 3D", input3);

// 4D 输入
static float input4[4] = {0, 0, 0, 0};
ImGui::InputFloat4("Input 4D", input4);

// 双精度
static double double_val = 0.0;
ImGui::InputDouble("Double", &double_val);
```

### 6.9 颜色编辑

颜色编辑控件是 ImGui 中最精美的控件之一。它提供了多种编辑方式：RGB/HSV 输入框、十六进制输入、色相轮、饱和度-亮度方框等。所有编辑方式是联动的——修改 RGB 会自动更新 HSV，反之亦然。

```cpp
// 颜色编辑（RGB）
static float color[3] = { 1.0f, 0.0f, 0.0f };
ImGui::ColorEdit3("Color RGB", color);

// 颜色编辑（RGBA）
static float color_alpha[4] = { 1.0f, 0.0f, 0.0f, 1.0f };
ImGui::ColorEdit4("Color RGBA", color_alpha);

// 颜色选择器（弹窗形式）
ImGui::ColorPicker3("Picker", color);
ImGui::ColorPicker4("Picker with Alpha", color_alpha);

// 只显示颜色按钮（不编辑）
ImGui::ColorButton("Color Button", ImVec4(color[0], color[1], color[2], 1.0f));

// 颜色编辑选项
ImGui::ColorEdit3("HDR Color", color,
    ImGuiColorEditFlags_HDR |
    ImGuiColorEditFlags_Float |
    ImGuiColorEditFlags_PickerHueWheel
);

// 显示颜色编辑选项菜单
ImGui::ColorEdit3("With Options", color, ImGuiColorEditFlags_AlphaPreview);
```

**ColorEdit 常用标志**：

```cpp
ImGuiColorEditFlags_NoAlpha         // 不显示 Alpha 通道
ImGuiColorEditFlags_NoPicker        // 不显示颜色选择器
ImGuiColorEditFlags_NoOptions       // 不显示选项按钮
ImGuiColorEditFlags_NoSmallPreview  // 不显示小预览
ImGuiColorEditFlags_NoInputs        // 不显示 RGB/HSV 输入
ImGuiColorEditFlags_NoTooltip       // 不显示提示
ImGuiColorEditFlags_NoLabel         // 不显示标签
ImGuiColorEditFlags_NoSidePreview   // 不显示侧边预览
ImGuiColorEditFlags_NoDragDrop      // 禁用拖拽
ImGuiColorEditFlags_NoBorder        // 无边框
ImGuiColorEditFlags_AlphaBar        // 显示 Alpha 条
ImGuiColorEditFlags_AlphaPreview    // 在预览中显示 Alpha
ImGuiColorEditFlags_AlphaPreviewHalf// 半透明显示 Alpha
ImGuiColorEditFlags_HDR             // HDR 模式
ImGuiColorEditFlags_DisplayRGB      // 以 RGB 格式显示
ImGuiColorEditFlags_DisplayHSV      // 以 HSV 格式显示
ImGuiColorEditFlags_DisplayHex      // 以十六进制显示
ImGuiColorEditFlags_Uint8           // 以 0-255 显示
ImGuiColorEditFlags_Float           // 以 0.0-1.0 显示
ImGuiColorEditFlags_PickerHueWheel  // 色轮模式
ImGuiColorEditFlags_PickerHueBar    // 色相条模式（默认）
```

### 6.10 下拉选择框

下拉选择框（Combo）用于从一组预定义选项中选择一个。它的 API 设计有些特殊——ImGui 提供了三种不同的方式来提供选项列表，以适应不同的使用场景。

```cpp
// 基础下拉框
static int selected_item = 0;
const char* items[] = { "Apple", "Banana", "Cherry", "Date" };
ImGui::Combo("Fruit", &selected_item, items, IM_ARRAYSIZE(items));

// 使用字符串分隔符
ImGui::Combo("Fruit (string)", &selected_item,
    "Apple\0Banana\0Cherry\0Date\0\0");

// 带回调的下拉框（大数据集）
static int big_selected = 0;
ImGui::Combo("Big List", &big_selected,
    [](void* data, int idx, const char** out_text) -> bool {
        // data 可以传递自定义数据
        if (idx < 0 || idx >= 1000) return false;
        static char buffer[32];
        sprintf(buffer, "Item %d", idx);
        *out_text = buffer;
        return true;
    }, nullptr, 1000);

// 自定义下拉框（使用 Selectable）
static bool combo_open = false;
static const char* current_item = "Select...";
if (ImGui::BeginCombo("Custom Combo", current_item)) {
    for (int i = 0; i < IM_ARRAYSIZE(items); i++) {
        bool is_selected = (current_item == items[i]);
        if (ImGui::Selectable(items[i], is_selected)) {
            current_item = items[i];
        }
        if (is_selected) {
            ImGui::SetItemDefaultFocus();
        }
    }
    ImGui::EndCombo();
}
```

### 6.11 列表框

列表框（ListBox）适合展示一组选项并允许用户选择其中一个或多个。与 Combo 不同，ListBox 始终展开显示所有选项（在可滚动区域内），而不是折叠为一个当前选项的显示。

```cpp
// 基础列表框
static int list_selected = 0;
const char* list_items[] = { "Item 1", "Item 2", "Item 3", "Item 4", "Item 5" };
ImGui::ListBox("List", &list_selected, list_items, IM_ARRAYSIZE(list_items));

// 指定高度
ImGui::ListBox("List (5 items height)", &list_selected,
    list_items, IM_ARRAYSIZE(list_items), 5);

// 自定义列表框
if (ImGui::BeginListBox("Custom List", ImVec2(-1, 200))) {
    for (int i = 0; i < 100; i++) {
        char label[32];
        sprintf(label, "Custom Item %d", i);
        bool is_selected = (list_selected == i);
        if (ImGui::Selectable(label, is_selected)) {
            list_selected = i;
        }
        if (is_selected) {
            ImGui::SetItemDefaultFocus();
        }
    }
    ImGui::EndListBox();
}
```

### 6.12 树形控件

树形控件用于展示层次结构数据，如文件系统、场景图、JSON 结构等。`TreeNode()` 创建一个可展开/折叠的节点，`TreePop()` 必须与之配对以标记节点的结束。

```cpp
// 简单树节点
if (ImGui::TreeNode("Section 1")) {
    ImGui::Text("Content in section 1");
    ImGui::Text("More content...");
    ImGui::TreePop();  // 必须配对调用
}

if (ImGui::TreeNode("Section 2")) {
    ImGui::Text("Content in section 2");
    ImGui::TreePop();
}

// 带 ID 的树节点（当标签不唯一时使用）
for (int i = 0; i < 5; i++) {
    // 使用 ## 隐藏 ID 部分，只显示标签部分
    if (ImGui::TreeNode((void*)(intptr_t)i, "Node %d", i)) {
        ImGui::Text("Node %d content", i);
        ImGui::TreePop();
    }
}

// 默认展开
if (ImGui::TreeNodeEx("Expanded", ImGuiTreeNodeFlags_DefaultOpen)) {
    ImGui::Text("This node is open by default");
    ImGui::TreePop();
}

// 叶子节点（无展开箭头）
if (ImGui::TreeNodeEx("Leaf", ImGuiTreeNodeFlags_Leaf)) {
    ImGui::Text("No arrow on this node");
    ImGui::TreePop();
}

// 可选择的树节点
static int selected_node = -1;
for (int i = 0; i < 5; i++) {
    ImGuiTreeNodeFlags flags = ImGuiTreeNodeFlags_OpenOnArrow |
                               ImGuiTreeNodeFlags_OpenOnDoubleClick;
    if (selected_node == i)
        flags |= ImGuiTreeNodeFlags_Selected;

    bool open = ImGui::TreeNodeEx((void*)(intptr_t)i, flags, "Selectable Node %d", i);
    if (ImGui::IsItemClicked()) {
        selected_node = i;
    }
    if (open) {
        ImGui::Text("Content...");
        ImGui::TreePop();
    }
}
```

### 6.13 折叠标题（CollapsingHeader）

折叠标题（CollapsingHeader）是树形控件的一种变体，常用于设置面板等场景。与 `TreeNode` 的主要区别是：
- `CollapsingHeader` 不能嵌套（它不像 TreeNode 那样用 TreePop 标记作用域）
- `CollapsingHeader` 默认带有边框，视觉上更像一个可折叠的区块
- `CollapsingHeader` 的展开状态由 ImGui 自动保存到 `.ini` 文件

```cpp
// 不可嵌套的折叠标题（类似树节点但带边框）
if (ImGui::CollapsingHeader("Header 1")) {
    ImGui::Text("Content in header 1");
    ImGui::Button("Button");
}

// 默认展开
if (ImGui::CollapsingHeader("Header 2", ImGuiTreeNodeFlags_DefaultOpen)) {
    ImGui::Text("This header is open by default");
}

// 带复选框的折叠标题
static bool header_enabled = true;
if (ImGui::CollapsingHeader("Header with Checkbox", &header_enabled)) {
    ImGui::Text("Content...");
}
```

---

## 7. 高级控件

基础控件可以满足大部分简单的 UI 需求，但当你需要构建更复杂的工具界面时——比如带有菜单栏的编辑器、可排序的表格、模态确认对话框、标签页设置面板——就需要用到本章介绍的高级控件。

这些控件在概念上并不比基础控件复杂，但它们的 API 通常有更多的配置选项和标志。建议在学习时先运行 `ImGui::ShowDemoWindow()` 查看实际效果，再对照本章的代码示例理解实现方式。

### 7.1 菜单栏

菜单栏是桌面应用中常见的导航元素。ImGui 支持两种菜单栏：窗口内的菜单栏和主菜单栏。

```cpp
ImGui::Begin("Window with Menu");

if (ImGui::BeginMenuBar()) {
    if (ImGui::BeginMenu("File")) {
        if (ImGui::MenuItem("New", "Ctrl+N")) {
            // 新建
        }
        if (ImGui::MenuItem("Open", "Ctrl+O")) {
            // 打开
        }
        ImGui::Separator();
        if (ImGui::MenuItem("Save", "Ctrl+S")) {
            // 保存
        }
        ImGui::Separator();
        if (ImGui::MenuItem("Exit")) {
            // 退出
        }
        ImGui::EndMenu();
    }

    if (ImGui::BeginMenu("Edit")) {
        if (ImGui::MenuItem("Undo", "Ctrl+Z", false, false)) {
            // 禁用状态（false 参数）
        }
        if (ImGui::MenuItem("Redo", "Ctrl+Y", false, false)) {
            // 禁用状态
        }
        ImGui::Separator();
        if (ImGui::MenuItem("Cut", "Ctrl+X")) { }
        if (ImGui::MenuItem("Copy", "Ctrl+C")) { }
        if (ImGui::MenuItem("Paste", "Ctrl+V")) { }
        ImGui::EndMenu();
    }

    ImGui::EndMenuBar();
}

ImGui::Text("Window content...");
ImGui::End();

// 主菜单栏（在窗口顶部）
if (ImGui::BeginMainMenuBar()) {
    if (ImGui::BeginMenu("File")) {
        if (ImGui::MenuItem("New")) { }
        ImGui::EndMenu();
    }
    ImGui::EndMainMenuBar();
}
```

### 7.2 工具提示

工具提示（Tooltip）是一种当鼠标悬停在某个控件上时显示的额外信息。它非常适合用来解释控件的功能、显示详细数据或提供快捷键提示。

```cpp
ImGui::Button("Hover Me");
if (ImGui::IsItemHovered()) {
    ImGui::BeginTooltip();
    ImGui::Text("This is a tooltip!");
    ImGui::Text("You can put any content here.");
    ImGui::EndTooltip();
}

// 简写形式
ImGui::Button("Hover Me Too");
if (ImGui::IsItemHovered()) {
    ImGui::SetTooltip("Simple tooltip with text: %s", "formatted");
}

// 带延迟的工具提示
ImGui::Button("Delayed Tooltip");
if (ImGui::IsItemHovered(ImGuiHoveredFlags_DelayNormal)) {
    ImGui::SetTooltip("Appears after a short delay");
}

// 总是显示的工具提示（用于教学）
ImGui::Button("Always");
if (ImGui::IsItemHovered(ImGuiHoveredFlags_ForTooltip)) {
    ImGui::SetTooltip("Standard tooltip");
}
```

### 7.3 弹窗（Modal / Popup）

弹窗是临时出现的浮动窗口，通常用于上下文菜单、确认对话框、下拉列表等场景。ImGui 的弹窗系统使用字符串 ID 来区分不同的弹窗，并通过 `OpenPopup()` 和 `BeginPopup()` 的配对来控制弹窗的显示。

```cpp
// 简单弹窗
if (ImGui::Button("Open Popup")) {
    ImGui::OpenPopup("MyPopup");
}

if (ImGui::BeginPopup("MyPopup")) {
    ImGui::Text("This is a popup!");
    if (ImGui::Button("Close")) {
        ImGui::CloseCurrentPopup();
    }
    ImGui::EndPopup();
}

// 上下文菜单（右键菜单）
ImGui::Button("Right Click Me");
if (ImGui::BeginPopupContextItem("ItemContextMenu")) {
    if (ImGui::MenuItem("Copy")) { }
    if (ImGui::MenuItem("Paste")) { }
    ImGui::Separator();
    if (ImGui::MenuItem("Delete")) { }
    ImGui::EndPopup();
}

// 窗口上下文菜单
ImGui::Begin("Window with Context Menu");
if (ImGui::BeginPopupContextWindow()) {
    if (ImGui::MenuItem("Option 1")) { }
    if (ImGui::MenuItem("Option 2")) { }
    ImGui::EndPopup();
}
ImGui::End();

// 模态对话框
if (ImGui::Button("Open Modal")) {
    ImGui::OpenPopup("Confirm Delete");
}

// 居中显示弹窗
ImVec2 center = ImGui::GetMainViewport()->GetCenter();
ImGui::SetNextWindowPos(center, ImGuiCond_Appearing, ImVec2(0.5f, 0.5f));

if (ImGui::BeginPopupModal("Confirm Delete", nullptr,
    ImGuiWindowFlags_AlwaysAutoResize)) {
    ImGui::Text("Are you sure you want to delete this item?");
    ImGui::Separator();

    if (ImGui::Button("OK", ImVec2(120, 0))) {
        // 执行删除
        ImGui::CloseCurrentPopup();
    }
    ImGui::SetItemDefaultFocus();
    ImGui::SameLine();
    if (ImGui::Button("Cancel", ImVec2(120, 0))) {
        ImGui::CloseCurrentPopup();
    }
    ImGui::EndPopup();
}
```

### 7.4 表格（Table）

表格是 ImGui 中功能最丰富的控件之一。它支持列宽调整、列重排序、列隐藏、排序、交替行背景色、边框控制等大量功能。表格 API 看起来有些冗长，但这是为了提供足够的灵活性来应对各种复杂的使用场景。

```cpp
// 创建表格
if (ImGui::BeginTable("MyTable", 3,  // 3 列
    ImGuiTableFlags_Borders |
    ImGuiTableFlags_RowBg |
    ImGuiTableFlags_Resizable |
    ImGuiTableFlags_Reorderable |
    ImGuiTableFlags_Hideable)) {

    // 设置表头
    ImGui::TableSetupColumn("Name", ImGuiTableColumnFlags_WidthStretch);
    ImGui::TableSetupColumn("Age", ImGuiTableColumnFlags_WidthFixed, 50.0f);
    ImGui::TableSetupColumn("Score", ImGuiTableColumnFlags_WidthStretch);
    ImGui::TableHeadersRow();

    // 数据行
    for (int row = 0; row < 10; row++) {
        ImGui::TableNextRow();

        ImGui::TableSetColumnIndex(0);
        ImGui::Text("Player %d", row);

        ImGui::TableSetColumnIndex(1);
        ImGui::Text("%d", 20 + row);

        ImGui::TableSetColumnIndex(2);
        ImGui::Text("%.1f", 100.0f - row * 5.0f);
    }

    ImGui::EndTable();
}

// 带选择的表格
if (ImGui::BeginTable("SelectableTable", 3,
    ImGuiTableFlags_Borders | ImGuiTableFlags_SizingStretchProp)) {

    ImGui::TableSetupColumn("ID");
    ImGui::TableSetupColumn("Name");
    ImGui::TableSetupColumn("Action");
    ImGui::TableHeadersRow();

    static int selected_row = -1;
    for (int row = 0; row < 5; row++) {
        ImGui::TableNextRow();

        // 使用SelectableSpanAllColumns让整行可点击
        ImGui::TableSetColumnIndex(0);
        char label[32];
        sprintf(label, "Row %d", row);
        if (ImGui::Selectable(label, selected_row == row,
            ImGuiSelectableFlags_SpanAllColumns)) {
            selected_row = row;
        }

        ImGui::TableSetColumnIndex(1);
        ImGui::Text("Name %d", row);

        ImGui::TableSetColumnIndex(2);
        if (ImGui::SmallButton("Edit")) {
            // 编辑行
        }
    }

    ImGui::EndTable();
}
```

**表格标志**：

```cpp
ImGuiTableFlags_Resizable          // 可调整列宽
ImGuiTableFlags_Reorderable        // 可重新排序列
ImGuiTableFlags_Hideable           // 可隐藏列
ImGuiTableFlags_Sortable           // 可排序
ImGuiTableFlags_SortMulti          // 多列排序
ImGuiTableFlags_RowBg              // 交替行背景色
ImGuiTableFlags_Borders            // 所有边框
ImGuiTableFlags_BordersH           // 水平边框
ImGuiTableFlags_BordersV           // 垂直边框
ImGuiTableFlags_BordersOuter       // 外边框
ImGuiTableFlags_BordersInner       // 内边框
ImGuiTableFlags_NoBordersInBody    // 主体无边框
ImGuiTableFlags_ScrollX            // 水平滚动
ImGuiTableFlags_ScrollY            // 垂直滚动
ImGuiTableFlags_SizingFixedFit     // 固定大小，适应内容
ImGuiTableFlags_SizingFixedSame    // 固定大小，所有列相同
ImGuiTableFlags_SizingStretchProp  // 拉伸，按比例分配
ImGuiTableFlags_SizingStretchSame  // 拉伸，所有列相同
```

### 7.5 Tab 标签页

标签页（TabBar/TabItem）用于在有限的空间内组织大量内容。用户可以通过点击不同的标签页来切换显示的内容区域。

```cpp
// 使用 BeginTabBar / BeginTabItem
if (ImGui::BeginTabBar("MyTabBar")) {

    if (ImGui::BeginTabItem("General")) {
        ImGui::Text("General settings...");
        ImGui::Checkbox("Option 1", &some_bool);
        ImGui::EndTabItem();
    }

    if (ImGui::BeginTabItem("Graphics")) {
        ImGui::Text("Graphics settings...");
        ImGui::SliderFloat("Brightness", &brightness, 0, 2);
        ImGui::EndTabItem();
    }

    if (ImGui::BeginTabItem("Audio")) {
        ImGui::Text("Audio settings...");
        ImGui::SliderFloat("Volume", &volume, 0, 1);
        ImGui::EndTabItem();
    }

    // 可关闭的标签页
    static bool show_advanced = true;
    if (show_advanced && ImGui::BeginTabItem("Advanced", &show_advanced)) {
        ImGui::Text("Advanced settings...");
        ImGui::EndTabItem();
    }

    ImGui::EndTabBar();
}
```

### 7.6 进度条

进度条用于展示某个长时间操作的完成进度，或者作为一个只读的状态指示器。

```cpp
// 基础进度条
static float progress = 0.0f;
progress += 0.01f;
if (progress > 1.0f) progress = 0.0f;

ImGui::ProgressBar(progress);
ImGui::ProgressBar(progress, ImVec2(0, 0), "Loading...");

// 模拟带动画的进度条
ImGui::ProgressBar(progress, ImVec2(-1, 0));
ImGui::SameLine(0, 0);
ImGui::Text("%.0f%%", progress * 100);
```

### 7.7 图像显示

图像显示需要使用你当前渲染 API 的纹理句柄。`ImTextureID` 是一个不透明的指针类型，在不同后端中对应不同的类型：DirectX 中是 `ID3D11ShaderResourceView*`，OpenGL 中是 `GLuint`，Vulkan 中是 `VkDescriptorSet`。这意味着显示图像的代码需要与特定的渲染后端配合。

```cpp
// 显示纹理（需要后端特定的纹理句柄）
// 在 DirectX 11 中，ImTextureID 是 ID3D11ShaderResourceView*
// 在 OpenGL 中，ImTextureID 是 GLuint

// 显示图像
ImTextureID my_texture = GetMyTexture();  // 你的纹理
ImVec2 image_size(200, 200);
ImGui::Image(my_texture, image_size);

// 可点击的图像
ImGui::ImageButton("image_button", my_texture, image_size);

// 带 UV 坐标的图像（用于图集）
ImVec2 uv0(0, 0);  // 左上角
ImVec2 uv1(1, 1);  // 右下角
ImGui::Image(my_texture, image_size, uv0, uv1);

// 带色调的图像
ImVec4 tint_color(1, 1, 1, 1);    // 正常显示
ImVec4 border_color(1, 0, 0, 1);  // 红色边框
ImGui::Image(my_texture, image_size, uv0, uv1, tint_color, border_color);
```

---

## 8. 绘图与自定义渲染

当你需要在 ImGui 窗口中绘制自定义图形时——比如游戏内的小地图、节点编辑器、波形图、自定义进度条——你就需要使用 ImDrawList API。这是 ImGui 中所有绘制操作的底层接口，控件库本身也是通过 ImDrawList 来绘制按钮、文本、滑块等所有视觉元素的。

理解 ImDrawList 的工作原理对于高级自定义渲染至关重要。ImDrawList 本质上是一个命令缓冲区和顶点缓冲区的组合：你调用 `AddLine()`、`AddRect()` 等函数时，实际上是在向列表中追加顶点和绘制命令。这些命令在 `Render()` 阶段被处理，最终转换为 `ImDrawData` 交给你的渲染后端执行。

### 8.1 ImDrawList 基础

每个窗口都有自己的 `ImDrawList`，可以通过 `ImGui::GetWindowDrawList()` 获取。此外还有一个背景层（Background）和前景层（Foreground）的 DrawList。

```cpp
// 获取当前窗口的绘制列表
ImDrawList* draw_list = ImGui::GetWindowDrawList();

// 获取背景绘制列表（在所有窗口下方）
ImDrawList* bg_draw_list = ImGui::GetBackgroundDrawList();

// 获取前景绘制列表（在所有窗口上方）
ImDrawList* fg_draw_list = ImGui::GetForegroundDrawList();
```

### 8.2 基本图形绘制

```cpp
ImDrawList* draw_list = ImGui::GetWindowDrawList();
ImVec2 window_pos = ImGui::GetWindowPos();

// 绘制线段
ImVec2 p1 = ImVec2(window_pos.x + 50, window_pos.y + 50);
ImVec2 p2 = ImVec2(window_pos.x + 200, window_pos.y + 100);
ImU32 red = IM_COL32(255, 0, 0, 255);
draw_list->AddLine(p1, p2, red, 2.0f);  // 2.0f 是线宽

// 绘制矩形
ImVec2 rect_min = ImVec2(window_pos.x + 50, window_pos.y + 150);
ImVec2 rect_max = ImVec2(window_pos.x + 200, window_pos.y + 250);
ImU32 green = IM_COL32(0, 255, 0, 255);
draw_list->AddRect(rect_min, rect_max, green);

// 填充矩形
ImU32 green_transparent = IM_COL32(0, 255, 0, 128);
draw_list->AddRectFilled(rect_min, rect_max, green_transparent);

// 圆角矩形
draw_list->AddRectFilled(rect_min, rect_max, green_transparent, 10.0f);  // 10 是圆角半径

// 绘制圆形
ImVec2 center = ImVec2(window_pos.x + 300, window_pos.y + 100);
float radius = 50.0f;
ImU32 blue = IM_COL32(0, 100, 255, 255);
draw_list->AddCircle(center, radius, blue, 32, 3.0f);  // 32 分段数，3.0f 线宽

// 填充圆形
draw_list->AddCircleFilled(center, radius, IM_COL32(0, 100, 255, 128), 32);

// 绘制三角形
ImVec2 tri_a = ImVec2(window_pos.x + 400, window_pos.y + 50);
ImVec2 tri_b = ImVec2(window_pos.x + 350, window_pos.y + 150);
ImVec2 tri_c = ImVec2(window_pos.x + 450, window_pos.y + 150);
draw_list->AddTriangle(tri_a, tri_b, tri_c, red);
draw_list->AddTriangleFilled(tri_a, tri_b, tri_c, IM_COL32(255, 0, 0, 128));

// 绘制四边形（多边形）
ImVec2 quad[4] = {
    ImVec2(window_pos.x + 500, window_pos.y + 50),
    ImVec2(window_pos.x + 550, window_pos.y + 80),
    ImVec2(window_pos.x + 520, window_pos.y + 150),
    ImVec2(window_pos.x + 470, window_pos.y + 120)
};
draw_list->AddQuad(quad[0], quad[1], quad[2], quad[3], green);
draw_list->AddQuadFilled(quad[0], quad[1], quad[2], quad[3], green_transparent);

// 绘制多边形
ImVec2 poly[5] = { /* ... 5 个点 ... */ };
draw_list->AddPolyline(poly, 5, red, ImDrawFlags_Closed, 2.0f);
draw_list->AddConvexPolyFilled(poly, 5, green_transparent);
```

### 8.3 绘制文本

```cpp
ImDrawList* draw_list = ImGui::GetWindowDrawList();
ImVec2 pos(window_pos.x + 50, window_pos.y + 300);
ImU32 white = IM_COL32(255, 255, 255, 255);

// 绘制文本
draw_list->AddText(pos, white, "Hello DrawList!");

// 使用特定字体
draw_list->AddText(ImGui::GetFont(), 24.0f, pos, white, "Big Text!");

// 绘制带背景阴影的文本
ImVec2 shadow_pos = ImVec2(pos.x + 1, pos.y + 1);
draw_list->AddText(shadow_pos, IM_COL32(0, 0, 0, 255), "Shadow Text");
draw_list->AddText(pos, white, "Shadow Text");
```

### 8.4 绘制曲线和贝塞尔曲线

```cpp
ImDrawList* draw_list = ImGui::GetWindowDrawList();

// 贝塞尔曲线
ImVec2 p1 = ImVec2(x + 50, y + 200);
ImVec2 cp1 = ImVec2(x + 150, y + 50);   // 控制点 1
ImVec2 cp2 = ImVec2(x + 350, y + 350);   // 控制点 2
ImVec2 p2 = ImVec2(x + 450, y + 200);
draw_list->AddBezierCubic(p1, cp1, cp2, p2, IM_COL32(255, 255, 0, 255), 2.0f, 50);

// 二次贝塞尔曲线
ImVec2 qcp = ImVec2(x + 250, y + 50);    // 单个控制点
draw_list->AddBezierQuadratic(p1, qcp, p2, IM_COL32(0, 255, 255, 255), 2.0f, 50);

// 绘制椭圆
void DrawEllipse(ImDrawList* draw_list, const ImVec2& center,
                 float rx, float ry, ImU32 col, int num_segments = 32) {
    for (int i = 0; i < num_segments; i++) {
        float a1 = (float)i / (float)num_segments * 2.0f * IM_PI;
        float a2 = (float)(i + 1) / (float)num_segments * 2.0f * IM_PI;
        ImVec2 p1(center.x + cosf(a1) * rx, center.y + sinf(a1) * ry);
        ImVec2 p2(center.x + cosf(a2) * rx, center.y + sinf(a2) * ry);
        draw_list->AddLine(p1, p2, col, 1.0f);
    }
}
```

### 8.5 使用画布进行交互式绘图

```cpp
// 交互式绘图区域
ImVec2 canvas_p0 = ImGui::GetCursorScreenPos();
ImVec2 canvas_sz = ImVec2(400, 400);
ImVec2 canvas_p1 = ImVec2(canvas_p0.x + canvas_sz.x, canvas_p0.y + canvas_sz.y);

// 绘制画布背景
ImDrawList* draw_list = ImGui::GetWindowDrawList();
draw_list->AddRectFilled(canvas_p0, canvas_p1, IM_COL32(30, 30, 30, 255));
draw_list->AddRect(canvas_p0, canvas_p1, IM_COL32(100, 100, 100, 255));

// 处理画布上的鼠标交互
ImGui::InvisibleButton("canvas", canvas_sz);
bool is_hovered = ImGui::IsItemHovered();
bool is_active = ImGui::IsItemActive();
ImVec2 mouse_pos = ImGui::GetIO().MousePos;
ImVec2 mouse_pos_in_canvas = ImVec2(mouse_pos.x - canvas_p0.x, mouse_pos.y - canvas_p0.y);

if (is_hovered) {
    // 在画布上绘制鼠标位置
    char buf[32];
    sprintf(buf, "(%.0f, %.0f)", mouse_pos_in_canvas.x, mouse_pos_in_canvas.y);
    draw_list->AddText(ImVec2(mouse_pos.x + 10, mouse_pos.y + 10),
        IM_COL32(255, 255, 255, 255), buf);

    // 绘制十字准星
    draw_list->AddLine(
        ImVec2(canvas_p0.x, mouse_pos.y),
        ImVec2(canvas_p1.x, mouse_pos.y),
        IM_COL32(100, 100, 100, 100)
    );
    draw_list->AddLine(
        ImVec2(mouse_pos.x, canvas_p0.y),
        ImVec2(mouse_pos.x, canvas_p1.y),
        IM_COL32(100, 100, 100, 100)
    );
}

if (is_active && ImGui::IsMouseDragging(ImGuiMouseButton_Left)) {
    // 在画布上拖拽绘制
    static ImVec2 last_pos = mouse_pos_in_canvas;
    draw_list->AddLine(
        ImVec2(canvas_p0.x + last_pos.x, canvas_p0.y + last_pos.y),
        mouse_pos,
        IM_COL32(255, 255, 0, 255), 2.0f
    );
    last_pos = mouse_pos_in_canvas;
}
```

**坐标系说明**：上面的例子使用了**屏幕坐标系**（通过 `GetCursorScreenPos()` 获取）。这是 ImDrawList 所使用的坐标系——所有绘制操作都使用屏幕绝对坐标，而不是窗口内的相对坐标。如果你需要相对于窗口内容区域左上角的坐标，可以使用 `ImGui::GetCursorPos()`，但记得手动加上 `window_pos` 偏移量再传给 DrawList。

**交互模式三件套**：`InvisibleButton()` + `IsItemHovered()` + `IsItemActive()` 是构建自定义交互组件的标准模式：
- `InvisibleButton()` 创建一个可交互区域，但不绘制任何内容
- `IsItemHovered()` 检测鼠标是否在这个区域内
- `IsItemActive()` 检测用户是否正在与这个区域交互（如按住鼠标）

这三者的组合让你可以完全控制一个区域的视觉表现和交互行为。配合 `IsMouseDragging()` 和 `GetMouseDragDelta()`，你可以实现拖拽、选择框、画笔等任意交互模式。

### 8.6 裁剪和遮罩

```cpp
ImDrawList* draw_list = ImGui::GetWindowDrawList();

// 推送裁剪矩形（只在这个矩形内绘制）
ImVec2 clip_min = ImVec2(x + 50, y + 50);
ImVec2 clip_max = ImVec2(x + 200, y + 200);
draw_list->PushClipRect(clip_min, clip_max, true);  // true = 相交模式

// 这里的绘制会被裁剪
ImVec2 center(x + 150, y + 150);
draw_list->AddCircleFilled(center, 80, IM_COL32(255, 0, 0, 255), 32);

// 恢复裁剪
draw_list->PopClipRect();
```

---

## 9. 样式与主题

ImGui 的样式系统非常灵活，允许你在全局层面定义整个应用的外观，也可以在局部层面临时修改特定控件的样式。理解样式系统的架构对于打造一致且美观的用户界面至关重要。

样式数据存储在 `ImGuiStyle` 结构中，通过 `ImGui::GetStyle()` 获取。这个结构包含两类数据：**标量值**（如圆角半径、间距大小）和**颜色值**（`Colors` 数组，包含约 50 种不同用途的颜色）。所有控件在绘制时都会引用这些样式值，这意味着一次全局修改就会影响整个界面。

样式修改有两种方式：**全局修改**（直接修改 `ImGui::GetStyle()`）和**临时修改**（使用 `PushStyleVar`/`PopStyleVar` 和 `PushStyleColor`/`PopStyleColor`）。前者适合定义应用的整体主题，后者适合对特定控件进行局部调整。

### 9.1 预设主题

```cpp
// ImGui 内置三种主题
ImGui::StyleColorsDark();      // 暗色主题（默认）
ImGui::StyleColorsLight();     // 亮色主题
ImGui::StyleColorsClassic();   // 经典主题（类似旧版 ImGui）
```

### 9.2 自定义样式

```cpp
ImGuiStyle& style = ImGui::GetStyle();

// 窗口相关
style.WindowRounding = 8.0f;           // 窗口圆角
style.WindowBorderSize = 1.0f;         // 窗口边框大小
style.WindowPadding = ImVec2(10, 10);  // 窗口内边距
style.WindowTitleAlign = ImVec2(0.5f, 0.5f); // 标题居中

// 控件间距
style.ItemSpacing = ImVec2(8, 4);      // 控件之间的间距
style.ItemInnerSpacing = ImVec2(4, 4); // 控件内部间距
style.FramePadding = ImVec2(4, 3);     // 框架内边距

// 框架和按钮
style.FrameRounding = 4.0f;            // 框架圆角
style.FrameBorderSize = 0.0f;          // 框架边框
style.ButtonTextAlign = ImVec2(0.5f, 0.5f); // 按钮文本居中

// 滚动条
style.ScrollbarSize = 14.0f;
style.ScrollbarRounding = 9.0f;

// 滑块
style.GrabMinSize = 10.0f;
style.GrabRounding = 4.0f;

// 弹窗
style.PopupRounding = 4.0f;
style.PopupBorderSize = 1.0f;

// 标签页
style.TabRounding = 4.0f;
style.TabBorderSize = 0.0f;

// 缩进
style.IndentSpacing = 21.0f;

// 抗锯齿
style.AntiAliasedLines = true;
style.AntiAliasedFill = true;

// 曲线细分
style.CircleTessellationMaxError = 0.30f;
```

### 9.3 自定义颜色

```cpp
ImGuiStyle& style = ImGui::GetStyle();
ImVec4* colors = style.Colors;

// 修改特定颜色
colors[ImGuiCol_Text]                   = ImVec4(1.00f, 1.00f, 1.00f, 1.00f);
colors[ImGuiCol_TextDisabled]           = ImVec4(0.50f, 0.50f, 0.50f, 1.00f);
colors[ImGuiCol_WindowBg]               = ImVec4(0.10f, 0.10f, 0.10f, 1.00f);
colors[ImGuiCol_ChildBg]                = ImVec4(0.00f, 0.00f, 0.00f, 0.00f);
colors[ImGuiCol_PopupBg]                = ImVec4(0.19f, 0.19f, 0.19f, 0.92f);
colors[ImGuiCol_Border]                 = ImVec4(0.19f, 0.19f, 0.19f, 0.29f);
colors[ImGuiCol_BorderShadow]           = ImVec4(0.00f, 0.00f, 0.00f, 0.24f);
colors[ImGuiCol_FrameBg]                = ImVec4(0.05f, 0.05f, 0.05f, 0.54f);
colors[ImGuiCol_FrameBgHovered]         = ImVec4(0.19f, 0.19f, 0.19f, 0.54f);
colors[ImGuiCol_FrameBgActive]          = ImVec4(0.20f, 0.22f, 0.23f, 1.00f);
colors[ImGuiCol_TitleBg]                = ImVec4(0.00f, 0.00f, 0.00f, 1.00f);
colors[ImGuiCol_TitleBgActive]          = ImVec4(0.06f, 0.06f, 0.06f, 1.00f);
colors[ImGuiCol_TitleBgCollapsed]       = ImVec4(0.00f, 0.00f, 0.00f, 1.00f);
colors[ImGuiCol_MenuBarBg]              = ImVec4(0.14f, 0.14f, 0.14f, 1.00f);
colors[ImGuiCol_ScrollbarBg]            = ImVec4(0.05f, 0.05f, 0.05f, 0.54f);
colors[ImGuiCol_ScrollbarGrab]          = ImVec4(0.34f, 0.34f, 0.34f, 0.54f);
colors[ImGuiCol_ScrollbarGrabHovered]   = ImVec4(0.40f, 0.40f, 0.40f, 0.54f);
colors[ImGuiCol_ScrollbarGrabActive]    = ImVec4(0.56f, 0.56f, 0.56f, 0.54f);
colors[ImGuiCol_CheckMark]              = ImVec4(0.33f, 0.67f, 0.86f, 1.00f);
colors[ImGuiCol_SliderGrab]             = ImVec4(0.34f, 0.34f, 0.34f, 0.54f);
colors[ImGuiCol_SliderGrabActive]       = ImVec4(0.56f, 0.56f, 0.56f, 0.54f);
colors[ImGuiCol_Button]                 = ImVec4(0.05f, 0.05f, 0.05f, 0.54f);
colors[ImGuiCol_ButtonHovered]          = ImVec4(0.19f, 0.19f, 0.19f, 0.54f);
colors[ImGuiCol_ButtonActive]           = ImVec4(0.20f, 0.22f, 0.23f, 1.00f);
colors[ImGuiCol_Header]                 = ImVec4(0.00f, 0.00f, 0.00f, 0.52f);
colors[ImGuiCol_HeaderHovered]          = ImVec4(0.00f, 0.00f, 0.00f, 0.36f);
colors[ImGuiCol_HeaderActive]           = ImVec4(0.20f, 0.22f, 0.23f, 0.33f);
colors[ImGuiCol_Separator]              = ImVec4(0.28f, 0.28f, 0.28f, 0.29f);
colors[ImGuiCol_SeparatorHovered]       = ImVec4(0.44f, 0.44f, 0.44f, 0.29f);
colors[ImGuiCol_SeparatorActive]        = ImVec4(0.40f, 0.44f, 0.47f, 1.00f);
colors[ImGuiCol_ResizeGrip]             = ImVec4(0.28f, 0.28f, 0.28f, 0.29f);
colors[ImGuiCol_ResizeGripHovered]      = ImVec4(0.44f, 0.44f, 0.44f, 0.29f);
colors[ImGuiCol_ResizeGripActive]       = ImVec4(0.40f, 0.44f, 0.47f, 1.00f);
colors[ImGuiCol_Tab]                    = ImVec4(0.00f, 0.00f, 0.00f, 0.52f);
colors[ImGuiCol_TabHovered]             = ImVec4(0.14f, 0.14f, 0.14f, 1.00f);
colors[ImGuiCol_TabActive]              = ImVec4(0.20f, 0.20f, 0.20f, 0.36f);
colors[ImGuiCol_TabUnfocused]           = ImVec4(0.00f, 0.00f, 0.00f, 0.52f);
colors[ImGuiCol_TabUnfocusedActive]     = ImVec4(0.14f, 0.14f, 0.14f, 1.00f);
colors[ImGuiCol_PlotLines]              = ImVec4(1.00f, 0.00f, 0.00f, 1.00f);
colors[ImGuiCol_PlotLinesHovered]       = ImVec4(1.00f, 0.00f, 0.00f, 1.00f);
colors[ImGuiCol_PlotHistogram]          = ImVec4(1.00f, 0.00f, 0.00f, 1.00f);
colors[ImGuiCol_PlotHistogramHovered]   = ImVec4(1.00f, 0.00f, 0.00f, 1.00f);
colors[ImGuiCol_TableHeaderBg]          = ImVec4(0.00f, 0.00f, 0.00f, 0.52f);
colors[ImGuiCol_TableBorderStrong]      = ImVec4(0.00f, 0.00f, 0.00f, 0.52f);
colors[ImGuiCol_TableBorderLight]       = ImVec4(0.28f, 0.28f, 0.28f, 0.29f);
colors[ImGuiCol_TableRowBg]             = ImVec4(0.00f, 0.00f, 0.00f, 0.00f);
colors[ImGuiCol_TableRowBgAlt]          = ImVec4(1.00f, 1.00f, 1.00f, 0.06f);
colors[ImGuiCol_TextSelectedBg]         = ImVec4(0.20f, 0.22f, 0.23f, 1.00f);
colors[ImGuiCol_DragDropTarget]         = ImVec4(0.33f, 0.67f, 0.86f, 1.00f);
colors[ImGuiCol_NavHighlight]           = ImVec4(1.00f, 0.00f, 0.00f, 1.00f);
colors[ImGuiCol_NavWindowingHighlight]  = ImVec4(1.00f, 0.00f, 0.00f, 0.70f);
colors[ImGuiCol_NavWindowingDimBg]      = ImVec4(1.00f, 0.00f, 0.00f, 0.20f);
colors[ImGuiCol_ModalWindowDimBg]       = ImVec4(1.00f, 0.00f, 0.00f, 0.35f);
```

**理解颜色语义**：`ImGuiCol_*` 枚举定义了约 50 种颜色槽位，每种都有特定的语义。命名遵循一致的规律：基础状态（如 `Button`）、悬停状态（`ButtonHovered`）、激活状态（`ButtonActive`）。这种三态设计贯穿整个 ImGui 控件系统：
- **基础态**：控件未被交互时的默认外观
- **悬停态**：鼠标悬停在控件上时的外观
- **激活态**：控件正在被交互（如按住鼠标）时的外观

理解这种三态模型后，自定义主题就会变得系统而有条理。你不需要猜测某个颜色用在哪里——只需要找到对应的 `*Hovered` 或 `*Active` 变体即可。

**关于 `ImVec4` 颜色格式**：颜色使用浮点数 RGBA 格式，每个分量范围是 0.0-1.0。这与许多图形 API（OpenGL、DirectX）使用的颜色格式一致。如果你需要与使用 0-255 整数的系统交互，只需除以 255.0f 即可转换。

### 9.4 临时修改样式

```cpp
// 使用 Push / Pop 临时修改样式（推荐）
ImGui::PushStyleColor(ImGuiCol_Button, ImVec4(1.0f, 0.0f, 0.0f, 1.0f));
ImGui::PushStyleColor(ImGuiCol_ButtonHovered, ImVec4(1.0f, 0.2f, 0.2f, 1.0f));
ImGui::PushStyleVar(ImGuiStyleVar_FrameRounding, 8.0f);

if (ImGui::Button("Styled Button")) {
    // ...
}

// 恢复之前的样式
ImGui::PopStyleColor(2);  // 弹出 2 个颜色
ImGui::PopStyleVar();     // 弹出 1 个变量

// 可用的 Push 函数
ImGui::PushStyleColor(ImGuiCol_XXX, color);     // 临时修改颜色
ImGui::PushStyleVar(ImGuiStyleVar_XXX, value);   // 临时修改变量
ImGui::PushFont(font);                           // 临时切换字体
ImGui::PushItemWidth(width);                     // 临时设置控件宽度
ImGui::PushTextWrapPos(wrap_pos);                // 临时设置文本换行位置
ImGui::PushClipRect(min, max, intersect);        // 临时裁剪区域
```

### 9.5 加载自定义字体

字体系统由 `ImFontAtlas` 管理，通过 `ImGuiIO::Fonts` 访问。字体加载必须在初始化阶段完成（在第一个 `NewFrame()` 之前），因为 ImGui 需要提前构建字体纹理图集。

字体加载有两个关键概念：字形范围（Glyph Ranges）和合并模式（Merge Mode）。字形范围决定了哪些字符会被包含在纹理图集中——默认只包含 ASCII 字符，要显示中文必须显式指定中文字形范围。合并模式允许将多个字体合并为一个图集，这在需要同时显示英文和中文时非常有用——你可以加载一个英文字体作为基础，然后用合并模式追加中文字体。

```cpp
ImGuiIO& io = ImGui::GetIO();

// 加载字体文件
ImFont* font1 = io.Fonts->AddFontFromFileTTF(
    "C:/Windows/Fonts/arial.ttf",  // 字体文件路径
    16.0f                           // 字体大小
);

// 加载不同大小的同一字体
ImFont* font2 = io.Fonts->AddFontFromFileTTF(
    "C:/Windows/Fonts/arial.ttf",
    24.0f
);

// 加载中文字体（需要指定字形范围）
ImFontConfig config;
config.MergeMode = true;  // 合并到现有字体
static const ImWchar chinese_ranges[] = {
    0x4E00, 0x9FFF,  // CJK 统一表意文字
    0x3000, 0x303F,  // CJK 标点符号
    0xFF00, 0xFFEF,  // 全角字符
    0,
};
ImFont* chinese_font = io.Fonts->AddFontFromFileTTF(
    "C:/Windows/Fonts/msyh.ttc",  // 微软雅黑
    16.0f,
    &config,
    chinese_ranges
);

// 构建字体纹理
io.Fonts->Build();

// 使用字体
ImGui::PushFont(font2);
ImGui::Text("Big Text");
ImGui::PopFont();
```

---

## 10. 高级功能

本章介绍 ImGui 中一些更底层的系统功能，包括窗口停靠、多视口、输入处理、焦点管理、剪贴板操作和配置持久化。这些功能通常不会在每个控件调用中使用，但它们是构建专业级工具界面的基础设施。

理解这些高级功能可以帮助你更好地控制 ImGui 的行为，实现更复杂的交互模式，以及将你的工具界面集成到更大的应用架构中。

### 10.1 窗口停靠（Docking）

> **注意**：Docking 功能在 docking 分支中提供，不是主分支。

```cpp
// 初始化时启用停靠
ImGuiIO& io = ImGui::GetIO();
io.ConfigFlags |= ImGuiConfigFlags_DockingEnable;

// 创建停靠空间（通常在主窗口中）
ImGuiViewport* viewport = ImGui::GetMainViewport();
ImGui::SetNextWindowPos(viewport->WorkPos);
ImGui::SetNextWindowSize(viewport->WorkSize);
ImGui::SetNextWindowViewport(viewport->ID);

ImGuiWindowFlags host_flags = 0;
host_flags |= ImGuiWindowFlags_NoTitleBar;
host_flags |= ImGuiWindowFlags_NoCollapse;
host_flags |= ImGuiWindowFlags_NoResize;
host_flags |= ImGuiWindowFlags_NoMove;
host_flags |= ImGuiWindowFlags_NoBringToFrontOnFocus;
host_flags |= ImGuiWindowFlags_NoNavFocus;
host_flags |= ImGuiWindowFlags_MenuBar;

ImGui::PushStyleVar(ImGuiStyleVar_WindowRounding, 0.0f);
ImGui::PushStyleVar(ImGuiStyleVar_WindowBorderSize, 0.0f);
ImGui::PushStyleVar(ImGuiStyleVar_WindowPadding, ImVec2(0.0f, 0.0f));

ImGui::Begin("DockSpace", nullptr, host_flags);
ImGui::PopStyleVar(3);

// 创建停靠空间
ImGuiID dockspace_id = ImGui::GetID("MainDockSpace");
ImGui::DockSpace(dockspace_id, ImVec2(0.0f, 0.0f),
    ImGuiDockNodeFlags_PassthruCentralNode);

// 首次运行时设置默认布局
static bool first_time = true;
if (first_time) {
    first_time = false;

    ImGui::DockBuilderRemoveNode(dockspace_id);
    ImGui::DockBuilderAddNode(dockspace_id,
        ImGuiDockNodeFlags_DockSpace);
    ImGui::DockBuilderSetNodeSize(dockspace_id,
        ImGui::GetMainViewport()->Size);

    // 分割停靠空间
    ImGuiID dock_main_id = dockspace_id;
    ImGuiID dock_left = ImGui::DockBuilderSplitNode(
        dock_main_id, ImGuiDir_Left, 0.25f, nullptr, &dock_main_id);
    ImGuiID dock_right = ImGui::DockBuilderSplitNode(
        dock_main_id, ImGuiDir_Right, 0.25f, nullptr, &dock_main_id);
    ImGuiID dock_bottom = ImGui::DockBuilderSplitNode(
        dock_main_id, ImGuiDir_Down, 0.25f, nullptr, &dock_main_id);

    // 将窗口停靠到相应位置
    ImGui::DockBuilderDockWindow("Hierarchy", dock_left);
    ImGui::DockBuilderDockWindow("Inspector", dock_right);
    ImGui::DockBuilderDockWindow("Console", dock_bottom);
    ImGui::DockBuilderDockWindow("Viewport", dock_main_id);

    ImGui::DockBuilderFinish(dockspace_id);
}

ImGui::End();
```

### 10.2 多视口（Multi-Viewports）

```cpp
// 启用多视口（窗口可以拖出主窗口）
ImGuiIO& io = ImGui::GetIO();
io.ConfigFlags |= ImGuiConfigFlags_ViewportsEnable;

// 在渲染阶段处理额外的视口
ImGui::Render();
ImDrawData* main_draw_data = ImGui::GetDrawData();

// 渲染主视口
ImGui_ImplDX11_RenderDrawData(main_draw_data);

// 渲染额外的视口
if (io.ConfigFlags & ImGuiConfigFlags_ViewportsEnable) {
    ImGui::UpdatePlatformWindows();
    ImGui::RenderPlatformWindowsDefault();
}
```

### 10.3 输入处理

ImGui 提供了一套与平台和后端无关的输入查询 API。这些函数可以在 UI 代码的任何位置调用，不仅限于控件内部。你可以用它们来实现全局快捷键、自定义交互行为、或者根据输入状态动态调整 UI。

输入查询分为两类：键盘输入和鼠标输入。键盘查询包括按键状态（按下、持续按住、释放）和组合键（如 Ctrl+S）。鼠标查询包括按钮状态、位置、拖拽和滚轮。

```cpp
ImGuiIO& io = ImGui::GetIO();

// 检查按键
if (ImGui::IsKeyDown(ImGuiKey_A)) {
    // A 键正被按住
}

if (ImGui::IsKeyPressed(ImGuiKey_Enter)) {
    // Enter 键被按下（只触发一次）
}

if (ImGui::IsKeyReleased(ImGuiKey_Escape)) {
    // Escape 键被释放
}

// 快捷键
if (ImGui::IsKeyChordPressed(ImGuiMod_Ctrl | ImGuiKey_S)) {
    // Ctrl+S 被按下
    SaveFile();
}

// 鼠标输入
if (ImGui::IsMouseDown(ImGuiMouseButton_Left)) {
    // 左键按住
}

if (ImGui::IsMouseClicked(ImGuiMouseButton_Right)) {
    // 右键点击
}

if (ImGui::IsMouseDoubleClicked(ImGuiMouseButton_Left)) {
    // 左键双击
}

// 鼠标拖拽
if (ImGui::IsMouseDragging(ImGuiMouseButton_Left)) {
    ImVec2 delta = ImGui::GetMouseDragDelta();
    // delta 是拖拽的距离
}

// 鼠标滚轮
float wheel = io.MouseWheel;
if (wheel != 0) {
    // 滚轮滚动
}

// 鼠标位置
ImVec2 mouse_pos = io.MousePos;
ImVec2 mouse_delta = io.MouseDelta;
```

### 10.4 焦点和激活

```cpp
// 设置下一项为默认焦点
ImGui::SetKeyboardFocusHere();
ImGui::InputText("Focused Input", buf, sizeof(buf));

// 检查项是否被激活（如正在编辑 InputText）
if (ImGui::IsItemActive()) {
    // 用户正在与这个控件交互
}

// 检查项是否被聚焦
if (ImGui::IsItemFocused()) {
    // 这个控件有键盘焦点
}

// 检查鼠标是否悬停
if (ImGui::IsItemHovered()) {
    // 鼠标悬停在这个控件上
}

// 带延迟的悬停检测
if (ImGui::IsItemHovered(ImGuiHoveredFlags_DelayShort)) {
    // 鼠标停留了一段时间
}

// 检查矩形区域是否被悬停
ImVec2 rect_min = ImGui::GetItemRectMin();
ImVec2 rect_max = ImGui::GetItemRectMax();
if (ImGui::IsMouseHoveringRect(rect_min, rect_max)) {
    // 鼠标在这个矩形内
}
```

### 10.5 剪贴板操作

ImGui 提供了简单的剪贴板读写接口，用于实现复制/粘贴功能。默认情况下，这些函数使用操作系统原生的剪贴板 API（由后端实现），但也可以通过 `ImGuiIO::GetClipboardTextFn` 和 `SetClipboardTextFn` 指针自定义行为。

在实际应用中，剪贴板操作最常见的使用场景是：在控制台中复制日志文本、在属性编辑器中复制数值、在表格中复制单元格内容。注意 `GetClipboardText()` 返回的指针是临时的，如果需要长期保存内容，应该立即复制到自己的缓冲区中。

```cpp
ImGuiIO& io = ImGui::GetIO();

// 获取剪贴板内容
const char* clipboard_text = ImGui::GetClipboardText();

// 设置剪贴板内容
ImGui::SetClipboardText("Hello from ImGui!");
```

### 10.6 数据持久化（.ini 文件）

```cpp
ImGuiIO& io = ImGui::GetIO();

// 启用/禁用 .ini 文件保存
io.IniFilename = "imgui.ini";  // 设置路径，设为 nullptr 禁用

// 手动保存和加载
ImGui::SaveIniSettingsToDisk("custom_layout.ini");
ImGui::LoadIniSettingsFromDisk("custom_layout.ini");

// 保存到内存
size_t size;
const char* ini_data = ImGui::SaveIniSettingsToMemory(&size);

// 从内存加载
ImGui::LoadIniSettingsFromMemory(ini_data, size);
```

---

## 11. 完整示例程序

前面的章节介绍了 ImGui 的各个控件和功能，但真实项目中的界面通常不是孤立地使用某一个控件，而是将多个控件组合成有组织的面板。本章提供三个完整的示例程序，展示如何在实际场景中组合使用 ImGui 的多种功能。

这些示例可以直接复制到你的项目中运行，也可以作为你自己构建界面的起点。

### 11.1 属性编辑器示例

```cpp
// 一个典型的游戏对象属性编辑器
void ShowPropertyEditor()
{
    ImGui::Begin("Property Editor");

    // 对象基本信息
    static char object_name[64] = "Player";
    ImGui::InputText("Name", object_name, sizeof(object_name));

    static int object_id = 1001;
    ImGui::InputInt("ID", &object_id, 1, 100,
        ImGuiInputTextFlags_ReadOnly);

    ImGui::Separator();

    // Transform 组件
    if (ImGui::CollapsingHeader("Transform", ImGuiTreeNodeFlags_DefaultOpen)) {
        static float position[3] = { 0.0f, 0.0f, 0.0f };
        static float rotation[3] = { 0.0f, 0.0f, 0.0f };
        static float scale[3] = { 1.0f, 1.0f, 1.0f };

        ImGui::DragFloat3("Position", position, 0.1f);
        ImGui::DragFloat3("Rotation", rotation, 1.0f);
        ImGui::DragFloat3("Scale", scale, 0.01f);
    }

    // 渲染组件
    if (ImGui::CollapsingHeader("Renderer")) {
        static bool enabled = true;
        static float color[4] = { 1.0f, 1.0f, 1.0f, 1.0f };
        static int render_layer = 0;

        ImGui::Checkbox("Enabled", &enabled);
        ImGui::ColorEdit4("Color", color);

        const char* layers[] = { "Default", "Transparent", "UI", "Overlay" };
        ImGui::Combo("Layer", &render_layer, layers, IM_ARRAYSIZE(layers));
    }

    // 物理组件
    if (ImGui::CollapsingHeader("Physics")) {
        static bool use_gravity = true;
        static float mass = 1.0f;
        static float drag = 0.0f;

        ImGui::Checkbox("Use Gravity", &use_gravity);
        ImGui::DragFloat("Mass", &mass, 0.1f, 0.0f, 1000.0f);
        ImGui::DragFloat("Drag", &drag, 0.01f, 0.0f, 1.0f);

        static int body_type = 0;
        ImGui::RadioButton("Dynamic", &body_type, 0); ImGui::SameLine();
        ImGui::RadioButton("Kinematic", &body_type, 1); ImGui::SameLine();
        ImGui::RadioButton("Static", &body_type, 2);
    }

    ImGui::End();
}
```

### 11.2 控制台示例

控制台是游戏和工具开发中最常用的调试界面之一。这个示例展示了一个功能完整的内嵌控制台，支持：彩色日志输出、文本过滤、自动滚动、命令输入和简单的命令解析。

设计要点：日志使用 `std::vector` 存储，每条日志带颜色信息；显示区域使用 `BeginChild` 创建独立的滚动视图；输入框使用 `EnterReturnsTrue` 标志捕获回车事件；过滤功能在渲染时实时应用，不删除原始数据。

```cpp
#include <vector>
#include <cstring>

struct ConsoleLog {
    std::string text;
    ImVec4 color;
};

static std::vector<ConsoleLog> console_logs;
static char console_input[256] = "";
static bool console_auto_scroll = true;

void AddLog(const char* fmt, ImVec4 color, ...) {
    char buf[1024];
    va_list args;
    va_start(args, color);
    vsnprintf(buf, sizeof(buf), fmt, args);
    va_end(args);
    console_logs.push_back({ buf, color });
}

void ShowConsole()
{
    ImGui::Begin("Console");

    // 工具栏
    if (ImGui::Button("Clear")) {
        console_logs.clear();
    }
    ImGui::SameLine();
    ImGui::Checkbox("Auto-scroll", &console_auto_scroll);
    ImGui::SameLine();

    // 过滤输入
    static char filter[64] = "";
    ImGui::InputText("Filter", filter, sizeof(filter));
    ImGui::Separator();

    // 日志显示区域
    ImGui::BeginChild("ScrollingRegion", ImVec2(0, -ImGui::GetFrameHeightWithSpacing()));

    for (const auto& log : console_logs) {
        // 应用过滤
        if (filter[0] != '\0' && log.text.find(filter) == std::string::npos)
            continue;

        ImGui::PushStyleColor(ImGuiCol_Text, log.color);
        ImGui::TextUnformatted(log.text.c_str());
        ImGui::PopStyleColor();
    }

    if (console_auto_scroll && ImGui::GetScrollY() >= ImGui::GetScrollMaxY()) {
        ImGui::SetScrollHereY(1.0f);
    }

    ImGui::EndChild();

    // 输入区域
    ImGui::Separator();
    ImGui::SetNextItemWidth(-1);
    if (ImGui::InputText("##ConsoleInput", console_input, sizeof(console_input),
        ImGuiInputTextFlags_EnterReturnsTrue)) {
        // 执行命令
        AddLog((std::string("> ") + console_input).c_str(),
               ImVec4(1, 1, 1, 1));

        // 模拟命令执行
        if (strcmp(console_input, "help") == 0) {
            AddLog("Available commands: help, clear, echo",
                   ImVec4(0, 1, 0, 1));
        } else if (strncmp(console_input, "echo ", 5) == 0) {
            AddLog(console_input + 5, ImVec4(1, 1, 1, 1));
        }

        console_input[0] = '\0';
        ImGui::SetKeyboardFocusHere(-1);  // 保持焦点
    }

    ImGui::End();
}
```

### 11.3 性能监控器

性能监控面板是实时应用（尤其是游戏和渲染引擎）不可或缺的工具。这个示例展示了一个典型的性能监控界面，包含 FPS 显示、帧时间历史图、内存使用直方图和 ImGui 自身绘制统计。

历史图使用环形缓冲区实现——新数据覆盖最旧的数据，通过 `values_offset` 记录写入位置。`PlotLines` 和 `PlotHistogram` 函数会自动处理这种环形数据，只需要传入正确的偏移量即可。

```cpp
void ShowPerformanceMonitor()
{
    ImGui::Begin("Performance");

    // FPS 显示
    ImGuiIO& io = ImGui::GetIO();
    ImGui::Text("FPS: %.1f", io.Framerate);
    ImGui::Text("Frame Time: %.3f ms", 1000.0f / io.Framerate);

    // 帧时间历史图
    static float values[90] = {};
    static int values_offset = 0;
    values[values_offset] = 1000.0f / io.Framerate;
    values_offset = (values_offset + 1) % IM_ARRAYSIZE(values);

    float average = 0;
    for (float v : values) average += v;
    average /= IM_ARRAYSIZE(values);

    char overlay[32];
    sprintf(overlay, "Avg: %.3f ms", average);

    ImGui::PlotLines("Frame Time", values, IM_ARRAYSIZE(values),
        values_offset, overlay, 0.0f, 50.0f, ImVec2(0, 80));

    // 内存使用（示例）
    ImGui::Separator();
    ImGui::Text("Memory:");
    static float memory_history[90] = {};
    static int mem_offset = 0;
    memory_history[mem_offset] = /* 你的内存统计 */ 0;
    mem_offset = (mem_offset + 1) % IM_ARRAYSIZE(memory_history);

    ImGui::PlotHistogram("Memory (MB)", memory_history,
        IM_ARRAYSIZE(memory_history), mem_offset,
        nullptr, 0.0f, 1024.0f, ImVec2(0, 60));

    // ImGui 自身统计
    ImGui::Separator();
    ImGui::Text("Draw Calls: %d", io.MetricsRenderWindows);
    ImGui::Text("Vertices: %d", io.MetricsRenderVertices);
    ImGui::Text("Indices: %d", io.MetricsRenderIndices);

    ImGui::End();
}
```

---

## 12. 最佳实践与性能优化

ImGui 的即时模式设计使得构建界面非常直观，但也带来了一些需要注意的问题。本章总结在实际项目中使用 ImGui 时最常见的最佳实践、性能优化技巧和常见陷阱。

遵循这些建议可以帮助你写出更健壮、更高效的 ImGui 代码，避免调试时花费大量时间追踪难以复现的 Bug。

### 12.1 使用 ID 栈避免冲突

```cpp
// 当同一标签被多次使用时，需要区分 ID
for (int i = 0; i < 10; i++) {
    ImGui::PushID(i);  // 将 i 压入 ID 栈

    ImGui::Button("Action");  // 每个按钮有唯一的 ID

    if (ImGui::BeginPopupContextItem()) {
        ImGui::Text("Popup for item %d", i);
        ImGui::EndPopup();
    }

    ImGui::PopID();  // 弹出 ID
}

// 也可以使用字符串或指针作为 ID
ImGui::PushID("unique_string");
ImGui::PushID(my_object_ptr);
ImGui::PushID(label, label + strlen(label));  // 范围
```

### 12.2 条件渲染

```cpp
// 只在需要时构建 UI
static bool show_debug = false;
if (show_debug) {
    ImGui::Begin("Debug");
    // ... 大量调试 UI
    ImGui::End();
}

// 使用 Begin 的返回值避免不必要的绘制
if (ImGui::Begin("Conditional Window")) {
    // 只有窗口可见时才执行
    ExpensiveOperation();
}
ImGui::End();
```

### 12.3 批量绘制

每个 ImGui 控件函数内部都有一定量的固定开销——计算布局、处理输入、注册到绘制列表等。当需要显示大量相似元素时，这些开销会累积。如果性能分析显示 UI 渲染占用了过多时间，可以考虑使用 ImDrawList 直接绘制。

ImDrawList API 绕过控件的完整处理流程，直接添加顶点和绘制命令。这种方式不适合需要交互的元素（因为没有输入处理），但对于纯显示内容（如日志列表、调试信息、波形图）可以显著提升性能。

```cpp
// 避免在循环中频繁调用单独的绘制函数
// 不好的做法：
for (int i = 0; i < 1000; i++) {
    ImGui::Text("Item %d", i);  // 每个 Text 调用都有开销
}

// 更好的做法：使用 ImDrawList 批量绘制
ImDrawList* draw_list = ImGui::GetWindowDrawList();
ImVec2 pos = ImGui::GetCursorScreenPos();
for (int i = 0; i < 1000; i++) {
    char buf[32];
    sprintf(buf, "Item %d", i);
    draw_list->AddText(pos, IM_COL32(255, 255, 255, 255), buf);
    pos.y += ImGui::GetTextLineHeight();
}
```

### 12.4 减少状态变化

`PushStyleColor` 和 `PushStyleVar` 函数虽然开销很小，但在循环中频繁调用仍然会产生不必要的负担。更关键的是，每次 Push/Pop 都会修改 ImGui 内部的样式栈，过多的栈操作可能影响缓存效率。

更好的做法是在循环外统一设置样式，在循环结束后统一恢复。如果不同元素需要不同的样式，考虑将同样式的元素分组处理。

```cpp
// 批量修改颜色时，使用 PushStyleColor 的数组形式
ImGui::PushStyleColor(ImGuiCol_Button, red);
for (int i = 0; i < 10; i++) {
    ImGui::Button("Red Button");
}
ImGui::PopStyleColor();

// 而不是每个按钮都 Push/Pop
for (int i = 0; i < 10; i++) {
    ImGui::PushStyleColor(ImGuiCol_Button, red);  // 低效
    ImGui::Button("Red Button");
    ImGui::PopStyleColor();
}
```

### 12.5 避免字符串格式化开销

`ImGui::Text()` 内部使用 `vsnprintf` 来格式化字符串，这意味着它必须解析格式字符串、处理各种格式说明符、进行数值到字符串的转换。对于每帧都在变化的文本（如帧计数器、计时器），这些格式化开销虽然不大，但在性能敏感的代码路径中值得优化。

`TextUnformatted()` 完全跳过格式化步骤，直接渲染传入的字符串。如果你的字符串已经提前格式化好了，或者本来就是纯文本，使用它可以获得略微更好的性能。对于静态文本（如标签、说明），编译器通常会优化掉格式化开销，所以两种方式的差别可以忽略。

```cpp
// 频繁变化的文本，使用 TextUnformatted 避免格式化
static char label[32];
sprintf(label, "Frame: %d", frame_number);
ImGui::TextUnformatted(label);  // 比 Text("Frame: %d", frame_number) 略快

// 对于静态文本，直接传递字符串字面量
ImGui::Text("Static Label");
```

### 12.6 使用 ChildWindow 分割内容

大型窗口如果包含大量控件，ImGui 需要处理所有控件的输入和布局计算，即使它们远在可视区域之外。使用 `BeginChild` 将内容分割为多个子窗口，可以利用子窗口的裁剪机制——不可见的子窗口内容会被跳过，不会参与布局和输入处理。

这种优化在编辑器类应用中尤为重要。例如，一个包含左侧文件树、中间代码编辑区、右侧属性面板的 IDE 式界面，如果不使用子窗口，整个界面的所有控件都会参与每帧的计算。使用三个 `BeginChild` 后，只有当前可见的子窗口内容会被处理。

```cpp
// 大型窗口使用子窗口可以优化裁剪
ImGui::Begin("Large Window");

// 左侧面板
ImGui::BeginChild("LeftPanel", ImVec2(200, 0), true);
// 绘制左侧面板内容
ImGui::EndChild();

ImGui::SameLine();

// 右侧面板
ImGui::BeginChild("RightPanel", ImVec2(0, 0), true);
// 绘制右侧面板内容
ImGui::EndChild();

ImGui::End();
```

### 12.7 延迟加载和虚拟化

当列表项数量超过几百个时，即使使用子窗口，每帧遍历所有项仍然会有不可忽略的开销。`ImGuiListClipper` 是 ImGui 提供的虚拟化工具，它只计算并渲染当前可见区域内的项，无论总项数是多少。

Clipper 的工作方式是：你告诉它总共有多少项，然后在一个 `while (clipper.Step())` 循环中，只渲染 `DisplayStart` 到 `DisplayEnd` 范围内的项。Clipper 会自动计算当前滚动位置下哪些项可见，即使你有一百万项，每帧也只会渲染几十项。

```cpp
// 对于大量列表项，使用 Clipper 只渲染可见项
if (ImGui::BeginListBox("Virtual List")) {
    ImGuiListClipper clipper;
    clipper.Begin(10000);  // 总共 10000 项
    while (clipper.Step()) {
        for (int i = clipper.DisplayStart; i < clipper.DisplayEnd; i++) {
            ImGui::Selectable("Item %d", i);
        }
    }
    ImGui::EndListBox();
}
```

### 12.8 常用调试技巧

ImGui 内置了一系列调试和诊断窗口，这些窗口不仅是学习 ImGui 的最佳资料，也是排查 UI 问题的有力工具。当你遇到布局异常、性能问题或不确定某个控件的正确用法时，优先查看这些内置窗口。

`ShowDemoWindow()` 尤其值得反复研读——它展示了几乎所有 ImGui 特性的使用方式，每个示例都是实际可运行的代码。当你不确定如何实现某个效果时，打开 Demo Window 找到类似的示例，然后查看 `imgui_demo.cpp` 中对应的实现代码。

```cpp
// 显示 ImGui 演示窗口（非常有教育意义）
ImGui::ShowDemoWindow();

// 显示样式编辑器
ImGui::ShowStyleEditor();

// 显示关于窗口
ImGui::ShowAboutWindow();

// 显示指标窗口
ImGui::ShowMetricsWindow();

// 显示调试日志
ImGui::ShowDebugLogWindow();

// 检查 ID 栈
ImGui::ShowIDStackToolWindow();

// 显示字体查看器
ImGui::ShowFontSelector("Font");
```

### 12.9 常见错误

即时模式的编程模型虽然简洁，但也带来了一些特有的陷阱。以下是使用 ImGui 时最常见的错误模式，理解它们的成因可以帮助你写出更健壮的代码。

核心原则是：**每个 `Begin` 必须有对应的 `End`，每个 `Push` 必须有对应的 `Pop`**。由于 ImGui 每帧都重新执行所有代码，控制流中的任何分支都必须保证配对的完整性。

```cpp
// 错误：Begin 没有对应的 End
if (some_condition) {
    ImGui::Begin("Window");
}  // 如果 some_condition 为 false，Begin 不会被调用
// 但下面的 End 总是被调用！
ImGui::End();  // 错误！

// 正确：
if (some_condition) {
    ImGui::Begin("Window");
    // ... UI ...
    ImGui::End();
}

// 错误：在 Begin/BeginChild 之间返回
void DrawUI() {
    ImGui::Begin("Window");
    if (some_error) {
        ImGui::Text("Error!");
        return;  // 忘记调用 End()！
    }
    ImGui::End();
}

// 正确：使用 RAII 或确保所有路径都调用 End
void DrawUI() {
    if (!ImGui::Begin("Window")) {
        ImGui::End();
        return;
    }
    // ... UI ...
    ImGui::End();
}

// 错误：在循环中修改正在迭代的数据
for (auto& item : items) {
    if (ImGui::Button("Delete")) {
        items.erase(std::remove(items.begin(), items.end(), item), items.end());
        // 迭代器失效！
    }
}

// 正确：标记要删除的项，循环结束后再删除
int item_to_delete = -1;
for (int i = 0; i < items.size(); i++) {
    if (ImGui::Button("Delete")) {
        item_to_delete = i;
    }
}
if (item_to_delete >= 0) {
    items.erase(items.begin() + item_to_delete);
}
```

---

## 附录

### 附录 A：ImGui 版本历史

| 版本 | 发布日期 | 主要特性 |
|------|----------|----------|
| 1.60 | 2018 | 表格系统改进 |
| 1.70 | 2019 | 导航改进，Tab 栏 |
| 1.80 | 2021 | 表格正式版，TreeNode 改进 |
| 1.90 | 2023 | 新输入系统，按键和弦 |
| 2.0  | TBD  | 未来版本 |

### 附录 B：资源链接

- **官方仓库**：https://github.com/ocornut/imgui
- **官方文档**：https://github.com/ocornut/imgui/wiki
- **交互式演示**：https://pbrfrat.com/post/imgui_playground/
- **Awesome ImGui**：https://github.com/ocornut/imgui/wiki/Useful-Extensions

### 附录 C：常用第三方扩展

| 扩展 | 描述 |
|------|------|
| **imnodes** | 节点编辑器（蓝图风格） |
| **implot** | 绘图库（线图、柱状图、散点图等） |
| **imguizmo** | 3D 操作器（平移、旋转、缩放） |
| **imgui-filebrowser** | 文件浏览器对话框 |
| **imgui_markdown** | Markdown 渲染 |
| **imgui_club** | 各种有用的代码片段 |

---

> 本文档基于 Dear ImGui 1.90+ 版本编写。部分 API 可能因版本不同而有所差异，请以官方文档为准。
