# C++ 各版本新特性总结（C++11 ~ C++26）

## C++11 (2011) — 现代 C++ 的基石

### 1. auto 类型推导
**说明**：编译器自动推导变量类型，简化代码书写。

**用法举例**：
```cpp
auto i = 42;           // int
auto d = 3.14;         // double
auto s = "hello";      // const char*
auto v = std::vector<int>{1, 2, 3};
for (auto it = v.begin(); it != v.end(); ++it) { }
```

**使用场景**：
- 迭代器类型过长时（如 `std::map<std::string, std::vector<int>>::iterator`）
- Lambda 表达式类型无法直接写出
- 模板编程中返回值类型复杂的情况

---

### 2. decltype
**说明**：获取表达式的类型，不做实际计算。

**用法举例**：
```cpp
int x = 42;
decltype(x) y = 10;        // y 的类型是 int

auto add = [](int a, int b) { return a + b; };
decltype(add) add2 = add;  // 获取 lambda 的类型
```

**使用场景**：
- 需要根据表达式推导类型，但不初始化变量
- 配合模板实现泛型代码时确定返回类型

---

### 3. 范围 for 循环 (Range-based for)
**说明**：简洁地遍历容器或数组。

**用法举例**：
```cpp
std::vector<int> vec = {1, 2, 3, 4, 5};
for (const auto& elem : vec) {
    std::cout << elem << " ";
}

// 修改元素
for (auto& elem : vec) {
    elem *= 2;
}
```

**使用场景**：
- 遍历整个容器且不需要迭代器时
- 数组、容器、初始化列表的统一遍历方式

---

### 4. Lambda 表达式
**说明**：内联定义匿名函数对象。

**用法举例**：
```cpp
auto square = [](int x) { return x * x; };

// 带捕获
int factor = 2;
auto multiply = [factor](int x) { return x * factor; };

// 捕获所有变量
auto func = [=]() { return factor; };     // 值捕获
auto func2 = [&]() { factor++; };         // 引用捕获

// 泛型 lambda (C++14 更完善，C++11 需指定参数类型)
std::sort(vec.begin(), vec.end(), [](int a, int b) { return a > b; });
```

**使用场景**：
- STL 算法自定义比较/操作函数
- 回调函数、事件处理
- 局部作用域的简短函数逻辑

---

### 5. 右值引用与移动语义
**说明**：通过 `&&` 区分右值，实现资源转移而非拷贝，大幅提升性能。

**用法举例**：
```cpp
class MyString {
    char* data;
public:
    // 拷贝构造函数
    MyString(const MyString& other) : data(new char[strlen(other.data) + 1]) {
        strcpy(data, other.data);
    }
    // 移动构造函数
    MyString(MyString&& other) noexcept : data(other.data) {
        other.data = nullptr;  // 转移所有权
    }
};

MyString createString() { return MyString("hello"); }
MyString s = createString();  // 调用移动构造函数，避免深拷贝
```

**使用场景**：
- 管理动态资源的类（string、vector、智能指针等）
- 函数返回大对象时
- 容器插入临时对象时（`emplace_back`）

---

### 6. std::move 与 std::forward
**说明**：`std::move` 将左值转为右值引用以便移动；`std::forward` 完美转发保持参数的值类别。

**用法举例**：
```cpp
std::vector<int> v1 = {1, 2, 3};
std::vector<int> v2 = std::move(v1);  // v1 变为空，资源移动到 v2

// 完美转发
template<typename T, typename Arg>
std::shared_ptr<T> make_shared(Arg&& arg) {
    return std::shared_ptr<T>(new T(std::forward<Arg>(arg)));
}
```

**使用场景**：
- 显式请求移动语义
- 工厂函数、包装函数中保持参数的原始类型和值类别

---

### 7. 智能指针
**说明**：自动内存管理，替代裸指针。

**用法举例**：
```cpp
// unique_ptr: 独占所有权
auto up = std::make_unique<int>(42);  // C++14 推荐，C++11 需直接 new
std::unique_ptr<int> up2(new int(10));

// shared_ptr: 共享所有权，引用计数
auto sp1 = std::make_shared<int>(20);
auto sp2 = sp1;  // 引用计数 +1

// weak_ptr: 弱引用，不增加引用计数
std::weak_ptr<int> wp = sp1;
if (auto locked = wp.lock()) {  // 使用前需 lock
    std::cout << *locked;
}
```

**使用场景**：
- `unique_ptr`：资源唯一拥有者（如工厂返回的对象）
- `shared_ptr`：共享资源（如图、树结构中的节点）
- `weak_ptr`：打破循环引用（如观察者模式）

---

### 8. nullptr
**说明**：类型安全的空指针，替代 `NULL` 和 `0`。

**用法举例**：
```cpp
void foo(int x) { }
void foo(int* p) { }

foo(NULL);      // 歧义！可能调用 foo(int)
foo(nullptr);   // 明确调用 foo(int*)

int* p = nullptr;
if (p == nullptr) { }
```

**使用场景**：
- 任何需要使用空指针的地方，避免重载解析歧义

---

### 9. constexpr
**说明**：编译期常量表达式，可在编译期计算。

**用法举例**：
```cpp
constexpr int square(int x) { return x * x; }
constexpr int arr_size = square(5);  // 25，编译期计算
int arr[arr_size];                    // 合法，arr_size 是编译期常量

constexpr int factorial(int n) {
    return n <= 1 ? 1 : n * factorial(n - 1);
}
```

**使用场景**：
- 数组大小、模板参数、枚举值等需要编译期常量的地方
- 提高运行时性能，将计算移到编译期

---

### 10. 初始化列表 (Initializer List)
**说明**：统一初始化语法，支持列表初始化。

**用法举例**：
```cpp
int arr[] = {1, 2, 3};
std::vector<int> vec = {1, 2, 3};  // 或 {1, 2, 3}
std::map<std::string, int> m = {{"a", 1}, {"b", 2}};

// 防止窄化转换
int x = 3.14;      // 警告但允许
int y{3.14};       // 编译错误！double 窄化为 int

// 默认初始化
int z{};           // 0
```

**使用场景**：
- 统一所有类型的初始化语法
- 禁止隐式的窄化转换，提高安全性

---

### 11. 委托构造函数
**说明**：构造函数可以调用同一个类的其他构造函数。

**用法举例**：
```cpp
class Rectangle {
    int width, height;
public:
    Rectangle(int w, int h) : width(w), height(h) {}
    Rectangle() : Rectangle(0, 0) {}           // 委托给双参构造
    Rectangle(int side) : Rectangle(side, side) {}  // 正方形
};
```

**使用场景**：
- 减少构造函数中的重复代码
- 一个主构造函数集中初始化逻辑

---

### 12. 继承构造函数
**说明**：派生类可以直接使用基类的构造函数。

**用法举例**：
```cpp
class Base {
public:
    Base(int x) {}
    Base(int x, double y) {}
};

class Derived : public Base {
public:
    using Base::Base;  // 继承所有构造函数
    // 隐式生成: Derived(int x) : Base(x) {}
};
```

**使用场景**：
- 派生类不添加新成员，无需重复编写转发构造函数

---

### 13. 类型别名 (using)
**说明**：替代 typedef，更清晰的语法，支持模板别名。

**用法举例**：
```cpp
// 替代 typedef
typedef std::vector<int> IntVec;
using IntVec2 = std::vector<int>;  // 等价但更清晰

// 模板别名（typedef 做不到）
template<typename T>
using Vec = std::vector<T>;

Vec<int> v;       // std::vector<int>
Vec<std::string> vs;
```

**使用场景**：
- 简化复杂类型名
- 创建模板化的类型别名

---

### 14. 变长模板 (Variadic Templates)
**说明**：模板参数数量可变。

**用法举例**：
```cpp
// 递归终止
void print() {}

template<typename T, typename... Args>
void print(T first, Args... args) {
    std::cout << first << " ";
    print(args...);  // 递归展开
}

print(1, "hello", 3.14);  // 输出: 1 hello 3.14

// sizeof... 获取参数个数
template<typename... Args>
void count(Args... args) {
    constexpr std::size_t n = sizeof...(Args);
    std::cout << n << "\n";
}
```

**使用场景**：
- 实现类型安全的变参函数（如 `printf`、`make_unique`）
- 元编程中的类型列表处理

---

### 15. 模板别名与 SFINAE
**说明**：Substitution Failure Is Not An Error，模板替换失败不算编译错误。

**用法举例**：
```cpp
template<typename T>
typename std::enable_if<std::is_integral<T>::value, T>::type
check(T t) {
    return t;
}

check(42);        // OK
check(3.14);      // 编译错误：没有匹配的函数
```

**使用场景**：
- 根据类型特性选择函数重载
- 编译期类型检查和限制

---

### 16. 强类型枚举 (enum class)
**说明**：枚举值有作用域限制，且底层类型可指定。

**用法举例**：
```cpp
enum class Color : unsigned char { Red, Green, Blue };
enum class Status { OK, Error };

Color c = Color::Red;     // 必须加作用域
// Color c2 = Red;        // 错误！Red 不在当前作用域

// 不隐式转换为 int
int x = static_cast<int>(Color::Red);
```

**使用场景**：
- 避免枚举名称冲突
- 类型安全，防止与整数混用

---

### 17. 静态断言 (static_assert)
**说明**：编译期断言，条件不满足时产生编译错误。

**用法举例**：
```cpp
static_assert(sizeof(int) == 4, "int must be 4 bytes");
static_assert(sizeof(long) >= 4, "long too small");

template<typename T>
class Container {
    static_assert(std::is_copy_constructible<T>::value,
                  "T must be copy constructible");
};
```

**使用场景**：
- 检查平台/编译器假设
- 模板约束的编译期检查

---

### 18. 类型特征 (Type Traits)
**说明**：编译期查询和操作类型属性。

**用法举例**：
```cpp
static_assert(std::is_integral<int>::value, "");
static_assert(std::is_pointer<int*>::value, "");
static_assert(std::is_same<int, int32_t>::value, "");

using decay_t = std::decay<int&>::type;  // int
using add_ref = std::add_lvalue_reference<int>::type;  // int&
```

**使用场景**：
- 模板元编程
- 编译期类型分支选择

---

### 19. 统一初始化 / 列表初始化
**说明**：花括号 `{}` 统一用于各种初始化场景。

**用法举例**：
```cpp
class Point {
public:
    int x, y;
    Point(int a, int b) : x(a), y(b) {}
};

Point p{1, 2};                    // 列表初始化
Point p2 = {1, 2};                // 拷贝列表初始化
std::vector<Point> pts = {{1, 2}, {3, 4}};
```

**使用场景**：
- 统一 POD 和类类型的初始化语法
- 避免最令人头痛的解析（Most Vexing Parse）

---

### 20. 原生字符串字面量 (R"()")
**说明**：原始字符串，不处理转义字符。

**用法举例**：
```cpp
std::string path = R"(C:\Program Files\App\file.txt)";
std::string json = R"({"name": "cpp", "version": 11})";
std::string regex = R"(\d+\.\d+)";

// 含括号的自定义分隔符
std::string str = R"xxx(包含)的字符串)xxx";
```

**使用场景**：
- 文件路径、正则表达式、JSON/XML 文本
- 任何包含大量反斜杠的字符串

---

### 21. 自定义字面量 (UDL — C++11 基础，C++14/17 扩展)
**说明**：为用户定义类型创建后缀字面量。

**用法举例**：
```cpp
// C++11 标准字面量
auto s = "hello"s;     // std::string
auto h = 60min;        // std::chrono::minutes
```

**使用场景**：
- 为自定义类型提供自然语法（C++14/17 可自定义后缀）

---

### 22. 并发支持
**说明**：标准库引入线程、互斥锁、条件变量、future 等。

**用法举例**：
```cpp
#include <thread>
#include <mutex>
#include <future>
#include <atomic>

std::mutex mtx;
int counter = 0;

void increment() {
    std::lock_guard<std::mutex> lock(mtx);  // RAII 锁
    ++counter;
}

std::thread t1(increment);
std::thread t2(increment);
t1.join();
t2.join();

// async
auto fut = std::async(std::launch::async, []() { return 42; });
int result = fut.get();

// atomic
std::atomic<int> atomic_counter{0};
atomic_counter.fetch_add(1);
```

**使用场景**：
- 多线程并行计算
- 异步任务执行
- 无锁数据结构

---

### 23. 正则表达式 (std::regex)
**说明**：标准正则表达式库。

**用法举例**：
```cpp
#include <regex>

std::string text = "Email: test@example.com";
std::regex pattern(R"((\w+@\w+\.\w+))");
std::smatch match;

if (std::regex_search(text, match, pattern)) {
    std::cout << match[0];  // test@example.com
}

// 替换
std::string result = std::regex_replace(text, pattern, "[hidden]");
```

**使用场景**：
- 文本验证、提取、替换

---

### 24. 随机数生成器
**说明**：取代 `rand()`，提供高质量的随机数。

**用法举例**：
```cpp
#include <random>

std::random_device rd;                       // 硬件熵源
std::mt19937 gen(rd());                      // Mersenne Twister 引擎
std::uniform_int_distribution<> dis(1, 6);   // 1-6 均匀分布

int dice = dis(gen);  // 模拟掷骰子

std::normal_distribution<> normal(0.0, 1.0); // 正态分布
double x = normal(gen);
```

**使用场景**：
- 科学计算、游戏、模拟仿真
- 需要特定分布的随机数时

---

### 25. std::tuple
**说明**：固定大小的异构值集合。

**用法举例**：
```cpp
std::tuple<int, std::string, double> t = std::make_tuple(1, "hello", 3.14);

int x = std::get<0>(t);
std::string s = std::get<1>(t);

// C++14 可按类型获取（若唯一）
// double d = std::get<double>(t);

// 解包
tie(x, s, std::ignore) = t;
```

**使用场景**：
- 函数返回多个值
- 异构数据的临时组合

---

### 26. std::array
**说明**：固定大小数组，有边界检查，支持 STL 接口。

**用法举例**：
```cpp
std::array<int, 5> arr = {1, 2, 3, 4, 5};
arr[0] = 10;
int x = arr.at(10);  // 抛出 out_of_range 异常

for (const auto& e : arr) {
    std::cout << e << " ";
}
```

**使用场景**：
- 需要固定大小且要有 STL 接口的数组
- 替代原生数组，获得边界安全

---

### 27. 右值引用与引用折叠
**说明**：`&&` 右值引用，配合引用折叠规则实现完美转发。

**使用场景**：
- 见第 5、6 条（移动语义、std::forward）

---

### 28. noexcept
**说明**：指定函数不抛出异常，用于优化和保证。

**用法举例**：
```cpp
void safe_func() noexcept { }
void maybe_throw() noexcept(false) { }

// 条件性 noexcept
template<typename T>
void func() noexcept(noexcept(T())) { }
```

**使用场景**：
- 移动构造函数/移动赋值运算符标记
- 异常安全保证（强保证、不抛异常保证）

---

### 29. 显式默认/删除函数 (=default, =delete)
**说明**：显式要求编译器生成默认函数，或禁止生成某函数。

**用法举例**：
```cpp
class NonCopyable {
public:
    NonCopyable() = default;
    NonCopyable(const NonCopyable&) = delete;           // 禁止拷贝
    NonCopyable& operator=(const NonCopyable&) = delete; // 禁止赋值
};

class POD {
public:
    POD() = default;           // 显式使用默认构造
    POD(const POD&) = default; // 显式使用默认拷贝
};
```

**使用场景**：
- 禁止拷贝语义（单例、资源句柄）
- 显式声明使用默认实现，增强可读性

---

### 30. 匿名联合体和匿名结构体成员
**说明**：更灵活的数据布局。

**使用场景**：
- 底层数据结构设计（较少见）

---

### 31. 对齐支持 (alignas, alignof)
**说明**：控制和对齐查询。

**用法举例**：
```cpp
alignas(16) char buffer[64];  // 16 字节对齐
constexpr std::size_t a = alignof(int);  // int 的对齐要求

struct alignas(32) Vec8 {
    float data[8];
};
```

**使用场景**：
- SIMD 数据对齐
- 硬件或协议要求特定对齐时

---

### 32. Unicode 字符串支持
**说明**：支持 UTF-8/16/32 字符串字面量。

**用法举例**：
```cpp
const char* utf8 = u8"中文";       // UTF-8 (C++11, C++20 改类型为 char8_t)
const char16_t* utf16 = u"中文";   // UTF-16
const char32_t* utf32 = U"中文";   // UTF-32
```

**使用场景**：
- 国际化应用

---

### 33.  long long 类型
**说明**：至少 64 位的整数类型。

**用法举例**：
```cpp
long long big = 9223372036854775807LL;
unsigned long long ubig = 18446744073709551615ULL;
```

**使用场景**：
- 需要大于 32 位整数范围时

---

### 34. 垃圾回收接口（声明，未强制实现）
**说明**：提供了 `std::declare_reachable` 等接口，但几乎没有编译器实现。

---

### 35. std::function 与 std::bind
**说明**：类型安全的可调用对象包装器和参数绑定。

**用法举例**：
```cpp
std::function<int(int, int)> op = std::plus<int>();
int r = op(2, 3);  // 5

op = [](int a, int b) { return a * b; };
r = op(2, 3);  // 6

// bind
auto add5 = std::bind(std::plus<int>(), 5, std::placeholders::_1);
int x = add5(10);  // 15，相当于 5 + 10
```

**使用场景**：
- 回调机制、事件系统
- 将成员函数绑定到对象
- 函数参数偏应用（currying）

---

### 36. 元组解包与 std::tie
**说明**：将元组元素绑定到变量。

**用法举例**：
```cpp
auto t = std::make_tuple(1, "hello", 3.14);
int x; std::string s; double d;
std::tie(x, s, d) = t;

// 忽略某些元素
std::tie(x, std::ignore, d) = t;
```

**使用场景**：
- 从 tuple 提取值
- 多返回值解包（C++17 structured binding 更优）

---

### 37. 枚举前置声明
**说明**：在知道底层类型时可以前置声明枚举。

**用法举例**：
```cpp
enum class Color : int;  // 前置声明

void foo(Color c);       // 使用前置声明

enum class Color : int { Red, Green, Blue };  // 定义
```

**使用场景**：
- 减少头文件依赖，降低编译时间

---

### 38. 局部/匿名类型作为模板参数
**说明**：C++11 前局部类不能作模板参数。

**用法举例**：
```cpp
void func() {
    struct Local { int value; };
    std::vector<Local> vec;  // C++11 开始合法
}
```

---

### 39. explicit 转换运算符
**说明**：禁止隐式调用转换运算符。

**用法举例**：
```cpp
class SafeBool {
public:
    explicit operator bool() const { return true; }
};

SafeBool sb;
// bool b = sb;   // 错误！隐式转换被禁止
bool b = static_cast<bool>(sb);  // OK
```

**使用场景**：
- 防止意外的隐式类型转换

---

## C++14 (2014) — 对 C++11 的补全

### 1. 泛型 Lambda
**说明**：Lambda 参数可用 `auto`。

**用法举例**：
```cpp
auto lambda = [](auto x, auto y) { return x + y; };
lambda(1, 2);        // int
lambda(1.0, 2.0);    // double
lambda(std::string("a"), "b");  // string
```

**使用场景**：
- 需要处理多种类型的通用 Lambda
- 模板函数的局部辅助逻辑

---

### 2. Lambda 捕获初始化
**说明**：在捕获列表中初始化新变量。

**用法举例**：
```cpp
int x = 10;
auto lambda = [value = x + 5]() { return value; };  // 捕获并计算新值

// 移动捕获（unique_ptr 等）
auto ptr = std::make_unique<int>(42);
auto lambda2 = [p = std::move(ptr)]() { return *p; };
```

**使用场景**：
- Lambda 内只需要某表达式的结果，而非变量本身
- 对只移类型进行捕获

---

### 3. 变量模板
**说明**：模板化的变量。

**用法举例**：
```cpp
template<typename T>
constexpr T pi = T(3.1415926535897932385);

double area = pi<double> * r * r;
float area_f = pi<float> * r * r;

// 类型特征常量
template<typename T>
constexpr bool is_integral_v = std::is_integral<T>::value;
```

**使用场景**：
- 类型依赖的常量
- 简化类型特征使用（C++17 已标准化 `_v` 后缀）

---

### 4. 二进制字面量
**说明**：以 `0b` 或 `0B` 开头的二进制整数。

**用法举例**：
```cpp
int flags = 0b1010'1100;  // 十进制 172
int mask = 0b1111'0000;   // 位掩码
```

**使用场景**：
- 位操作、标志位定义时更直观

---

### 5. 数字分隔符 (')
**说明**：单引号作为数字分隔符，提高可读性。

**用法举例**：
```cpp
int billion = 1'000'000'000;
long long hex = 0xDEAD'BEEF;
int binary = 0b1010'1111'0000;
double pi = 3.141'592'653'589;
```

**使用场景**：
- 大数字的可读性提升

---

### 6. 放宽的 constexpr 限制
**说明**：constexpr 函数可包含更多语句。

**用法举例**：
```cpp
constexpr int factorial(int n) {
    int result = 1;      // C++11 不允许变量声明
    for (int i = 1; i <= n; ++i) {  // C++11 不允许循环
        result *= i;     // C++11 只允许 return 语句
    }
    return result;
}
```

**使用场景**：
- 更复杂的编译期计算

---

### 7. constexpr 成员函数默认隐式 const
**说明**：C++14 中 constexpr 成员函数不再隐式 const（C++11 隐式 const，会限制修改成员）。

**使用场景**：
- constexpr 类可以修改自身状态（编译期可变的语义）

---

### 8. std::make_unique
**说明**：创建 unique_ptr 的安全方式。

**用法举例**：
```cpp
auto p = std::make_unique<int>(42);
auto arr = std::make_unique<int[]>(100);  // 数组版本
```

**使用场景**：
- 替代 `new`，避免内存泄漏，异常安全

---

### 9. std::integer_sequence
**说明**：编译期整数序列。

**用法举例**：
```cpp
template<typename T, T... Ints>
void print_sequence(std::integer_sequence<T, Ints...>) {
    ((std::cout << Ints << " "), ...);  // C++17 fold expression
}

print_sequence(std::make_index_sequence<5>{});  // 0 1 2 3 4
```

**使用场景**：
- 展开参数包时的索引生成
- 元编程中的序列操作

---

### 10. std::exchange
**说明**：替换值并返回旧值。

**用法举例**：
```cpp
std::unique_ptr<int> p(new int(5));
auto old = std::exchange(p, nullptr);  // p 变为 nullptr，old 持有原值

// 移动构造中的用法
MyClass(MyClass&& other) noexcept
    : ptr(std::exchange(other.ptr, nullptr)) {}
```

**使用场景**：
- 实现移动语义时简化代码
- 原子地获取并清除值

---

### 11. std::quoted
**说明**：字符串的带引号 IO 操作。

**用法举例**：
```cpp
std::stringstream ss;
std::string name = "hello world";
ss << std::quoted(name);  // 输出: "hello world"

std::string restored;
ss >> std::quoted(restored);  // 读取带引号的字符串
```

**使用场景**：
- CSV、配置文件等含空格的字符串 IO

---

### 12. 返回类型推导（普通函数）
**说明**：普通函数可用 `auto` 推导返回类型。

**用法举例**：
```cpp
auto add(int a, int b) {  // C++14 开始支持
    return a + b;
}

// 多返回语句需一致
auto abs(int x) {
    if (x >= 0) return x;   // return int
    else return -x;          // return int
}
```

**使用场景**：
- 返回类型复杂或依赖参数时
- 与 Lambda 泛型配合

---

### 13. decltype(auto)
**说明**：使用 decltype 规则推导类型，保留引用和 cv 限定。

**用法举例**：
```cpp
int x = 42;
int& ref = x;

auto a = ref;              // int（值）
decltype(auto) b = ref;    // int&（引用）

int&& rref = 1;
decltype(auto) c = rref;   // int&&
```

**使用场景**：
- 包装函数中完美返回引用
- 需要精确保留表达式类型的场景

---

## C++17 (2017) — 大幅提升易用性

### 1. 结构化绑定 (Structured Binding)
**说明**：将 tuple、pair、struct 等解包到命名变量。

**用法举例**：
```cpp
std::tuple<int, std::string, double> t = {1, "hello", 3.14};
auto [id, name, score] = t;

std::map<int, std::string> m = {{1, "one"}, {2, "two"}};
for (const auto& [key, value] : m) {
    std::cout << key << ": " << value << "\n";
}

struct Point { int x; int y; };
Point p{10, 20};
auto [x, y] = p;
```

**使用场景**：
- 多返回值函数的解包
- 遍历 map 时直接获得 key-value
- 结构体字段提取

---

### 2. if/switch 初始化语句
**说明**：在 if/switch 条件前添加初始化语句。

**用法举例**：
```cpp
if (auto it = m.find(key); it != m.end()) {
    // 使用 it
}  // it 在此处析构

switch (auto c = getchar(); c) {
    case 'q': break;
    case 'x': break;
}

if (std::lock_guard<std::mutex> lock(mtx); shared_data.ready) {
    // 已加锁且条件满足
}
```

**使用场景**：
- 限制变量作用域
- 先获取锁再检查条件
- 查找后检查结果

---

### 3. 内联变量 (inline variable)
**说明**：头文件中定义全局变量，不违反 ODR。

**用法举例**：
```cpp
// header.hpp
inline int global_counter = 0;  // 所有翻译单元共享同一变量
inline constexpr double PI = 3.14159;

// 替代方案：之前需要 extern + 某个 cpp 中定义
```

**使用场景**：
- 头文件中定义全局常量/变量
- 单例模式的简化实现

---

### 4. 折叠表达式 (Fold Expressions)
**说明**：简洁处理变参模板。

**用法举例**：
```cpp
// 一元右折叠
.template<typename... Args>
auto sum(Args... args) {
    return (args + ...);  // ((a + b) + c) + d
}

// 一元左折叠
.template<typename... Args>
auto sum_left(Args... args) {
    return (... + args);  // a + (b + (c + d))
}

// 二元折叠（带初始值）
.template<typename... Args>
auto sum_with_zero(Args... args) {
    return (0 + ... + args);  // 0 + a + b + c
}

// 逗号折叠执行操作
.template<typename... Args>
void print_all(Args... args) {
    (std::cout << ... << args);  // 输出所有参数
}
```

**使用场景**：
- 变参模板的简洁展开
- 实现可变参数的运算符应用

---

### 5. constexpr if
**说明**：编译期条件分支，不会实例化未选中的分支。

**用法举例**：
```cpp
template<typename T>
auto get_value(T t) {
    if constexpr (std::is_pointer_v<T>) {
        return *t;  // 指针才解引用
    } else {
        return t;   // 非指针直接返回
    }
}

get_value(42);      // 返回 int
int x = 10;
get_value(&x);      // 返回 int&
```

**使用场景**：
- 编译期类型分支，替代 SFINAE
- 模板中根据类型选择不同实现

---

### 6. 类模板参数推导 (CTAD)
**说明**：构造类模板时自动推导模板参数。

**用法举例**：
```cpp
std::pair p(1, "hello");           // std::pair<int, const char*>
std::tuple t(1, 2.0, "hi");        // std::tuple<int, double, const char*>
std::vector v = {1, 2, 3};         // std::vector<int>
std::lock_guard lock(mtx);         // 无需写 std::lock_guard<std::mutex>
```

**使用场景**：
- 简化容器和工具类的构造
- 减少冗余的类型书写

---

### 7. std::string_view
**说明**：只读的字符串引用，零拷贝。

**用法举例**：
```cpp
void process(std::string_view sv) {  // 不拥有数据
    std::cout << sv.substr(0, 5);    // 轻量级切片
}

std::string s = "hello world";
process(s);               // 隐式转换
process("literal");       // 不需要分配内存
```

**使用场景**：
- 函数参数接受字符串，但不需要拥有它
- 大量字符串处理，避免不必要的拷贝
- 注意：不保证数据生命周期！

---

### 8. std::optional
**说明**：表示可能不存在的值。

**用法举例**：
```cpp
std::optional<int> maybe_value;

maybe_value = 42;
if (maybe_value.has_value()) {
    std::cout << *maybe_value;  // 解引用
}

// 安全访问
std::cout << maybe_value.value_or(0);  // 无值时返回默认值

// 函数返回可能失败的结果
std::optional<int> parse_int(const std::string& s) {
    // ...
    if (success) return result;
    return std::nullopt;  // 表示无值
}
```

**使用场景**：
- 函数可能返回无效结果
- 延迟初始化
- 替代返回特殊值（如 -1, nullptr）表示失败

---

### 9. std::variant
**说明**：类型安全的联合体。

**用法举例**：
```cpp
std::variant<int, double, std::string> v;
v = 42;
v = "hello";  // 改变类型

// 访问
if (std::holds_alternative<int>(v)) {
    std::cout << std::get<int>(v);
}

// 安全访问
std::string* s = std::get_if<std::string>(&v);
if (s) std::cout << *s;

// std::visit (C++17)
std::visit([](auto&& arg) {
    std::cout << arg;
}, v);
```

**使用场景**：
- 一个变量可能是几种类型之一
- 替代 void*/union + 类型标记的方案
- 状态机、AST 节点、消息类型

---

### 10. std::any
**说明**：类型擦除容器，可存放任意类型。

**用法举例**：
```cpp
std::any a = 42;
a = std::string("hello");
a = 3.14;

if (a.type() == typeid(int)) {
    int x = std::any_cast<int>(a);
}

// 不安全转换会抛 std::bad_any_cast
try {
    std::string s = std::any_cast<std::string>(a);  // 当前是 double，抛异常
} catch (const std::bad_any_cast&) { }
```

**使用场景**：
- 类型完全未知的场景（如配置文件值、脚本绑定）
- 与动态类型系统交互
- 注意：有类型擦除开销，优先用 variant/optional

---

### 11. std::filesystem
**说明**：标准文件系统库。

**用法举例**：
```cpp
namespace fs = std::filesystem;

fs::path p = "C:/Users/test/file.txt";
std::cout << p.filename();      // file.txt
std::cout << p.stem();          // file
std::cout << p.extension();     // .txt

// 查询
if (fs::exists(p)) { }
if (fs::is_regular_file(p)) { }
auto size = fs::file_size(p);
auto time = fs::last_write_time(p);

// 操作
fs::create_directory("new_dir");
fs::copy(p, "backup.txt");
fs::remove(p);
fs::rename("old.txt", "new.txt");

// 遍历目录
for (const auto& entry : fs::directory_iterator(".")) {
    std::cout << entry.path() << "\n";
}

// 递归遍历
for (const auto& entry : fs::recursive_directory_iterator(".")) { }
```

**使用场景**：
- 所有文件和目录操作
- 跨平台路径处理

---

### 12. 嵌套命名空间
**说明**：紧凑的嵌套命名空间语法。

**用法举例**：
```cpp
namespace A::B::C {  // C++17 前需要层层嵌套
    void func() {}
}

// 等价于：
// namespace A { namespace B { namespace C { void func() {} } } }
```

**使用场景**：
- 库的分层命名空间声明

---

### 13. __has_include
**说明**：预处理指令，检查头文件是否存在。

**用法举例**：
```cpp
#if __has_include(<optional>)
#  include <optional>
#  define HAS_OPTIONAL 1
#elif __has_include(<experimental/optional>)
#  include <experimental/optional>
#  define HAS_OPTIONAL 1
#else
#  define HAS_OPTIONAL 0
#endif
```

**使用场景**：
- 编写跨版本/跨平台的兼容性代码

---

### 14. 编译期 static_assert 无需消息
**说明**：`static_assert` 的第二个参数变为可选。

**用法举例**：
```cpp
static_assert(sizeof(int) == 4);  // C++17 前需要第二个消息参数
```

---

### 15. 保证拷贝省略 (Guaranteed Copy Elision) / 强制拷贝省略
**说明**：纯右值的实质化规则改变，某些情况下省略拷贝是强制的。

**用法举例**：
```cpp
T func() { return T(); }  // 纯右值表达式
T x = func();             // C++17 保证不调用移动/拷贝构造
```

**使用场景**：
- 返回值优化成为标准保证
- 不可拷贝不可移动的类型也能作为返回值

---

### 16. 捕获 *this 和 [=, *this]
**说明**：显式按值捕获当前对象。

**用法举例**：
```cpp
class Widget {
    int value = 42;
public:
    auto get_callback() {
        return [*this]() { return value; };  // 按值捕获对象的副本
        // return [=]() { return value; };   // C++17 前 = 按值捕获 this 指针
    }
};
```

**使用场景**：
- 异步回调中安全地持有对象快照
- 避免 Lambda 执行时原对象已销毁

---

### 17. constexpr Lambda
**说明**：Lambda 隐式为 constexpr（满足条件时）。

**用法举例**：
```cpp
constexpr auto square = [](int x) { return x * x; };
constexpr int result = square(5);  // 25
```

**使用场景**：
- 编译期计算中使用 Lambda

---

### 18. 属性列表 (Attribute) 扩展 [[nodiscard]], [[maybe_unused]], [[fallthrough]]
**说明**：标准化的编译器提示。

**用法举例**：
```cpp
[[nodiscard]] int allocate_resource() { return 1; }
allocate_resource();  // 警告：忽略了返回值

[[maybe_unused]] void old_function() {}  // 显式标记可能未使用

void func(int x) {
    switch (x) {
        case 1:
            do_something();
            [[fallthrough]];  // 显式声明有意 fallthrough
        case 2:
            do_other();
            break;
    }
}

[[deprecated("Use new_func instead")]] void old_func() {}
```

**使用场景**：
- `[[nodiscard]]`：错误码、资源句柄、纯查询函数
- `[[fallthrough]]`：消除 switch fallthrough 警告
- `[[maybe_unused]]`：消除未使用变量/函数警告

---

### 19. 动态内存分配对齐 (Aligned new/delete)
**说明**：支持对齐的堆分配。

**用法举例**：
```cpp
// 分配 1024 字节、256 字节对齐的内存
void* p = ::operator new(1024, std::align_val_t{256});
::operator delete(p, std::align_val_t{256});

// new 表达式
struct alignas(64) CacheLine { int data[16]; };
auto* obj = new CacheLine;  // 自动使用对齐的 new
```

**使用场景**：
- SIMD、GPU 等需要特定内存对齐的数据

---

### 20. std::byte
**说明**：表示原始字节的类型，非算术类型。

**用法举例**：
```cpp
std::byte b{0x0F};
auto shifted = std::byte{0x01} << 1;  // std::byte{0x02}

// 与整数转换
int i = std::to_integer<int>(b);  // 15

std::vector<std::byte> buffer(1024);  // 原始字节缓冲区
```

**使用场景**：
- 二进制数据处理，替代 `char`/`unsigned char`
- 禁止无意义的算术运算

---

### 21. 并行算法 (STL Parallel Algorithms)
**说明**：STL 算法支持并行执行策略。

**用法举例**：
```cpp
#include <execution>

std::vector<int> v(1000000);
std::iota(v.begin(), v.end(), 1);

// 顺序执行
std::sort(std::execution::seq, v.begin(), v.end());

// 并行执行
std::sort(std::execution::par, v.begin(), v.end());

// 并行 + 向量化
std::sort(std::execution::par_unseq, v.begin(), v.end());

// 简单写法
std::for_each(std::execution::par, v.begin(), v.end(), [](int& x) {
    x *= x;
});
```

**使用场景**：
- 大数据量处理需要并行加速
- 注意：需确保操作是线程安全的

---

### 22. 模板参数 auto (非类型模板参数用 auto)
**说明**：非类型模板参数类型自动推导。

**用法举例**：
```cpp
template<auto N>
struct integral_constant {
    static constexpr auto value = N;
};

integral_constant<42> i;      // N 推导为 int
integral_constant<'a'> c;     // N 推导为 char

// 数组大小作为模板参数
template<auto& Arr>
void process() {
    constexpr std::size_t n = std::size(Arr);
}

int arr[10];
process<arr>();
```

**使用场景**：
- 更灵活的编译期值模板

---

### 23. using 声明的 pack 展开
**说明**：在 using 声明中展开参数包。

**用法举例**：
```cpp
template<typename... Mixins>
class Widget : public Mixins... {
public:
    using Mixins::Mixins...;  // 继承所有基类的构造函数
    using Mixins::func...;    // 引入所有基类的 func
};
```

**使用场景**：
- Mixin 模式的简洁实现

---

### 24. 显式推导指引 (Deduction Guides)
**说明**：指导类模板参数推导。

**用法举例**：
```cpp
template<typename T>
struct Container {
    Container(T) {}
};

// 推导指引：使 Container(5) 推导为 Container<int>
// 通常 CTAD 自动处理，复杂场景需要显式指引
template<typename Iter>
Container(Iter, Iter) -> Container<typename std::iterator_traits<Iter>::value_type>;

std::vector<int> v = {1, 2, 3};
Container c(v.begin(), v.end());  // 推导为 Container<int>
```

**使用场景**：
- 自定义类的 CTAD 行为
- 迭代器范围构造的推导

---

## C++20 (2020) — 重大革新

### 1. 模块 (Modules)
**说明**：替代头文件，更快的编译速度，更强的封装。

**用法举例**：
```cpp
// math.cppm (模块接口单元)
export module math;

export int add(int a, int b) {
    return a + b;
}

export namespace math {
    template<typename T>
    T square(T x) { return x * x; }
}

// 私有模块片段
module :private;  // 之后的内容不导出

// main.cpp (模块消费者)
import math;

int main() {
    auto r = math::square(5);  // 25
    return add(1, 2);
}
```

**使用场景**：
- 大型项目，编译时间优化
- 更清晰的接口边界（无需 include guard）
- 宏隔离

---

### 2. 协程 (Coroutines)
**说明**：支持无栈协程，实现异步/生成器模式。

**用法举例**：
```cpp
// 简单生成器（需要自定义 promise_type）
generator<int> fibonacci() {
    co_yield 0;  // 产出值，挂起
    co_yield 1;
    int a = 0, b = 1;
    while (true) {
        int next = a + b;
        co_yield next;
        a = b;
        b = next;
    }
}

// 异步任务（伪代码，需要协程框架支持）
task<void> fetch_data() {
    auto data = co_await async_read();  // 非阻塞等待
    process(data);
    co_return;
}
```

**使用场景**：
- 异步 IO、网络编程
- 生成器/迭代器（无穷序列）
- 状态机简化

---

### 3. 概念 (Concepts)
**说明**：模板约束，编译期接口契约。

**用法举例**：
```cpp
template<typename T>
concept Addable = requires(T a, T b) {
    { a + b } -> std::same_as<T>;
};

template<typename T>
concept Numeric = std::integral<T> || std::floating_point<T>;

// 使用概念约束模板
template<Addable T>
T add(T a, T b) { return a + b; }

// 简写语法
template<Numeric T>
T square(T x) { return x * x; }

// 约束自动推导 (abbreviated function template)
auto multiply(Numeric auto a, Numeric auto b) {
    return a * b;
}

// requires 子句
template<typename T>
    requires std::copy_constructible<T>
T clone(const T& t) { return t; }
```

**使用场景**：
- 替代复杂的 SFINAE
- 清晰的模板约束文档
- 更好的编译错误信息

---

### 4. 三路比较运算符 <=> (Spaceship Operator)
**说明**：一次比较生成所有关系运算符结果。

**用法举例**：
```cpp
struct Point {
    int x, y;

    // 编译器自动生成 ==, !=, <, <=, >, >=
    auto operator<=>(const Point&) const = default;
    bool operator==(const Point&) const = default;
};

Point a{1, 2}, b{1, 3};
auto cmp = a <=> b;  // std::strong_ordering::less
if (cmp < 0) { /* a < b */ }

// 自定义比较
struct Person {
    std::string name;
    int age;

    std::strong_ordering operator<=>(const Person& other) const {
        if (auto cmp = name <=> other.name; cmp != 0) return cmp;
        return age <=> other.age;
    }
};
```

**使用场景**：
- 需要全部六种比较运算符时大幅简化代码
- 排序键的链式比较

---

### 5. 指定初始化 (Designated Initializers)
**说明**：按名称初始化聚合类型的成员。

**用法举例**：
```cpp
struct Point { int x; int y; int z; };

Point p = {.x = 1, .y = 2};      // z 默认初始化
Point p2 = {.y = 5, .x = 3};     // C++20: 必须按声明顺序

// 数组也可以
int arr[3] = {[0] = 1, [2] = 3};
```

**使用场景**：
- 提高结构体初始化的可读性
- 大型配置结构体，只设置部分字段

---

### 6. Lambda 的模板语法
**说明**：显式模板参数列表。

**用法举例**：
```cpp
// C++14 泛型 Lambda 的显式版本
auto lambda = []<typename T>(T x, T y) { return x + y; };
lambda(1, 2);      // T = int
lambda(1.0, 2.0);  // T = double

// 概念约束的 Lambda
auto add = []<std::integral T>(T a, T b) { return a + b; };
// add(1.0, 2.0);  // 编译错误
```

**使用场景**：
- 需要显式指定模板参数的场景
- Lambda 中使用概念约束

---

### 7. consteval 与 constinit
**说明**：`consteval` 强制编译期计算；`constinit` 强制静态初始化。

**用法举例**：
```cpp
consteval int compile_time_only(int x) {
    return x * x;
}

constexpr int a = compile_time_only(5);  // OK
// int b = compile_time_only(runtime_val);  // 错误！必须编译期求值

// constinit: 必须有常量初始化器，但并非常量
constinit int global = 42;        // 静态初始化，运行时仍可修改
// constinit int bad;             // 错误：没有初始化器
```

**使用场景**：
- `consteval`：编译期函数，防止意外运行时调用
- `constinit`：避免静态初始化顺序问题（无动态初始化）

---

### 8. 立即函数 (consteval)
**说明**：同上，consteval 函数只能在编译期调用。

---

### 9. 范围库 (Ranges)
**说明**：函数式风格的管道操作，惰性求值。

**用法举例**：
```cpp
#include <ranges>

std::vector<int> nums = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};

// 管道操作
auto result = nums
    | std::views::filter([](int n) { return n % 2 == 0; })
    | std::views::transform([](int n) { return n * n; })
    | std::views::take(3);

// result: 4, 16, 36
for (int n : result) {
    std::cout << n << " ";
}

// 其他视图
auto evens = std::views::iota(1)  // 无限序列 1, 2, 3, ...
    | std::views::filter([](int n) { return n % 2 == 0; });

// 范围算法
std::ranges::sort(nums);
if (std::ranges::find(nums, 5) != nums.end()) { }
```

**使用场景**：
- 数据转换管道
- 惰性求值的无穷序列
- 更声明式的数据处理

---

### 10. std::format
**说明**：类型安全的格式化字符串（类似 Python f-string）。

**用法举例**：
```cpp
#include <format>

std::string s = std::format("Hello, {}! You have {} messages.",
                            "Alice", 42);
// Hello, Alice! You have 42 messages.

// 格式化选项
std::format("{:>10}", "hi");      // "        hi" (右对齐，宽10)
std::format("{:.2f}", 3.14159);   // "3.14"
std::format("{:04d}", 5);         // "0005"
std::format("{:#x}", 255);        // "0xff"

// 打印到 stdout（C++23 有 std::print）
std::cout << std::format("Value: {}\n", value);
```

**使用场景**：
- 替代 printf/sprintf，类型安全
- 国际化字符串格式化
- 日志输出

---

### 11. std::span
**说明**：连续序列的轻量级非拥有视图。

**用法举例**：
```cpp
void process(std::span<int> data) {
    for (auto& x : data) x *= 2;
}

int arr[] = {1, 2, 3, 4, 5};
process(arr);                      // 从数组构造

std::vector<int> vec = {1, 2, 3};
process(vec);                      // 从 vector 构造
process({vec.data(), 3});          // 子范围

// 固定大小 span
void fixed(std::span<int, 4> data) { }
int arr4[4] = {1, 2, 3, 4};
fixed(arr4);
```

**使用场景**：
- 函数参数接受连续数据，不关心容器类型
- 替代 `T*` + `size_t` 的参数对

---

### 12. 日历和时区 (std::chrono 扩展)
**说明**：完整的日期时间库。

**用法举例**：
```cpp
#include <chrono>

using namespace std::chrono;

// 日期
year_month_day date = 2024y / March / 15d;
auto today = std::chrono::system_clock::now();

// 时长
auto d = days{5} + hours{3};
auto sleep = 200ms + 500us;

// 时钟和时间点
auto tp = system_clock::now() + 24h;

// 时区（需要 TZDB）
// auto zt = zoned_time{"Asia/Shanghai", system_clock::now()};
```

**使用场景**：
- 金融、日程、日志系统中的日期计算
- 跨时区应用

---

### 13. jthread 与停止令牌 (std::jthread, std::stop_token)
**说明**：可自动 join 的线程，支持协作式取消。

**用法举例**：
```cpp
void worker(std::stop_token st) {
    while (!st.stop_requested()) {
        // 执行任务
        std::this_thread::sleep_for(100ms);
    }
}

std::jthread t(worker);  // 自动管理生命周期
// ...
t.request_stop();        // 请求停止
// t 析构时自动 join
```

**使用场景**：
- 替代 std::thread，避免忘记 join/detach
- 需要优雅终止的长时间任务

---

### 14. 原子智能指针 (std::atomic<std::shared_ptr>)
**说明**：shared_ptr 的原子操作特化。

**用法举例**：
```cpp
std::atomic<std::shared_ptr<int>> atomic_ptr;
auto p = std::make_shared<int>(42);
atomic_ptr.store(p);

auto loaded = atomic_ptr.load();
auto expected = p;
atomic_ptr.compare_exchange_strong(expected, std::make_shared<int>(100));
```

**使用场景**：
- 无锁数据结构中共享所有权

---

### 15. 同步设施：信号量 (std::counting_semaphore)、锁存器 (std::latch)、屏障 (std::barrier)
**说明**：更多底层同步原语。

**用法举例**：
```cpp
// 锁存器：等待指定次数的计数到达
std::latch latch{3};
for (int i = 0; i < 3; ++i) {
    std::jthread([&latch] {
        // 工作...
        latch.count_down();  // 计数减一
    });
}
latch.wait();  // 等待三个线程都 count_down

// 屏障：多轮同步
std::barrier barrier{4};
for (int i = 0; i < 4; ++i) {
    std::jthread([&barrier] {
        // 阶段 1
        barrier.arrive_and_wait();
        // 阶段 2
        barrier.arrive_and_wait();
    });
}

// 信号量
std::counting_semaphore<10> sem{5};  // 最大 10，初始 5
sem.acquire();   // P 操作，信号量减一
sem.release();   // V 操作，信号量加一
```

**使用场景**：
- `latch`：一次性同步多个线程
- `barrier`：分阶段算法（如并行迭代）
- `semaphore`：资源池限流

---

### 16. std::source_location
**说明**：编译期获取源码位置信息。

**用法举例**：
```cpp
void log(const std::string& msg,
         std::source_location loc = std::source_location::current()) {
    std::cout << loc.file_name() << ":"
              << loc.line() << " ("
              << loc.function_name() << "): "
              << msg << "\n";
}

log("something happened");  // 自动记录调用位置
```

**使用场景**：
- 日志库、调试信息
- 替代 `__FILE__`、`__LINE__` 宏

---

### 17. 位运算工具 (std::bit_cast, std::endian, std::has_single_bit 等)
**说明**：标准位操作工具。

**用法举例**：
```cpp
// bit_cast: 类型双关（type punning），安全地 reinterpret
double d = 3.14;
auto u = std::bit_cast<std::uint64_t>(d);  // 查看 double 的位模式

// endian
if constexpr (std::endian::native == std::endian::little) {
    // 小端平台
}

// 位运算辅助
std::has_single_bit(8u);     // true，是 2 的幂
std::bit_ceil(5u);           // 8，不小于 5 的最小 2 的幂
std::popcount(0b10110u);     // 3，二进制中 1 的个数
std::countr_zero(8u);        // 3，尾部零的个数
```

**使用场景**：
- 序列化/反序列化
- 底层协议实现
- 高性能位运算算法

---

### 18. 特性测试宏
**说明**：标准特性测试宏，检查编译器支持。

**用法举例**：
```cpp
#ifdef __cpp_modules
    import my_module;
#else
    #include "my_header.hpp"
#endif

#if __cpp_concepts >= 202002L
    // 使用 C++20 concepts
#endif
```

**使用场景**：
- 编写跨标准版本的库代码

---

### 19. 打包展开在更多上下文
**说明**：更通用的 pack expansion。

**用法举例**：
```cpp
// Lambda 捕获中的包展开
template<typename... Args>
void capture_all(Args... args) {
    auto lambda = [args...] {  // C++20 前某些场景受限
        (std::cout << ... << args);
    };
    lambda();
}

// 结构化绑定中的包
template<typename... Ts>
void process(Ts... args) {
    auto&& [...elems] = std::forward_as_tuple(args...);
}
```

---

### 20. volatile 弃用部分用法
**说明**：volatile 的一些操作被标记为弃用。

---

### 21. 新增聚合初始化规则
**说明**：基类也可以被聚合初始化。

**用法举例**：
```cpp
struct Base { int x; };
struct Derived : Base { int y; };

Derived d{{10}, 20};  // C++20: 初始化基类 + 派生类成员
```

---

### 22. using enum
**说明**：将枚举值引入当前作用域。

**用法举例**：
```cpp
enum class Color { Red, Green, Blue };

void func() {
    using enum Color;  // C++20
    Color c = Red;     // 不需要 Color:: 前缀
}
```

**使用场景**：
- switch 语句中减少前缀书写

---

### 23. 类类型的非类型模板参数 (NTTP Class Types)
**说明**：更多类型可作为非类型模板参数。

**用法举例**：
```cpp
struct Id {
    int value;
    bool operator==(const Id&) const = default;  // 需要字面量类型的条件
};

template<Id id>
void process() { }

process<Id{42}>();  // C++20
```

---

### 24. 协程相关关键字 co_await, co_yield, co_return
**说明**：见第 2 条协程。

---

## C++23 (2023) — 标准化现代实践

### 1. std::expected
**说明**：表示可能失败的运算结果，比异常更高效。

**用法举例**：
```cpp
std::expected<int, std::string> divide(int a, int b) {
    if (b == 0) return std::unexpected("division by zero");
    return a / b;
}

auto result = divide(10, 2);
if (result) {
    std::cout << *result;  // 5
} else {
    std::cout << result.error();
}

// 链式操作
auto r = divide(10, 2)
    .and_then([](int x) { return divide(100, x); })  // 100 / 5
    .or_else([](const std::string& e) {
        return std::expected<int, std::string>(0);
    });
```

**使用场景**：
- 错误处理替代异常（性能敏感路径）
- 显式错误传播

---

### 2. std::print / std::println
**说明**：格式化输出到标准流。

**用法举例**：
```cpp
#include <print>

std::print("Hello, {}!\n", "World");       // 无换行
std::println("Value: {}", 42);             // 自动添加换行
std::println("{:.2f}", 3.14159);          // 3.14

// 输出到指定流
std::print(std::cerr, "Error: {}\n", msg);
```

**使用场景**：
- 替代 `std::cout <<`，更简洁高效
- 类型安全的格式化输出

---

### 3. std::mdspan
**说明**：多维数组的非 owning 视图。

**用法举例**：
```cpp
int data[12] = {0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11};

// 3x4 的二维视图
std::mdspan<int, std::extents<std::size_t, 3, 4>> matrix(data);

std::println("element (1,2) = {}", matrix(1, 2));  // 7

// 动态维度
std::mdspan<int, std::dextents<std::size_t, 2>> dyn_matrix(
    data, 3, 4);

// 切片
cout << std::submdspan(matrix, 1, std::full_extent);  // 第 1 行
```

**使用场景**：
- 科学计算、矩阵运算
- 多维数据的统一接口
- 与 C/Fortran 数组交互

---

### 4. std::flat_map / std::flat_set
**说明**：基于有序容器的 map/set，缓存友好。

**用法举例**：
```cpp
std::flat_map<int, std::string> map;  // 底层是 vector + 排序
map[3] = "three";
map[1] = "one";
map[2] = "two";

// 遍历时按 key 排序，且内存连续
for (const auto& [k, v] : map) {
    std::println("{}: {}", k, v);
}

// flat_set 同理
std::flat_set<int> set = {3, 1, 4, 1, 5};
```

**使用场景**：
- 小数据量、遍历多修改少的场景
- 缓存敏感型应用（内存连续）
- 注意：插入复杂度为 O(N)，适合读多写少

---

### 5. std::generator（协程生成器）
**说明**：标准协程生成器类型。

**用法举例**：
```cpp
std::generator<int> fib() {
    co_yield 0;
    co_yield 1;
    int a = 0, b = 1;
    while (true) {
        int next = a + b;
        co_yield next;
        a = b;
        b = next;
    }
}

for (auto n : fib() | std::views::take(10)) {
    std::print("{} ", n);
}
```

**使用场景**：
- 无穷序列
- 懒加载数据流

---

### 6. if consteval
**说明**：更精确的编译期分支。

**用法举例**：
```cpp
auto func(auto x) {
    if consteval {       // 仅在编译期执行
        return compile_time_impl(x);
    } else {            // 仅在运行期执行
        return runtime_impl(x);
    }
}
```

**使用场景**：
- 编译期和运行期需要不同实现时，比 `if constexpr` 更清晰

---

### 7. 显式 this 参数 (Deducing this / Explicit Object Parameter)
**说明**：将 this 作为显式参数，统一 const/non-const 重载。

**用法举例**：
```cpp
struct Widget {
    int value;

    // C++23 前需写两个重载
    // int get() const { return value; }
    // int& get() { return value; }

    // C++23: 一个函数处理所有情况
    auto&& get(this auto&& self) {
        return std::forward<decltype(self)>(self).value;
    }
};

Widget w{42};
int x = w.get();       // 拷贝
int& r = w.get();      // 引用
const Widget cw{10};
int y = cw.get();      // const 访问
```

**使用场景**：
- CRTP 基类代码简化
- 避免 const/non-const 的重复代码

---

### 8. 隐式 lambda 捕获 this 的改动
**说明**：`[=]` 不再隐式捕获 this 指针（改为按值捕获所有）。

---

### 9. 多维下标运算符 (operator[] 多个参数)
**说明**：`operator[]` 支持多个参数。

**用法举例**：
```cpp
struct Matrix {
    int& operator[](size_t row, size_t col) {
        return data[row * cols + col];
    }
    std::vector<int> data;
    size_t cols;
};

Matrix m;
int x = m[1, 2];  // 调用 operator[](1, 2)
```

**使用场景**：
- 多维数组类
- 替代 `operator()` 的下标访问

---

### 10. std::to_underlying
**说明**：将枚举转换为其底层整数类型。

**用法举例**：
```cpp
enum class Color : unsigned char { Red, Green, Blue };

auto val = std::to_underlying(Color::Red);  // unsigned char，值为 0
```

**使用场景**：
- 将 enum class 转为整数进行位运算或输出

---

### 11. std::byteswap
**说明**：字节序交换。

**用法举例**：
```cpp
std::uint32_t val = 0x12345678;
auto swapped = std::byteswap(val);  // 0x78563412
```

**使用场景**：
- 网络协议、文件格式的字节序转换

---

### 12. 预处理语句 #elifdef / #elifndef
**说明**：预处理器条件分支增强。

**用法举例**：
```cpp
#ifdef FOO
// ...
#elifdef BAR    // C++23，等价于 #elif defined(BAR)
// ...
#elifndef BAZ   // 等价于 #elif !defined(BAZ)
// ...
#endif
```

---

### 13. std::views::zip, std::views::chunk, std::views::slide, std::views::stride 等范围适配器
**说明**：更多标准范围视图。

**用法举例**：
```cpp
std::vector<int> a = {1, 2, 3};
std::vector<char> b = {'a', 'b', 'c'};

// zip: 同时遍历多个范围
for (auto [x, y] : std::views::zip(a, b)) {
    std::println("{}, {}", x, y);  // (1,a), (2,b), (3,c)
}

// chunk: 分块
for (auto chunk : std::views::iota(1, 10) | std::views::chunk(3)) {
    // {1,2,3}, {4,5,6}, {7,8,9}
}

// slide: 滑动窗口
for (auto win : std::views::iota(1, 6) | std::views::slide(3)) {
    // {1,2,3}, {2,3,4}, {3,4,5}, {4,5,6}
}

// stride: 步进采样
for (auto n : std::views::iota(1, 10) | std::views::stride(2)) {
    // 1, 3, 5, 7, 9
}
```

**使用场景**：
- 数据处理管道
- 科学计算中的批量处理

---

### 14. std::basic_string::contains
**说明**：字符串包含检查。

**用法举例**：
```cpp
std::string s = "hello world";
if (s.contains("world")) { }   // true
if (s.contains('o')) { }       // true
if (s.contains("xyz")) { }     // false
```

**使用场景**：
- 简化子串存在性检查，替代 `find != npos`

---

### 15. std::stacktrace
**说明**：标准堆栈跟踪。

**用法举例**：
```cpp
#include <stacktrace>

void show_stack() {
    auto trace = std::stacktrace::current();
    std::println("{}", trace);
    // 或逐帧分析
    for (const auto& entry : trace) {
        std::println("{}: {} in {}",
            entry.source_file(),
            entry.source_line(),
            entry.description());
    }
}
```

**使用场景**：
- 调试、日志记录、崩溃报告

---

### 16. std::out_ptr / std::inout_ptr
**说明**：与 C 风格 API 交互的智能指针辅助。

**用法举例**：
```cpp
// C API: void get_resource(int** out);
std::unique_ptr<int> ptr;
C_api_get_resource(std::out_ptr(ptr));
// ptr 现在持有 C API 分配的内存
```

**使用场景**：
- C++ 智能指针与 C 风格出参 API 的桥接

---

### 17. std::move_only_function / std::copyable_function
**说明**：支持只移/可复制函数对象的 std::function 替代。

**用法举例**：
```cpp
std::move_only_function<int()> f;
auto ptr = std::make_unique<int>(42);
f = [p = std::move(ptr)] { return *p; };
// f 可移不可拷

// 注意：std::function 不支持只移的 callable
// std::move_only_function 支持
```

**使用场景**：
- unique_ptr 捕获的 Lambda 作为回调

---

### 18. std::optional 的 monadic 操作 (and_then, or_else, transform, value_or)
**说明**：函数式风格的 optional 链式调用。

**用法举例**：
```cpp
std::optional<int> opt = 5;

auto result = opt
    .transform([](int x) { return x * 2; })     // 有值时转换
    .and_then([](int x) -> std::optional<int> {
        if (x > 5) return x;
        return std::nullopt;
    })
    .value_or(0);  // 无值时的默认值
```

**使用场景**：
- 链式处理可能失败的计算

---

### 19. std::ranges::to
**说明**：将范围转换为容器。

**用法举例**：
```cpp
auto vec = std::views::iota(1, 10)
    | std::views::filter([](int n) { return n % 2 == 0; })
    | std::ranges::to<std::vector>();  // {2, 4, 6, 8}

auto set = std::views::iota(1, 5)
    | std::ranges::to<std::set>();
```

**使用场景**：
- 范围管道结果物化为容器

---

### 20. 大小写不敏感字符串比较支持（std::basic_string 增加 starts_with 等重载，或单独用 ranges）
**说明**：配合 ranges 的视图实现。

---

### 21. 更多的 constexpr
**说明**：大量标准库组件 constexpr 化。

**用法举例**：
```cpp
constexpr std::string str = "hello";       // C++20 起部分支持，C++23 更完善
constexpr std::vector<int> vec = {1,2,3}; // C++23 起支持
```

---

## C++26 (2026) — 部分已确定特性

### 1. 包索引 (Pack Indexing)
**说明**：直接访问参数包中的第 N 个元素。

**用法举例**：
```cpp
template<typename... Args>
void get_first(Args... args) {
    auto first = args...[0];      // C++26
    auto last = args...[sizeof...(args) - 1];
}
```

**使用场景**：
- 变参模板中随机访问参数
- 替代递归展开获取特定参数

---

### 2. 结构化绑定的扩展
**说明**：支持更多的结构化绑定场景。

**用法举例**：
```cpp
// 绑定到私有成员（通过 get<> 友元）
// 更好的 lambda 参数结构化绑定支持
```

---

### 3. Contracts（契约，正在标准化中）
**说明**：前置条件、后置条件、断言。

**用法举例**：
```cpp
// 草案语法，最终可能有变化
int divide(int a, int b)
    [[pre: b != 0]]           // 前置条件
    [[post r: r * b == a]]    // 后置条件
{
    return a / b;
}
```

**使用场景**：
- 接口契约定义
- 运行时/编译期断言

---

### 4. 更多的 constexpr
**说明**：虚函数、try-catch、new/delete 等在 constexpr 中的支持。

**用法举例**：
```cpp
constexpr int allocate_and_compute() {
    int* p = new int(42);   // C++26 constexpr new
    int result = *p * 2;
    delete p;
    return result;
}
```

**使用场景**：
- 编译期动态内存分配
- 更复杂的编译期计算

---

### 5. 反射 (Static Reflection, 草案阶段)
**说明**：编译期查询类型的元信息。

**用法举例**：
```cpp
// 草案语法，最终形式待定
struct Point { int x; int y; };

// 遍历成员
// template for (auto member : Point.members) {
//     std::println("{}", member.name());
// }
```

**使用场景**：
- 序列化/反序列化自动生成
- ORM、绑定生成
- 编译期元编程

---

### 6. 更安全的范围访问
**说明**：如 `std::ranges::first`, 更安全的访问器等。

---

### 7. std::chrono 的更多扩展
**说明**：如 `std::chrono::days_in_month` 等。

---

### 8. std::simd (并行向量类型，已存在但未完全标准化)
**说明**：标准 SIMD 类型。

**用法举例**：
```cpp
std::simd<float, 4> a = {1.0f, 2.0f, 3.0f, 4.0f};
std::simd<float, 4> b = {2.0f, 2.0f, 2.0f, 2.0f};
auto c = a * b;  // 向量乘法: {2, 4, 6, 8}
```

**使用场景**：
- 数值计算向量化
- 多媒体、科学计算

---

## 版本快速对照表

| 版本 | 年份 | 核心主题 |
|------|------|----------|
| C++11 | 2011 | 现代 C++ 基础：auto、Lambda、智能指针、右值引用、并发 |
| C++14 | 2014 | C++11 的补完：泛型 Lambda、变量模板、constexpr 放宽 |
| C++17 | 2017 | 易用性飞跃：结构化绑定、if 初始化、optional/variant/filesystem、折叠表达式 |
| C++20 | 2020 | 重大革新：模块、协程、概念、范围库、format、三路比较 |
| C++23 | 2023 | 标准化现代实践：expected、print、flat_map、mdspan、deducing this |
| C++26 | 2026 | 反射、包索引、更完善的 constexpr、contracts（预计） |

---

## 编译器支持提示

- **GCC**: 用 `-std=c++20` / `-std=c++23` / `-std=c++2c` 启用
- **Clang**: 类似 GCC 的选项
- **MSVC**: 用 `/std:c++20` / `/std:c++latest`
- 部分 C++20/23 特性可能需要较新版本的编译器（GCC 13+, Clang 17+, MSVC 2022 17.8+）

建议在实际项目中使用 C++17 作为基线，有条件时迁移到 C++20，新实验性项目可以尝试 C++23。
