# Toon Diffuse Ramp 数学模型

## 1. 核心算法思想

用光照强弱去查一张提前画好的颜色渐变表（Ramp Atlas），查到的透明度和颜色决定物体哪里亮、哪里暗、暗部是什么色，从而做出可控的卡通明暗。

## 2. 应用场景

卡通渲染的漫反射着色，比如二次元角色、风格化场景，用来替代传统平滑漫反射，得到硬边阴影和分层明暗。

## 3. 符号与参数说明

| 符号 | 含义 | 来源 / 计算方式 | 范围 |
|------|------|----------------|------|
| $N \cdot L$ | 法线与光照方向点积 | 光照计算 | $[-1, 1]$ |
| $NoL_{half}$ | 半 Lambert 映射 | $\frac{N \cdot L + 1}{2}$ | $[0, 1]$ |
| $Offset$ | 明暗交界线偏移量 | `DiffuseColorRampUVOffset` 映射而来 | $[-Range, Range]$ |
| $G$ | 加入偏移后的光照梯度 | $\mathrm{Clamp}(NoL_{half} + Offset, 0, 1)$ | $[0, 1]$ |
| $AO$ | 环境光遮蔽 | GBuffer | $[0, 1]$ |
| $Shadow$ | 阴影系数 | 阴影计算 | $[0, 1]$ |
| $S$ | AO 与 Shadow 合并结果 | $\min(AO, Shadow, 1)$ | $[0, 1]$ |
| $S'$ | 转换到 Cosine 空间后的 $S$ | $\cos(S\pi) \times (-0.5) + 0.5$ | $[0, 1]$ |
| $RampU$ | 最终横向查询坐标 | $\min(G, S')$ | $[0, 1]$ |
| $Height$ | Ramp Atlas 总行数 | 纹理尺寸 | 整数 |
| $Index$ | 当前 Ramp 行号 | Material 参数 `DiffuseColorRampIndex` | $[0, Height-1]$ |
| $R$ | Atlas 采样结果 | $R = (r, g, b, a)$ | RGBA |
| $A$ | 明暗混合因子 | $Ramp.a$ | $[0, 1]$（1=亮面，0=暗面） |
| $C_{\mathrm{Ramp}}$ | 颜色调制 | $Ramp.rgb$ | RGB |
| $C_s$ | 阴影颜色 | Material 参数 `ShadowColor` | RGB |
| $C_b$ | 基础颜色 | Material 参数 `BaseColor` | RGB |
| $L$ | 灯光颜色与衰减 | 光照计算 | RGB |

> **备注**：`Offset` 的完整计算为
>
> $$
> Offset = (OffsetInput \times 2 - 1) \times Range
> $$
>
> 其中 $OffsetInput$ 来自 Material 的 `DiffuseColorRampUVOffset`，范围 $[0,1]$。
>
> `RampV` 坐标计算为
>
> $$
> RampV = \frac{Index + 0.5}{Height}
> $$
>
> 其中 $+0.5$ 是为了采样行中心，避免采到边界。

## 4. 数学推导

### 4.1 光照梯度计算

由 $N \cdot L$ 计算半 Lambert 值：

$$
NoL_{half} = \frac{N \cdot L + 1}{2}
$$

### 4.2 加入偏移量

加入偏移量 $Offset$ 并截断到 $[0,1]$：

$$
G = \mathrm{Clamp}(NoL_{half} + Offset, 0, 1)
$$

### 4.3 合并 AO 与 Shadow

将环境光遮蔽和阴影合并，得到遮蔽上限：

$$
S = \min(AO, Shadow, 1)
$$

### 4.4 转换到 Cosine 空间

因为 $N \cdot L$ 本质是 $\cos(\theta)$，属于非线性的 Cosine 空间，而 $AO$ 和 $Shadow$ 是线性空间，所以需要将 $S$ 转换到同一分布：

$$
S' = \cos(S\pi) \times (-0.5) + 0.5
$$

该映射满足：

| $S$ | $S'$ |
|-----|------|
| 0   | 0    |
| 0.5 | 0.5  |
| 1   | 1    |

### 4.5 计算 Ramp 查询坐标

光照梯度决定明暗趋势，AO/Shadow 作为上限截断：

$$
RampU = \min(G, S')
$$

### 4.6 采样 Ramp Atlas

从 Atlas 采样得到 RGBA：

$$
R = (r, g, b, a)
$$

提取混合因子与颜色调制：

$$
A = \mathrm{Ramp}.a
$$

$$
C_{\mathrm{Ramp}} = \mathrm{Ramp}.rgb
$$

### 4.7 混合颜色与最终光照

在阴影色与基础色之间按 $A$ 插值：

$$
C_{\mathrm{Toon}} = (1 - A)C_s + A C_b
$$

最终漫反射光照：

$$
C_{\mathrm{final}} = C_{\mathrm{Toon}} \times C_{\mathrm{Ramp}} \times L
$$

## 5. Ramp Atlas 结构示意

```text
        U (光照输入, 0 → 1)
        ──────────────────────►
Row 0   ██████████░░░░░░░   ← Index = 0, 二值化阴影
Row 1   ████████░░░░░░░░░   ← Index = 1, 3色阶过渡
Row 2   ██████░░░░░░░░░░░   ← Index = 2, 软过渡
...
Row N   ███░░░░░░░░░░░░░░   ← Index = N
```

- **U 轴**：光照输入，对应 $RampU$。
- **V 轴**：Ramp Index，每一行是一条独立的 Color Curve。
- **RGB 通道**：漫反射颜色调制。
- **Alpha 通道**：明暗混合因子，1 为亮面（Base Color），0 为暗面（Shadow Color）。

## 6. 核心公式总结

$$
NoL_{half} = \frac{N \cdot L + 1}{2}
$$

$$
G = \mathrm{Clamp}(NoL_{half} + Offset, 0, 1)
$$

$$
S = \min(AO, Shadow, 1)
$$

$$
S' = \cos(S\pi) \times (-0.5) + 0.5
$$

$$
RampU = \min(G, S')
$$

$$
R = \mathrm{Atlas}\left(RampU, \frac{Index + 0.5}{Height}\right)
$$

$$
C_{\mathrm{Toon}} = (1 - A)C_s + A C_b
$$

$$
C_{\mathrm{final}} = C_{\mathrm{Toon}} \times C_{\mathrm{Ramp}} \times L
$$

## 7. 参考

- MooaToon 官方文档 “控制明暗颜色过渡”：  
  https://mooatoon.com/docs/Tutorial/ControlLightShadowColorTransition
