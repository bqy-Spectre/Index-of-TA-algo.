# M_TV_ToonSkin_Master 卡通皮肤材质 · 美术使用手册

> 适用对象：会画贴图、但不写着色器的材质/角色美术。
> 文中英文均为材质里的真实参数名或贴图名，第一次出现时附中文解释。
> 所有默认值都来自材质本身的连接与参数默认值（母材质默认值，未被实例覆盖）。
> 链路说明按当前节点、实际资产与 UE 5.8 源码共同核对；画面效果由美术确认，不代表已做视觉验收。父默认是继承值，不是已经签核的效果配方。
> 资源目录：母材质保持在 `/Game/Toon/MaterialLibrary/Masters/`；皮肤函数、Profile、专属贴图和组件分别归入 `Functions/ToonSkin/`、`Profiles/ToonSkin/`、`Support/ToonSkin/`、`Blueprints/ToonSkin/`；中性法线位于 `Support/Shared/`。

---

## 0. 开工前先看：四个总开关默认是关的，而且是「静态开关」

这套材质有四个 `StaticSwitchParameter`（静态开关参数），**默认全部为 False**：

| 开关名 | 中文含义 | 默认 | 控制的功能 |
|---|---|---|---|
| `UseDetailNormal` | 启用细节法线 | **False** | 关掉时只用主法线，细节法线整条链路被跳过 |
| `UseRim` | 启用边缘光 | **False** | 关掉时边缘光完全不出现 |
| `UseSphereHighlight` | 启用卡通高光球 | **False** | 关掉时高光球完全不出现 |
| `UseFaceSDF` | 启用脸部 SDF 阴影 | **False** | 关掉时图案 UV 走普通 `PatternUVScale` 那条路 |

**静态开关和标量参数不一样，请注意两点：**

1. 它们是在**材质实例（MI）**里勾的，勾完会**触发着色器重新编译**，不能在运行中动态切换。也就是说：`RimIntensity` 可以在运行时动画，但 `UseRim` 不行。
2. 因为要重编译，调试时不要反复来回勾——一次勾完再看效果。

**另外一批默认关闭或贡献为零的参数**（画了贴图没效果时要先查）：

| 参数 | 中文含义 | 默认 |
|---|---|---|
| `BlushStrength` | 腮红强度 | **0** |
| `CavityStrength` | 凹陷（腔隙）强度 | **0** |
| `RimIntensity` | 边缘光强度 | **0** |
| `SphereHighlightIntensity` | 高光球强度 | **0** |
| `FaceSDFDataValid` | 脸部 SDF 数据有效 | **0** |
| `OilSpecularBoost` / `NoseSpecularBoost` | 出油 / 鼻头高光提升 | 0 |
| `NoseRoughnessBias` | 鼻头粗糙度偏移 | 0 |
| `RoughnessBias` | 整体粗糙度偏移 | 0 |
| `OilRoughnessBias` | 出油区粗糙度偏移 | **−0.1**（只有 Effects.G 非零时参与；默认图 G=0） |

**其他父默认值（有些仍需开关和贴图配合）：**

| 参数 | 默认 | 说明 |
|---|---|---|
| `NormalStrength` | 1 | 主法线满强度 |
| `DetailNormalStrength` | 0.2 | 启用细节法线后参与强度计算，不是推荐起点 |
| `DetailUVScale` | 10 | 细节法线平铺 10 次 |
| `DetailFadeNear` / `DetailFadeFar` | 350 / 1300 | 细节法线的距离淡出区间（单位厘米） |
| `SpecularBase` | 0.5 | 高光基准值 |
| `SpecularMultiplier` / `RoughnessMultiplier` / `AOMultiplier` | 1 | 三个总倍率 |
| `MinRoughness` / `MaxRoughness` | 0.18 / 0.90 | 粗糙度钳制区间 |
| `RimPower` | 3 | 边缘光收束程度 |
| `FaceSDFStrength` / `FaceSDFSoftness` | 1 / 0.02 | |
| `PatternUVScale` / `SphereUVScale` | 1 / 1 | |
| `BlushTint` | (1, 0.53, 0.48) | 偏粉的腮红色 |
| `BaseColorTint` / `RimColor` / `SphereHighlightTint` | 白 (1,1,1) | 不改色 |

---

## 1. 这套材质能做出什么效果

它是一套 **Substrate（UE 的新材质框架）+ Toon BSDF（卡通着色）** 的皮肤母材质，最终输出一个 `SubstrateToonBSDF`。能做的五件事：

1. **皮肤本体**：底色 + 凹陷（腔隙，比如鼻翼、眼窝、嘴唇缝）+ 腮红 + 主法线 + 细节法线（毛孔）+ 粗糙度/高光分区。
2. **局部质感微调**：出油区（额头、鼻翼、T 区）和鼻头，各自有独立的粗糙度偏移和高光提升。
3. **卡通边缘光**（Rim）：沿轮廓描一圈光。注意它进的是 **EmissiveColor（自发光）**，强度不由场景灯光决定，不是逐灯计算的真实高光；最终显示仍受渲染和后处理影响。
4. **卡通高光球**（Sphere Highlight）：按视空间法线采样球面图，同样进自发光。图案随表面朝向和观察方向变化，不是固定在脸部 UV 上的贴花，也不是随场景灯光移动的真实高光。
5. **脸部 SDF 阴影**（Face SDF）：按主光方向和头部朝向，从 SDF 图里取阴影阈值，算出一张"Ramp 偏移载体 UV"，喂给卡通着色节点去推动明暗分界。这是这套材质最复杂、也最像"正统二次元面部阴影"的部分。

**做不到 / 别在这里找的（拓扑里没有的东西）：**

- **金属**：`Metallic` 接的是常量 **0**，改不了。
- **各向异性 / 拉丝**：BSDF 的 `Anisotropy`（各向异性）和 `Tangent`（切线）两个输入**都是空的**，没有连线。皮肤不需要，但别指望能开。
- **透明、半透、镂空**：`BLEND_Opaque`（不透明）、`MD_Surface`（表面域）、`TwoSided = False`（单面）。
- **菲涅尔节点**：整张图里没有 Fresnel 节点。边缘光是用 `dot(顶点法线, 相机向量)` 自己算的，效果类似但实现不同。
- **自动读取场景灯光**：`FaceKeyLightDirectionWS`（主光方向）是一个**手填的向量参数**，材质不会自己去读场景里的灯。同理 `FaceHeadForwardWS` / `FaceHeadRightWS`（头部朝向/右向）也是手填的。这三个由每角色自己的 MID（动态材质实例）接收。已有 `BPC_TV_ToonSkin_FaceSDF` 组件负责更新，配置方法见 §3.7；材质本身不会自动读取骨骼和主光。

---

## 2. 贴图怎么制作

### 2.0 八张贴图总览

| # | 贴图参数名 | 采样器类型 | 默认资源 | 实际用到的通道 | UV |
|---|---|---|---|---|---|
| 1 | `BaseColorTexture` | Color（颜色） | `WhiteSquareTexture` | RGB | UV0 |
| 2 | `SkinSurface_MSK` | Masks（遮罩） | `T_TV_ToonSkin_Default_Surface_MSK` | **R=AO、G=粗糙度、B=高光遮罩、A=凹陷**（四通道全用） | UV0 |
| 3 | `SkinEffects_MSK` | Masks | `T_TV_ToonSkin_Default_Effects_MSK` | **R=腮红、G=出油、B=鼻头、A=细节法线遮罩**（四通道全用） | UV0 |
| 4 | `SkinStylize_MSK` | Masks | `T_TV_ToonSkin_Default_Stylize_MSK` | **R=高光球遮罩、G=边缘光遮罩、A=总闸门**（**B 未用**） | UV0 |
| 5 | `NormalTexture` | Normal（法线） | `T_TV_Neutral_N` | RGB | UV0 |
| 6 | `DetailNormalTexture` | Normal（法线） | `T_TV_Neutral_N` | RGB | UV0 × `DetailUVScale`(10) |
| 7 | `SphereHighlightTexture` | **Color（颜色）** | `T_TV_ToonSkin_Default_Sphere_BC` | RGB=颜色、**A=强度** | **高光球 UV**（随视角算出来） |
| 8 | `FaceSDF_MSK` | Masks | `T_TV_ToonSkin_Default_FaceSDF_MSK` | **R=右侧来光阈值、G=左侧来光阈值、B=权重** | **脸部 UV**（UV0 经 `FaceSDFUVTransform` 变换） |

**默认占位图实际是什么（已导出源像素确认）**

五张皮肤专用默认图都是 1×1，下面是源 RGBA 的 8 位数值：

| 默认资源 | 源 RGBA | 默认贡献 |
|---|---|---|
| `T_TV_ToonSkin_Default_Surface_MSK` | (255,140,255,255) | AO/B/A 为 1；G 为 140/255≈0.54902 |
| `T_TV_ToonSkin_Default_Effects_MSK` | (0,0,0,255) | 腮红、出油、鼻头遮罩为 0；细节法线遮罩为 1 |
| `T_TV_ToonSkin_Default_Stylize_MSK` | (0,0,0,255) | 总闸门 A 为 1，但高光球 R 和边缘光 G 都为 0 |
| `T_TV_ToonSkin_Default_Sphere_BC` | (0,0,0,0) | RGB 与 Alpha 都为 0，没有球图发光贡献 |
| `T_TV_ToonSkin_Default_FaceSDF_MSK` | (0,0,0,255) | B 为 0，关闭脸部 SDF；不是可照着画的脸部范例 |

默认出油**没有开启**：虽然 `OilRoughnessBias=-0.1`，默认 Effects.G=0，乘出来仍是 0。风格化图 A=1 也不等于功能开启，还要 R/G、静态开关和强度配合。默认高光球贴图本身为零，只提高强度也不会出现图案。

这些是**源像素**，不是平台压缩后每次采样的精确保证；Surface.G 是基础粗糙度输入，最终还要经过倍率、三项偏移和 Min/Max 钳制。

---

### 2.1 `BaseColorTexture`（底色贴图）

**画什么**

皮肤本身的颜色。脸颊、额头、脖子、手背都要画出**血色差异**：脸颊偏红、额头偏黄、眼周偏青紫（卡通皮肤常画一层淡淡的冷色）、嘴唇更红。关节、指节、锁骨也可以画得偏红一点。

**画完什么效果**

链路（在 `MF_TV_ToonSkin_Color` 函数里）：
```
Tinted（染色后）     = BaseColorTexture.RGB × BaseColorTint（底色染色，默认白）
CavityFactor（凹陷系数） = Lerp(1, SkinSurface_MSK.A, Saturate(CavityStrength))
CavityColor（凹陷后）  = Tinted × CavityFactor
```

- 底色是**直接相乘**，没有反相、没有重映射。白色贴图 = 保留 `BaseColorTint` 的颜色，黑色 = 变黑。
- 默认 `CavityStrength = 0`，所以 `CavityFactor = 1`，凹陷这一步默认不改变任何东西。
- `BaseColorTint` 大于 1 **仍能提亮未到上限的颜色**：例如 0.2×2=0.4。最终 `FinalColor = Saturate(...)` 只把每个通道限制在 0~1；到 1 后继续增加不会再提高该通道的底色输入。

**⚠ 关于腮红的一个反直觉点**

腮红**不是**乘在底色上的，而是：
```
最终底色 = Lerp(凹陷后的底色, BlushTint（腮红色，默认 (1,0.53,0.48)）, 腮红权重)
```
也就是说：**腮红权重等于 1 时，该处颜色会完全变成 `BlushTint`，底色的所有细节（包括凹陷）全部被覆盖掉。** 这和"加一层红"完全不同——它是**替换**。因此要同时看 `SkinEffects_MSK.R` 与 `BlushStrength`：R=1 只有在强度足以让权重到 1 时才完全替换；例如 R=1、Strength=0.2，权重仍为 0.2。

**怎么配合其他贴图**

- 底色是表面的基础颜色，不只管正对相机的画面；最终明暗仍会受光照、Profile 和观察方向影响。额外边缘光见 §3.4，高光球见 §2.6。
- 底色**不管**粗糙度、AO、出油、鼻头——那些全在 `SkinSurface_MSK` 和 `SkinEffects_MSK` 上。

**怎么导入 / 怎么检查**

采样器类型 `SAMPLERTYPE_Color`。当前默认 `WhiteSquareTexture` 的实际设置见 §2.8；替换底色使用 **sRGB 开启、Compression=Default** 的颜色图，只用 RGB，用 UV0。Mip 和地址模式按新图检查，不把采样节点的默认 Mip 模式当成贴图已有 Mip 的证明。
颜色不对 → 先看 `BaseColorTint` 是否被实例改过；再确认 `BlushStrength` 是不是被调大了（腮红会整片替换颜色，很容易误判成底色错了）。

---

### 2.2 `SkinSurface_MSK`（表面图）★四通道全用

通道分工由 `ComponentMask`（通道遮罩）节点明确指定：

| 通道 | 作用 | 链路 |
|---|---|---|
| **R** | AO（环境光遮蔽） | `AO = Saturate(R × AOMultiplier)` → 直接进材质根节点的 `AmbientOcclusion` |
| **G** | 粗糙度 | `RoughnessBase = G × RoughnessMultiplier`（后面还有三道加法，见 3.2） |
| **B** | 高光遮罩 | `Specular = Saturate(高光链 × B)` |
| **A** | 凹陷 / 腔隙 | `CavityFactor = Lerp(1, A, Saturate(CavityStrength))`（乘进底色） |

**画什么**

- **R（AO）**：画缝隙和遮挡处。鼻翼两侧、鼻孔、眼窝深处、嘴唇缝、耳廓内部、指缝、脖子与衣领交界 → 画黑；平坦的额头、脸颊 → 留白。
- **G（粗糙度）**：画"反光散不散"。
  - 鼻头、额头、颧骨这类容易出油的地方 → 画暗（光滑，出锐利高光）；
  - 脸颊、脖子、手背的哑光区 → 画亮（粗糙，高光糊开）；
  - 嘴唇 → 画得比周围暗（湿润感）。
- **B（高光遮罩）**：画哪些部位需要抑制真实高光的 Specular 输入，不是所有亮斑的总遮罩。需要抑制的区域画黑，其他区域留白；**画黑 = 该处 Specular 输入归零，不负责关闭自发光高光球或边缘光。**
- **A（凹陷）**：画"结构性暗部"。眼窝、鼻孔内侧、人中、法令纹、锁骨窝、指关节的褶皱 → 画黑（会被压暗）；平坦区域 → 留白。**默认 `CavityStrength = 0`，这张通道默认不起作用。**

**画完什么效果（四个通道都没有反相）**

| 通道 | 白色（1） | 黑色（0） |
|---|---|---|
| R（AO） | 默认倍率 1 时 AO 输入为 1 | AO 输入为 0；不保证直射光、自发光也一起变黑 |
| G（粗糙度） | 提高基础粗糙度 | 降低基础粗糙度；最终值还受倍率、偏移及 Min/Max 限制 |
| B（高光遮罩） | 保留前面算出的 Specular 输入 | Specular 输入为 0，不关闭自发光或其他渲染路径 |
| A（凹陷） | 不压暗这一步的底色 | 强度 1 时凹陷后的底色为 0，之后仍可能被腮红替换 |

**怎么配合其他贴图**

- R 只进 AO 通道，**不会**去影响底色。
- G 是粗糙度的**基础值**，之后还会被 `RoughnessBias`、`OilRoughnessBias`、`NoseRoughnessBias` 三道加法修改（见 3.2）。
- B 是 **Specular 输入**的最后一道区域闸门：前面算出的基础值和两项提升都要乘它；实际高光还受粗糙度、法线、灯光和 Profile 影响。
- A 在图内只直接修改底色，不修改 Roughness 或 Specular 输入；不等于最终所有光照表现都与底色无关。

**怎么导入 / 怎么检查**

采样器类型 `SAMPLERTYPE_Masks`，**关闭 sRGB，Compression=Masks**，四通道分别制作，用 UV0。当前默认双轴 Clamp、Mip=FromTextureGroup；TC_Masks 是压缩预设，不保证所有平台固定使用 BC7。默认 Surface 的源像素为 (255,140,255,255)，G≈0.54902。
改了没反应 → 确认画对通道（R/G/B/A 语义完全不同）；AO 太重用 `AOMultiplier` 调，凹陷没出来先查 `CavityStrength` 是否为 0。

---

### 2.3 `SkinEffects_MSK`（效果图）★四通道全用

| 通道 | 作用 | 链路 |
|---|---|---|
| **R** | 腮红遮罩 | `BlushWeight = Saturate(R × BlushStrength)` |
| **G** | 出油区 | 粗糙度 `+ G × OilRoughnessBias`；高光 `+ G × OilSpecularBoost` |
| **B** | 鼻头区 | 粗糙度 `+ B × NoseRoughnessBias`；高光 `+ B × NoseSpecularBoost` |
| **A** | 细节法线遮罩 | `DetailStrength = Saturate(A × DetailNormalStrength × 距离淡出)` |

**画什么**

- **R（腮红）**：用灰度分布画泛红区域和过渡。白色给完整遮罩，不一定完全换色，实际权重是 R×`BlushStrength` 再钳到 0~1；权重达到 1 才完全取 `BlushTint`。灰度和强度需配合，不给未经签核的峰值配方。
- **G（出油）**：额头、鼻翼、鼻梁、下巴的 T 区 → 画白；脸颊、脖子 → 画黑。
- **B（鼻头）**：只有鼻头（鼻尖到鼻翼那一小块）→ 画白；其余全黑。
- **A（细节法线遮罩）**：哪里要看到毛孔。鼻子、鼻翼、额头、脸颊 → 画白；嘴唇、眼睑这些光滑处 → 画黑。**这是细节法线的区域闸门，画黑的地方毛孔完全不出现。**

**画完什么效果**

- **R**：见 2.1 的替换说明。默认 `BlushStrength = 0` → 权重 0 → 腮红完全不动。
- **G（出油）**：两条路同时生效，`OilRoughnessBias` 默认 **−0.1**、`OilSpecularBoost` 默认 0。
  - **默认贴图 G=0，出油两路都没有贡献**。换图后 G 为白、仍用父默认时，粗糙度在最终钳制前减 0.1，但 Specular 输入不增加；粗糙度变化本身仍会改变高光形状。`OilSpecularBoost` 为正时才额外提高出油区的 Specular 输入。
- **B（鼻头）**：两个参数默认都是 0，所以默认完全不起作用。想让鼻头更亮更油 → 调正 `NoseSpecularBoost`、调负 `NoseRoughnessBias`。
- **A**：细节法线的闸门，配合 2.5 一起看。

**关于"出油"和"鼻头"两组参数的关系**

这两组有独立遮罩和参数，**没有优先级或替换关系，但会加到同一个粗糙度／Specular 输出上**。遮罩重叠时两组贡献相加，最后一起受 Clamp／Saturate 限制；其中一组把结果推到上限或下限后，另一组可能看不出增量。你可以只用其中一组——比如不要鼻头，就把 `SkinEffects_MSK.B` 全画黑，或者把两个 Nose 参数都留 0。

**怎么导入 / 怎么检查**

关闭 sRGB，Compression=Masks，用 UV0，当前默认双轴 Clamp、Mip=FromTextureGroup。默认源像素为 (0,0,0,255)，G=0，出油未开启。
出油没效果 → 先检查实例是否覆盖正确，再看 G 是否非零；随后分别检查粗糙度偏移和 Specular 提升。只检查 `OilSpecularBoost` 不够，最后还要看粗糙度钳制和 Surface.B。

---

### 2.4 `SkinStylize_MSK`（风格化图）★一图管两处，A 是总闸门

| 通道 | 作用 | 链路 |
|---|---|---|
| **R** | 高光球遮罩 | `SphereMask = R × Saturate(A)` |
| **G** | 边缘光遮罩 | `RimMask = G × Saturate(A)` |
| **A** | **总闸门** | `StyleA = Saturate(A)`，同时乘进上面两个遮罩 |
| B | **未使用** | 画了无效 |

**画什么**

- **A（总闸门）**：哪些部位允许出现卡通风格化效果（高光球 + 边缘光）。脸、脖子、手 → 画白；被头发遮住的区域、衣服下面的皮肤、不想发光的地方 → 画黑。**这一张通道同时关掉两个效果。**
- **R（高光球遮罩）**：想要"卡通高光形状"的区域。额头、鼻梁、脸颊、肩膀 → 画白；眼窝深处、暗部 → 画黑。
- **G（边缘光遮罩）**：想要轮廓描光的位置。脸部外轮廓、肩膀边缘、手臂外缘 → 画白；朝向内侧、被遮挡的部分 → 画黑。

**画完什么效果**

- 三个通道都是**纯乘法，无反相**：白色 = 效果最强，黑色 = 完全关闭。
- **A 是双闸门**：`SphereMask` 和 `RimMask` 都要再乘一次 `Saturate(A)`。所以 A 画黑的地方，R 和 G 画得再白也没用。
- 这两个遮罩还要分别乘上 `UseSphereHighlight` / `UseRim` 两个静态开关，以及 `SphereHighlightIntensity` / `RimIntensity` 两个标量（默认都 0）。**四道条件缺一不可。**

**怎么导入 / 怎么检查**

关闭 sRGB，Compression=Masks，只需要画 **R / G / A**，用 UV0；当前默认双轴 Clamp、Mip=FromTextureGroup。默认源像素为 (0,0,0,255)：A=1，但 R/G=0，两个风格化效果的区域贡献仍为零。
高光球或边缘光不出现 → 按顺序查：① `UseSphereHighlight` / `UseRim` 静态开关有没有勾；② `SphereHighlightIntensity` / `RimIntensity` 是不是 0；③ `SkinStylize_MSK.A` 是不是画黑了；④ 对应的 R 或 G 通道是不是黑。

---

### 2.5 `NormalTexture`（主法线）与 `DetailNormalTexture`（细节法线）

**画什么**

- **主法线（2.5a）**：大的结构起伏。法令纹、眉弓、鼻梁的骨感、锁骨的凹陷、手背的青筋、指关节的褶皱。
- **细节法线（2.5b）**：皮肤毛孔、细小的皮纹。它会被平铺 **10 次**（`DetailUVScale` 默认 10），所以要画成一张**可平铺的小单元图**，不是覆盖全身的大图。

**画完什么效果**

主法线先由独立参数 `NormalStrength` 控制，再安全归一化，作为混合的基础层：

```
MainStrength = Saturate(NormalStrength)              // 注意是 Saturate，超过 1 无效
MainLerp     = Lerp(平面法线 (0,0,1), NormalTexture.RGB, MainStrength)
```

细节法线那一路：
```
DetailMasked   = SkinEffects_MSK.A × DetailNormalStrength（默认 0.2）
Fade           = SmoothStep(DetailFadeNear(350), DetailFadeFar(1300), 相机距离cm)
NearWeight     = 1 − Fade
DetailStrength = Saturate(DetailMasked × NearWeight)
DetailLerp     = Lerp(平面法线, DetailNormalTexture.RGB, DetailStrength)
```

- **`DetailFade` 的方向要记牢**：`NearWeight = 1 − SmoothStep(350, 1300, 距离)`。距离 **350cm 以内 Fade=0 → NearWeight=1 → 毛孔最强**；距离 **1300cm 以外 Fade=1 → NearWeight=0 → 毛孔消失**。
  → 也就是**离近了才看得见毛孔，离远了自动淡掉**。参数名叫 Near/Far 但它是"近距离权重"，别看反。
- 两层最终用 **RNM（Reoriented Normal Mapping，重导向法线混合）** 合成，再安全归一化。这是在主法线方向上叠加细节，不是直接相加；细节能否清楚看见仍取决于输入法线、强度、Mip 和观察距离，不保证所有细节都不会被弱化。
- 链路里有 `SafeNormalize`（安全归一化）：向量长度过小时使用指定回退方向，主法线／细节法线回退为 `(0,0,1)`，混合结果回退为已处理的主法线。它不能把所有错误编码都修好；黑色法线贴图也不等于解码后的零向量。出现异常先检查法线编码、压缩和绿通道方向，不用强度掩盖坏数据。
- **`UseDetailNormal` 静态开关默认 False**：关掉时走"只算主法线"的分支，细节法线、距离淡出、`SkinEffects_MSK.A` 闸门**全部被跳过**。

**怎么配合其他贴图**

- 细节法线的区域由 `SkinEffects_MSK.A` 决定（2.3）。
- 细节法线的密度由 `DetailUVScale` 决定（默认 10），它**只影响细节法线**，不影响主法线。
- 最终法线会喂给三处：BSDF 的 `Normal`、脸部 SDF 的世界法线、以及**高光球 UV**。所以开细节法线会让高光球的流动变得更碎——想让高光球干净，毛孔要克制。

**怎么导入 / 怎么检查**

两张都是切线空间法线贴图，**sRGB 关闭、Compression=Normalmap**；当前默认均为 `Support/Shared/T_TV_Neutral_N`，实际地址与 Mip 设置见 §2.8。Normalmap 是预设，实际平台格式由构建决定，不承诺固定 BC5／BC7。`bTangentSpaceNormal = True`；采样后按法线解码，不把源 RGB 当作普通颜色。
- 主法线没反应 → 查 `NormalStrength`（默认 1，被 Saturate 钳住所以 >1 无效）；查模型有没有切线/UV；查绿通道方向。
- 细节法线没反应 → 先查实例覆盖和 `UseDetailNormal`（默认 False）；再查 `DetailNormalStrength`、`SkinEffects_MSK.A`、实际细节贴图是否仍是中性图，最后检查距离淡出。

---

### 2.6 `SphereHighlightTexture`（卡通高光球贴图）★RGB 是颜色，A 控制强度

**它不用 UV0。** UV 是实时算出来的（在 `MF_TV_ToonSkin_SphereUV` 里）：

```
视空间法线 = 把最终法线从切线空间变换到视空间，取 XY
SphereUV   = 视空间法线.XY × SphereUVScale × 0.5 + 0.5 + SphereUVOffset
```

也就是经典的球面贴图采样：**坐标跟着视角走**，转相机时图案会在脸上流动。

**画什么**

一张球面图，画你想要的"卡通高光形状"：

- 常见的二次元做法：在中间偏上画一个（或几个）柔和的亮块，其他区域是黑。
- 想要硬边的卡通色块 → 用**硬边**形状，不要用大范围渐变。
- 想要条纹状、星形、花瓣状的高光 → 直接画形状即可，它是一张普通的颜色图。

**画完什么效果（完整链路）**

```
SphereTint    = SphereHighlightTexture.RGB × SphereHighlightTint（默认白）
SphereAlpha   = SkinStylize_MSK.R × Saturate(SkinStylize_MSK.A) × SphereHighlightTexture.A
SphereWeight  = SphereAlpha × Max(SphereHighlightIntensity, 0)
SphereColor   = SphereTint × SphereWeight
→ UseSphereHighlight 开关（默认关）→ 加到 EmissiveColor（自发光）
```

逐个说明：

- **RGB 是颜色**，和 `SphereHighlightTint` 相乘。`SphereHighlightTint` 默认纯白，所以默认就是贴图本来的颜色。
- **A 通道是强度**，直接乘进权重。所以**画黑 = 那里完全不发光**，即使 RGB 很亮也没用。**这张图的 RGB 和 A 都要画。**
- 权重还乘了 `SkinStylize_MSK` 的 R 和 A（2.4），以及 `SphereHighlightIntensity`（默认 0，且被 `Max(..., 0)` 兜住不会变负）。
- **它是自发光的高光形状，不是灯光计算的真实高光**。图内不读取灯光或阴影，所以背光时也可能有贡献；`SkinStylize_MSK` 只能固定哪些角色部位允许它出现，不能自动跟踪当前背光区域。曝光、后处理等仍会影响最终显示。
- `SphereUVScale`（默认 1）**围绕 UV 中心缩放表面朝向的映射**，不是在角色 UV 上调平铺密度；`SphereUVOffset` 默认 (0,0,0,0)，只取 RG 负责平移。Scale 增大，会在相同法线变化下跨过更大的 UV 范围；Scale=0 时，全表面采样 `(0.5,0.5)+Offset.RG`。改变覆盖角度不等于额外改变中心位置。
- 它依赖**最终法线**，所以 `UseDetailNormal` 一开，毛孔会让高光球的边缘变碎。

**怎么导入**

采样器类型是 **`SAMPLERTYPE_Color`（颜色）**。当前默认及替换要求为 **sRGB 开启、Compression=Default**；RGB 按颜色空间处理，Alpha 是强度数据，不做同样的 sRGB 颜色解码。**RGB 和 A 都要准备。** 默认 `T_TV_ToonSkin_Default_Sphere_BC` 是 (0,0,0,0)，没有高光图案。
当前默认双轴 **Clamp**，越界延伸边缘，不重复；Mip=FromTextureGroup，节点跟随资产采样设置。换成 Wrap 会重复，换图后要检查新图地址。纯黑默认图不能用来观察这些纹样差异。

**怎么检查**（按顺序）

1. `UseSphereHighlight` 静态开关有没有勾（默认 False）；
2. `SphereHighlightIntensity` 是不是 0（默认就是 0）；
3. `SkinStylize_MSK.A` 和 `.R` 在你观察的位置是不是白；
4. 贴图的 **A 通道**是不是画了内容（全黑的话完全不亮）；
5. 检查 `SphereUVScale` 是否为 0、图案是否为常量、UV 是否落在 Clamp 的恒定边缘，再看模型和观察方向是否实际变化。中性法线贴图仍保留模型几何法线，不能据此认定图案不会随视角变。

---

### 2.7 `FaceSDF_MSK`（脸部 SDF 阴影图）★需要贴图、Face Profile 和运行数据配套

**它仍使用 UV0。** 在 UV0 上再做一次缩放和偏移：

```
FaceUV = UV0 × FaceSDFUVTransform.RG + FaceSDFUVTransform.BA
```

`FaceSDFUVTransform` 默认 `(R=1, G=1, B=0, A=0)`，也就是 RG 是缩放、BA 是偏移，默认不缩放不偏移。如果你的脸部 SDF 图是按某种 atlas（图集）布局排的，就靠这四个数把脸那块区域框出来。

**画什么**

这张图沿用脸部 **SDF** 的命名，但本实现读取的是 **0~1 的受光阈值数据**，不是直接把固定的有符号距离当阴影。数值越高，该像素在主光从正面转向背面时越晚进入阴影：

- **R 通道**：光从**角色头部右侧**来时使用的阈值。
- **G 通道**：光从**角色头部左侧**来时使用的阈值。左右由 `FaceHeadRightWS` 定义，不是屏幕左右；两通道各自覆盖完整脸部 UV，允许左右不对称。
  - 代码里的取法：`sdf = saturate(side >= 0 ? SDFSample.r : SDFSample.g)`——根据主光方向落在头的哪一侧，自动选 R 还是 G。
- **B 通道**：**覆盖权重**。白色给完整覆盖，黑色保留原生明暗，灰色部分混合，并非只有纯白才能生效。按角色设计画脸部覆盖；眼睛、眉毛、嘴唇、脖子等需要保留原生明暗的部位画黑。它不读取额发投影，也不能用固定头发覆盖区替代动态 VSM 阴影。

**SDF 图怎么画**：R/G 都存线性 0~1 阈值；不是“0.5 永远分开亮面和暗面”。当 `FaceSDFBias=0`，主光投影正前方时阈值中心为 `−w`，侧面为 `0.5`，正后方为 `1+w`。图值高于当前阈值更偏受光，低于阈值更偏阴影；只有侧面入射时 0.5 才是过渡中心，软边宽度由 `FaceSDFSoftness` 决定。

先由 TA 根据脸部 UV 和各光照角度想要的边界制作阈值图，再交美术审核形状。当前默认源像素是 (0,0,0,255)，B=0，**只是关闭占位图，不是角色 SDF 制作范例**；当前库内没有已核实的角色范例可供照画。

**画完什么效果（完整链路，来自 Custom 节点的 HLSL）**

```
1. 用 FaceHeadForwardWS / FaceHeadRightWS 构造头部坐标系（正交化）
2. 把 FaceKeyLightDirectionWS 投影到这个坐标系，得到光相对头的方位角 a
   a = saturate(acos(dot(光方向, 头前方向)) / π + FaceSDFBias)
3. w   = Max(FaceSDFSoftness, 0.0001)（父默认 0.02）
   t   = Lerp(−w, 1+w, a)
4. lit = smoothstep(t−w, t+w, sdf)          ← 用 SDF 图切出明暗
5. q   = Clamp(dot(归一化世界法线, 归一化光方向), −1, 1) × 0.5 + 0.5
                                               ← 原生 Ramp 的主光坐标参考，不是最终亮度
6. delta = weight × (lit − q)               ← SDF 与真实光照的差值
7. weight = Saturate(SDFSample.b) × Saturate(FaceSDFStrength)
            × Saturate(FaceSDFDataValid) × basisValid × verticalFade
8. 返回 CarrierUV = (0.5 + delta × 0.25, 0.5)
```

翻译成人话：**它比较“SDF 想要的受光值”和“主光原本的 Ramp 坐标”，把差距编码到载体 UV，再由 Face Profile 读回漫反射偏移。** 它不把 SDF 乘进底色，也不改法线、粗糙度或 Specular 输入；原生 Ramp 仍负责将修正后的坐标变成颜色。

几个必须知道的点：

- **`FaceSDFDataValid` 默认是 0 → weight = 0 → delta = 0 → CarrierUV = (0.5, 0.5)，材质计算的偏移为零。** 这是数据有效标记，不是美术强度旋钮。还必须选择 Face Profile、准备 B 非零的有效阈值图，并让 `FaceSDFStrength` 非零；仅开静态开关和有效标记不够。
- **`basisValid`**：头部前向量、右向量、光方向、法线这几个向量只要有一个长度接近 0，SDF 权重就为 0，返回中性的载体 UV `(0.5,0.5)`。头部前向和右向互相平行也会触发保护。默认值 `FaceHeadForwardWS = (1,0,0)`、`FaceHeadRightWS = (0,1,0)`、`FaceKeyLightDirectionWS = (1,0,0)` 都是单位向量，不会触发这个保护——但它们**是占位值，不代表真实的头和光**。
- **`verticalFade`**：当主光方向和头部"上下轴"完全平行时（光从正上方或正下方打来），这个淡出项会归零。这是为了避免光在头顶正上方时方位角失去意义。
- **`FaceKeyLightDirectionWS` 由运行数据驱动**：方向指向光源，Directional Light 取 Forward 的负值。默认 (1,0,0) 与头前向相同，是有效的正面入射，不是退化坐标系；只是占位方向，不会自行跟随场景。默认静态开关关闭、DataValid=0、贴图 B=0，所以不能说默认一定出现固定阴影。
- `FaceSDFBias`（默认 0）偏移归一化角度阈值：正值让同一图值更早进入阴影，负值更晚；它不旋转头部或光源。`FaceSDFSoftness`（默认 0.02）控制过渡宽度，越小越硬，实际最低使用 0.0001。
- **脸部实例必须覆盖为 `TP_TV_ToonSkin_Face`**。Master 默认是 Body Profile，没有偏移载体；误用 Body Profile 时算出了 CarrierUV 也不会得到所需偏移。Face Profile 的载体、偏移 Strength=2／Size=1 和阴影处理属于配套设置，不拿来当普通花纹调参。
- 单主光适用条件下，偏移用于把主光 Ramp 坐标推向 SDF 结果；多光源下会修正共享明暗，不能承诺只影响指定主光。原生 VSM 遮挡仍由引擎处理，材质不自行读取额发阴影；角色动画同步与 VSM 的既有未完成验收，不因这份手册核查标记通过。
- 关掉 `UseFaceSDF` 时，`PatternUVs` 走 `UV0 × PatternUVScale`；开启时该通道被载体占用，不能同时作为独立偏移花纹或排线坐标。

**怎么导入**

采样器类型 `SAMPLERTYPE_Masks`，**关闭 sRGB，Compression=Masks，双轴 Clamp，Mip=FromTextureGroup**。准备 R/G 阈值与 B 覆盖，A 保留不读取；使用变换后的 UV0。默认 `T_TV_ToonSkin_Default_FaceSDF_MSK` 的 B=0，不是有效脸部图。

**怎么检查**（按顺序）

1. `UseFaceSDF` 静态开关有没有勾（默认 False）；
2. 实例是否选择 `TP_TV_ToonSkin_Face`，不要仍继承 Body Profile；
3. `FaceSDF_MSK.B`、`FaceSDFStrength` 是否非零，R/G 是否为有效阈值数据，UV 是否对应脸；
4. 组件是否配置有效，并写入 `FaceSDFDataValid=1`（默认 **0**，失效时应归零）；
5. 三个方向是否有效且跟着实际头骨和主光更新，近垂直光淡出是否是当前情形。

---

### 2.8 导入与采样汇总（节点与实际默认资产已核对）

| 项 | 实际连接与采样说明 |
|---|---|
| UV 通道 | `BaseColorTexture` / `SkinSurface_MSK` / `SkinEffects_MSK` / `SkinStylize_MSK` / `NormalTexture` 用 **UV0**；`DetailNormalTexture` 用 **UV0 × `DetailUVScale`(10)**；`SphereHighlightTexture` 用**实时算出的高光球 UV**；`FaceSDF_MSK` 用 **UV0 经 `FaceSDFUVTransform` 变换后的脸部 UV** |
| 平铺 | 只有一个：`DetailUVScale`（默认 10）。`SphereUVScale`（默认 1）只作用于高光球 UV，不是常规平铺 |
| 采样器来源 | 全部 `SSM_FromTextureAsset`（跟随贴图资产自身的采样设置） |
| Mip | 八个采样节点均为 `MipValueMode = TMVM_None`，没有显式 LOD／Bias；这不等于贴图已具有完整 Mip，需另看资产尺寸、Mip 设置和平台构建 |
| 采样器类型 | `BaseColorTexture`、`SphereHighlightTexture` = **Color**；`NormalTexture`、`DetailNormalTexture` = Normal；`SkinSurface_MSK`、`SkinEffects_MSK`、`SkinStylize_MSK`、`FaceSDF_MSK` = Masks |
| 未使用通道 | `SkinStylize_MSK` 的 **B** |
| 颜色空间与压缩 | 当前资产实际设置见下表；Color、Masks、Normal 的 sampler 需与替换图设置兼容。压缩预设不等于固定平台编码，不把 TC_Masks 当作 BC7 保证 |

**八个槽的当前默认资产设置**（不是测试实例的 override）：

| 贴图槽 | 当前默认资源 | sRGB | Compression 预设 | Address X/Y | MipGenSettings |
|---|---|---|---|---|---|
| `BaseColorTexture` | Engine `WhiteSquareTexture` | 开 | Default | Wrap / Wrap | Sharpen4 |
| `SkinSurface_MSK` | `T_TV_ToonSkin_Default_Surface_MSK` | 关 | Masks | Clamp / Clamp | FromTextureGroup |
| `SkinEffects_MSK` | `T_TV_ToonSkin_Default_Effects_MSK` | 关 | Masks | Clamp / Clamp | FromTextureGroup |
| `SkinStylize_MSK` | `T_TV_ToonSkin_Default_Stylize_MSK` | 关 | Masks | Clamp / Clamp | FromTextureGroup |
| `NormalTexture` | `T_TV_Neutral_N` | 关 | Normalmap | Wrap / Wrap | FromTextureGroup |
| `DetailNormalTexture` | `T_TV_Neutral_N` | 关 | Normalmap | Wrap / Wrap | FromTextureGroup |
| `SphereHighlightTexture` | `T_TV_ToonSkin_Default_Sphere_BC` | 开 | Default | Clamp / Clamp | FromTextureGroup |
| `FaceSDF_MSK` | `T_TV_ToonSkin_Default_FaceSDF_MSK` | 关 | Masks | Clamp / Clamp | FromTextureGroup |

**Wrap 处理越界重复，Clamp 延伸边缘，都不负责改变密度。** 细节法线若要平铺，新图应保留 Wrap；其他图以实际 UV、边缘内容和用途选择地址。五张专用默认图只有 1×1，FromTextureGroup 不会凭空产生更多尺寸级别；生产图换成更大尺寸后，要检查实际构建的 Mip。

---

## 3. 效果怎么调整

> 按"你想改什么画面"来查。标量参数建议在**材质实例（MI）**上覆盖；四个静态开关必须在实例上勾（会触发重编译）。

### 3.1 高光太弱 / 太强

送入 Toon BSDF 的 **Specular 输入**是这条链（在 `MF_TV_ToonSkin_Response` 里），不是最终画面亮度：

```
SpecOil   = SpecularBase（默认 0.5）+ SkinEffects_MSK.G × OilSpecularBoost
SpecAll   = SpecOil + SkinEffects_MSK.B × NoseSpecularBoost
SpecGain  = SpecAll × SpecularMultiplier（默认 1）
Specular  = Saturate(SpecGain × SkinSurface_MSK.B)
```

| 你想做的 | 改什么 |
|---|---|
| 改真实高光的基础响应 | `SpecularBase`（默认 0.5），不改变底色或自发光球图 |
| 整体再乘一个倍率 | `SpecularMultiplier`（默认 1） |
| 出油区更亮 | `OilSpecularBoost`（默认 0）+ `SkinEffects_MSK.G` |
| 鼻头更亮 | `NoseSpecularBoost`（默认 0）+ `SkinEffects_MSK.B` |
| 某处 Specular 输入归零 | `SkinSurface_MSK.B` 画黑，不关闭自发光球图 |

注意最后有 `Saturate`，**限制到 0~1 的是 Specular 输入，不是屏幕高光亮度**。输入达到 1 后继续增加没有输入增量；最终高光仍由法线、粗糙度、光照和 Profile 共同决定。

### 3.2 粗糙度（基础值加三项偏移，最后被钳住）

```
RoughBase   = SkinSurface_MSK.G × RoughnessMultiplier
RoughBiased = RoughBase + RoughnessBias（默认 0）
RoughOil    = RoughBiased + SkinEffects_MSK.G × OilRoughnessBias（默认 −0.1）
RoughAll    = RoughOil + SkinEffects_MSK.B × NoseRoughnessBias（默认 0）
Roughness   = Clamp(RoughAll, MinRoughness(0.18), MaxRoughness(0.90))
```

基础值先乘 `RoughnessMultiplier`，随后是**整体、出油、鼻头三项加法偏移**；不是四项。偏移单位是粗糙度数值，最终由 Min/Max 钳住，正常请保持 `0≤MinRoughness≤MaxRoughness≤1`。

- 想让整张脸更光滑 → `RoughnessBias` 给负值（比如 −0.1）。
- 出油区需要 Effects.G 非零；默认 G=0，偏移没有贡献。`OilRoughnessBias` 更负会进一步降低该区粗糙度，但到 Min 后继续减不会改变输出。
- **改不动通常是被两端的 Clamp 钳住了**：检查 `MinRoughness`（0.18）和 `MaxRoughness`（0.90）。例如默认基础值 140/255≈0.54902、效果遮罩为零，Bias=−0.5 时钳制前约为 0.04902，最终为 0.18。不同基础值不一定都会被钳到同一端。这个例子只解释运算，不是推荐配置。

### 3.3 腮红与凹陷

- **腮红**：`BlushStrength`（默认 0）当总开关，`SkinEffects_MSK.R` 当区域，`BlushTint`（默认粉 `(1,0.53,0.48)`）当颜色。
  - **记住它是替换不是叠加**：权重 1 = 整块变成 `BlushTint`。降低 `BlushStrength` 或遮罩灰度，会减少替换比例；只有乘积钳制后达到 1 才完全替换，不给固定起调配方。
- **凹陷（腔隙）**：`CavityStrength`（默认 0）当总开关，`SkinSurface_MSK.A` 当区域（黑 = 压暗）。
  - 强度 1 时，凹陷系数 = 贴图 A，A=0 会把**腮红混合之前的底色**压为零；腮红、自发光及真实光照仍可能使最终画面不黑。减小强度或把 A 提亮，会减轻这一步压暗。

### 3.4 边缘光（Rim）

```
Edge   = 1 − Saturate(dot(顶点法线, 相机方向))
Shape  = Pow(Edge, Max(RimPower, 0.001))
Weight = Shape × RimMask(SkinStylize_MSK.G × Saturate(A)) × Max(RimIntensity, 0)
EmissiveAdd = Weight × RimColor
```

- **用的是顶点法线，不是法线贴图**：所以边缘光**不受法线贴图影响**，只跟模型的几何轮廓走。想改变发光区域，先画 `SkinStylize_MSK.G`；它能控制局部覆盖，但不能替代模型几何法线来改变轮廓方向。
- `RimPower`（默认 3）越大，边缘光越细越贴边；越小越宽越糊。它有一个 `Max(..., 0.001)` 兜底，设成 0 不会出错（但会变得极宽）。
- 四个开关/参数：`UseRim`（静态，默认关）+ `RimIntensity`（默认 0）+ `SkinStylize_MSK.A` + `SkinStylize_MSK.G`，**缺一不可**。
- 它进的是**自发光**，所以背光、暗部也会亮。可用 `SkinStylize_MSK.G` 固定限制发光部位，但静态贴图不能自动判断当前受光侧或动态遮挡。

### 3.5 卡通高光球

见 2.6。三个旋钮：`SphereHighlightIntensity`（默认 0，总强度）、`SphereUVScale`（默认 1，球面范围）、`SphereUVOffset`（默认 0，平移）。颜色改 `SphereHighlightTint` 或贴图本身。

**和边缘光同时开时**：Master 将两路直接相加，`Emissive = Rim + Sphere`，没有两路互相压制、遮挡或总上限。叠加处可能获得更大的自发光数值；是否过曝取决于强度、图案、曝光和后处理，由美术确认，不把同时开启写成必然过曝或必然安全。

### 3.6 卡通图案 UV（PatternUVs）

这是一个**二选一**的输入，由 `UseFaceSDF` 静态开关决定：

- **关（默认）**：`PatternUVs = UV0 × PatternUVScale`
- **开**：`PatternUVs = CarrierUV`（脸部 SDF 算出来的偏移，见 2.7）

**它不是贴花槽。** 原生 Toon 用这套坐标读取漫反射 Ramp 偏移、高光 Ramp 偏移和阴影排线资源；网点是相应纹理的图案形态，不是仅调 Scale 自动生成的功能。

- **当前 Body Profile**：三种图案资源均为 None，强度均为 0；`PatternUVScale` 改了也不会凭空生成花纹。原有漫反射与高光曲线仍参与正常着色。
- **当前 Face Profile**：漫反射偏移使用 `T_TV_ToonSkin_FaceRampCarrier`，Strength=2、Size=1；高光偏移和阴影排线资源均为 None、强度为 0。载体用于读回 SDF 信号，不是装饰图。Face 的 `bDiffuseRampIncludeShadow=False`，真实遮挡在 Ramp 外生效；Body 为 True。
- 载体 UV 和 Profile 图集在引擎中仍有打包、烘焙与采样精度限制；“计算偏移为零”不等于承诺最终读取逐位为零。若边界出现异常偏移，交 TA 核对载体和 Profile，不让美术用 Bias 补偿传输误差。
- 开启 `UseFaceSDF` 时 `PatternUVScale` 不参与载体 UV；不要同时把 Face Profile 当普通排线或偏移图案配置使用。Profile 的资源与曲线修改交 TA 管理，具体画面由美术确认。

### 3.7 脸部 SDF 阴影

见 §2.7。转头或主光变化时，这条功能需要更新运行数据，而不是只在编辑器手填一次：

- `FaceKeyLightDirectionWS` ← 场景主光方向
- `FaceHeadForwardWS` / `FaceHeadRightWS` ← 头部朝向

项目已有组件 `/Game/Toon/MaterialLibrary/Blueprints/ToonSkin/BPC_TV_ToonSkin_FaceSDF`：每角色使用自己的 MID，不能用一个共享 MPC 存所有角色的头部方向。配置 `TargetMesh`、`HeadSocketName`、`HeadForwardLocal`／`HeadRightLocal`、`KeyLight`（指定 Directional Light）、`FaceMaterialSlotNames`，并提供已启用 `UseFaceSDF`、覆盖 Face Profile 的 `FaceParentMI`。

由 TA 检查槽名、Socket 和动画更新策略后开启 `ConfigurationVerified`。组件读取头部世界旋转和指定主光，更新三个方向，最后写 `FaceSDFDataValid`；无效时归零。美术不要用固定方向或强行写 Valid=1 掩盖未配置的数据。组件设置在骨骼组件之后更新，但真实骨骼动画同帧同步的既有验收仍未完成，不宣称已经验证所有动画流程。

---

## 4. 常见问题排查

**Q1：勾了 `UseRim`，边缘光还是没有。**
`RimIntensity` 默认是 0，勾开关只是让链路通了，强度还是 0。同时确认 `SkinStylize_MSK.A`（总闸门）和 `.G`（边缘光遮罩）在你观察的位置是白的。

**Q2：腮红一开，整块脸变成一坨纯粉色，底色全没了。**
这是设计如此，不是 bug——腮红是 `Lerp` 到 `BlushTint`，是**替换**不是叠加。先检查遮罩×`BlushStrength` 的权重是否达到 1，再降低强度或该区域灰度，直到保留所需底色细节；不把测试值当统一配方。

**Q3：毛孔（细节法线）看不到。**
三层原因：① `UseDetailNormal` 静态开关没勾（默认 False）；② `SkinEffects_MSK.A` 是黑的；③ 相机离得超过 1300cm（`DetailFadeFar`），自动淡出了。再检查 `DetailNormalStrength` 和实际贴图内容：默认细节法线是中性图，只改强度不会产生毛孔。

**Q4：毛孔离远了还在，很脏。**
`DetailFadeNear` / `DetailFadeFar` 默认是 350 / 1300 厘米。把 Far 调小（比如 600）能让毛孔在更近的距离就淡掉。**注意方向：`NearWeight = 1 − SmoothStep(Near, Far, 距离)`，正常必须保持 `Near<Far`；相等或反置不作为可依赖的淡出／反转功能。**

**Q5：脸部 SDF 阴影完全没出来。**
两个开关：`UseFaceSDF` 静态开关（默认 False）+ `FaceSDFDataValid`（默认 **0**）。还需选 Face Profile，检查有效 SDF 图的 B 和 Strength 非零，以及组件是否配置通过并更新三个方向；默认 SDF 图 B=0，不会生效。

**Q6：转头/转灯光，脸的阴影不跟着变。**
`FaceKeyLightDirectionWS` / `FaceHeadForwardWS` / `FaceHeadRightWS` 都是**手填的向量参数**，材质不会自动读场景。先检查 §3.7 的现有组件是否正确配置并更新每角色 MID，不要只手填固定方向。

**Q7：高光球不随视角流动。**
按 §2.6 检查 Scale=0、常量／默认全零球图、Clamp 边缘及实际观察方向变化。`NormalStrength=0` 或中性法线只取消贴图扰动，模型几何法线仍参与视空间坐标；不能据此认定球图不动。

**Q8：想把脸的暗部压暗，改了 AO 没用。**
AO（`SkinSurface_MSK.R`）只进环境光遮蔽通道，**不影响底色**。要压暗底色请用**凹陷**（`SkinSurface_MSK.A` + `CavityStrength`）。

**Q9：想要金属、拉丝、透明，能做吗？**
不能。`Metallic` 硬编码 0；BSDF 的 `Anisotropy` 和 `Tangent` 都没接线；材质是 `BLEND_Opaque` + `TwoSided = False`。

**Q10：边缘光/高光球在背光面也亮，看起来假。**
这是正常的——它们进的是**自发光**，不受光照影响。可以用 `SkinStylize_MSK` 的 G/R 固定限制角色上的发光区域；它们不能随场景灯光自动识别背光或额发投影。需要动态受光限制时交 TA 评估，不用静态图冒充动态阴影。

**Q11：反复勾静态开关，编辑器很卡。**
静态开关会触发着色器重编译。请一次性把要勾的都勾完，不要来回切换。
