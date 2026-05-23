# 第 22 章 四元数（Quaternions）

---

第 1 章引入了**向量**——有序 3 元组的实数，定义几何上有用的运算；第 2 章引入了**矩阵**——实数表，可以表示线性变换和仿射变换。本章引入第三类对象：**四元数**。

**核心结论**：**单位四元数可以表示 3D 旋转**，而且具有**便于插值**的优良性质——这是骨骼动画的数学基石。

---

## 22.1 复数回顾

四元数可以视为复数的推广。先理解**复数乘法实现 2D 旋转**，再过渡到四元数表示 3D 旋转就自然多了。

### 22.1.1 定义

**复数**：有序实数对 `z = (a, b)`。`a` 是**实部**，`b` 是**虚部**。

| 运算 | 定义 |
|------|------|
| 相等 | `(a,b) = (c,d) ⟺ a=c 且 b=d` |
| 加减 | `(a,b) ± (c,d) = (a±c, b±d)` |
| 乘 | `(a,b)(c,d) = (ac-bd, ad+bc)` |

**实数嵌入**：`x ≡ (x, 0)`，与复数相乘相当于向量缩放。

**虚单位**：`i = (0, 1)`，且 `i² = (0,1)(0,1) = (-1, 0) = -1`。所以 `i² = -1`，`i` 是 `x² = -1` 的解。

**共轭**：`z = (a, b)` 的共轭 `z* = (a, -b)`。

**a+ib 形式**：

```
a + ib = (a, 0) + (0, 1)(b, 0) = (a, b)
```

复数加法即 2D 向量加法。

### 22.1.2 几何意义

**模**：`|a+ib| = √(a² + b²)`，复数对应的 2D 向量长度。模为 1 的复数称为**单位复数**。

### 22.1.3 极坐标与旋转

把复数用极坐标写：

```
a + ib = r(cosθ + i·sinθ)    其中 r = |a+ib|
```

**乘法的几何意义**：

```
z1·z2 = r1·r2 · [cos(θ1+θ2) + i·sin(θ1+θ2)]
```

如果 `r2 = 1`，则 `z1·z2` 把 `z1` 沿原点旋转了 `θ2`！

> **关键观察**：**单位复数与一个复数相乘 → 在 2D 平面旋转**。四元数的角色就是把这个观察推广到 3D。

---

## 22.2 四元数代数

### 22.2.1 定义

**四元数**：有序 4 元组 `q = (x, y, z, w) = (q1, q2, q3, q4)`。常缩写为：

```
q = (u, w)    其中 u = (x, y, z) 是虚向量部，w 是实部
```

| 运算 | 定义 |
|------|------|
| 相等 | `(u, a) = (v, b) ⟺ u=v 且 a=b` |
| 加减 | `(u, a) ± (v, b) = (u±v, a±b)` |
| **乘** | `(u, a)(v, b) = (av + bu + u×v, ab - u·v)` |

乘法看着"古怪"，但**这个定义最终能表示旋转**——和矩阵乘法一样，定义看起来怪但用起来好。

```
设 p = (u, p4)，q = (v, q4)
则 r = pq 的展开式（矩阵形式）：

[r1]   [ p4  -p3   p2   p1] [q1]
[r2] = [ p3   p4  -p1   p2] [q2]
[r3]   [-p2   p1   p4   p3] [q3]
[r4]   [-p1  -p2  -p3   p4] [q4]
```

### 22.2.2 特殊乘积

令 `i = (1,0,0,0), j = (0,1,0,0), k = (0,0,1,0)`，则：

```
i² = j² = k² = -1
ij = k,  ji = -k
jk = i,  kj = -i
ki = j,  ik = -j
```

这与 3D 叉积的反对称性一致（注意 `ij ≠ ji`）。

### 22.2.3 性质

- 四元数乘法**不满足交换律**（如 `ij = -ji`）；
- 但满足**结合律**（继承自矩阵乘法）；
- **单位元**：`e = (0, 0, 0, 1)`，即 `eq = qe = q`；
- 满足**分配律**：`p(q+r) = pq + pr`。

### 22.2.4 标量、向量与四元数的关系

| 对象 | 嵌入四元数 |
|------|------------|
| 实数 `s` | `(0, 0, 0, s)` |
| 向量 `u = (x,y,z)` | `(x, y, z, 0)` |
| 单位元 `1` | `(0, 0, 0, 1)` |

**纯四元数**：实部为 0 的四元数。向量自然嵌入为纯四元数。

实数与四元数的乘法可交换：

```
s·q = (0,0,0,s)·q = (sq1, sq2, sq3, sq4) = q·s
```

### 22.2.5 共轭与范数

**共轭**：`q = (u, q4)` 的共轭 `q* = (-u, q4)`——**虚向量部取反**。

性质：

- `(p+q)* = p* + q*`
- `(pq)* = q*p*`（注意顺序反过来）
- `q + q* = (0, 0, 0, 2q4)` 是实数
- `qq* = q*q = (0, 0, 0, ||q||²)` 是实数

**范数**：

```
||q|| = √(q1² + q2² + q3² + q4²)
```

**单位四元数**：`||q|| = 1`。

性质：

- `||q*|| = ||q||`
- `||pq|| = ||p||·||q||` ← **两个单位四元数的积仍是单位四元数**！

### 22.2.6 逆

对非零四元数 `q`：

```
q⁻¹ = q* / ||q||²
```

验证：`q·q⁻¹ = q·q*/||q||² = (0,0,0,||q||²)/||q||² = 1`。

**对单位四元数**：`||q||² = 1`，所以 `q⁻¹ = q*` ← 极为重要的简化！

### 22.2.7 极坐标表示

若 `q = (u, q4)` 是单位四元数，则 `||u||² + q4² = 1`，存在 `θ ∈ [0, π]` 使 `q4 = cos θ`，进而 `||u|| = sin θ`。

设 `n = u / ||u||`（单位向量），则：

```
q = (sin θ · n,  cos θ)         ← 极坐标表示
```

> **下一节会看到**：`n` 就是**旋转轴**，可以通过对 `n` 取反实现"反向旋转"。

---

## 22.3 单位四元数与旋转

### 22.3.1 旋转算子

**核心定理**：设 `q = (u, w) = (sin θ · n, cos θ)` 是单位四元数，`v` 是 3D 点/向量，把 `v` 看成纯四元数 `(v, 0)`，则：

```
R_q(v) = q · v · q⁻¹    (单位四元数下 q⁻¹ = q*)
```

**绕轴 `n` 旋转角度 `2θ`**！（注意是 2θ，不是 θ）

**推导大意**（实部恒为 0，省略）：

```
qvq* = (2(u·v)u + (w² - u·u)v + 2w(u×v),  0)
     = (2sin²θ·(n·v)n + cos2θ·v + 2sinθ·cosθ·(n×v),  0)
     = ((1-cos2θ)·(n·v)n + cos2θ·v + sin2θ·(n×v),  0)
```

后者正是 **轴-角旋转公式 Rₙ(v)**，对应绕 `n` 旋转 `2θ`。

**构造旋转四元数**：给定轴 `n` 和角 `θ`：

```
q = (sin(θ/2) · n,  cos(θ/2))
```

> 除以 2 是为了**补偿** R_q 实际产生的旋转是 `2 × (θ/2) = θ`。

### 22.3.2 四元数 → 旋转矩阵

把旋转算子展开成矩阵：

```
       [1-2q2²-2q3²    2q1q2+2q3q4    2q1q3-2q2q4]
Q  =   [2q1q2-2q3q4    1-2q1²-2q3²    2q2q3+2q1q4]
       [2q1q3+2q2q4    2q2q3-2q1q4    1-2q1²-2q2²]
```

> 行向量与列向量约定不同，列向量约定下取转置。

### 22.3.3 旋转矩阵 → 四元数

设旋转矩阵 `R`：

1. **计算 trace**：`tr(R) = R11 + R22 + R33 = 4q4² - 1`，求出 `q4`；
2. **从反对角元素差分**求 `q1, q2, q3`：

```
R23 - R32 = 4·q1·q4   →   q1 = (R23 - R32) / (4·q4)
R31 - R13 = 4·q2·q4   →   q2 = (R31 - R13) / (4·q4)
R12 - R21 = 4·q3·q4   →   q3 = (R12 - R21) / (4·q4)
```

3. 若 `q4 = 0`，则切换到**最大对角元素**主导的分支，避免除零。

### 22.3.4 复合旋转

两次单位四元数旋转的复合就是它们的乘积：

```
R_p ∘ R_q  ⟺  Rotation by pq
```

因为 `||pq|| = ||p||·||q|| = 1`，乘积仍是单位四元数。

---

## 22.4 四元数插值（SLERP）

**动画需求**：把一个朝向平滑过渡到另一个。

把单位四元数视为 4D 单位球面上的点。两单位四元数 `p, q` 的点积：

```
p·q = ||p||·||q||·cos θ = cos θ
```

`θ` 是两四元数在 4D 球面上的夹角，**度量它们的接近程度**。

### 球面线性插值（Spherical Linear Interpolation, SLERP）

要从 `a` 沿球面弧线插值到 `b`，参数 `t ∈ [0, 1]`：

```
slerp(a, b, t) = (sin((1-t)·θ) / sin θ) · a + (sin(t·θ) / sin θ) · b
```

`θ = arccos(a·b)`。

### 小角度退化

当 `θ ≈ 0`，`sin θ ≈ 0`，除法不稳。此时回退到**线性插值 + 归一化（LERP+NLERP）**：

```
slerp 退化为 normalize((1-t)·p + t·q)
```

> 大角度时纯线性插值会让旋转**忽快忽慢**（球面投影非线性），而 SLERP 保持**匀速旋转**。

### q 与 -q 的歧义

```
R_q   绕 n 旋转 θ
R_-q  绕 -n 旋转 (2π - θ)    ← 同一终点！
```

**几何上 `q` 与 `-q` 表示同一朝向**，但插值路径不同：

- `slerp(a, b, t)`：可能走长弧（绕远路，物体旋转过头）
- `slerp(a, -b, t)`：可能走短弧

**判断方式**：比较 `||a-b||²` 与 `||a-(-b)||² = ||a+b||²`，谁小走谁。

### C++ 实现

```cpp
public static Quaternion LerpAndNormalize(Quaternion p, Quaternion q, float s)
{
    return Normalize((1.0f - s) * p + s * q);
}

public static Quaternion Slerp(Quaternion p, Quaternion q, float s)
{
    // 选择短弧
    if (LengthSq(p - q) > LengthSq(p + q))
        q = -q;

    float cosPhi = DotP(p, q);

    // 极小角度时退化为 LERP
    if (cosPhi > (1.0f - 0.001f))
        return LerpAndNormalize(p, q, s);

    float phi    = (float)Math.Acos(cosPhi);
    float sinPhi = (float)Math.Sin(phi);

    return ((float)Math.Sin(phi*(1.0-s))/sinPhi) * p
         + ((float)Math.Sin(phi*s)    /sinPhi) * q;
}
```

---

## 22.5 DirectX Math 四元数函数

DirectX Math 用 `XMVECTOR`（4 个浮点）存储四元数。常用函数：

```cpp
// 四元数点积
XMVECTOR XMQuaternionDot(XMVECTOR Q1, XMVECTOR Q2);

// 单位元 (0, 0, 0, 1)
XMVECTOR XMQuaternionIdentity();

// 共轭
XMVECTOR XMQuaternionConjugate(XMVECTOR Q);

// 模 / 归一化
XMVECTOR XMQuaternionLength(XMVECTOR Q);
XMVECTOR XMQuaternionNormalize(XMVECTOR Q);

// 乘积 Q1·Q2
XMVECTOR XMQuaternionMultiply(XMVECTOR Q1, XMVECTOR Q2);

// 轴-角 → 四元数
XMVECTOR XMQuaternionRotationAxis  (XMVECTOR Axis, FLOAT Angle);
XMVECTOR XMQuaternionRotationNormal(XMVECTOR NormalAxis, FLOAT Angle);
// Normal 版本要求轴已单位化，更快

// 旋转矩阵 ↔ 四元数
XMVECTOR XMQuaternionRotationMatrix(XMMATRIX M);
XMMATRIX XMMatrixRotationQuaternion(XMVECTOR Q);

// 四元数 → 轴-角
VOID XMQuaternionToAxisAngle(XMVECTOR* pAxis, FLOAT* pAngle, XMVECTOR Q);

// 球面插值
XMVECTOR XMQuaternionSlerp(XMVECTOR Q0, XMVECTOR Q1, FLOAT t);
```

---

## 22.6 Demo：骷髅头旋转

本章 Demo 让一个骷髅模型在场景中做"位置 + 朝向 + 缩放"动画：

- **朝向**用四元数表示，用 SLERP 插值；
- **位置**和**缩放**用线性插值。

这也是下一章角色动画的"热身"。

### 关键帧（Keyframe）

```cpp
struct Keyframe {
    Keyframe();
    ~Keyframe();

    float    TimePos;
    XMFLOAT3 Translation;
    XMFLOAT3 Scale;
    XMFLOAT4 RotationQuat;  // 四元数
};
```

### 骨骼动画（BoneAnimation）

按时间排序的关键帧列表（用"Bone"是为下一章铺垫）：

```cpp
struct BoneAnimation {
    float GetStartTime() const;
    float GetEndTime  () const;

    // 在时间 t 上插值得到变换矩阵
    void Interpolate(float t, XMFLOAT4X4& M) const;

    std::vector<Keyframe> Keyframes;
};
```

### 插值实现

```cpp
void BoneAnimation::Interpolate(float t, XMFLOAT4X4& M) const
{
    // t 在起点之前：返回第一帧
    if (t <= Keyframes.front().TimePos) {
        XMVECTOR S = XMLoadFloat3(&Keyframes.front().Scale);
        XMVECTOR P = XMLoadFloat3(&Keyframes.front().Translation);
        XMVECTOR Q = XMLoadFloat4(&Keyframes.front().RotationQuat);
        XMVECTOR zero = XMVectorSet(0.0f, 0.0f, 0.0f, 1.0f);
        XMStoreFloat4x4(&M,
            XMMatrixAffineTransformation(S, zero, Q, P));
    }
    // t 在终点之后：返回最后一帧
    else if (t >= Keyframes.back().TimePos) {
        // 同上，用 back()
    }
    // t 在两帧之间：插值
    else {
        for (UINT i = 0; i < Keyframes.size() - 1; ++i) {
            if (t >= Keyframes[i].TimePos &&
                t <= Keyframes[i+1].TimePos)
            {
                float lerpPercent =
                    (t - Keyframes[i].TimePos) /
                    (Keyframes[i+1].TimePos - Keyframes[i].TimePos);

                XMVECTOR s0 = XMLoadFloat3(&Keyframes[i].Scale);
                XMVECTOR s1 = XMLoadFloat3(&Keyframes[i+1].Scale);
                XMVECTOR p0 = XMLoadFloat3(&Keyframes[i].Translation);
                XMVECTOR p1 = XMLoadFloat3(&Keyframes[i+1].Translation);
                XMVECTOR q0 = XMLoadFloat4(&Keyframes[i].RotationQuat);
                XMVECTOR q1 = XMLoadFloat4(&Keyframes[i+1].RotationQuat);

                // 位置/缩放用 LERP，朝向用 SLERP
                XMVECTOR S = XMVectorLerp(s0, s1, lerpPercent);
                XMVECTOR P = XMVectorLerp(p0, p1, lerpPercent);
                XMVECTOR Q = XMQuaternionSlerp(q0, q1, lerpPercent);

                XMVECTOR zero = XMVectorSet(0.0f, 0.0f, 0.0f, 1.0f);
                XMStoreFloat4x4(&M,
                    XMMatrixAffineTransformation(S, zero, Q, P));
                break;
            }
        }
    }
}
```

### 仿射变换构造器

```cpp
XMMATRIX XMMatrixAffineTransformation(
    XMVECTOR Scaling,
    XMVECTOR RotationOrigin,
    XMVECTOR RotationQuaternion,   // 四元数表示朝向
    XMVECTOR Translation);
```

### 定义关键帧（5 帧动画）

```cpp
float          mAnimTimePos = 0.0f;
BoneAnimation  mSkullAnimation;

void QuatApp::DefineSkullAnimation()
{
    // 4 个朝向四元数（轴-角构造）
    XMVECTOR q0 = XMQuaternionRotationAxis(
        XMVectorSet(0,1,0,0), XMConvertToRadians( 30.0f));
    XMVECTOR q1 = XMQuaternionRotationAxis(
        XMVectorSet(1,1,2,0), XMConvertToRadians( 45.0f));
    XMVECTOR q2 = XMQuaternionRotationAxis(
        XMVectorSet(0,1,0,0), XMConvertToRadians(-30.0f));
    XMVECTOR q3 = XMQuaternionRotationAxis(
        XMVectorSet(1,0,0,0), XMConvertToRadians( 70.0f));

    mSkullAnimation.Keyframes.resize(5);

    mSkullAnimation.Keyframes[0].TimePos     = 0.0f;
    mSkullAnimation.Keyframes[0].Translation = XMFLOAT3(-7,0,0);
    mSkullAnimation.Keyframes[0].Scale       = XMFLOAT3(0.25f, 0.25f, 0.25f);
    XMStoreFloat4(&mSkullAnimation.Keyframes[0].RotationQuat, q0);

    // ... 第 1~4 帧 (TimePos = 2, 4, 6, 8s)

    mSkullAnimation.Keyframes[4].TimePos     = 8.0f;
    mSkullAnimation.Keyframes[4].Translation = XMFLOAT3(-7,0,0);
    mSkullAnimation.Keyframes[4].Scale       = XMFLOAT3(0.25f, 0.25f, 0.25f);
    XMStoreFloat4(&mSkullAnimation.Keyframes[4].RotationQuat, q0);  // 回到初始朝向
}
```

### 每帧更新

```cpp
void QuatApp::UpdateScene(float dt)
{
    mAnimTimePos += dt;
    if (mAnimTimePos >= mSkullAnimation.GetEndTime())
        mAnimTimePos = 0.0f;  // 循环

    mSkullAnimation.Interpolate(mAnimTimePos, mSkullWorld);
}
```

---

## 本章小结

| 主题 | 核心要点 |
|------|---------|
| **复数推广** | `i² = -1` → 单位复数乘法 = 2D 旋转 |
| **四元数定义** | `q = (u, w) = (x, y, z, w)` |
| **乘法** | `(u,a)(v,b) = (av + bu + u×v, ab - u·v)`，**不交换、结合** |
| **共轭/范数** | `q* = (-u, w)`，`\|\|q\|\| = √(x²+y²+z²+w²)` |
| **逆** | `q⁻¹ = q*/\|\|q\|\|²`；单位四元数 `q⁻¹ = q*` |
| **极坐标** | 单位四元数 `q = (sin θ · n, cos θ)`，`n` 是旋转轴 |
| **旋转算子** | `R_q(v) = qvq*`，绕 `n` 旋转 **2θ** |
| **构造公式** | 绕轴 `n` 转角度 `θ`：`q = (sin(θ/2)·n, cos(θ/2))` |
| **q 与 -q** | 同一朝向，但插值路径不同（短弧 vs 长弧） |
| **SLERP** | `(sin((1-t)θ)·a + sin(tθ)·b) / sin θ`，匀速球面插值 |
| **退化处理** | 小角度退化为 LERP+normalize |
| **DXMath** | `XMQuaternionSlerp` 等函数封装了所有运算 |

---

## 课后练习思路

1. 复数运算练习：加减乘除、共轭、模长。
2. 用复数乘法把 `(2, 1)` 旋转 30°。
3. 证明 `det M = 1 且 M⁻¹ = M^T` ⟺ `M` 是旋转矩阵。
4. 四元数运算：`(1,2,3,4)` 与 `(2,-1,1,-2)` 的各种运算。
5. 求绕 `(1,1,1)` 转 45°、绕 `(0,0,-1)` 转 60° 的单位四元数。
6. 证明 `(pq)* = q*p*`、`\|\|pq\|\| = \|\|p\|\|·\|\|q\|\|` 等性质。

---

> **下一步**：第 23 章将把单四元数旋转推广到**角色骨骼层级**——蒙皮 mesh、关键帧动画、final transform 计算和顶点蒙皮。
