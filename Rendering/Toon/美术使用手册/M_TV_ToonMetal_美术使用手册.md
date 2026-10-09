# M_TV_ToonMetal_Master 卡通金属材质 · 美术使用手册

> 适用对象：会画贴图、但不写着色器的材质/角色美术。
> 文中英文均为材质里的真实参数名或贴图名，第一次出现时给中文解释。
> 所有默认值都来自材质本身的连接与参数默认值（master material 的默认值，未被实例覆盖）。
> 链路说明按节点、实际资产与 UE 源码共同核对；画面效果仍由美术确认。文中的计算例子只解释参数作用，不是已经签核的效果配方。
> 资源目录：共用中性贴图位于 `/Game/Toon/MaterialLibrary/Support/Shared/`，金属 Profile 位于 `/Game/Toon/MaterialLibrary/Profiles/ToonMetal/`；母材质仍在 `Masters/` 根目录。

---

## 0. 开工前先看：几个默认是「关」的开关

这套材质里有一批参数默认值是 **0**，意味着贴图画对了也可能一点效果都没有。开工前先确认：

| 参数名 | 中文含义 | 默认值 | 默认是开是关 |
|---|---|---|---|
| `AnisotropyEnable` | 拉丝总开关 | **0** | 关（拉丝整体无效） |
| `AnisotropyScale` | 拉丝强度基数 | **0** | 关（与开关相乘） |
| `AnisotropyTangentMapStrength` | 用切线贴图代替顶点切线的程度 | **0** | 关（用模型顶点切线） |
| `AnisotropyShift` | 拉丝方向整体旋转量 | 0 | 不旋转 |
| `AnisotropyShiftNoiseScale` | 拉丝方向噪声扰动量 | 0 | 关 |
| `HoyoSphereBlendStrength` | 卡通金属球染色强度 | **0** | 关（金属球贴图不影响颜色） |
| `DetailNormalStrength` | 细节法线强度 | **0** | 关 |
| `SpecularMultiplier` | 非金属 F0 输入倍率（不是所有金属高光总开关） | **0.5** | 父默认；含义见 2.6 |
| `NormalStrength` | 主法线强度 | 1 | 开 |
| `MetallicMultiplier` / `RoughnessMultiplier` / `AOMultiplier` | 金属度 / 粗糙度 / AO 总强度 | 1 | 开 |
| `MetalRegionMask` | 金属区域总强度（标量） | 1 | 开 |
| `AnisotropyMask` | 拉丝区域总强度（标量） | 1 | 开 |
| `SpecularF0Mask` | 高光区域总强度（标量） | 1 | 开 |
| `PatternUVScale` | 图案 UV 缩放 | 1 | 原尺寸 |
| `MetalSphereBrightness` / `MetalSphereTileScaleX` | 金属球亮度 / 横向缩放 | 1 / 1 | 原值 |
| `BaseColorTint` | 底色染色（乘在底色上） | 白 (1,1,1) | 不改色 |
| `MetalDarkColor` / `MetalLightColor` | 金属球暗部色 / 亮部色 | 白 (1,1,1) | 不改色 |

**另外两个"默认状态"要心里有数：**

- `ORMTexture`（金属度/粗糙度/AO 三合一贴图）的默认图是 **纯白** `T_TV_NeutralWhite_MSK`，所以**默认状态是：金属度 1、粗糙度 1（最粗糙）、AO 1（无遮蔽）**，全表面都是金属。
- `BaseColorTexture` 的默认图是引擎白块 `WhiteSquareTexture`，`NormalTexture` 默认是中性法线 `T_TV_Neutral_N`（平的）。

---

## 1. 这套材质能做出什么效果

它是一套 **Substrate（UE 的新材质框架）+ Toon BSDF（卡通着色）** 的金属母材质，最终输出一个 `SubstrateToonBSDF` 节点。

能做的：

1. **卡通金属本体**：底色 + 金属度 + 粗糙度 + 法线，走卡通着色，适合机甲装甲板、机械零件、金属装饰件。
2. **拉丝 / 各向异性高光**（Anisotropy，各向异性）：让高光被拉成条状或环形，是金属最出效果的一块。可以做：
   - 沿模型顶点切线方向的整体拉丝（默认方式，靠模型 UV 的切线方向）；
   - 用贴图逐像素指定拉丝方向（靠 `AnisotropyTangentTexture`）；
   - 用贴图控制哪些地方有拉丝、拉丝多强（`AnisotropyMaskTexture` / `AnisotropyScaleTexture`）；
   - 用噪声打乱拉丝方向（`AnisotropyShiftNoiseTexture`），做出"手工打磨/不均一"的感觉。
3. **卡通金属球染色**（Hoyo Sphere，日式卡通里常见的高光球）：用一张球面图把金属分成"暗部色 / 亮部色"两档，并随视角移动，是这类二次元金属的标志性效果。
4. **图案 UV**（`PatternUVs`）：把 UV 做 `Frac`（取小数）后喂给卡通着色节点，为 Toon Profile 的明暗分档偏移图案和阴影排线提供重复坐标。它不是独立贴花槽，具体用法见 3.5。

**不适合 / 做不到的（拓扑里没有的东西，别在这里找）：**

- 透明、半透、镂空：材质是 `BLEND_Opaque`（不透明）、`MD_Surface`（表面域）、`TwoSided = False`（单面）。
- 自发光：`SubstrateToonBSDF` 的 `EmissiveColor`（自发光）输入**没有连线**，画了也无效。
- 独立边缘光 / 轮廓光：图里没有暴露 Fresnel 边缘染色或独立轮廓光控制。**没有 Fresnel 节点，不等于最终高光不随视角变化**；金属球是额外的视角染色路径，不是全部视角响应的来源。
- 材质内平铺（Tiling）：表面图用 UV0，没有全局平铺参数；金属球另用视空间法线坐标和 MetalSphereTileScaleX。**Wrap 只让越界坐标重复，不会增加 UV 密度**；表面图的密度要在 DCC 的 UV／图案内容上处理。

---

## 2. 贴图怎么制作

### 2.0 十一张贴图总览

| # | 贴图参数名 | 采样器类型 | 默认资源 | 实际用到的通道 | 用的 UV |
|---|---|---|---|---|---|
| 1 | `BaseColorTexture` | Color（颜色） | `WhiteSquareTexture` | RGB | UV0 |
| 2 | `ORMTexture` | Masks（遮罩） | `T_TV_NeutralWhite_MSK` | R=AO、G=粗糙度、B=金属度 | UV0 |
| 3 | `NormalTexture` | Normal（法线） | `T_TV_Neutral_N` | RGB | UV0 |
| 4 | `DetailNormalTexture` | Normal（法线） | `T_TV_Neutral_N` | RGB | UV0 |
| 5 | `MetalRegionMaskTexture` | Masks | `T_TV_NeutralWhite_MSK` | R | UV0 |
| 6 | `SpecularF0MaskTexture` | Masks | `T_TV_NeutralWhite_MSK` | R | UV0 |
| 7 | `MetalSphereTexture` | Masks | `T_TV_NeutralWhite_MSK` | R | **不用 UV0**，用视空间法线算出来的球面 UV |
| 8 | `AnisotropyMaskTexture` | Masks | `T_TV_NeutralWhite_MSK` | R | UV0 |
| 9 | `AnisotropyScaleTexture` | Masks | `T_TV_NeutralWhite_MSK` | R | UV0 |
| 10 | `AnisotropyShiftNoiseTexture` | Masks | `T_TV_NeutralBlack_MSK` | R | UV0 |
| 11 | `AnisotropyTangentTexture` | Masks | `T_TV_NeutralTangent_MSK` | RGB | UV0 |

> 注意：第 8、9 两张贴图在链路里**只是相乘**，拓扑没有给它们分工（详见 2.8 / 2.9）。

---

### 2.1 `BaseColorTexture`（底色贴图）

**画什么**

就是这块金属的底色。金属区域的底色还参与金属反射颜色，**并非必须偏暗、偏灰**。按角色色设画主体色、拼色和装饰，不把某个灰度值当成这套材质的规定。

- 银白装甲板：按角色色设画银白或灰色；纯白不会由算法必然变脏，实际观感还受光照、粗糙度与 Profile 影响。
- 深色机甲骨架：画设计需要的深色及冷暖倾向，不用固定色号代替角色色设。
- 刀刃：靠近刃口画一道更亮的窄条，刀身中段画稍暗的灰，做出"刃口更亮"的卡通读法。

**画完什么效果**

进入链路的第一步是：`BaseColorRaw（底色原始值）= BaseColorTexture.RGB × BaseColorTint（底色染色，向量参数，默认白）`。

- 就是**直接相乘**，没有反相、没有重映射。白色贴图 = 保留 `BaseColorTint` 的颜色，黑色 = 变黑。
- `BaseColorTint` 默认是白 `(1,1,1)`，所以默认情况下底色贴图是什么色就是什么色；想整体调色就改 Tint，不要改贴图。

**怎么配合其他贴图**

- 底色之后还会经过金属球染色的加权混合（2.7）。只有 Sphere 权重非零、渐变乘数不为 1 时才改变底色；不是默认就再乘一遍。
- 底色图不直接控制金属度；金属度由 ORM.B、MetallicMultiplier 和 Region 共同决定。底色仍参与金属区域的反射颜色。

**怎么导入**

- 采样器类型是 `SAMPLERTYPE_Color`（颜色）。当前默认 `WhiteSquareTexture` 实际为 **sRGB 开启、Compression Settings=Default（`TC_Default`）、双轴 Wrap、Mip 设置 Sharpen4**。替换成自己的底色图时，使用 sRGB 开启的颜色贴图并保留正常 mip；不要把这套颜色设置套到遮罩或方向图上。
- 只用 RGB，A 通道不使用。
- 用 UV0，材质内不平铺。

**怎么检查**

画了底色但颜色不对/发白 → 先看 `BaseColorTint` 是不是被实例改成了别的颜色（默认应纯白）；再看是否被 `MetalSphereTexture` 又乘了一遍（把 `HoyoSphereBlendStrength` 归零就能排除干扰）。

---

### 2.2 `ORMTexture`（三合一：AO / 粗糙度 / 金属度）★最核心的一张

通道分工由三个 `ComponentMask`（通道遮罩）节点**明确指定**，不是猜的：

- **R 通道 → AO（环境光遮蔽）**：`AOFinal = R × AOMultiplier`，直接接到材质根节点的 `AmbientOcclusion`（环境光遮蔽）。
- **G 通道 → Roughness（粗糙度）**：`RoughnessFinal = G × RoughnessMultiplier` → 接到 BSDF 的 `Roughness`。
- **B 通道 → Metallic（金属度）**：`MetallicRaw = B × MetallicMultiplier`。
- **A 通道：未使用**（拓扑里没有任何一路取 A）。

**画什么**

- **R（AO）**：画缝隙和凹陷。装甲板之间的接缝、铆钉周围、螺丝孔内侧、刀刃与刀镡的交界处 → 画黑（数值低）；大面积平面 → 留白（数值 1）。
- **G（粗糙度）**：画高光散不散。要更光滑的抛光或磨损露光泽区域 → G 画得比周围**更暗**；要更粗糙的磨砂或铸件 → 更亮。别把“画面更亮”理解成“粗糙度图更亮”。
- **B（金属度）**：金属装甲和刀身画白，漆面、橡胶、布料、塑料件画黑。分区按材质边界设计；灰度表示输入混合，不是算法禁止的值，也不能断言灰度一定显脏。抗锯齿、过滤和实际过渡按制作需求处理。

**画完什么效果（三个通道都没有反相，全是直接相乘）**

| 通道 | 白色（1） | 黑色（0） | 灰色（0.5） |
|---|---|---|---|
| R（AO） | AO 输入为 1，不额外遮蔽 | AO 输入为 0，遮蔽最强；不保证任何光照下全黑 | AO 输入为 0.5，不等于画面亮度减半 |
| G（粗糙度） | 父倍率为 1 时粗糙度输入为 1，高光更分散，不等于没有高光 | 输入为 0，高光更集中；引擎着色仍有安全下限 | 输入为 0.5 |
| B（金属度） | 其他倍率为 1 时为完全金属，反射颜色由底色参与决定；Sphere 另看权重 | 金属度归零，但 Region 非零时仍可能被 Sphere 染色 | 金属／非金属输入混合，不是必定显脏 |

特别注意：默认纯白 ORM 在父默认下给出金属度 1、粗糙度 1、AO 1。它不等于“默认没有高光”；是否显得哑光还要看灯光、底色和 Profile。

**怎么配合其他贴图**

- `ORMTexture.B` 决定"金属度的底色"，而 `MetalRegionMaskTexture`（金属区域遮罩）会**再乘一次**到金属度上。最终：`Metallic = ORM.B × MetallicMultiplier × MetalRegionMaskTexture.R × MetalRegionMask`。
  → 所以：**ORM 的 B 管"金属度的分布和浓淡"，`MetalRegionMaskTexture` 管"整体开关和区域裁剪"**。区域划分建议放在 `MetalRegionMaskTexture` 上（改起来快、一改改两处），ORM 的 B 用来做细节浓淡。
- 粗糙度**只受** ORM.G 和 `RoughnessMultiplier` 影响，其他贴图都管不到它。
- AO 不直接乘进 BaseColor 或 Specular 输入，而是接 AmbientOcclusion。渲染阶段会使用它处理间接光及相关遮蔽，**不能据此保证画面颜色／反射永远不受影响**。

**怎么导入**

- 采样器类型 `SAMPLERTYPE_Masks`（遮罩）。当前默认 `T_TV_NeutralWhite_MSK` 实际为 **sRGB 关闭、Compression Settings=Masks（`TC_Masks`）、双轴 Wrap**。你的 ORM 也要关闭 sRGB、使用 Masks；**Masks 是压缩预设，不等于固定 BC7**，实际 GPU 格式由平台构建决定，打包后仍要检查三通道有没有明显损失。
- 三个通道必须各画各的，导入前确认没有被打包工具重新排列通道。
- 用 UV0。

**怎么检查**

- 金属区域不对 → 先确认画在 **B 通道**（不是 R 或 G）；把 `MetallicMultiplier` 拉到 0 / 1 做 A/B 对比，能立刻看出金属度是不是生效。
- 完全没反应 → 用材质编辑器的预览或者把 `RoughnessMultiplier` 从 1 改到 0 试试，确认这张 ORM 到底有没有被实例上的贴图槽覆盖（母材质默认是一张纯白图）。
- AO 太重 → 调 `AOMultiplier`，不要重画贴图。

---

### 2.3 `NormalTexture`（主法线贴图）

**画什么**

大面积的结构起伏：装甲板的分块、倒角、铆钉、螺栓、散热槽、刀身上的锻打纹。凹凸幅度按常规法线贴图规格画（切线空间法线，材质里 `bTangentSpaceNormal = True`）。

**画完什么效果**

链路是：`BaseNormal（基础法线）= Lerp(平面法线 (0,0,1), NormalTexture.RGB, NormalStrength)`。

- 这是一个**从"完全平"到"完全用你的贴图"的插值**，不是乘法：
  - `NormalStrength = 1`（默认）→ 完整使用贴图；
  - `NormalStrength = 0` → 完全平，贴图无效；
  - `NormalStrength > 1` → 会**放大**凹凸（可以做出夸张的效果，但过大会出现噪点状高光）；
  - 平面法线常量在拓扑里确认是 `(0,0,1)`，所以贴图里的"中性值"必须用常规中性法线色（约 `#8080FF`）。

**怎么配合其他贴图**

- 主法线会和 `DetailNormalTexture` 用 **BlendAngleCorrectedNormals**（角度校正法线混合，引擎自带的混合函数）合成，合成结果再做 `Normalize`（归一化）后输出，所以两张法线的强度可以按常规思路叠加，不采用简单相加；细节是否明显仍取决于强度、法线内容、采样距离和光照。
- 这张法线的结果同时喂给三处：BSDF 的 `Normal`、金属球的视空间法线、拉丝的正交化基准。也就是说**改法线会连带改变金属球明暗和拉丝方向**，这是正常的联动。

**怎么导入**

- 采样器类型 `SAMPLERTYPE_Normal`（法线）。当前默认 `T_TV_Neutral_N` 实际为 **sRGB 关闭、Compression Settings=Normalmap（`TC_Normalmap`）、双轴 Wrap、Mip 设置 FromTextureGroup**。你的法线也使用关闭 sRGB 的 Normalmap 预设；不要因为想保留 RGB 就擅自改成普通 Masks 或颜色压缩，最终 GPU 压缩格式由目标平台决定。
- 默认是 `T_TV_Neutral_N`（中性法线），画之前先看一眼这张图的中性色基准，和它保持一致，避免整块模型出现统一偏色。
- 用 UV0。

**怎么检查**

看不到凹凸 → 先确认 `NormalStrength` 不为 0（默认 1）；再看模型有没有切线/UV（`bTangentSpaceNormal = True` 依赖切线空间）；最后确认法线贴图的绿色通道方向（OpenGL / DirectX）和项目规范一致，方向反了会看到"凹陷变凸起"。

---

### 2.4 `DetailNormalTexture`（细节法线贴图）

**画什么**

近距离才看得到的小纹理：金属的拉丝细纹、磨砂颗粒、铸造麻点、细微的划痕。是叠加在 2.3 之上的"第二层细节"。

**画完什么效果**

链路：`AdditionalNormal（附加法线）= Lerp(平面法线 (0,0,1), DetailNormalTexture.RGB, DetailNormalStrength)`。

- 和主法线**完全同样的插值算法**，但 **`DetailNormalStrength` 默认是 0**，也就是说这张贴图默认完全不起作用。
- 它作为"附加层"通过 BlendAngleCorrectedNormals 叠在主法线上，不是简单相加，不保证任何强度和距离下细节都清楚可见。

**怎么配合其他贴图**

- 主法线管大形，细节法线管细纹，两者共用一个 UV0（材质里没有给细节法线单独的平铺参数）。想让细节更密，只能在贴图里画得更密，或者改 UV。
- 细节法线**不影响**金属度/粗糙度，只影响明暗和拉丝方向。

**怎么导入**

同 2.3（法线贴图规范），默认同样是 `T_TV_Neutral_N`。

**怎么检查**

画了没反应 → 先确认实例覆盖已勾选、贴图已绑定，再查 DetailNormalStrength 是否非零，最后检查图案和采样距离；默认 0 是关闭值，不把它写成有统计支持的诊断概率。

---

### 2.5 `MetalRegionMaskTexture`（金属区域遮罩）★一图管两处

**画什么**

"这块模型上，哪些地方是金属"的分区图。举例：

- 一整套机甲：外装甲板画白，内衬的黑色橡胶/布料画黑，涂装色带画黑。
- 一把刀：刀身画白，缠绳的刀柄、皮革刀鞘画黑；刀镡上的宝石装饰画黑。
- 装甲边缘的磨损露金属：在漆面（黑区）的边缘往外扩一点白色，做出"漆被磨掉露出金属"的读法。

**画完什么效果**

`MetalMask（金属遮罩值）= MetalRegionMaskTexture.R × MetalRegionMask（标量，默认 1）`，然后这个值**被用到两个地方**：

1. **乘进金属度**：`Metallic = ORM.B × MetallicMultiplier × MetalMask`
2. **乘进金属球染色权重**：`金属球权重 = HoyoSphereBlendStrength × MetalMask`

所以：

- **白色（1）** → 保留 Region 权重；实际金属度还乘 ORM.B 和 MetallicMultiplier，Sphere 是否染色还看 BlendStrength 与两档颜色。
- **黑色（0）** → 这里不是金属：金属度归零（走非金属），且**完全不受金属球影响**（非金属区域不会被卡通高光球染色）。
- **灰色** → 按数值减弱两路输入，例如 R=0.5、MetalRegionMask=1 时 Region 权重为 0.5；不是保证画面亮度减半。

这是一个**一图管两处**的设计：画一次，金属度和卡通染色同时生效。

**怎么配合其他贴图**

- 它和 `ORMTexture.B` 是**相乘**关系（重复控制）；两者任一为 0，那个像素的金属度就是 0。
- 分工建议：`MetalRegionMaskTexture` 画"区域"（大块黑白，改起来快），`ORMTexture.B` 画"浓淡细节"（细微的强弱变化）。
- 它不直接乘进 Specular 或各向异性输入，但改变 Metallic 会改变最终反射颜色与能量分配；不要理解成画面高光完全不受 Region 影响。

**怎么导入**

采样器类型 `SAMPLERTYPE_Masks`，默认纯白 `T_TV_NeutralWhite_MSK`，只用 R 通道，用 UV0。当前默认图实际 **sRGB 关闭、Compression Settings=Masks、双轴 Wrap**；替换图也按线性 Masks 导入。白色表示保留金属区域权重，实际金属度还要乘 ORM.B 和两个标量（见上方链路）。

**怎么检查**

- 金属区域边界对不上 → 确认画在 R 通道；用 `MetalRegionMask` 标量从 1 拉到 0，整块变非金属，能确认链路通不通。
- 只想改金属度、不想改 Sphere 权重 → 保留 Region，改 ORM.B 或 MetallicMultiplier。只改 Region 会影响两路，但不需要为这件事重写材质。

---

### 2.6 `SpecularF0MaskTexture`（非金属 F0 输入遮罩，不是所有高光总开关）

**画什么**

这张图改变送进 BSDF 的 Specular 输入，主要用于非金属／部分金属区域的 F0（正视反射率）变化，不负责把所有金属高光统一压掉。先确认目标位置的金属度，再画局部强弱。

- 非金属漆层、塑料或介电涂层：R 白保留 Specular 输入，灰黑减弱；要让高光更锐，还要把 ORM.G 降低。
- 磨砂的分散感先由粗糙度控制，别只靠这张图画黑代替粗糙度。
- 完全金属区域的反射颜色由 BaseColor 主导；这张图不是改变金属反射强弱的通用手段。

**画完什么效果**

链路：`Specular = SpecularMultiplier（默认 0.5）× (SpecularF0MaskTexture.R × SpecularF0Mask（标量，默认 1)）`

- **纯乘法，没有反相**：白色保留 Specular 输入，灰黑减弱，黑色使这个输入为 0。**这不等于最终所有高光为 0**：F0 在非金属端取 0.08×Specular，在金属端取底色，中间按 Metallic 混合。
- 画白是保留输入，不是让高光必定更锐；锐利程度主要看 Roughness。完全金属处这张图可能没有预期的“总亮度”作用。
- 父默认 SpecularMultiplier=0.5、两项 F0 遮罩=1 时，Specular 输入为 0.5；Metallic=0 时对应 F0=0.04，不是“画面高光亮度 0.5”。

**怎么配合其他贴图**

- 它和粗糙度是不同输入：粗糙度控制高光分散，Specular 参与非金属 F0。最终表现还受 Metallic、光照与 Profile 影响。
- **不能说和金属度无关**：金属度决定 Specular 与底色怎样共同形成 F0。金属度=1 时，Specular=0 仍保留由底色形成的金属 F0。

**怎么导入**

采样器类型 `SAMPLERTYPE_Masks`，默认纯白，只用 R 通道，用 UV0。当前默认 `T_TV_NeutralWhite_MSK` 实际 **sRGB 关闭、Compression Settings=Masks、双轴 Wrap**；自己的图也用线性 Masks，实际 GPU 格式不是固定 BC7。

**怎么检查**

画了没变化 → 先确认实例覆盖、贴图绑定和 R 通道，再查 SpecularMultiplier／SpecularF0Mask；最后检查目标区域 Metallic。完全金属处别用这条输入验证“关闭所有高光”。

---

### 2.7 `MetalSphereTexture`（卡通金属球贴图）★视角相关的关键张

这张是整套材质里唯一**不用 UV0** 的贴图——它的 UV 是实时算出来的：

```
视空间法线 = 把最终法线从切线空间变换到视空间，取 XY
球面 UV    = (视空间法线.XY × (MetalSphereTileScaleX, 1)) × 0.5 + 0.5
```

也就是经典的"球面环境贴图"采样方式：**贴图坐标跟着视角走**，转动相机时明暗会在模型表面上流动，这就是二次元金属那种"高光球"的观感。

**画什么**

标准的球面渐变图：

- 中心（对应正对相机的面）画一个亮区，向四周渐变到暗（这是最常见的日式金属球），或者反过来，取决于你想要"正面亮"还是"边缘亮"。
- 想做多段色阶（卡通的分档感）：用**硬边**的同心圆环/色块，而不是平滑渐变——值会被直接当成插值权重，硬边会得到干净的色块分界。
- 想做条带状高光（比如刀身上那道长条光）：在球面图上画一条带。

**画完什么效果**

完整链路（全部来自拓扑，无猜测）：

```
SphereFactor（球面因子） = MetalSphereTexture.R × MetalSphereBrightness（默认 1）
金属渐变色               = Lerp(MetalDarkColor（暗部色）, MetalLightColor（亮部色）, SphereFactor)
带球的底色               = BaseColorRaw × 金属渐变色
最终底色                 = Lerp(BaseColorRaw, 带球的底色, HoyoSphereBlendStrength × MetalMask)
```

- **`MetalSphereBrightness = 1` 时**：贴图黑色（0）→ 用 `MetalDarkColor`；白色（1）→ 用 `MetalLightColor`；中间灰 → 两色之间插值。**没有反相。**Brightness 改了，白色对应的结果也会改，不再保证正好是亮部色。
- 这个渐变色是**乘在底色上的**（`带球的底色 = 底色 × 渐变色`），所以 `MetalDarkColor` / `MetalLightColor` 要按"乘数"来给：想让暗部压暗就给小于 1 的灰，想让亮部提亮就给大于 1 的值。**默认两色都是纯白 1，渐变乘数恒为 1；不管球面图和 Brightness 怎么变，这组默认色都不会改底色。**两色相同时，球面图也不能制造两色之间的明暗分布。
- HoyoSphereBlendStrength 默认 0 → Sphere 的混合权重为 0，输出保留原底色。**这是数值上的关闭，不代表编译后保证不采样或省掉整条计算**。没反应先查权重。
- Sphere 权重只乘 Region，不乘 ORM.B 或 MetallicMultiplier。**Region 为黑才关闭这条染色**；如果用 ORM.B 或 MetallicMultiplier 把金属度降到 0、Region 仍非零，Sphere 仍能染色。
- `MetalSphereBrightness` 是乘在球面因子上的，**不是直接乘亮度**。因子在 0~1 时，是两色之间插值；超过 1 就会越过亮部色，低于 0 就会越过暗部色，链路里**没有 clamp（钳制）**。例如某一通道暗部色 0.2、亮部色 0.8，贴图白色且 Brightness=1.5，算出的乘数是 1.1，已经越过 0.8。往哪个颜色方向过冲取决于两色，不保证数值越大画面越亮；最终观感还受底色和光照影响。
- `MetalSphereTileScaleX` 只缩放 **X 方向**（Y 固定为 1），可以把球面渐变横向压扁/重复；Y 方向在拓扑里是常量 1，改不了。

**怎么配合其他贴图**

- 它**乘在底色上**：底色画得越暗，金属球的效果越不明显。底色建议画中等明度的灰，把明暗交给这张球面图。
- 它**依赖法线**：法线贴图会改变视空间法线，从而改变球面 UV。所以法线画得越碎，金属球的明暗流动越碎——想让高光球干净，法线就要克制。
- Sphere 不直接修改 Anisotropy 或 Tangent，但它改 BaseColor，可能改变金属反射颜色；两路也共用最终法线，不能保证画面完全互不影响。

**怎么导入**

采样器类型 `SAMPLERTYPE_Masks`，**只用 R 通道**。当前默认纯白 `T_TV_NeutralWhite_MSK` 实际 **sRGB 关闭、Compression Settings=Masks、X/Y 都是 Wrap**。白色在 Brightness=1 时取 `MetalLightColor`；其他 Brightness 按前面的插值关系计算。
不需要按模型 UV 画（它用的是算出来的球面 UV）。当前节点的采样器**跟随贴图资产**：`MetalSphereTileScaleX > 1` 可能让 X 坐标越出 0~1，Wrap 会重复图案；换成 Clamp 则延伸贴图边缘，不重复。默认图是纯白，重复和延伸看起来一样，不能用它判断球面图有没有平铺。换成自己的球面图后，要在那张图里明确设置地址模式，而不是以为材质参数会自动选 Wrap。

**怎么检查**（按顺序）

1. `HoyoSphereBlendStrength` 是否为 0 → 设为 1。
2. `MetalRegionMaskTexture` / `MetalRegionMask` 在你观察的位置是否为白（黑色区域完全不受影响）。
3. `MetalDarkColor` / `MetalLightColor` 是否都是纯白 → 都是白时，球面图连明暗也改不了；先确认两色确实不同，再检查球面图的分布。
4. 转相机看采样是否变化；不变化时先看两色是否相同、图是否常量、权重是否非零及观察表面朝向。中性法线或 NormalStrength=0 只取消贴图扰动，模型几何法线仍在，**不能据此认定球面 UV 不随相机变化**。

---

### 2.8 `AnisotropyMaskTexture`（拉丝区域遮罩）

**画什么**

"哪些地方有拉丝"。举例：

- 刀身的长条拉丝 → 刀身画白，刀柄皮革画黑。
- 圆形金属件（齿轮、轮毂）的同心旋切纹 → 画白，配合切线贴图让方向绕圈。
- 装甲板上：抛光面板画白，磨砂和喷漆区画黑。

**画完什么效果**

`AnisoMask（拉丝遮罩）= AnisotropyMaskTexture.R × AnisotropyMask（标量，默认 1）`

最终拉丝强度：`Anisotropy = AnisotropyEnable × AnisotropyScale × AnisotropyScaleTexture.R × AnisoMask`

- **纯乘法，无反相**：白色 = 拉丝最强，黑色 = 完全没有拉丝（各向异性归零，退化成普通高光），灰色 = 按比例减弱。
- 注意别把这张图画成"拉丝方向图"——方向由切线（`AnisotropyTangentTexture` 或模型顶点切线）决定，这张只管强度和区域。
- **最终各向异性为负，不是把方向向量倒过来。**UE 的这条高光计算会把切线和副切线两轴的粗糙程度交换：同样绝对值的 +0.5 / −0.5，是换伸展轴，不是把箭头翻成反方向。具体高光能否明显看出换轴还取决于灯光、粗糙度和 Profile；不要把它当成所有光照路径下固定“转了90°”的效果保证。强度为 0 是取消各向异性，不是旋转方向。

**怎么配合其他贴图**

- 拓扑上它和 `AnisotropyScaleTexture`（2.9）在链路里**只是相乘关系**，节点连接图没有给它们不同的职责。两者都是"强度"控制，功能上是重复的。
- 分工建议（这是使用建议，不是拓扑规定）：`AnisotropyMaskTexture` 当"区域开关"（硬边黑白），`AnisotropyScaleTexture` 当"强度细节"（灰度微调）。

**怎么导入**

采样器类型 `SAMPLERTYPE_Masks`，默认纯白，只用 R 通道，用 UV0。当前默认 `T_TV_NeutralWhite_MSK` 实际 **sRGB 关闭、Compression Settings=Masks、双轴 Wrap**；自己的图也使用线性 Masks。

**怎么检查**

没反应 → 依次确认 `AnisotropyEnable` 不为 0、`AnisotropyScale` 不为 0、这张图的 R 通道不为黑、`AnisotropyMask` 标量不为 0，并检查 2.9 的强度图。**乘法链任意一项为 0，拉丝就是 0。**Scale 为负仍可能产生各向异性，不要把负值当关闭。

---

### 2.9 `AnisotropyScaleTexture`（拉丝强度贴图）

**画什么**

在已经有拉丝的区域内部，做强度的细微变化：比如同一块装甲上，中间拉丝明显、靠近边缘被磨损得拉丝变弱；或者做出"手工打磨不均匀"的斑驳感。用低对比度灰度画。

**画完什么效果**

`拉丝强度 = AnisotropyScale（标量，默认 0）× AnisotropyScaleTexture.R`，再和 `AnisotropyEnable`、拉丝遮罩相乘。

- 白色（1）= 完整强度；黑色 = 无拉丝；灰色 = 按比例。
- 这张同样是**纯乘法无反相**。贴图仍画 0~1；如果标量让最终强度变成负值，含义是 2.8 说明的两轴粗糙度交换，不是遮罩反相，也不是切线向量反向。

**关于"和 2.8 是不是重复"**

**如实说明：是的，拓扑上重复。** 两条链路的终点是同一个乘法链，`AnisotropyMaskTexture.R × AnisotropyMask` 和 `AnisotropyScaleTexture.R × AnisotropyScale` 之间没有任何其他运算（没有最大值、没有覆盖、没有遮罩优先级），只是相乘。节点连接图里**没有**标注它们各自的设计意图，所以上面"一个管区域、一个管强度"只是使用建议，不是材质本身的规定。你完全可以用其中任意一张单独完成全部工作，把另一张留成纯白。

**怎么导入 / 怎么检查**

同 2.8。默认纯白 `T_TV_NeutralWhite_MSK`。

---

### 2.10 `AnisotropyShiftNoiseTexture`（拉丝方向扰动噪声）

**画什么**

一张灰度噪声图，用来打乱拉丝方向。细密噪声 = 细碎的打磨感；大的柔和斑块 = 大面积的方向不均。不要画成高频纯白噪声（会让高光闪烁），建议柔和的中低频噪声。

**画完什么效果**

链路：`ShiftFinal（最终旋转量）= AnisotropyShift（标量，默认 0）+ (AnisotropyShiftNoiseTexture.R × AnisotropyShiftNoiseScale（默认 0)）`

然后拉丝方向（切线）会**绕着法线旋转** `ShiftFinal` 这么多（`RotateAboutAxis` 节点，旋转轴 = 归一化后的世界空间法线）。

- **纯加法，无反相**：贴图白色（1）的贡献 = `AnisotropyShiftNoiseScale`；黑色（0）= 不产生扰动。
- 默认图是**纯黑** `T_TV_NeutralBlack_MSK`，且 `AnisotropyShiftNoiseScale` 默认 0 → **双重默认关闭**，画了没效果优先查这个标量。
- 噪声值是 0~1，**倍率为正时贡献正向旋转，倍率为负时贡献反向旋转**。默认这一路没有自动以 0.5 居中；想让黑白两端向两侧偏，可设 `AnisotropyShift = −AnisotropyShiftNoiseScale / 2`。例如 NoiseScale=0.2、Shift=−0.1：黑色为−0.1圈，中灰为0圈，白色为+0.1圈；这是计算关系，具体噪声图的手感仍由美术看效果决定。
- **旋转量按“圈”输入，不是按弧度输入。**当前节点 `Period=1`，UE 编译时才把输入乘 `2π` 送给内部旋转函数：0.25=90°、0.5=180°、1=360°。负值反向旋转；这里描述的是方向向量的旋转量，不保证每次转动在最终高光上都同样明显。

**怎么配合其他贴图**

它只改**方向**，不改强度；强度由 2.8 / 2.9 管。它和 `AnisotropyTangentTexture` 可以叠加使用（先定方向，再整体/按噪声旋转）。

**怎么导入**

采样器类型 `SAMPLERTYPE_Masks`，默认**纯黑** `T_TV_NeutralBlack_MSK`，只用 R 通道，用 UV0。实际 **sRGB 关闭、Compression Settings=Masks、双轴 Wrap**；自己的噪声图也按线性 Masks 导入，不做颜色 gamma 转换。

**怎么检查**

没扰动 → 查 `AnisotropyShiftNoiseScale` 是否非 0，以及拉丝本身是否已经开启（`AnisotropyEnable`、`AnisotropyScale`）。

---

### 2.11 `AnisotropyTangentTexture`（拉丝方向贴图）

**画什么**

逐像素指定拉丝的**方向**，而不是强度。方向以切线空间向量的形式存进 RGB：

- 贴图值先做 `×2 − 1` 的重映射（拓扑里确认：`RGB × 常量 2.0 − (1,1,1)`），也就是把 0~1 的贴图值映射成 −1~1 的向量。
- 然后变换到世界空间，再**减去法线方向上的分量**（对法线做正交化），最后归一化。

**只有贴着最终法线切平面的方向有效**。作图常用 R/G 表达平面内走向，B 常设约 0.5；但投影针对扰动后的最终法线，并非固定删掉 TS 的 Z。**B 参与完整 RGB 解码和空间转换，不能一概说没用。**

举例：

- 沿刀身长度方向的直拉丝 → 全图统一填一个方向色（比如解码后是 +U，对应 R=1、G=0.5）。
- 圆形轮毂的同心旋切纹 → 让方向色随 UV 绕圈变化（R/G 按角度做 sin/cos）。
- S 形、波浪形的装饰拉丝 → 按形状画方向向量图。

**画完什么效果**

最终切线方向在两条来源之间**插值**：

```
最终方向 = Lerp(来自模型顶点切线的方向, 来自这张贴图的方向, AnisotropyTangentMapStrength（默认 0）)
```

- `AnisotropyTangentMapStrength = 0`（默认）→ **完全使用模型顶点切线**（`VertexTangentWS`），这张贴图不起作用。拉丝方向由**模型的 UV 切线方向**决定，画贴图改不了。
- `= 1` → 完全使用这张贴图的方向。
- 中间值 → 两者混合（方向会互相拉扯，通常不要停在中间）。
- 顶点切线那条支路里做了正交化 + 安全的归一化（长度太小会自动退化到另一个构造方式），所以即使模型切线有问题也不会崩，只是方向不可控。

**怎么配合其他贴图**

- 强度由遮罩乘法链控制；方向先在模型切线与贴图方向之间混合、投影归一化，再由 Shift 旋转。法线是共同基准，方向混合抵消时会回退，不能概括成三路完全互不影响。
- 改法线会改变正交化基准，从而改变最终方向——拉丝方向不对时，先确认法线是不是平的。

**怎么导入**

采样器类型 `SAMPLERTYPE_Masks`，**sRGB 关闭、Compression Settings=Masks、双轴 Wrap**（当前默认资产实际设置）。**虽然是 Masks 采样器，它存的是 TS（切线空间）方向向量，不要按“黑关闭、白开启”的遮罩思路画。**用 UV0；自己的方向图也必须保持线性编码。
默认 T_TV_NeutralTangent_MSK 的实际源图导出 RGB 为 **(255,128,128)**，尺寸 1×1。导出 TGA 为不透明 24 位 RGB，RGBA 表示中的 Alpha=255 是补值，未独立读回；算法不读取它。理想 +U 的 (1,0,0) 编码为 (1,0.5,0.5)，8 位 128/255≈0.50196，因此解码约 (1,0.00392,0.00392)，后续还投影和归一化，平台压缩也可能引入误差，不是逐位精确 +U。
默认 `AnisotropyTangentMapStrength=0`，这张方向图不接管输出，仍使用模型切线；“默认图接近 +U”不等于“默认输出强制沿 +U”。

**怎么检查**

- 贴图方向改了但画面不变 → 查 `AnisotropyTangentMapStrength` 是否 > 0（默认 0）。
- 方向对了但整体歪了 → 用 `AnisotropyShift` 做整体旋转微调。
- 默认顶点切线方向不对 → 要么在 DCC 里调整 UV/切线后重新导入，要么把 `AnisotropyTangentMapStrength` 拉到 1 用贴图完全接管。

---

### 2.12 导入规范汇总（节点 + 当前默认资产核对）

| 项 | 已核对的内容 |
|---|---|
| UV 通道 | 除 `MetalSphereTexture` 外，全部使用 **UV0**（`ConstCoordinate = 0`、`CoordinateIndex = 0`）；`MetalSphereTexture` 使用实时计算的球面 UV |
| 平铺 | 材质内 `UTiling / VTiling = 1`，**没有平铺参数**；`MetalSphereTexture` 只有 `MetalSphereTileScaleX`（仅 X 方向）可控 |
| 采样器来源 | 全部 `SamplerSource = SSM_FromTextureAsset`（跟随贴图资产自身的采样设置） |
| Mip | 节点均 `AutomaticViewMipBias = True`、`MipValueMode = TMVM_None`，但这**不等于资产都有 mip**：默认黑／白遮罩为 NoMipmaps，法线和方向图为 FromTextureGroup，引擎白块为 Sharpen4。美术新做的有细节贴图保留正常 mip，不照搬常量占位图的 NoMipmaps |
| 采样器类型 | `BaseColorTexture` = Color；`NormalTexture` / `DetailNormalTexture` = Normal；其余 8 张 = Masks |
| 颜色空间与压缩 | 已读实际默认资产：BaseColor 的引擎白块为 sRGB=true / TC_Default；主／细节法线为 false / TC_Normalmap；其余8个槽的默认控制／方向图为 false / TC_Masks。Default、Normalmap、Masks 是**压缩预设**，不保证底层固定采用 BC7 或其他单一格式 |
| 地址模式 | 当前默认纹理全部 X/Y=Wrap；节点跟随资产。换图后按新图实际设置，Wrap 重复、Clamp 延伸边缘；这不会替你缩放模型 UV |
| 未使用通道 | ORM 用 R/G/B，A 未用；Tangent 图用 RGB，A 未用；其余六张 Masks 图只用 R |

---

## 3. 效果怎么调整

> 按"你想改什么画面"来查，不按参数名罗列。所有数值都建议在**材质实例（MI）**上覆盖，不要改母材质。

### 3.1 高光太弱 / 太强 / 形状不对

| 你想做的 | 改什么 | 说明 |
|---|---|---|
| 调非金属／部分金属的 F0 输入 | SpecularMultiplier（默认 0.5） | 不是完全金属高光总开关；先看 Metallic |
| 改非金属区域 F0 输入 | SpecularF0MaskTexture.R | 白保留、灰黑减弱；不负责锐度 |
| 让高光更锐（聚成一个点） | `ORMTexture` 的 **G 通道画暗** | 粗糙度越低越锐 |
| 让高光更散（抹开） | `ORMTexture` 的 **G 通道画亮** | 粗糙度越高越散 |
| 快速整体调粗糙度 | `RoughnessMultiplier`（默认 1） | 不想改贴图时用 |
| 高光被拉成条状（拉丝） | 见 3.4 | 拉丝会完全改变高光形状 |

### 3.2 粗糙度

只有一条链路：`Roughness = ORMTexture.G × RoughnessMultiplier`。
**任何其他贴图都影响不到粗糙度**——如果发现粗糙度"改不动"，检查是不是在改错通道（R 是 AO，B 是金属度）。

### 3.3 金属质感（金属度 + 卡通金属球 + 金属区域）

三层叠加，按这个顺序排查：

1. **决定哪里是金属**：`MetalRegionMaskTexture`（区域，一图管两处）+ `ORMTexture.B`（浓淡）。两者相乘，任一为 0 就不是金属。总开关：`MetalRegionMask`（标量，默认 1）、`MetallicMultiplier`（默认 1）。
2. **决定底色与金属反射颜色**：先取 BaseColorTexture×BaseColorTint，再按 Sphere 权重混合染色；不是默认无条件乘完整球色。
3. **决定卡通高光球的明暗流动**：
   - 先把 `HoyoSphereBlendStrength` 从 0 调到 1（**默认关闭**）；
   - 用 `MetalSphereTexture` 控制明暗分布（Brightness=1 时黑=暗部色，白=亮部色）；
   - 用 `MetalDarkColor` / `MetalLightColor` 控制两档颜色（它们是**乘数**，默认纯白）；
   - 用 `MetalSphereBrightness` 改两色间的插值因子（因子>1会越过亮部色，不保证整体更亮，见2.7的过冲例子）；
   - 用 `MetalSphereTileScaleX` 横向压缩/重复球面渐变。

### 3.4 拉丝 / 各向异性（开启步骤，缺一不可）

拉丝是四个数相乘，任何一个为 0 就没有拉丝：

```
Anisotropy = AnisotropyEnable × AnisotropyScale × AnisotropyScaleTexture.R × (AnisotropyMaskTexture.R × AnisotropyMask)
```

**开启流程（照着做）：**

1. `AnisotropyEnable` = 1（默认 0，总开关）
2. 确认 AnisotropyScale 非零（父默认 0），由美术按目标效果决定强度；本文不给未经签核的起调区间。
   - 如果使用负值，先看 2.8：它交换高光计算的两轴粗糙度，不是把拉丝方向向量反过来；最终画面由美术确认。
3. 确认 `AnisotropyMaskTexture.R` 是白的（默认纯白，OK）、`AnisotropyMask` = 1
4. 看方向对不对：默认用**模型顶点切线**。方向不对 → 在 DCC 里调 UV/切线重新导入，或把 `AnisotropyTangentMapStrength` 拉到 1 并用 `AnisotropyTangentTexture` 接管方向
5. 想整体转个角度 → `AnisotropyShift`（按圈输入，0.25=90°；负值反向）
6. 想做不均匀的手工感 → `AnisotropyShiftNoiseTexture` + `AnisotropyShiftNoiseScale`（默认 0，需调）
7. 想只在某些区域有拉丝 → 把 `AnisotropyMaskTexture` 对应位置画黑

### 3.5 视角相关 / 边缘类的效果

**先说清楚：这套材质里没有菲涅尔节点，也没有边缘光节点。**

**金属球是额外的视角染色路径，不是全部视角响应的来源**。它用视空间法线采图，相机改变可能使染色分布变化；BSDF 高光本身也依赖视线。要设计球面染色，外圈画亮或画暗由目标决定，不承诺任何模型和视角都出现固定轮廓亮边。

另外 `PatternUVs`（图案 UV）走的是 `Frac(UV0 × PatternUVScale)`，默认 Scale=1；调大是在同一份模型 UV 上增加坐标重复。**它是“用什么坐标采图”，不是“自动长出一张纹样”，也不是独立贴花槽。**UE Toon BSDF 把这份坐标交给 Profile 的三条图案路径：

- **漫反射 Ramp 偏移图**：改变明暗分档的位置。当前 `TP_TV_ToonMetal` 的 `DiffuseRampOffsetTexture` 未绑定，`DiffuseRampOffsetStrength=0`，这一路关闭。
- **高光 Ramp 偏移图**：改变高光分档的位置。当前 `SpecularRampOffsetTexture` 未绑定，`SpecularRampOffsetStrength=0`，这一路关闭。
- **阴影排线图**：在阴影里按明暗程度使用图案通道。当前 `ShadowHatchingPatternTexture` 未绑定，虽然 `ShadowHatchingPatternStrength=1`，也没有提供一张排线图来产生空间纹样。

所以**当前默认 Profile 下，只改 `PatternUVScale` 不会凭空生成网点或排线**。需要这类效果时，交 TA 在获批准的 Profile 里配置图案资源及相应强度；Ramp 偏移路径需要强度>0，阴影排线路径还受阴影分布控制。美术在实例里调 Scale 只管重复密度，Profile 自身的图案尺寸也会参与缩放。最终像网点、排线还是其他形状，由绑定的图、Profile 曲线和光照共同决定，本文不宣称已做画面签核。

### 3.6 凹凸

- 大形 → NormalTexture，强度 NormalStrength（默认 1）；0 只取消主贴图扰动，模型几何法线和细节层仍在。大于 1 是插值外推，不保证每个方向都只是等比放大。
- 细纹 → `DetailNormalTexture`，强度 `DetailNormalStrength`（**默认 0，必须手动开**）。
- 两张用角度校正法线混合，再归一化；不是简单相加，也不保证所有强度／距离下细节都同样明显。

---

## 4. 常见问题排查

**Q1：拉丝贴图全画好了，一点反应都没有。**
按这个顺序查（每一步都是"为 0 就整条链断掉"）：
1. `AnisotropyEnable` 是不是 0（默认就是 0）→ 设 1；
2. `AnisotropyScale` 是不是 0（默认也是 0）→ 设非零；
3. `AnisotropyMask`（标量）和 `AnisotropyMaskTexture.R` 是否为 0；
4. 确认 AnisotropyScaleTexture.R 非零，再检查粗糙度、灯光和 Profile；Region 黑区不直接关闭各向异性，非金属也能使用它。
5. 模型有没有正确的顶点切线和 UV（默认方向来自 `VertexTangentWS`）。

**Q2：金属球球面图画好了，颜色完全不变。**
1. `HoyoSphereBlendStrength` 默认 0 → 设 1；
2. 该处 `MetalRegionMaskTexture` 是否为黑（黑区权重为 0，不受影响）；
3. `MetalDarkColor` / `MetalLightColor` 是否都是纯白（都是白时，球面图连明暗也不会改变）；
4. 底色是不是画得太暗（金属球是**乘**在底色上的）。

**Q3：整块模型看起来又灰又哑、没有金属感。**
这是默认状态（ORM 默认纯白 = 金属度 1、粗糙度 1）。改：把 `ORMTexture` 的 G 通道压暗（变光滑），B 通道确认是白，再按 3.3 打开金属球。

**Q4：某些区域发黑、发死，完全没有高光。**
先确认目标 Metallic，再查 F0 遮罩及 SpecularMultiplier。黑色使 Specular 输入归零，不保证完全金属高光消失；若是金属反射发黑，还要检查底色、Sphere 染色、光照及 Profile。

**Q5：缝隙没有深度感。**
`ORMTexture` 的 **R 通道**才是 AO，黑色 = 有遮蔽。确认没画到 G 或 B 上去；整体强度用 `AOMultiplier` 调，不要重画贴图。

**Q6：细节法线画了没反应。**
先确认实例覆盖和贴图，再查 DetailNormalStrength 是否非零（父默认 0）；本文不把某个起调区间当成效果配方。

**Q7：法线看起来反了（该凸的地方凹了）。**
法线贴图是切线空间（`bTangentSpaceNormal = True`），检查绿通道方向（OpenGL / DirectX）是否与项目一致；也可以用 `NormalStrength` 拉到 0 和 1 之间做对比确认。

**Q8：改了法线之后，金属球的明暗和拉丝方向也跟着变了。**
这是正常的联动：金属球的 UV 和拉丝的正交化基准都用同一个最终法线。想让高光球干净，就把法线做得克制一点。

**Q9：想要自发光 / 边缘光 / 半透明，这套材质能做吗？**
没有公开自发光、独立边缘光或透明接口：EmissiveColor 未连线，材质为 Opaque／单面。没有 Fresnel 节点不代表现有高光不随视角变化；需要新增独立功能时交 TA。

**Q10：贴图想在材质里平铺，找不到参数。**
材质没有表面图的全局平铺参数。Wrap 只处理越界，不改变密度；表面密度在 UV 或图案内容上处理。Sphere 的 MetalSphereTileScaleX 是另一套坐标的例外，只管 X。

---
