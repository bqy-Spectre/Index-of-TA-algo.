# M_TV_ToonCloth_Master 卡通布料材质 · 美术使用手册

> 适用对象：会画贴图、但不写着色器的材质/角色美术。
> 文中英文均为材质里的真实参数名或贴图名，第一次出现时附中文解释。
> 所有默认值都来自材质本身的连接与参数默认值（母材质默认值，未被实例覆盖）。
> 本文结合当前节点、实际资产与 UE 5.8 源码共同核对；画面效果仍由美术确认，不代表已做视觉验收。

---

## 0. 开工前先看：这套材质和金属版最大的三个不同

如果你刚做完金属那套，先把这三个差异记住，不然会一直按错误的习惯去画：

1. **布料没有金属度。** `SubstrateToonBSDF` 的 `Metallic`（金属度）接的是一个**常量 0**，没有任何贴图或参数能改它。`ORMTexture` 的 B 通道在这套材质里**完全没接线**——画了也没用。
2. **默认关闭有不同方式**，先检查具体闸门：
   - `Stocking_MSK`（丝袜遮罩）默认是**纯黑** `T_TV_NeutralBlack_MSK` → 丝袜整条链路权重为 0，全关。
   - `DetailNormalStrength`（细节法线强度）默认 **0** → 细节法线无效。
3. **有三个通道是"0.5 才是中性值"**，不是白色。它们都做了 `×2 − 1` 或 `×2` 的重映射，画纯白会**过冲**：
   - ClothEffectsA_MSK.R（粗糙度倍率，×2；**Scale=1 时 0.5 为单位倍率**）
   - `ClothWeave_MSK.B`（织纹微观粗糙度，×2−1，**0.5 = 不增不减**）
   - `Stocking_MSK.B`（丝袜微观粗糙度，×2−1，**0.5 = 不增不减**）

**另外三个"默认状态"：**

- ORM 默认纯白，在父默认下给出 AO=1、基础粗糙度=1；后者还经过 Effects、织纹、绒面、丝袜和上下限，**不是最终固定为 1 或没有高光**。
- `BaseColorTexture` 默认白块；`NormalTexture` / `DetailNormalTexture` / `ClothWeave_N` 默认都是中性法线（平的）。
- `ClothEffectsA_MSK` 默认源 RGBA=(147,64,0,255)，R≈0.57647，解码粗糙度倍率≈**1.15294，不是 1**；它不是最终粗糙度。
- `ClothEffectsB_MSK` 默认源 RGBA=(179,13,0,255)，**B=0，区域丝袜控制也关闭**。只换白色 Stocking_MSK 不够，Effects B 的 B 也必须非零。

---

## 1. 这套材质能做出什么效果

它是一套 **Substrate（UE 的新材质框架）+ Toon BSDF（卡通着色）** 的布料母材质，最终输出一个 `SubstrateToonBSDF`。链路分五段，各自负责一种质感：

1. **布料本体**：底色 + 法线 + 细节法线 + 粗糙度 + AO，走卡通着色。适合衣服、披风、裙摆、布料装饰。
2. **绒面 / 天鹅绒边缘光泽**（Velvet，天鹅绒）：掠射角处的视角权重更大，颜色向 VelvetTint 混合，同时参与粗糙度和高光处理。**不保证边缘更亮**，还要看 Tint、原底色、强度及后续丝袜染色。见 3.3。
3. **织纹微观结构**（Weave，织物）：一层高频织纹法线 + 一层微观粗糙度扰动，做出"近距离能看到布料经纬"的感觉。它自带独立的 UV 缩放（`WeaveUVScale` 默认 **12**）。
4. **缎面拉丝 / 各向异性**（Anisotropy，各向异性）：改变高光方向性。**切线输入固定，但强度正负可交换两轴粗糙度**，见 2.10。
5. **丝袜**（Stocking）：正常视角范围下，正视保留绒面后的底色，掠射朝 StockingDarkColor 混合，再处理高光和粗糙度。名字叫暗部色不代表一定比当前底色暗。

卡通图案由 **Toon Profile（卡通光照配置）**提供，PatternUV 只给它坐标，不是独立贴花槽。当前 Profile 未绑定图案且相关强度均为 0，只调 Scale 不会产生网点或排线。见 3.7。

**做不到 / 别在这里找的（拓扑里没有的东西）：**

- 金属：`Metallic` 硬编码为 0，无法开启。
- 自发光：`EmissiveColor`（自发光）接的是一个**常量黑 (0,0,0)**，改不了。
- 透明、半透、镂空：材质是 `BLEND_Opaque`（不透明）、`MD_Surface`（表面域）、`TwoSided = False`（单面）。
- 所有表面图用 UV0，主图没有平铺参数。**两个缩放参数控制三张采样图**：DetailUVScale 控制细节法线，WeaveUVScale 同时控制织纹法线和织纹遮罩。
- 图里没有独立 Fresnel 边缘染色节点，绒面和丝袜以法线·相机向量计算额外视角响应；**不是最终高光只有这两处随视角变化**，BSDF 着色本身也依赖视线。

---

## 2. 贴图怎么制作

### 2.0 九张贴图总览

| # | 贴图参数名 | 采样器类型 | 默认资源 | 实际用到的通道 | UV |
|---|---|---|---|---|---|
| 1 | `BaseColorTexture` | Color（颜色） | `WhiteSquareTexture` | RGB | UV0 |
| 2 | `ORMTexture` | Masks（遮罩） | `T_TV_NeutralWhite_MSK` | **R=AO、G=粗糙度**（B、A 未用） | UV0 |
| 3 | `NormalTexture` | Normal（法线） | `T_TV_Neutral_N` | RGB | UV0 |
| 4 | `DetailNormalTexture` | Normal（法线） | `T_TV_Neutral_N` | RGB | UV0 × `DetailUVScale` |
| 5 | `ClothEffectsA_MSK` | Masks | `T_TV_ToonCloth_Default_EffectsA_MSK`，源 RGBA=(147,64,0,255) | **R / G / B 三个都用** | UV0 |
| 6 | `ClothEffectsB_MSK` | Masks | `T_TV_ToonCloth_Default_EffectsB_MSK`，源 RGBA=(179,13,0,255) | **R / G / B 三个都用** | UV0 |
| 7 | `ClothWeave_N` | Normal（法线） | `T_TV_Neutral_N` | RGB | UV0 × `WeaveUVScale`(12) |
| 8 | `ClothWeave_MSK` | Masks | `T_TV_NeutralWhite_MSK` | **只有 B 和 A**（R、G 未用） | UV0 × `WeaveUVScale`(12) |
| 9 | `Stocking_MSK` | Masks | `T_TV_NeutralBlack_MSK` | **R / G / B / A 四个都用** | UV0 |

Effects 默认图的 RGB 来自本次实际资产源图导出；导出器输出不透明 RGB，表中的 Alpha=255 是补值，不是独立 Alpha 读回。Effects 的 Alpha 没有接入算法，不影响上述默认控制值。

---

### 2.1 `BaseColorTexture`（底色贴图）

**画什么**

布料本身的颜色。主体色、花纹、拼色和绣线都画这里，明度与饱和度按角色色设决定；算法没有规定布料一定比金属更亮或更饱和。

**画完什么效果**

链路：`BaseColorRaw（底色原始值）= Saturate(（钳制到 0~1） BaseColorTexture.RGB × BaseColorTint（底色染色，默认白）)`

- **直接相乘，没有反相、没有重映射**。白色贴图 = 保留 `BaseColorTint` 的颜色，黑色 = 变黑。
- Saturate 限制这一步底色在 0~1，**不是禁止 Tint>1 提亮**。例如某通道底色 0.2、Tint=2，结果为 0.4；超过 1 的部分才截断。这是底色输入，不保证最终画面亮度翻倍。

**怎么配合其他贴图**

- 底色之后**先绒面、再丝袜**：先向 VelvetTint 混合（3.3），丝袜再以绒面后的底色做视角染色（2.9）。白色 Tint 不保证任何底色都提亮，DarkColor 也不保证比当前底色暗；两路不是永远相反的开关。
- 底色**不管**粗糙度、AO、织纹——那些在别的图上。

**怎么导入 / 怎么检查**

- 使用 Color sampler，**sRGB 开启、TC_Default**。当前默认 WhiteSquareTexture 双轴 Wrap，Mip 为 Sharpen4。只用 RGB，UV0；新图的 Mip 策略由 TA 确定。
- 颜色不对 → 先看 `BaseColorTint` 是否被实例改过（默认应纯白）；再把绒面和丝袜两条关掉排除干扰（绒面：`VelvetTintStrength` 归 0；丝袜：确认 `Stocking_MSK` 为黑）。

---

### 2.2 `ORMTexture`（AO / 粗糙度二合一）★注意：B 通道没用

通道分工由 `ComponentMask`（通道遮罩）节点**明确指定**：

- **R 通道 → AO（环境光遮蔽）**：`AO = Saturate(R × AOMultiplier)`，直接进材质根节点的 `AmbientOcclusion`。
- **G 通道 → 粗糙度**：`RoughnessRaw = Saturate((G × RoughnessMultiplier) + RoughnessBias)`。
- **B 通道、A 通道：拓扑里没有接线，画了无效。**（金属版里 B 是金属度，这里不是，别画错。）

**画什么**

- **R（AO）**：画褶皱深处、布料层叠处、缝线沟、衣领与脖子的交界、口袋和腰带压住的位置 → 画黑；大面积平坦布面 → 留白。
- **G（粗糙度）**：画"反光散不散"。
  - 需要更光滑的丝绸、缎面区域 → 相对周围画暗；
  - 需要更粗糙的棉麻、毛呢区域 → 相对周围画亮；这是输入方向，不是已签核的数值配方。
  - 同一件衣服上被磨亮的位置（袖口、肘部、屁股坐的地方）→ 画得比周围暗一点，做出"磨光"的读法。

**画完什么效果（两个通道都没有反相，全是直接相乘）**

| 通道 | 白色（1） | 黑色（0） | 灰色（0.5） |
|---|---|---|---|
| R（AO） | 父倍率为 1 时输入为 1，不额外遮蔽 | 输入为 0，遮蔽最强；不保证任何光照下全黑 | 输入为 0.5，不等于画面亮度减半 |
| G（粗糙度） | 父倍率／偏置下基础值为 1 | 基础值为 0，后续还有偏移和上下限 | 基础值为 0.5 |

默认白图使基础粗糙度为 1、AO 为 1，**不保证整件布完全哑光**。判断最终粗糙度要沿 3.2 的后续链路检查。

**关于 `RoughnessBias`（粗糙度偏移，默认 0）**

它是**加法**：G×RoughnessMultiplier+RoughnessBias，然后钳 0~1。负 Bias 不一定使结果为 0，例如 G=0.5、倍率=1、Bias=−0.1，结果仍为 0.4；合成值低于 0 才截断。

**这张图不是粗糙度的终点**

`RoughnessRaw` 之后还会经过织纹、绒面、丝袜三道加工（见 3.2）。所以 ORM 的 G 是"**粗糙度的基础值**"，不是最终值。

**怎么导入**

使用 **sRGB 关闭、TC_Masks**，匹配 Masks sampler，UV0。当前默认白遮罩双轴 Wrap、NoMipmaps；这是常量占位图的设置，新画的细节图应准备 Mip。Masks 是预设，**不等于固定 BC7**，平台格式由纹理构建决定。

**怎么检查**

改了没反应 → 确认画在 **G** 通道（不是 B！）；确认这张 ORM 有没有被实例上的贴图槽覆盖；用 `RoughnessMultiplier` 从 1 拉到 0 做 A/B 对比。

---

### 2.3 `NormalTexture`（主法线贴图）

**画什么**

布料的大形起伏：褶皱、缝线、纽扣、拉链、口袋边缘、布料层叠的厚度。按常规切线空间法线规格画（材质里 `bTangentSpaceNormal = True`）。

**画完什么效果**

进入混合前，主法线本身**不做强度插值**——它直接作为"基础法线"送进 `BlendAngleCorrectedNormals`（角度校正法线混合，引擎自带函数）。也就是说：

- **这张图没有强度参数**，画多强就是多强。想减弱只能改贴图本身。
- 强度参数 `NormalStrength` 在这套布料材质里**不存在**（金属版有，这里没有）。

**怎么配合其他贴图**

- 细节法线先与主法线混合，织纹再叠一层，使用角度校正混合；不是简单相加，也不保证所有强度和距离下细节都清楚可见。
- 最终进 BSDF 的是**织纹那一层**的输出，也就是三层都叠完之后的结果。
- 法线会改变 `dot(法线, 相机向量)`，从而**连带影响绒面边缘光和丝袜的视角分档**。褶皱画得越碎，绒面/丝袜的明暗流动也越碎——想让边缘光干净，法线要克制。

**怎么导入 / 怎么检查**

法线使用 **sRGB 关闭、TC_Normalmap**，匹配 Normal sampler。当前默认 T_TV_Neutral_N 双轴 Wrap，Mip 为 FromTextureGroup。用 UV0；压缩预设不等于固定的平台编码格式。
看不到凹凸 → 检查模型有没有切线/UV；检查绿通道方向（OpenGL / DirectX）是否与项目一致。

---

### 2.4 `DetailNormalTexture`（细节法线贴图）

**画什么**

近距离才看得到的布料细纹：纤维走向、细密的编织纹、起球的颗粒感、皮革的毛孔。

**画完什么效果**

链路：`附加法线 = Lerp(（插值）平面法线 (0,0,1), DetailNormalTexture.RGB, DetailNormalStrength)`

- 和金属版完全同样的插值算法，但 **`DetailNormalStrength` 默认 0** → 默认完全不起作用。
- 平面法线常量在拓扑里确认是 `(0,0,1)`，所以贴图的中性值必须用常规中性法线色（约 `#8080FF`）。

**怎么配合其他贴图**

- 它和织纹（2.7）是**两个独立的细节层**：这层用 UV0 × `DetailUVScale`（默认 1），织纹那层用 UV0 × `WeaveUVScale`（默认 12）。想要"大而疏的纤维纹 + 小而密的织纹"就两层都用；嫌麻烦只用织纹那层也行。
- 它不直接写粗糙度或 Specular 基础值，但会改变绒面视角权重，**间接影响最终粗糙度和 Specular**，也会改变丝袜染色视角。没有直接连线不等于完全无联动。

**怎么导入 / 怎么检查**

同 2.3。没反应先查实例覆盖、贴图、DetailNormalStrength 是否非零（默认 0），再查图案和密度，不给没有统计依据的诊断概率。

---

### 2.5 `ClothEffectsA_MSK`（效果控制图 A）★一图管三路

这张图的三个通道分别喂给三条完全不同的链路，由 `ComponentMask`（通道遮罩）明确指定：

| 通道 | 输出名 | 计算 | 喂给谁 |
|---|---|---|---|
| **R** | `RoughnessControl`（粗糙度控制） | **R × 2.0** | 乘进粗糙度（见 3.2） |
| **G** | `SpecularControl`（高光控制） | 原值 | 乘进 Specular 输入（见 3.1） |
| **B** | `AnisotropyControl`（拉丝控制） | 原值 | 乘进缎面拉丝强度（见 3.5） |

**画什么**

- **R（粗糙度分区 ×2）**：在同一件衣服上区分不同材质区域。缎面裙摆画暗（<0.5，更光滑），毛呢外套画亮（>0.5，更粗糙），**普通区域画 0.5（正好不改变）**。
- **G（高光分区）**：哪里反光更强。金属丝绣线、亮片、漆皮 → 画白；哑光棉麻 → 画灰到黑。
- **B（缎面拉丝分区）**：只有缎面、丝绸这类有方向性反光的布料才画白；棉布、毛呢画黑（否则会出现不该有的条状高光）。

**画完什么效果（重点：R 通道有 ×2 重映射）**

- **R 通道**：`RegionRoughness（区域粗糙度）= Max(（取较大值）R × 2.0 × EffectRoughnessScale, 0)`，然后 `RoughnessAfterWeave = Clamp(（钳制）RoughnessRaw × RegionRoughness + 织纹微观粗糙度, MinRoughness, MaxRoughness)`。
  - R=0.5、EffectRoughnessScale=1 → 基础粗糙度乘项倍率为 1；只是不改这一步乘项，不关闭后续加工。
  - **R = 1.0（纯白）** → `×2 = 2.0` → 粗糙度**翻倍**（会更粗糙，且很容易被 `MaxRoughness` 钳住）
  - R=0 → 基础粗糙度乘项为 0；织纹偏移、绒面、丝袜和 Min/Max 仍生效，**不等于最终完全镜面**。
  - 注意 `RegionRoughness` 是**乘**在 `RoughnessRaw` 上的，所以它管的是"在 ORM 基础上的倍率"，不是绝对值。ORM 的 G 是底子，这张是倍率。
  - 还有一层 `Max(..., 0)`，所以负值会被压成 0，不会出现负粗糙度。
- **G 通道**：原值参与 Specular 输入乘法（见 3.1），白色保留、灰黑减弱；没有反相，不保证画面亮度与灰度成正比。
- **B 通道**：原值乘进拉丝强度，白色 = 拉丝更强，**没有反相**。

**关于"是否和 `Effect*Scale` 重复"**

贴图决定局部数值，Scale 是全实例倍率。EffectControls 只解码（R×2，其余直出）；Scale 在各路消费处参与：Roughness 乘后取 Max(0)，Specular 基础乘项后 Saturate，Anisotropy 乘后 Clamp(−1,1)。不能概括成“中间没有最大值或限制”。

贴图画局部分布，Scale 调整体倍率，父默认均为 1。**0~2 是 UI 滑条范围，不是所有链路的硬限制**；输入更大的数仍按节点运算，可能饱和、钳制或在后续 Lerp 外插。局部错误先查图，不用全局倍率补另一块区域。

**怎么导入 / 怎么检查**

使用 **sRGB 关闭、TC_Masks**，UV0。当前默认双轴 **Clamp**，Mip 为 **LeaveExistingMips（保留已有 Mip）**；不是织纹的 Wrap，也不是自动重新生成效果图 Mip。
默认 T_TV_ToonCloth_Default_EffectsA_MSK 已从当前资产导出源像素：1×1，RGBA=**(147,64,0,255)**。R=147/255≈0.57647，默认 Scale=1 时解码倍率≈**1.15294，不是 1**；G≈0.25098，B=0。这些是源数据解码值，平台压缩和采样可能带来误差；最终粗糙度还经过 ORM、织纹、绒面、丝袜和 Clamp，不能直接写成 1.15294。

---

### 2.6 `ClothEffectsB_MSK`（效果控制图 B）★一图管三路

同样由 `ComponentMask` 明确指定，三个通道分别喂给另外三条链路：

| 通道 | 输出名 | 计算 | 喂给谁 |
|---|---|---|---|
| **R** | `WeaveControl`（织纹控制） | 原值 | 织纹法线和微观粗糙度的总闸门 |
| **G** | `FiberControl`（绒面控制） | 原值 | 绒面边缘光的强度 |
| **B** | `StockingControl`（丝袜控制） | 原值 → `× EffectStockingScale` → `Saturate` | 丝袜的总权重 |

**画什么**

- **R（织纹分区）**：哪里要看到织物纹理。外层面料画白，里衬、皮革、金属扣件画黑。**这是织纹的总开关**——画黑的地方，织纹法线和织纹粗糙度**同时**失效。
- **G（绒面分区）**：哪里要天鹅绒/丝绒的边缘泛白。丝绒领子、天鹅绒裙摆画白；棉布、皮革、金属画黑。**画黑 = 完全不出现绒面边缘光。**
- **B（丝袜分区）**：只有丝袜/紧身袜覆盖的身体部位画白，其余全部画黑。丝袜链路的**一切**都乘以这个权重。

**画完什么效果**

- 三个通道直接参与控制，没有反相；白色保留更大输入，灰度减弱，黑色关闭对应区域贡献。有效控制还受 Scale 和后续 Saturate 影响，**不是画面强度始终与灰度成比例**。
- R 与 EffectWeaveScale 相乘后 Saturate；G 与 EffectFiberScale 在绒面权重链相乘后再 Saturate，还受视角、破碎和粗糙度影响。它们不是与下游运算无关的纯乘法闸门。
- B 通道多一步：`RegionStocking（丝袜区域）= Saturate(ClothEffectsB_MSK.B × EffectStockingScale)`，被钳到 0~1，所以它**不会过冲**。

**这张图和 `Stocking_MSK` 的分工（重要，别搞混）**

- `ClothEffectsB_MSK.B` = "**这件衣服上哪块区域是丝袜**"（大块分区，决定丝袜链路开不开）
- Stocking_MSK（2.9）控制丝袜内部覆盖、高光和粗糙度细节，可设计厚薄观感；**不提供透明、皮肤透射或真实厚度**。
- 两者**相乘**：前者为 0 的地方，后者画得再花也没用。

**怎么导入 / 怎么检查**

使用 **sRGB 关闭、TC_Masks**，UV0。当前默认双轴 **Clamp**，Mip 为 **LeaveExistingMips**。T_TV_ToonCloth_Default_EffectsB_MSK 已导出源像素：1×1，RGBA=**(179,13,0,255)**，R≈0.70196、G≈0.05098、**B=0**。区域丝袜控制默认关闭，只换 Stocking_MSK 不会绕过它。源值不等于平台压缩后读回值。

---

### 2.7 `ClothWeave_N`（织纹法线贴图）

**画什么**

单个"经纬交错的织纹单元"。斜纹、平纹、缎纹、针织罗纹、甚至皮革纹都可以。它是一张**会被重复很多次的小图**，不是覆盖整件衣服的大图。

**画完什么效果**

链路：
```
织纹强度 = WeaveNormalStrength（默认 0.35）× Saturate(EffectWeaveScale × ClothEffectsB_MSK.R)
叠加法线 = Lerp(平面法线 (0,0,1), ClothWeave_N.RGB, 织纹强度)
最终法线 = BlendAngleCorrectedNormals(基础法线 = 主+细节法线, 附加法线 = 叠加法线)
```

- 这是平面法线与织纹法线的插值。父默认 WeaveNormalStrength=0.35，还要乘有效闸门；默认 EffectsB.R=179/255 时有效强度约 **0.24569**。默认织纹图是中性法线，所以有非零强度也不会凭空生成针织起伏。
- 织纹强度还乘了一道闸门 `Saturate(EffectWeaveScale × ClothEffectsB_MSK.R)`，所以 **2.6 的 R 通道是总开关**——那里画黑，这里画了也白画。

**UV 缩放（重要）**

`ClothWeave_N` 用的是 **UV0 × `WeaveUVScale`，默认 12**。也就是说织纹默认被重复 12 次。这是这套材质里唯一一个默认就很大的平铺值。

- 想让织纹更密 → 调大 `WeaveUVScale`；更疏 → 调小。
- 这个值**同时作用于 `ClothWeave_N` 和 `ClothWeave_MSK`**（两张图共用一个 UV 计算），所以两张图的纹理密度必须匹配。

**怎么导入 / 怎么检查**

使用 sRGB 关闭、TC_Normalmap，匹配 Normal sampler。当前默认 T_TV_Neutral_N 双轴 Wrap，Mip 为 FromTextureGroup；平铺织纹法线要有 Mip，不把预设写成固定 BC5／BC7。
看不到织纹 → 依次查：`ClothEffectsB_MSK.R` 在你观察的位置是不是白；`WeaveNormalStrength` 是不是被调成 0；`WeaveUVScale` 是不是大到糊成一片。

---

### 2.8 `ClothWeave_MSK`（织纹遮罩图）★只有 B 和 A 通道有用

拓扑确认：这张图以 RGBA（V4）采样，但**只取了 B 和 A 两个通道**，R 和 G 没有接线。

| 通道 | 输出名 | 计算 | 作用 |
|---|---|---|---|
| **B** | 微观粗糙度 | `B × 2 − 1`（重映射）→ `× WeaveRoughnessAmplitude` → `× Saturate(EffectWeaveScale × ClothEffectsB_MSK.R)` | 加进最终粗糙度 |
| **A** | `FiberBreakup`（纤维破碎） | 原值 | 打散绒面边缘光的均匀度 |
| R、G | — | **未使用** | 画了无效 |

**画什么**

- **B（微观粗糙度 ×2−1）**：默认正幅度下，**亮于 0.5 增加粗糙度，暗于 0.5 降低粗糙度**。要让磨光凸起更光滑就画更暗，别画反。0.5 是理想零偏差，白色解码 +1；幅度改负数时方向会反过来。
- **A（纤维破碎）**：一张噪声图，用来让绒面边缘光**不均匀**。柔和的中低频噪声最合适；画成高频纯白噪声会让边缘光闪烁。**默认纯白 = 1 = 均匀无破碎。**

**画完什么效果**

- B 通道的完整链路：`MicroRoughness（微观粗糙度）= (B × 2 − 1) × WeaveRoughnessAmplitude（默认 0.05）× 闸门`
  - 默认幅度 0.05、有效闸门在 0~1 时，这一步最多 ±0.05；当前默认 EffectsB.R=179/255，白 B 偏移约 **+0.03510**。改幅度或被 Clamp 截断时，不继续把 ±0.05 当固定上限。
  - 增大幅度能放大差异；但全白 B 只给恒定正偏移，不会产生起伏，起伏要画不同灰度。
- A 通道：`FiberBreakup` 会进绒面链路，和 `VelvetFiberBreakupStrength`（默认 0.5）做插值：`Lerp(1, FiberBreakup, VelvetFiberBreakupStrength)`。
  - A=1（白）→ 结果 1 → 绒面**不受影响**；
  - A=0、BreakupStrength=0.5 → 破碎乘项为 0.5；最终 Saturate 未触顶时权重减半，**不保证画面亮度或饱和后的权重减半**。
  - 也就是说：**黑 = 削弱绒面，白 = 绒面完整**。这是个反直觉的点，注意别画反。

**怎么导入 / 怎么检查**

使用 **sRGB 关闭、TC_Masks**，只需画 B 和 A，UV 同织纹法线。当前默认白遮罩双轴 **Wrap**、NoMipmaps；真正有细节的平铺图应保留 Wrap 并准备 Mip，不要照搬 Effects 的 Clamp 或占位图的无 Mip 设置。
改了 B 没反应 → 查 `WeaveRoughnessAmplitude` 是否为 0、查 `ClothEffectsB_MSK.R` 闸门是否为白。

---

### 2.9 `Stocking_MSK`（丝袜遮罩图）★四个通道全用，且默认全黑

拓扑确认：这张图以 RGBA（V4）采样，**R / G / B / A 四个通道全部接线**，各管一件事：

| 通道 | 输出名 | 计算 | 作用 |
|---|---|---|---|
| **R** | `StockingWeight`（丝袜权重） | `Saturate(R) × RegionStocking` → `Saturate` | 丝袜链路的总权重，**决定丝袜有没有效果** |
| **G** | `StockingSpecularDetail`（丝袜高光细节） | `Saturate(G)` | 该处高光的倍率 |
| **B** | 微观粗糙度偏移 | `B × 2 − 1`（重映射） | 丝袜区域的粗糙度微调 |
| **A** | `StockingAnisotropyMask`（丝袜拉丝遮罩） | `Saturate(A)` | 该处是否放大缎面拉丝 |

**画什么**

- **R（丝袜覆盖观感，最重要）**：覆盖区画白，不覆盖处画黑，灰度按比例改变权重。它不是物理厚度或透明度；Opaque 材质不会因灰度变化露出真实皮肤。
  - **默认整张图是纯黑 → R=0 → 丝袜完全关闭。** 这是这套材质最大的一个"画了没效果"陷阱。
- **G（高光细节）**：丝袜的高光强弱分布。想让高光在膝盖、小腿前侧更明显 → 那里画白；膝盖窝、阴影处画灰。
  - **G=0 不一定关掉高光**：它将 Specular 输入乘 1−丝袜权重，只在权重=1 时该输入归零；部分覆盖不能套用全覆盖结论。
- **B（微观粗糙度 ×2−1）**：丝袜本身的表面质感微调。0.5 = 中性。默认黑 → `×2−1 = −1` → 全额的负偏移（但因为 R=0 权重为 0，实际不起作用）。
- **A（已有缎面各向异性的倍率覆盖）**：白用 StockingAnisotropyScale，黑保留倍率 1；**黑不是关闭拉丝**。原输入为 0 时，丝袜倍率也不能凭空产生各向异性。

**画完什么效果（完整链路，全部来自拓扑）**

```
丝袜权重   StockingWeight = Saturate( Saturate(R) × Saturate(ClothEffectsB_MSK.B × EffectStockingScale) )

颜色：     视角项 = SmoothStep(（平滑阶梯）StockingViewMin(0.15), StockingViewMax(0.85), Saturate(法线·相机向量))
          分档色 = Lerp(StockingDarkColor（丝袜暗部色，默认 (0.35,0.35,0.40)), 绒面后的底色, 视角项)
          最终   = Lerp(绒面后的底色, 分档色, StockingWeight × StockingColorStrength（默认 0.35）)

高光：     FinalSpecular = Saturate(绒面后的高光 × Lerp(1, StockingSpecularScale（默认 1.25）× Saturate(G), StockingWeight))

粗糙度：   StockingRoughnessOffset = StockingWeight × (StockingRoughnessBias（默认 −0.05）
                                    + (B × 2 − 1) × StockingMicroRoughnessAmplitude（默认 0.04）)
          FinalRoughness = Clamp(绒面后的粗糙度 + StockingRoughnessOffset, MinRoughness, MaxRoughness)

拉丝：     FinalAnisotropy = Lerp(缎面拉丝值, Clamp(缎面拉丝值 × Lerp(1, StockingAnisotropyScale（默认 1）, Saturate(A)), −1, 1), StockingWeight)
```

逐个通道翻译成画面语言：

- **R**：权重。**0 = 丝袜完全不生效**（颜色、高光、粗糙度、拉丝四处全部走原值）；1 = 丝袜全额生效。
  - 颜色混合系数是丝袜权重×StockingColorStrength。默认 0.35、完整区域控制且 R=1 时，向分档色混合 35%；正视时分档色就是原色，因此仍可能不变。调大强度加重混合，系数越界会外插，不保证只是更暗。
  - **视角方向要注意**：保持 StockingViewMin < StockingViewMax，默认 **0.15 / 0.85**。N·V≤Min 时分档色取 DarkColor；N·V≥Max 时取**绒面后的底色**。最终还乘丝袜权重和颜色强度，不是整片换色。DarkColor 比这份底色暗才会变暗，比它亮就会变亮。
  - **不要反置 Min/Max 当反转开关**：当前直接接到 SmoothStep，没有排序／保护；变化输入由编译器发出 HLSL smoothstep。Min>Max 或相等都不作为可靠的正常设置，本文未对非法顺序做 GPU 实测。改变边缘明暗请改 DarkColor。
- **G**：高光倍率。因为它是 `Lerp(1, StockingSpecularScale × G, 权重)` 里的 `B` 端，所以：
  - G=0 → 倍率=1−权重：权重 0／0.5／1 时为 1／0.5／0；只在全权重时该 Specular 输入归零，不保证任何角度的所有反射消失。
  - G=1、默认 SpecularScale=1.25 → 倍率=1+0.25×权重：权重 0／0.5／1 时为 1／1.125／1.25。后续 Saturate 和光照还影响结果，不是画面必定亮 25%。
  - 想让丝袜比周围亮得更多 → 调大 `StockingSpecularScale`。
- **B**：默认 Bias=−0.05、Amplitude=0.04 时，B=0／0.5／1 的偏移分别为 **−0.09／−0.05／−0.01，再乘丝袜权重**。B=1 不是零偏移；B=0.5 只消微观项，不消 Bias。默认权重=0 时实际偏移为 0，最终还受 Clamp 约束。
- **A**：已有各向异性的倍率覆盖。默认 Scale=1 时黑白都不改倍率；Scale<1 可削弱，>1 可增强；A=0 保留原倍率，不关闭原输入。

**怎么导入 / 怎么检查**

使用 **sRGB 关闭、TC_Masks**，UV0。当前默认黑遮罩双轴 Wrap、NoMipmaps；新画的细节图应准备 Mip。默认纯黑，加上 Effects B 的 B=0，两道闸门都关闭。
丝袜完全没效果 → 按这个顺序查：① `Stocking_MSK` 是否被实例换成你画的图（默认是纯黑）；② R 通道是不是画了白；③ `ClothEffectsB_MSK.B` 对应区域是不是白；④ `EffectStockingScale` 是不是被调成 0；⑤ `StockingColorStrength` 是不是 0（颜色不变化但高光会变）。

---

### 2.10 关于缎面拉丝的方向：它是写死的，改不了

这是这套材质最需要注意的一点，写在贴图章节末尾因为很多美术会去找"方向贴图"：

```
AnisotropyFinal = Clamp(（钳制）ClothEffectsA_MSK.B × 常量 1.0 × EffectAnisotropyScale, −1, 1)
TangentFinalTS  = 常量 (1, 0, 0)        ← 无方向贴图；UE 通过模型切线基底转到世界空间
```

- **切线输入固定为 TS +X（通常对应 UV 的 U 方向）**。图里无方向贴图或显式顶点切线节点，但母材质开启 Tangent Space Normal，UE 仍用模型切线基底转到世界空间；不等于“与模型切线无关”。
- `AnisotropyMask`（拉丝遮罩）接的是一个**常量 1.0**，不是参数，改不了。
- MI 没有连续旋转切线的接口；切线走向需与模型组在 DCC 中处理 UV／切线。强度正负不是自由旋转方向的旋钮。
- 缎面输入钳到 **−1~1**。对应各向异性高光计算中，同等绝对值改变符号会**交换切线／副切线两轴粗糙度，不是切线向量反向**。归零是取消各向异性；丝袜还会塑形这个值（2.9）。不同光源／计算路径不保证都能明显看到轴变化。

---

### 2.11 导入与采样汇总（节点和默认资产已核对）

| 项 | 拓扑确认的内容 |
|---|---|
| UV 通道 | 全部使用 **UV0**（`ConstCoordinate = 0`、`CoordinateIndex = 0`） |
| 平铺 | 只有两个：`DetailUVScale`（默认 1，作用于 `DetailNormalTexture`）和 `WeaveUVScale`（默认 **12**，同时作用于 `ClothWeave_N` 和 `ClothWeave_MSK`）。其余贴图无平铺参数 |
| 采样器来源 | 全部 `SSM_FromTextureAsset`（跟随贴图资产自身的采样设置） |
| Mip 采样 | 节点均为 TMVM_None，常规自动选级；**不等于资产都有多级 Mip**，还要看下表 |
| 采样器类型 | `BaseColorTexture` = Color；`NormalTexture` / `DetailNormalTexture` / `ClothWeave_N` = Normal；`ORMTexture` / `ClothEffectsA_MSK` / `ClothEffectsB_MSK` / `ClothWeave_MSK` / `Stocking_MSK` = Masks |
| 未使用通道 | `ORMTexture` 的 B、A；`ClothWeave_MSK` 的 R、G |
| 颜色空间与压缩 | BaseColor 开 sRGB、TC_Default；三张法线关 sRGB、TC_Normalmap；五张控制图关 sRGB、TC_Masks。预设不等于固定 BC7／BC5 平台格式 |

九个槽的**当前默认资源**如下，不是要求新图照抄占位图的 Mip 设置：

| 槽位 | 默认资源 | sRGB | Compression | Address X/Y | MipGenSettings |
|---|---|---|---|---|---|
| BaseColorTexture | /Engine/EngineResources/WhiteSquareTexture | 开 | TC_Default | Wrap / Wrap | Sharpen4 |
| NormalTexture | T_TV_Neutral_N | 关 | TC_Normalmap | Wrap / Wrap | FromTextureGroup |
| DetailNormalTexture | T_TV_Neutral_N | 关 | TC_Normalmap | Wrap / Wrap | FromTextureGroup |
| ClothWeave_N | T_TV_Neutral_N | 关 | TC_Normalmap | Wrap / Wrap | FromTextureGroup |
| ORMTexture | T_TV_NeutralWhite_MSK | 关 | TC_Masks | Wrap / Wrap | NoMipmaps |
| ClothWeave_MSK | T_TV_NeutralWhite_MSK | 关 | TC_Masks | Wrap / Wrap | NoMipmaps |
| Stocking_MSK | T_TV_NeutralBlack_MSK | 关 | TC_Masks | Wrap / Wrap | NoMipmaps |
| ClothEffectsA_MSK | T_TV_ToonCloth_Default_EffectsA_MSK | 关 | TC_Masks | Clamp / Clamp | LeaveExistingMips |
| ClothEffectsB_MSK | T_TV_ToonCloth_Default_EffectsB_MSK | 关 | TC_Masks | Clamp / Clamp | LeaveExistingMips |

项目资源位于 /Game/Toon/MaterialLibrary/Support/。**Effects 默认 Clamp，织纹默认 Wrap**；换图后节点跟随新资产的寻址设置。LeaveExistingMips 只保留已有链，不保证外部新图已备好正确的效果图 Mip，交 TA 检查。

---

## 3. 效果怎么调整

> 按"你想改什么画面"来查。所有数值建议在**材质实例（MI）**上覆盖，不要改母材质。

### 3.1 整体高光太弱 / 太强

最终高光是三道计算叠出来的，从后往前排查：

```
VelvetSpecular = Saturate( Saturate(BaseSpecular（默认 0.5） × ClothEffectsA_MSK.G × EffectSpecularScale)
                           × (1 − 绒面权重 × VelvetSpecularSuppress（默认 0.5）) )
FinalSpecular  = Saturate( VelvetSpecular × Lerp(1, StockingSpecularScale × Stocking_MSK.G, 丝袜权重) )
```

| 你想做的 | 改什么 |
|---|---|
| 调非金属 F0 输入 | BaseSpecular（父默认 0.5），最终还受绒面和丝袜影响 |
| 某区域高光更强/更弱 | `ClothEffectsA_MSK.G`（白=强，黑=无） |
| 整体再乘一个倍率 | EffectSpecularScale（默认 1，UI 滑条 0~2，不是节点硬限制） |
| 边缘（掠射角）高光太强 | `VelvetSpecularSuppress`（默认 0.5，调大=压制更多） |
| 丝袜区域的高光 | `StockingSpecularScale`（默认 1.25）+ `Stocking_MSK.G` |

### 3.2 粗糙度（这套材质里链路最长的一个）

粗糙度会被**四道**依次加工，最终还被钳在 `MinRoughness`（0.08）~ `MaxRoughness`（0.95）之间：

1. **基础值**：`ORMTexture.G × RoughnessMultiplier + RoughnessBias`，钳 0~1
2. **区域倍率与织纹偏移**：基础值乘 Max(EffectsA.R×2×EffectRoughnessScale,0)，再加 **(ClothWeave_MSK.B×2−1)×WeaveRoughnessAmplitude×Saturate(EffectWeaveScale×EffectsB.R)**，然后钳到 MinRoughness／MaxRoughness。
3. **绒面响应**：Lerp(上一步,1,绒面权重×VelvetRoughnessResponse)。正常 0~1 系数下朝 1 混合，权重为 0 不变；系数越界可能外插，不保证永远只是抬升。
4. **丝袜偏移**：`+ 丝袜权重 × (StockingRoughnessBias（默认 −0.05）+ (Stocking_MSK.B × 2 − 1) × StockingMicroRoughnessAmplitude（默认 0.04）)`，最后再钳一次

排查要点：先查实例覆盖、ORM.G、EffectsA.R 和各倍率，再查 **MinRoughness≤MaxRoughness**（默认 0.08／0.95）。当前没有自动排序，触及上下限时继续改倍率可能看不出变化。

### 3.3 绒面 / 天鹅绒边缘光

绒面权重（控制边缘视角响应，**不等于必定泛白**）的完整公式：

```
绒面权重 = Saturate(
    Pow( 1 − Saturate(法线·相机向量), VelvetGrazingPower（默认 2.0） )
    × ClothEffectsB_MSK.G × EffectFiberScale（默认 1）
    × Lerp(1, ClothWeave_MSK.A, VelvetFiberBreakupStrength（默认 0.5）)
    × Lerp(1, Saturate(织纹后的粗糙度), VelvetRoughnessResponse（默认 0.35）)
)
```

它同时驱动三件事：颜色向 VelvetTint 偏（VelvetTintStrength 默认 0.2）、粗糙度向 1 混合（VelvetRoughnessResponse 默认 0.35）、高光受抑制（VelvetSpecularSuppress 默认 0.5）。

颜色顺序是 **底色 → Velvet Tint → Stocking 视角染色**。正常 0~1 混合系数下，Tint 比底色亮才提亮，较暗则压暗；白色不保证任何输入都更亮，尤其底色已白或参数越界时。丝袜再朝 DarkColor 混合，两路不是永远相反；强度越界还可能外插，不能只凭“白”“暗”判断最终明暗。

**FiberWeight 对外输出未接，不等于权重无效。** 当前 Master 的 VelvetAdvanced 调用未连接它的 FiberWeight 输出；但函数内部同一权重已驱动颜色、粗糙度和 Specular，这三路输出都接到下游。参数通过三路影响材质，不需要补线。

| 你想做的 | 改什么 |
|---|---|
| 边缘染色更明显 | 先确认 Tint 与底色有差异，再增大 VelvetTintStrength（父默认 0.2）；不保证提亮 |
| 改边缘染色的颜色 | VelvetTint（默认纯白），最后还受丝袜染色影响 |
| 视角响应范围更宽/更窄 | VelvetGrazingPower（默认 2.0，正常正值范围内越小越宽） |
| 只有某区域有绒面 | `ClothEffectsB_MSK.G`（黑=完全关闭）+ `EffectFiberScale`（总开关） |
| 让绒面边缘不均匀、更自然 | `ClothWeave_MSK.A` + `VelvetFiberBreakupStrength`（默认 0.5） |
| 粗糙的布绒感更强 / 更弱 | `VelvetRoughnessResponse`（默认 0.35） |

### 3.4 织纹

```
织纹强度  = WeaveNormalStrength（默认 0.35）× Saturate(EffectWeaveScale × ClothEffectsB_MSK.R)
织纹粗糙度 = (ClothWeave_MSK.B × 2 − 1) × WeaveRoughnessAmplitude（默认 0.05）× 同一个闸门
```

- 织纹的**法线强度**和**粗糙度幅度**共用同一个闸门 `ClothEffectsB_MSK.R`——一个开关管两处，画黑就全关。
- 纹理密度改 `WeaveUVScale`（默认 12）。
- 织纹法线画在 `ClothWeave_N`，粗糙度差异画在 `ClothWeave_MSK.B`。

### 3.5 缎面拉丝

```
AnisotropyFinal = Clamp(ClothEffectsA_MSK.B × EffectAnisotropyScale, −1, 1)   // 缎面阶段；TS 切线固定 +X
```

- 只有**两个**可控量：`ClothEffectsA_MSK.B`（分区）+ `EffectAnisotropyScale`（总倍率，默认 1）。`AnisotropyMask` 是常量 1，改不了。
- MI 无连续旋转切线接口，切线走向需模型组处理；同等绝对值改正负交换高光两轴粗糙度，不是切线反向（2.10）。
- 丝袜区域还经过 StockingAnisotropyScale 和 Stocking_MSK.A 的塑形及权重混合（2.9）。**上式只是缎面阶段，不是最终 BSDF 输入。**

### 3.6 丝袜

见 2.9 的完整公式。最常用的四个旋钮：

| 你想做的 | 改什么 |
|---|---|
| 丝袜整体开/关 | `Stocking_MSK.R`（画白）+ `ClothEffectsB_MSK.B`（分区） |
| 掠射目标色（不保证更暗） | StockingDarkColor（默认 (0.35,0.35,0.40)），与绒面后的底色比较 |
| 视角染色强弱 | StockingColorStrength（默认 0.35），还乘丝袜权重 |
| 染色出现的视角范围 | StockingViewMin / Max（默认 0.15 / 0.85）：保持 **Min<Max**，此前提下越接近过渡越硬 |

没有自动排序／保护，不把反置或相等当反转功能。正常顺序下正视保留绒面后的底色，掠射朝 DarkColor 混合；综合明暗由颜色和权重决定。

### 3.7 卡通图案 UV

`PatternUVs = Frac(（取小数）UV0 × PatternUVScale)`，`PatternUVScale` 默认 1。调大可以按 UV 网格重复图案。
它给 Profile 的**漫反射 Ramp 偏移图案、高光 Ramp 偏移图案、阴影排线**提供坐标，不是独立贴花槽。前两类改变光照曲线的取值偏移，排线参与阴影，不是直接覆盖衣服底色。

当前绑定 TP_TV_ToonCloth 的实际配置：

| 用途 | 资源 | Strength | Size |
|---|---|---|---|
| Diffuse Ramp Offset（漫反射曲线偏移） | 未绑定 | 0 | 1 |
| Specular Ramp Offset（高光曲线偏移） | 未绑定 | 0 | 1 |
| Shadow Hatching（阴影排线） | 未绑定 | 0 | 1 |

**当前只调 PatternUVScale 不会自动产生纹样。** 先由 TA 在批准的 Profile 中准备图案和启用强度，再调密度；具体外观还取决于图案、曲线和光照。共享 Profile 不为单个实例随意修改。

---

## 4. 常见问题排查

**Q1：丝袜全画好了，一点反应都没有。**
1. `Stocking_MSK` 是不是还挂着默认的**纯黑**图（R=0 → 权重 0，全链路关闭）；
2. R 通道是不是真的画了白（画到 G/B/A 上没用）；
3. `ClothEffectsB_MSK.B` 对应区域是不是白（这是丝袜的区域总开关）；
4. `EffectStockingScale` 是不是被调成 0；
5. 颜色没变但高光变了 → 查 `StockingColorStrength` 是不是 0。

**Q2：织纹画了看不到。**
1. `ClothEffectsB_MSK.R` 对应区域是不是黑（这是织纹总开关，画黑则法线和粗糙度双关）；
2. `WeaveNormalStrength` 是不是 0（默认 0.35，通常不是）；
3. `WeaveUVScale` 默认 12，检查是不是大到糊成一片噪声，或你的织纹贴图本身是不是中性法线。

**Q3：想让粗糙度变化更明显，改了贴图却几乎看不出。**
默认 WeaveRoughnessAmplitude=0.05，实际偏移还乘织纹闸门。先查闸门和 Clamp；全白 B 是恒定偏移，要有起伏需画变化。基础值查 ORM.G，区域倍率查 EffectsA.R，不把它们当同一条控制。

**Q4：把 `ClothEffectsA_MSK` 画成纯白，结果整件衣服变得又亮又怪。**
纯白 R 解码倍率为 2；**Scale=1 时理想单位倍率是 R=0.5**。8 位 #808080 的值为 128/255≈0.50196，解码约 1.00392，不是精确 1。织纹和丝袜 B 的理想零偏差同为 0.5，8 位 128 解码约 +0.00392，再乘各自幅度／权重；别把近似中灰写成无损中性。

**Q5：边缘泛白（绒面）该有的地方没有。**
查 Effects B 的 G 是否为黑、EffectFiberScale 或 VelvetTintStrength 是否为 0；再查 Tint 与底色有无差异、丝袜是否覆盖染色。权重非零不保证泛白，外部 FiberWeight 未接也不是关闭原因。

**Q6：丝袜区域完全没有高光，看起来发死。**
先查丝袜权重和 G；G=0 时输入乘 1−权重，不是任何覆盖度下都无高光。**G 已为 0 时，单独增大 StockingSpecularScale 仍乘到 0，不能恢复这条输入**，先给 G 非零值；再查 BaseSpecular、EffectsA.G、绒面抑制和粗糙度。

**Q7：细节法线画了没反应。**
先确认实例覆盖、贴图和 DetailNormalStrength 是否非零（默认 0），再查图案内容和 UV 密度；不把固定起调区间当视觉配方。

**Q8：改了法线之后，绒面边缘光和丝袜的明暗也跟着乱了。**
这是正常联动：绒面和丝袜都基于 `dot(法线, 相机向量)` 计算，法线越碎，视角项越碎。想让边缘光干净，主法线和织纹都要克制。

**Q9：想要金属、自发光、半透明，这套能做吗？**
不能。`Metallic` 硬编码 0；`EmissiveColor` 硬编码黑；材质是 `BLEND_Opaque` + `TwoSided = False`。

**Q10：拉丝方向不对，找不到方向贴图。**
没有方向贴图或连续旋转切线的 MI 参数。TS 切线固定 +X，世界方向仍依赖模型切线基底；与模型组检查 UV／切线。强度正负交换高光两轴，不是切线反向（2.10）。

**Q11：想在材质里给底色/法线平铺。**
除 DetailUVScale 和 WeaveUVScale 外无表面平铺参数。Wrap 只处理越界，不增加密度；主图密度在 DCC 的 UV 或图案内容上处理。
