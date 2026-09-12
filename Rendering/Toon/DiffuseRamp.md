由 $N \cdot L$ 计算：

$$
NoL_{half} = \frac{N \cdot L + 1}{2}
$$

加入偏移量 $Offset$ 并截断得到：

$$
G = Clamp(NoL_{half} + Offset, 0, 1)
$$

合并 $AO$ 与 $Shadow$ 得到：

$$
S = \min(AO, Shadow, 1)
$$

将 $S$ 转换到 Cosine 空间得到：

$$
S' = \cos(S\pi) \times (-0.5) + 0.5
$$

计算最终 Ramp 查询坐标：

$$
RampU = \min(G, S')
$$

从 Atlas 采样得到：

$$
R = (r, g, b, a)
$$

提取：

$$
A = \mathrm{Ramp}.a
$$

与：

$$
C_{\mathrm{Ramp}} = \mathrm{Ramp}.rgb
$$

混合颜色：

$$
C_{\mathrm{Toon}} = (1 - A)C_s + A C_b
$$

计算最终光照：

$$
C_{\mathrm{final}} = C_{\mathrm{Toon}} \times C_{\mathrm{Ramp}} \times L
$$
