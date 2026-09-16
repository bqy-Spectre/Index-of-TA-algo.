# UE5.8 Toon 渲染管线架构 v0

> **本版为第一版执行基线，冻结生效。**
>
> 状态：**框架层（第一部分）已全部冻结，待拍板项已全部关闭。**
> 已关闭：D-032 ~ D-039（8 项，见 12.6）。
> 剩余未决项全部为 `待查源码`（Codex）与 `待实验`（跑起来看），见 0.6。
>
> 本文档此前版本（v0.3 ~ v0.8 的全部技术内容）已合并归位，
> 历史修订记录保留在 12.9，仅作溯源用，**不影响本版条款**。
>
> 判断一切取舍的唯一标准：**改一次要不要重新编译、改错了会不会牵动全局。**
>
> 改动本版任何已锁定决策，必须先走 12.7 决策变更流程。

---

# 0. 怎么读这份文档

## 0.1 状态标记（本版重做）

此前把三种性质完全不同的情况都写成"待验证"，导致它们永远清不掉——因为其中一类**再查一百遍源码也还是未决，它缺的是决策不是信息**。本版强制拆开。

| 标记 | 含义 | 谁能关闭它 | 关闭动作 |
| --- | --- | --- | --- |
| **已确定** | 架构决策，除非出现新证据否则不改 | 决策变更流程 | 走 12.7 流程 |
| **已确认** | 经 UE5.8 真实源码验证，可直接写代码 | 已完成 | 无需再查 |
| **待查源码** | Codex 还没查，实现前必须查 | Codex | 源码审查 |
| **待拍板** | 事实已清楚，缺的是**选择**不是信息 | 人 | 做决策、写进决策表 |
| **待实验** | 必须跑起来看，源码查不出来 | 实现后验收 | Golden Scene / 回归用例 |
| 后置 | 第一版产品画面不依赖 | — | — |
| 不支持 | 当前版本明确不进入范围 | — | — |

**使用规则**：

* 任何项在关闭时，必须同时更新第 0.6 节待办总表。
* "待拍板"不得写成"待查源码"——查了也不会有答案。
* "待实验"不得写成"待查源码"——源码里没有画面后果。

## 0.2 文档分层原则

引擎改动分三类，性质完全不同：

| 类别 | 判据 | 改一次代价 | 文档位置 |
| --- | --- | --- | --- |
| 框架契约 | 改一次要 C++ rebuild 或牵动整套数据链 | 极高，牵一发动全身 | 第一部分，现在定死 |
| 数据语义 | 不改代码，但一变所有资产或下游全部作废 | 高 | 第一部分，现在定死 |
| Shader 实现 | 热重载可改，改错只影响画面 | 低，可反复迭代 | 第二部分，推迟设计 |

因此：

* GBuffer 位分配、ShadingModelID、Permutation、View Uniform 属于**框架契约**。
* 资产通道含义、参数通道、数据语义属于**数据语义**。
* 具体算法、公式、参数默认值属于**Shader 实现**，第一部分不设计。

**第一部分不出现任何算法公式，第二部分不重复任何框架契约。**

## 0.3 源码审查基线

本版所有"已确认"结论均来自对下列基线的真实源码审查：

| 项 | 值 |
| --- | --- |
| 引擎版本 | UE 5.8.2 |
| CL | 56702186 |
| 分支 | `++UE5+Release-5.8` |
| 审查方式 | Codex 直连 GitHub 私有仓库读取 |
| 对照源 | MooaToon-Engine 5.7 分支（仅作落点与算法参考） |

引用规则：

* 标"已确认"的内容可以直接写代码，不需要再查。
* 标"待查源码"的内容只是假设，实现前必须查。
* 未标任何状态的内容视为架构推理，**不得当作事实使用**。

## 0.4 常见 UE 渲染词（小白向）

| 名词 | 小白理解 |
| --- | --- |
| BasePass | 第一个工位：把这个像素是什么材质、什么法线、什么颜色写进 GBuffer |
| GBuffer | 延迟渲染用的一组"像素资料表"，后面的工位都读它 |
| Deferred Lighting | 灯光工位：读 GBuffer，算最终直接光 |
| CustomData | GBufferD，引擎留给我们自定义管线的"自留地"，只有 4 个字节 |
| ShadingModel | 材质类型，如塑料、金属、皮肤；我们要新增一个"卡通" |
| Permutation | Shader 的不同编译组合，如 Opaque / Masked / 不同 ShadingModel |
| CPD | Custom Primitive Data，每个模型组件自带的 36 个 float 插槽，原生功能 |
| RDG | UE 管理 Render Pass、Texture 依赖关系的系统 |
| Lumen | UE 的动态全局光照系统 |
| TSR | UE 的时间超分辨率与抗锯齿系统 |
| UNorm8 | 每个通道只有 8 位、共 256 级的存储格式 |
| 八面体编码 | 把一个 3D 方向压进 2 个 8 位通道的方法 |
| SSS | 次表面散射，逆光时耳廓、发梢透出暖色的效果 |

## 0.5 给执行 Agent 的总规则

1. 严格按第 10 节的分阶段计划执行，一次只做一个阶段。
2. 每个阶段完成后必须停下，提交验收结果，等待人工确认后才进入下一阶段。
3. 禁止跨阶段提前实现任何功能。
4. 禁止在某一阶段验收未通过时继续推进。
5. 遇到本文档未覆盖的情况，停下来提问，不要自行发挥。
6. 每次修改本文档，必须在 12.9 追加修改摘要。
7. 修改任何已锁定决策，必须先走 12.7 决策变更流程。
8. 第 12.10 节 Codex 审查清单中未标记"已完成"的项，禁止开始对应实现。
9. **第一部分未全部冻结前，禁止进入第二部分的任何算法设计。**
   * **状态：第一部分（框架层）已于 v0 定版全部冻结**——D-032 ~ D-039 全部关闭，待拍板栏为空。
   * 因此**第二部分（Shader 实现层）算法设计已解锁**，可按第 10 章阶段计划逐阶段展开。
   * 解锁不等于可以改框架：第二部分仍**不得重新定义数据来源**，只能引用第一部分已冻结的契约字段。
10. **每次回填文档后必须做"三问巡检"**（本版新增，防止增量回填再次产生自相矛盾）：
    1. 新结论是否推翻了文档里某处旧标记？
    2. 是否有 P0 项没编号、没归属？
    3. 本轮对话想清楚的东西，是否都写进了文档？
11. 关闭任何"待查源码 / 待拍板 / 待实验"项后，必须同步更新 0.6 待办总表。

## 0.6 当前待办总表（一眼看清还差什么）

本表是全局进度的单一入口。任何项状态变化都必须同步此处。

### 待拍板（缺的是选择，不是信息——人来做）

**本栏为空。D-032 ~ D-039 已于 v0 定版中全部关闭，决策内容与强制约束见 12.6。**

出现新的待拍板项时，必须编号、写入 12.6、并在此处登记。

### 待查源码（Codex 来做）

| 编号 | 事项 | 阻塞 | 建议时机 |
| --- | --- | --- | --- |
| C-10 | Screen Outline 时机与 UV 映射 | B6 | Outline 开工前 |
| C-12 | Eyebrow DepthBias 材质侧方案 | Eyebrow | Eyebrow 开工前 |
| C-13 | Dither 与 VSM 阴影：有无可复用开关、修改代价 | Fade | Fade 开工前 |
| C-02 | MooaToon 5.7 → Epic 5.8 rebase 对账（收敛到 GI / GBuffer / Deferred） | 阶段 7 | 阶段 7 前 |
| Q1 | flags 8 位是否够用（已用 7，剩 1） | 否 | 框架冻结前 |
| Q2 | GBufferD 每通道 8 位是否长期方案 | 否 | 框架冻结前 |
| Q3 | 原生 CPD 36 float 是否够用（现用 6） | 否 | 框架冻结前 |
| Q4 | GI 最小原型只改 Composite 的判据与失败信号 | B5 | B5 前 |
| **Q5** | **5.8.2 默认 material translator profile（Classic / MIR）与开关名、默认值** | **B0 验收可信度** | **B0 开工前** |
| **Q6** | **GBuffer 各 MRT 的精确分工与格式（A/B/C/D/E 分别装什么）、Selective Outputs 在 5.8 的表现** | **否** | **B3 前** |

### 待实验（必须跑起来看，源码查不出）

| 事项 | 验收方式 | 阶段 |
| --- | --- | --- |
| KEY Light 数据链实现是否正确（方案本身为架构推断） | Debug View，验证多 View 与删除回落 | 4 |
| 8 bit FaceSDF 是否出现色阶 | Golden Scene B | 5 |
| 八面体精度下极窄头发高光是否闪烁 | 极窄高光 + 运动镜头用例 | 6 |
| Ramp 明暗硬边界在 TSR 下是否拖影 | Golden Scene F | 6 |
| Dither 在 TSR 下是否 ghosting | Golden Scene F | 10 |
| 描边在 TSR 下是否断线 / 闪烁 | Golden Scene E / F | 8 |

### 非技术前置（卡住一切）

| 事项 | 归属 |
| --- | --- |
| 从主程处取得编译好的引擎源码，本地路径与其一致 | 人 |
| 确认编译责任人、push 分支与提交批次 | 人 |
| **团队共享 DDC / 统一编译环境** | 人 |
| **规定：涉及 shader 的里程碑（M1 粉色球、M2 完整角色）必须在干净 DDC 或真 cook 下复验一次** | 人 |

> 最后两条的理由：不共享缓存时，"我这跑通了"只在同机器同缓存成立，里程碑验收结论不可复现。

---

# 第一部分：UE 框架层

> **状态：已冻结（v0 定版）。**
>
> 本部分全部属于框架契约与数据语义。所有待拍板项（D-032 ~ D-039）已关闭，
> 改动任何内容必须先走 12.7 决策变更流程。

---

# 1. 项目边界与路线

## 1.1 边界决策

| 项 | 决策 | 状态 |
| --- | --- | --- |
| UE | UE5.8 Source Fork | 已确定 |
| RHI | DX12 / SM6 | 已确定 |
| Renderer | Deferred | 已确定 |
| Toon | 自定义 ShadingModel | 已确定 |
| Substrate | 关闭 | 已确定 |
| MegaLights | 关闭 | 已确定 |
| GI | Lumen | 已确定 |
| AA / Upscale | TSR | 已确定 |
| 第一平台 | PC | 已确定 |
| 原生实验性 Toon | 不支持 | 已确定，见 1.2 |
| Forward | 不支持 | 已确定 |
| Path Tracing | 不支持 | 已确定 |
| Mobile | 后置 | 已确定 |
| 水彩 / Kuwahara / SNN | 后置 | 已确定 |

## 1.2 为什么不用 UE5.8 原生实验性 Toon

### 1.2.1 能用的部分

UE5.8 原生 Substrate Toon BSDF 提供：Ramp 漫反射与高光、自阴影控制、各向异性高光、GI 缩放、多光源参与、材质级 Toon / PBR 共存。

### 1.2.2 决定性缺口

| 缺口 | 影响 |
| --- | --- |
| 描边缺失（Silhouette / Edges 不属于当前 BSDF 定义） | 描边仍需自己实现 |
| 已知实验性问题不回移植 | 固定 5.8 也不能规避 |
| Slab 混合受限（Toon 不与其他 slabs 混合） | 不满足完整材质设计 |

### 1.2.3 源码审查补充结论（已确认）

* `MATERIAL_SHADINGMODEL_TOON` 仅在 Translator 检测到 `ESubstrateBsdfFeature::Toon` 时设置，**不是传统 `EMaterialShadingModel` 独立项**。
* GBuffer 侧 ID 为 `SHADINGMODELID_SUBSTRATE_TOON = 13`，由 Substrate BSDF 特性导出。
* `ShadingModels.ush` 中 ID 13 的 `ToonBxDF` 分支被 `#if SUBSTRATE_ENABLED` 包围。

结论：原生 Toon 属于 Substrate 体系，与我们要做的非 Substrate 自定义 Toon 是**两条完全不同的路径**。我们关闭 Substrate 后，原生 Toon 路径整体不可用。

### 1.2.4 结论

采用自定义 ShadingModel。原生 Toon 仅作效果与接口对照参考，**不得复用其任何宏、ID、BxDF 或 pack / unpack 函数**。

## 1.3 第一版成功定义（按执行顺序）

1. 角色进入普通 UE5.8 Lumen 场景
2. Custom Toon ShadingModel 正常
3. Base、ILM、Ramp 正常
4. 方向光产生稳定 Toon 明暗
5. Face SDF 正常
6. Point 与 Spot 只补亮
7. VSM 正常
8. Hair、Eye、Rim 正常
9. 角色 GI 可独立调节
10. Screen Outline 稳定
11. TSR 与相机运动无阻塞级闪烁

---

# 2. 引擎改动施工图

本节是整个框架层的核心，所有改动先在此汇总。定位一律使用**符号名**，不使用行号（行号会随 rebase 漂移）。

## 2.1 改动总览

| 编号 | 改动 | 施工类型 | 状态 |
| --- | --- | --- | --- |
| E-1 | Toon ShadingModel 注册链 | Engine rebuild | 必须，10 个文件 |
| E-2 | CustomData 白名单 | Engine rebuild + shader | 必须，**5 处** |
| E-3 | Material Toon 输入引脚 | Engine rebuild | 必须，8 处 |
| E-4 | GBuffer / CustomData 配置链 | 无源码修改 | 只做审核验证 |
| E-5 | KEY Light 进入 View Uniform | Engine rebuild | 必须，4 处 |
| S-1 | BasePass CustomData 编解码 | shader-only | 必须 |
| S-2 | Direct Toon Lighting | shader-only（固定粉色）/ Engine rebuild（正式 Ramp） | 必须 |
| S-3 | GI Toon 化 | shader-only | 必须 |

### 后置或可选

| 改动 | 状态 | 原因 |
| --- | --- | --- |
| GI 分流 C++ 路径（`IndirectLightRendering.cpp`） | 后置 | Composite 单点已足够（D-026） |
| `LumenScreenProbeGather.usf` | 后置 | 原型阶段无证据需要 |
| `PrimitiveSceneProxy` / Desc 扩展 HeadBasis | 后置 | CPD 已确认足够（36 float） |
| `ToonShadingCommon.ush` / `ToonShadingModel.ush` | 可选 | 只是代码组织方式。**但建议第一版就建公共 helper**，见 2.5.6 |
| Geometry Outline MeshPass | 后置 | P2 |
| Geometry HairShadow Pass | 后置 | P2 |
| Anisotropy 路径研究 | 后置 | P2 |

## 2.2 E-1 ShadingModel 注册链（10 个文件）

| 文件 | 函数 / Struct / 宏 | 插入位置 | 要改什么 | 不做的后果 |
| --- | --- | --- | --- | --- |
| `EngineTypes.h` | `EMaterialShadingModel` | `MSM_Strata` 之后、`MSM_NUM` 之前 | 加入 `MSM_CustomToon` | 材质系统无法选择 |
| `MaterialShader.cpp` | `GetShadingModelString` | `MSM_Strata` 分支之后 | 返回稳定名称 | 日志 / 统计显示 Unknown |
| `HLSLMaterialTranslator.cpp` | `FHLSLMaterialTranslator::ShadingModel` | 原生 Toon / Strata 分支附近 | **显式翻译为 packed ID 14** | Classic 生成错误 SMID |
| `HLSLMaterialTranslator.cpp` | `FHLSLMaterialTranslator::GetMaterialEnvironment` | `MATERIAL_SHADINGMODEL_TOON` 等 define 处 | 设置 `MATERIAL_SHADINGMODEL_CUSTOM_TOON` | 白名单与 shader 分支识别不到 |
| `MaterialIRModuleBuilder.cpp` | `MP_ShadingModel` 固定值路径 | `ConstantInt(GetFirstShadingModel())` 处 | **对 CustomToon 显式写 14** | MIR 固定 SM 材质 SMID 错 |
| `MaterialExpressionsToMIR.cpp` | `UMaterialExpressionShadingModel::Build`、局部 `FShadingModel::ToHLSL` | 现有 SM 映射 switch 内 | **对 CustomToon 显式写 14** | 动态 SM 表达式输出错误 ID |
| `MaterialIRToHLSLTranslator.cpp` | `GetShadingModelParameterName` | 现有参数名 switch 内 | 返回 `MATERIAL_SHADINGMODEL_CUSTOM_TOON` | MIR 材质缺 define，进 `UE_MIR_UNREACHABLE` |
| `ShaderMaterial.h` | `FShaderMaterialPropertyDefines` | `MATERIAL_SHADINGMODEL_TOON` 位附近 | 新增独立 CustomToon 位 | C++ 环境无法保存该 define |
| `ShaderGenerationUtil.cpp` | `ApplyFetchEnvironmentInternal` | `FETCH_COMPILE_BOOL` SM 列表内 | 取回 CustomToon define | DDC 恢复时 define 丢失 |
| `ShadingCommon.ush` | `SHADINGMODELID_*`、`SHADINGMODELID_NUM` | `SUBSTRATE_TOON 13` 之后 | 定义 14、NUM=15，15 保留 | BasePass / Deferred 解释不一致 |

## 2.3 E-2 CustomData 白名单（五处，缺一即静默失败）

| 文件 | 符号 | 层 |
| --- | --- | --- |
| `BasePassCommon.ush` | `WRITES_CUSTOMDATA_TO_GBUFFER` | shader 写入 |
| `ShaderMaterialDerivedHelpers.cpp` | `CalculateDerivedMaterialParameters` | C++ 派生 |
| `ShaderGenerationUtil.cpp` | `DetermineUsedMaterialSlots` | C++ slot 分析 |
| `ShaderGenerationUtil.cpp` | `SetSlotsForShadingModelType` | C++ SM slot |
| `DeferredShadingCommon.ush` | `HasCustomGBufferData` | shader 解码 |

后两处同属 `ShaderGenerationUtil.cpp`，但**函数不同，必须分别改**。

任一处缺失的表现都是"GBufferD 为零或未分配"，**且不报 shader 错误**——这是全套改动中最隐蔽的坑。

## 2.4 E-3 / E-4 / E-5 施工图

### E-3 Material Input（8 处）

| 文件 | 符号 | 要改什么 |
| --- | --- | --- |
| `SceneTypes.h` | `EMaterialProperty` | 新增 Float4 `MP_ToonData`（普通属性末尾、`MP_MaterialAttributes` 之前），同步 `MP_MAX` 断言 |
| `MaterialAttributeDefinitionMap.cpp` | `InitializeAttributeMap` | 注册 GUID、Float4 类型、默认值、显示名 |
| `Material.cpp` | `IsPropertyActive_Internal` | `MP_ToonData` 仅对 CustomToon 可见；`MP_Tangent` 对 CustomToon 启用 |
| `HLSLMaterialTranslator.cpp` | 构造函数 `SharedPixelProperties`、`TranslateMaterial` | 注册并编译 `MP_ToonData` |
| `MaterialAggregate.cpp` | `FAttributePropertyIndexMap` 构造 | `PushAttribute(MP_ToonData)`，MIR 属性遍历基准 |
| `MaterialExpressions.cpp` | `UMaterialExpressionMakeMaterialAttributes::{GetExpressionInput,GetConnectedInputs,Compile,CompilePreview}`；`UMaterialExpressionBreakMaterialAttributes::BuildPropertyToIOIndexMap` | 增加 ToonData 输入 / 输出映射，更新 `MP_MAX` 静态断言 |
| `MaterialExpressionsToMIR.cpp` | `UMaterialExpressionMakeMaterialAttributes::Build` | 加入 ToonData assignment |
| `Material.h` / `MaterialShared.cpp` | `UMaterialEditorOnlyData`、Material Input 编译函数 | ToonData 必须保持 Float4 |

**硬约束**：不得用 `FColorMaterialInput` 承载 ToonData，其描述路径为 Float3，会丢 alpha。

### E-4 GBuffer

**不修改 GBuffer 格式或 MRT 数量**（已确认）。复用现有 `GBS_CustomData`、GBufferD 与 SceneTexture 绑定。实际写入只在 `ShadingModelsMaterial.ush::SetGBufferForShadingModel` 的 ID14 分支完成。

仅需做审核验证，无源码修改。

### E-5 KEY Light 通道（4 处）

| 文件 | 符号 | 要改什么 |
| --- | --- | --- |
| `SceneView.h` | `FSceneView` | 增加 `FVector4f ToonKeyLightDirectionAndValid`，每 View 独立 |
| `SceneView.h` | `VIEW_UNIFORM_BUFFER_MEMBER_TABLE` | 同名 Float4 uniform 字段，原生 `DirectionalLightDirection` 附近 |
| `SceneRendering.cpp` | `FViewInfo::SetupUniformBufferParameters` | 无条件复制该字段 |
| 项目模块 | `ISceneViewExtension::SetupView` | 每次回调先写无效默认值，再查 Key Light；删除 / 替换时回落 |

## 2.5 S-1 / S-2 / S-3 施工图

| 编号 | 文件:符号 | 要改什么 |
| --- | --- | --- |
| S-1 | `ShadingModelsMaterial.ush::SetGBufferForShadingModel` | ID14 分支内按契约填 `GBuffer.CustomData`（R=flags，GB=oct tangent，A=双语义） |
| S-1 | `DeferredShadingCommon.ush::EncodeGBuffer` / `DecodeGBufferData` | 不改格式，只保证白名单启用；Toon 位解析放入共享 helper |
| S-2 | `ShadingModels.ush::IntegrateBxDF` | 显式 `case SHADINGMODELID_CUSTOM_TOON`，调用独立 `CustomToonBxDF`，置于原生 Toon 前且不受 `SUBSTRATE_ENABLED` 包围 |
| S-3 | `DiffuseIndirectComposite.usf::MainPS` | Legacy GBuffer 分支内显式分流 ID14，不复用 ID13 分支 |

### 2.5.1 关于公共 shader helper 的建议

`ToonShadingCommon.ush` 本身是代码组织方式，非功能前提。但**编码 / 解码 / 位解析必须集中在共享 helper**（无论叫不叫这个名字），理由是：不同 shader 各写一套 floor / 截断 / 不同 epsilon，会产生舍入漂移，而这种漂移不报错、只在某些像素上出错，极难排查。

**建议第一版就建立**，哪怕只放几个函数签名。这不是"空壳类"（假装实现了功能），而是公共头文件（结构性的，晚建则所有已写 shader 都要回头加 include）。

## 2.6 提交批次

| 批次 | 内容 | 施工类型 | 验证方法 |
| --- | --- | --- | --- |
| **B0 粉色球** | E-1 全链 + E-2 全白名单 + 最小 S-1 写入 + S-2 ID14 固定粉色分支 | Engine rebuild | Opaque 与真实 Masked 均粉色；SMID=14 非 13；GBufferD 非零；DefaultLit 不变 |
| B1 材质数据契约 | E-3 全部 8 处 | Engine rebuild | 直接根输入与 Make Material Attributes 一致；Classic / MIR 一致；四分量互异值完整到达 BasePass |
| B2 KEY Light | E-5 Engine View/UB + 项目 SVE | Engine rebuild | 多 View 不同方向；删除 / 替换后 w=0；无上一帧残留 |
| B3 完整 CustomData | S-1 byte / oct pack-unpack、A 双语义 | shader-only | RGBA 字节 round-trip；Opaque / Masked 一致；Face / non-Face 分流正确 |
| B4 正式直接光 | `CustomToonBxDF` + Ramp atlas 绑定 | Engine rebuild | Directional / Point / Spot、VSM、Ramp 行中心采样、Hair / Eye tangent 对照 |
| B5 GI | S-3 | shader-only | Lumen 开 / 关；只改变 ID14 |
| B6 Outline | 项目 RDG pass、Post shader、SMID14 / GBufferD 绑定 | project-module | ScreenPercentage、分屏、VR、动态分辨率；DefaultLit 负对照 |

施工类型标签：

| 标签 | 含义 | 提交策略 |
| --- | --- | --- |
| Engine rebuild | 需主程重新编译引擎 | 批量提交，减少编译次数 |
| Shader-only | 只改 shader，可热重载 | 可本地快速迭代 |
| Project-module | 只动项目代码 | 项目自行编译 |

## 2.7 B0：最快看到粉色球的最小批次（当前可开工）

**必须包含**：

* `MSM_CustomToon` 枚举
* Classic 与 MIR 的 ID14 映射（四个位置全部）
* `MATERIAL_SHADINGMODEL_CUSTOM_TOON` define 及其 C++ 镜像
* C++ 与 shader 全部 CustomData 白名单（五处）
* `SetGBufferForShadingModel` 写 ID14 与固定非零测试值
* `IntegrateBxDF` 独立 ID14 case，调用固定粉色 `CustomToonBxDF`

**明确排除**：新材质引脚（E-3）、FaceSDF、KEY Light（E-5）、Ramp texture、GI、Outline。

**构建成本**：一次 Engine rebuild + 相关 shader 编译。

**验收**：Opaque 球与 Masked 球为粉色；DefaultLit 不变；SMID=14；未进入 ID13 原生 Toon。

**Masked 验收陷阱**：必须使用**非恒定 OpacityMask**（如棋盘纹理）。若 `WritesEveryPixel()==true`，Masked 会被编译成 Solid，形成"Masked 已验证"的假象。

---

# 3. 整帧时序与 Pass 挂点

本节只规定执行顺序与挂点，不涉及任何算法。

## 3.1 整帧 FrameGraph

1. Depth / Visibility
2. BasePass（写 BaseColor、Normal、ToonSMID、CustomData）
3. VSM / Shadows
4. Lumen GI 求解
5. Direct Toon Lighting（KEY、FILL、Ramp、Face、Hair / Eye / Rim）
6. Toon Indirect Composite
7. SceneColor HDR
8. Screen Outline
9. Overlay / Translucency
10. HDR Toon Post
11. TSR / Tonemap
12. Display Post
13. Screen

## 3.2 光照 Lane 与真实 Renderer 路径

* KEY 与 FILL 属于 Direct Lighting。
* CHARACTER INDIRECT 属于 Lumen / Diffuse Indirect。
* 三者属于统一艺术模型，但**不是同一个 Pass**。
* 不得为了概念统一把三条 Lane 强塞进单个 `ShadingModels.ush` 函数。

## 3.3 Post FrameGraph 顺序

1. HDR SceneColor
2. Optional SNN / Kuwahara（滤波）
3. **Screen Outline**
4. Watercolor Pigment
5. Toon Bloom
6. TSR / Tonemap / ColorGrade
7. Paper / Grain / Vignette
8. Display

**顺序依据（D-038，已确定）**：滤波**只修改颜色，不修改深度与法线**；Screen Outline 依赖深度 / 法线 / Mask 做边缘检测，因此先滤波**不影响描边定位**。

收益：描边画在已柔化的色块之上，线保持锐利——这正是二游的观感（色块柔、线要利）。代价为零，仅调整顺序，不加 mask、不加仲裁权重。

Screen Outline 固定在 TSR 之前，理由见 3.4。该约束与"先滤波后描边"不冲突——滤波与描边都在 TSR 之前。

## 3.4 Screen Outline 时机约束（已确认）

TSR 与上采样之后，SceneColor 与原始 GBuffer 可能不再同分辨率，直接共用像素坐标会产生错位。

因此：

* Screen Outline 优先放在 **TSR 之前**。
* 必须显式处理 `ViewRect`、`BufferSize` 与 UV 映射。
* 详细坐标换算规则由 C-10 审查确认后补充。

## 3.5 时序冲突登记（本版新增）

时序属于框架层——因为**挪一个 Pass 的位置可能比加参数更省事，也可能推翻整个方案**。以下为已识别的时序冲突：

| 编号 | 冲突 | 现状 | 状态 |
| --- | --- | --- | --- |
| **D-038** | **描边 × 后处理滤波** | **已确定：先滤波，后描边**（3.3 顺序已改）。滤波只动颜色、不动深度/法线，故先滤波不影响描边定位；线压在柔化后的色块上更锐利，符合二游观感 | **已确定** |
| C2 | **Ramp 明暗硬边界 × TSR** | 与"描边 × TSR"是**两个独立问题**：描边是线、这个是面 | 待实验，见 0.6 |
| — | 描边 × TSR | 已通过"放 TSR 前"规避 | 已确认 |
| — | Paper / Grain × TSR | 已通过"放 TSR 后"规避 | 已确认 |
| — | 滤波 × TSR | 滤波在 TSR 前，其人造边缘是否 ghosting 未验证 | 待实验 |
| D-039 | 描边 × Dither Fade | 淡出时描边是否同步消失 | **已确定：接受**（按 D-039 政策） |

## 3.6 RDG Pass Contract

| Pass | 输入 | 输出 | 类型 | 生命周期 |
| --- | --- | --- | --- | --- |
| ToonScreenOutline | Depth / Normal / Mask | SceneColor | RDG Fullscreen | Frame |
| ToonBloomDownsample | HDR Color | BloomMip | RDG Compute | Frame |
| ToonBloomUpsample | BloomMip | Bloom | RDG Compute | Frame |
| WatercolorPigment | HDR Color | WC Color | RDG Compute | Frame |
| PaperComposite | DisplayColor | FinalColor | RDG Fullscreen | Frame |
| GeometryOutline | Mesh | SceneColor | MeshPass | Frame |
| HairShadowMask | Hair Mesh | R8 Mask | MeshPass | 后置 |

规则：`Resource → Pass = Read`，`Pass → Resource = Write`。临时 RT / Buffer 生命周期交给 RDG。

## 3.7 后处理 Pass 强制契约

每个 Post Pass 必须声明：

| 字段 | 示例 |
| --- | --- |
| Input | HDR SceneColor |
| Output | StylizedColor |
| Resolution | Full / Half / Quarter |
| Color Space | Pre-exposed HDR / Scene-referred / Display |
| History | Yes / No |
| Temporal Sensitive | Yes / No |
| Exposure Dependency | Yes / No |

## 3.8 SceneViewExtension 签名约束（已确认）

5.8 有效签名：

```text
SubscribeToPostProcessingPass(Pass, const FSceneView&, Delegates, Enabled)
```

**关键警告**：旧的无 `FSceneView` 重载自 5.5 起已弃用，**不会被调用，且不报错**。使用旧签名会导致后处理看起来完全没生效，且极难排查。第一版必须锁定新签名（D-028）。

---

# 4. BasePass 数据契约

本节属于框架契约，改动代价最高。任何位分配变动都会同时影响 BasePass、Deferred、GI、Outline、Debug。

## 4.1 CustomData 位分配（已确认）

GBufferD 格式为 `PF_B8G8R8A8`（`GBT_Unorm_8_8_8_8`），CustomData 为 RGBA 各 8 位 UNorm，单通道精度约 `1/255`。

| 通道 | 内容 | 消费方 |
| --- | --- | --- |
| R | 8 个 Toon Flags | Direct Lighting / Outline |
| G | Oct Tangent X | Hair / Eye |
| B | Oct Tangent Y | Hair / Eye |
| A | IsFace 时为 FaceSDFResult | Lighting |
| A | 非 Face 时低 4 位 **RampRowID（首版保留、固定填 0）** | Lighting |
| A | 非 Face 时高 4 位 MaterialType | Lighting |

### 4.1.1 强制约束：RampRowID 位保留（D-034 附带，不可违反）

首版采用单 Ramp（D-034），但 **A 通道低 4 位的 RampRowID 必须保留，第一版固定填 0，禁止腾作他用。**

理由（这是成本决策，不是洁癖）：

| | 删掉这 4 位 | 保留并填 0 |
| --- | --- | --- |
| 第一版成本 | 一样 | 一样 |
| 将来加多行色阶 | **A 级**：位语义变更，BasePass / Deferred / GI / Outline / Debug 全部返工，已产出资产作废 | **C 级**：多画几行 ramp + 改 `RowCount` 常量 |

首版选单 Ramp 的原因是"当前无美术、TA 为唯一决策人"，**不是因为论证过单行够用**。美术进场后第一个大概率被挑战的就是"皮肤、头发、衣服暗部颜色凭什么一样"（二游中皮肤暗部偏暖红、头发暗部偏冷紫是标配）。

配套要求：

* Ramp 仍以**贴图资源**形式存在，`RowCount = 1`，V 采样固定 0.5（复用 4.1 的 `(RowID + 0.5) / RowCount` 寻址，一行都不用改）。
* **禁止**把 ramp 退化成 shader 里的常量数组——那样将来加行要改架构。
* 违反此约束（把该位挪作他用）视同修改框架契约，必须走 12.7 决策变更流程。

## 4.2 Flags 位定义

| 位 | 含义 |
| --- | --- |
| 0 | IsFace |
| 1 | IsHair |
| 2 | IsEye |
| 3 | UseOutline |
| 4 | UseHairHighlight |
| 5 | UseEyeHighlight |
| 6 | UseRim |
| 7 | Reserved（**仅剩 1 位**，见 Q1） |

## 4.3 CustomData.A 双语义（已确定）

```text
if IsFace:
    A = 完整 8 bit FaceSDFResult
else:
    A 低 4 bit = RampRowID
    A 高 4 bit = MaterialType / Aux
```

代价：Face SDF 只由 KEY 决定，Point / Spot 不重新翻转脸部阴影（D-009）。

收益：不新增额外 RT、不增加 GBuffer 带宽、非 Face 分支额外获得 16 个 MaterialType 值，支撑"单 SM 多材质类型"而不占额外 SMID（D-019）。

## 4.4 UNorm8 编解码协议（已确认）

所有离散数据必须先恢复为完整 byte 再做位运算。**禁止直接在归一化 float 上测试单个 bit。**

```text
连续值编码：q = floor(saturate(x) * 255.0 + 0.5)，写出 q / 255.0
整数字节编码：整数限制到 [0,255] 后写出 q / 255.0
解码：q = floor(saturate(encoded) * 255.0 + 0.5)
```

关键约束：

* 量化步长 `1/255`，最近舍入最大误差 `0.5/255`。
* Flags 与 ID 的 GBuffer 读取必须使用 point / load 语义，**不得线性过滤**——线性过滤会混合相邻像素的 flags，任何阈值都无法可靠恢复。
* 编解码必须集中为共享函数，禁止不同 shader 各自使用 floor、截断或不同 epsilon。

八面体编码采用 UE 原生函数：

```text
编码：UnitVectorToOctahedron(normalize(V)) * 0.5 + 0.5
解码：OctahedronToUnitVector(Encoded * 2.0 - 1.0)
```

5.8 源码给出的 oct 8:8 误差为平均 `0.33709`、最大 `0.94424`，Hair / Eye 已使用同类编码。首版接受该精度，但必须加入极窄头发高光与运动镜头的视觉回归用例（待实验）。

FaceSDF 精度：

* 使用全部 256 级。
* 最终明暗分界必须在读取后完成，保留至少数个量化级宽度的 feather。
* 若出现色阶，优先用阈值区间重映射或对象 / UV 稳定抖动，**不扩大 GBuffer**。
* 禁止在 BasePass 提前把 FaceSDF 压成单个布尔明暗结果。

RampRowID 寻址：

* 解码后必须作为整数使用。
* Ramp 纹理 V 坐标按 `(RowID + 0.5) / RowCount` 定位行中心。
* 禁止直接把 A 的归一化值当作 Ramp UV（会因高四位复用而寻址错误）。

## 4.5 CustomData 白名单（硬性步骤，已确认）

注册新 ShadingModel **不会自动获得 CustomData 写入权**。`WRITES_CUSTOMDATA_TO_GBUFFER` 与 `HasCustomGBufferData` 都是显式白名单。

不加的后果：

```text
SM 注册成功
材质面板也能选
但 CustomData 永远为 0
```

**且不报错。** 画面就是不对，且很难查——因为代码路径全对，只是数据没写进去。

## 4.6 GBuffer 修改规则

任何 GBuffer 格式改动必须同步检查 8 处：`GBufferInfo`、`SceneTexturesConfig`、`SceneTextures`、Shader GBuffer Encode / Decode、BasePass、Deferred Decode、Pixel Inspector、Debug View。

MooaToon 5.7 已修改：`GBufferInfo.cpp/.h`、`GBufferHelpers.ush`、`SceneTexturesConfig.cpp/.h`、`SceneTextures.cpp`、`BasePassPixelShader.usf`、`BasePassCommon.ush`。

说明：`GBufferHelpers.ush` 在 5.8 属重构路径检查点，非 Substrate 经典路径的直接实现仍在 `DeferredShadingCommon.ush`。所有 MooaToon 参考均需 rebase 到 5.8。

## 4.7 CustomData producer → consumer 全链（已确认）

```text
Material Texture / Vertex Data
→ Material Graph
→ MP_ToonData + MP_Tangent
→ Classic / MIR Material Translation
→ FPixelMaterialInputs
→ SetGBufferForShadingModel
→ FGBufferData.CustomData
→ EncodeGBuffer
→ GBufferD
→ SceneTexturesStruct.GBufferDTexture
→ DecodeGBufferData
→ IntegrateBxDF / CustomToonBxDF
→ DiffuseIndirectComposite / Post Outline
```

| 段 | Producer | Consumer | 数据形态 |
| --- | --- | --- | --- |
| 纹理 / 顶点 → 材质图 | Texture Sample、Vertex Color | 材质根输入或 Make Material Attributes | ILM RGBA、FaceSDF sample、VertexColor |
| 材质图 → ToonData | 材质图约定 | `MP_ToonData`、既有 `MP_Tangent` | Float4 + Float3 |
| Classic Translation | `HLSLMaterialTranslator::TranslateMaterial` | `FPixelMaterialInputs` / property getter | 编译后表达式 |
| MIR Translation | `MaterialAggregate.cpp`、`MaterialExpressionsToMIR.cpp` | `MaterialIRModuleBuilder` | MIR attribute → PixelMaterialInputs |
| → BasePass | `CalcMaterialParameters` | `SetGBufferForShadingModel` | ToonData、WorldTangent、CPD、View KeyDir |
| → GBuffer struct | ID14 分支 | `EncodeGBuffer` | `FGBufferData.CustomData = float4` |
| → GBufferD | `EncodeGBuffer` | MRT GBufferD | `PF_B8G8R8A8` |
| → SceneTexture | BasePass MRT | `FSceneTextureUniformParameters::GBufferDTexture` | RDG texture |
| → Deferred decode | `SceneTexturesStruct.GBufferDTexture` | `DecodeGBufferData` | 整数像素 `.Load()` 后 normalized float4 |
| → Lighting | `DeferredLightPixelMain` | `IntegrateBxDF` → `CustomToonBxDF` | CustomData + ID14 |
| → GI | `DiffuseIndirectComposite.usf::MainPS` | ID14 GI 分支 | 已解码 GBuffer / SMID |
| → Post / Outline | `FSceneTextureShaderParameters.SceneTextures` | 项目 outline shader | `SceneTexturesStruct.GBufferDTexture.Load()` |

### 强制接口结论

| 问题 | 结论 |
| --- | --- |
| ILM 四通道经哪个引脚进入 | 5.8 无原生 ILM 引脚。推荐 `MP_ToonData` + 既有标准属性；ILM.G/B 由 CustomToon 重解释 Metallic / Specular（见 D-033） |
| FaceSDF 经哪个引脚进入 | 不传纹理资源，材质 / BasePass 采样后只把最终标量送入（见 D-035） |
| Hair / Eye tangent | 复用 `MP_Tangent` → `MaterialParameters.WorldTangent`，需把 CustomToon 加入属性活跃门控 |
| VertexColor.G | 必须在材质图内先与 ILM shadow / ramp 控制合成，GBuffer 之后取不到原始值 |
| **VertexColor.A** | **当前契约没有存储，Post 无法取得逐顶点描边宽度**（见 D-036） |
| SceneTexture 名称 | C++：`FSceneTextureUniformParameters::GBufferDTexture`；HLSL：`SceneTexturesStruct.GBufferDTexture` |
| `FSceneTextureShaderParameters` 成员 | 成员是 `SceneTextures`，GBufferD 位于嵌套的 Scene Texture uniform 中 |
| Lighting 谁消费 | `IntegrateBxDF` 的 ID14 case 调用 `CustomToonBxDF` |
| Ramp atlas 谁提供 | RowID 只是索引，正式版还需 Deferred Light pass 参数绑定（见 D-034） |
| Post 如何安全读取 | pass 请求 GBufferD，shader 用整数像素 `.Load()`，并以 GBufferB 中 SMID==14 为前置条件 |

### 静默失败点（不报错，但画面错）

| 静默失败点 | 检查方法 |
| --- | --- |
| ToonData 未加入 `IsPropertyActive_Internal` | Material Editor 检查引脚可见性 |
| 只改 Classic 未改 MIR | 分别强制验证 Classic 与 MIR 生成结果 |
| 漏 `MaterialAggregate` | MIR debug 输出确认属性进入 aggregate |
| 漏 Make / Break Attributes | 同材质分别用直接输入与 Make Attributes 比较 |
| 用 `FColorMaterialInput` 承载 ToonData | 给四通道写互异测试值并在 BasePass 可视化 |
| 把 FaceSDF 纹理当可传递属性 | 确认传递的是最终标量 |
| ILM.G 放进 AO | 开 / 关 `ALLOW_STATIC_LIGHTING` 比较解码值 |
| 漏任一 CustomData 白名单 | Buffer Visualization 检查 GBufferD |
| 未做 byte round-trip | 写入 0、1、127、128、254、255 逐值回读 |
| A 未先按 IsFace 分流 | 分别测试 Face / non-Face 且使用相同 A 字节 |
| Post 使用过滤采样 | 确认使用 `.Load()` |
| Post 坐标未映射到 SceneTexture extent | 非 100% ScreenPercentage、分屏、VR 验证 |
| Pass 未请求 GBufferD | RDG capture 检查绑定资源 |
| Post 不检查 SMID14 | 场景放 DefaultLit 与 ID13 材质做负对照 |
| Key Light valid 未检查 | 运行时删除和替换 Key Light |
| GI 无 ID14 分流 | Lumen 开 / 关；CustomToon 与 DefaultLit 对照 |

---

# 5. ShadingModel 与 Permutation

## 5.1 编号分配（已确认）

| 项 | 值 |
| --- | --- |
| C++ 枚举 | `MSM_CustomToon`，追加在 `MSM_Strata` 与 `MSM_NUM` 之间 |
| GBuffer / HLSL ID | `SHADINGMODELID_CUSTOM_TOON = 14` |
| `SHADINGMODELID_NUM` | 更新为 `15` |
| ID 15 | 保留，不分配给项目 Shading Model |

* C++ 枚举值与 GBuffer ID **不要求相等**，必须显式映射，禁止依赖枚举序号自动映射。
* `SHADINGMODELID_MASK = 0xF`，仅 4 位。ID 15 等于全 1 值，源码诊断会把 `NUM` 与 `MASK` 作为特殊值分别显示，因此保留不用。

## 5.2 最高风险断点（已确认）

`MSM_CustomToon` 追加在 `MSM_Strata`(12) 与 `MSM_NUM`(13) 之间，**其 C++ 枚举自然值就是 13**——而 13 正是原生 `SHADINGMODELID_SUBSTRATE_TOON`。

UE 5.8 的 Classic Translator 与 MIR Translator 都存在**直接把 `EMaterialShadingModel` 数值输出为 HLSL 常量**的路径。若不显式映射，所有 Custom Toon 像素会被写成 ID 13，直接命中原生 Substrate Toon 分支。

必须在以下**四个位置**全部显式映射到 packed ID 14：

| 路径 | 位置 |
| --- | --- |
| Classic 固定 SM | `HLSLMaterialTranslator.cpp:FHLSLMaterialTranslator::ShadingModel` |
| MIR 固定下拉框 | `MaterialIRModuleBuilder.cpp`：`MP_ShadingModel` 的 `Emitter.ConstantInt(...)` |
| MIR 动态 SM 表达式 | `MaterialExpressionsToMIR.cpp:UMaterialExpressionShadingModel::Build`，局部 `FShadingModel::ToHLSL` |
| HLSL 侧 | `ShadingCommon.ush:SHADINGMODELID_CUSTOM_TOON` |

漏掉任意一个，都会出现"部分材质写 14、部分写 13"的隐蔽不一致。

## 5.3 与原生 Substrate Toon 的隔离（已确认）

**不得复用**：`MATERIAL_SHADINGMODEL_TOON`、`SHADINGMODELID_SUBSTRATE_TOON`、`ESubstrateBsdfFeature::Toon`、`ToonBxDF`、`PackToonCustomData` / `UnpackToonCustomData`、`COMPLEXPATH_MODE_TOON`、`SUBSTRATE_OUTPUT_TOON_DATA`、`SUBSTRATE_LIGHTPASS_APPLIES_TOON_DIFFUSE_RAMP`、`FDeferredLightApplyToonDiffusePS`、`SUBSTRATE_EXPERIMENTAL_TOON_*`。

**项目使用独立符号**：`MSM_CustomToon`、`MATERIAL_SHADINGMODEL_CUSTOM_TOON`、`SHADINGMODELID_CUSTOM_TOON`、`CustomToonBxDF`、`PackCustomToonData` / `UnpackCustomToonData`。

隔离原则：

* 所有 Custom Toon 判断必须**显式比较 ID 14**。
* 即使关闭 Substrate，也**不能依赖 `SUBSTRATE_ENABLED=0` 作为唯一隔离手段**（D-030）——5.8 仍有部分未受该宏保护的 ID 13 检查，分布在 CustomData、transmission、shadow、Lumen 与 GI 判断中。
* 只要始终写 ID 14 并保持独立命名，就不会误入 ID 13 分支。

## 5.4 单 SM 多材质类型（已确认）

Hair、Eye、ClearCoat 都是原生"一个 SMID + CustomData / 功能位"的先例：Eye 在 `CustomData.yz` 存八面体法线并以 `IRIS_NORMAL` 控制编译路径；ClearCoat 由 `CLEAR_COAT_BOTTOM_NORMAL` 控制变体。

因此"一个 Toon SM + GBuffer flags / MaterialType"符合原生模式，不需要为 Face / Hair / Eye 分配多个 SMID（D-019）。

**关键约束**：Deferred Lighting 是读取 GBuffer 的 Global Shader，**不会继承某个材质的 Static Switch define**。因此 Deferred 所需的类型信息必须显式写入 GBuffer，不能依赖材质静态分支。

## 5.5 Permutation 全链（已确认，D-022 闭合）

**状态：已确认，无剩余源码架构阻塞点。阶段 1 可开工。**

### 5.5.1 完整链路

```text
MSM_CustomToon
├─ Classic Translator
│  ├─ FHLSLMaterialTranslator::PrepareEnvironmentDefines
│  ├─ FHLSLMaterialTranslator::GetMaterialEnvironment
│  └─ FHLSLMaterialTranslator::ShadingModel：C++ enum → packed ID 14
├─ MIR Translator
│  ├─ GetShadingModelParameterName
│  ├─ FMaterialIRToHLSLTranslation::Run
│  ├─ MaterialIRModuleBuilder 固定 SM：C++ enum → packed ID 14
│  └─ UMaterialExpressionShadingModel::Build：动态 SM → packed ID 14
↓
MATERIAL_SHADINGMODEL_CUSTOM_TOON
↓
FShaderMaterialPropertyDefines
↓
ApplyFetchEnvironmentInternal
↓
CalculateDerivedMaterialParameters
↓
BasePass
├─ MATERIALBLENDING_SOLID
└─ MATERIALBLENDING_MASKED
↓
WRITES_CUSTOMDATA_TO_GBUFFER
↓
SetGBufferForShadingModel
↓
GBuffer.ShadingModelID = 14 / GBuffer.CustomData = Toon RGBA8
↓
HasCustomGBufferData(14)
↓
IntegrateBxDF
↓
case SHADINGMODELID_CUSTOM_TOON → CustomToonBxDF
```

### 5.5.2 Classic Translator 必改

* `PrepareEnvironmentDefines`：确认枚举位被 `for (i < MSM_NUM)` 自动收集。
* `GetMaterialEnvironment`：增加 `HasShadingModel(MSM_CustomToon)` → `MATERIAL_SHADINGMODEL_CUSTOM_TOON=1`。
* `ShadingModel`：**显式映射到 packed ID 14**（见 5.2）。

### 5.5.3 MIR Translator 必改

只改 Classic 不算完成。必须同时覆盖：

* `GetShadingModelParameterName`：缺 case 会进入 `UE_MIR_UNREACHABLE`，MIR 材质翻译直接失败。
* `FMaterialIRToHLSLTranslation::Run`：现有 `for (i < MSM_NUM)` 自动遍历。
* `MaterialIRModuleBuilder.cpp`：固定 `MP_ShadingModel` 的 `Emitter.ConstantInt(...)` 必须输出 14。
* `MaterialExpressionsToMIR.cpp`：`FShadingModel::ToHLSL` 当前输出 `(uint32)Id`，必须输出 14。

### 5.5.4 Shader Compile Environment 镜像

必须同步修改，否则 HLSL 有 define 但派生系统认为 Toon=false：

* `ShaderMaterial.h:FShaderMaterialPropertyDefines`：增加 `MATERIAL_SHADINGMODEL_CUSTOM_TOON` 位。
* `ShaderGenerationUtil.cpp:ApplyFetchEnvironmentInternal`：增加 `FETCH_COMPILE_BOOL(...)`。
* `ShaderMaterialDerivedHelpers.cpp:CalculateDerivedMaterialParameters`：加入 `WRITES_CUSTOMDATA_TO_GBUFFER`。
* `ShaderGenerationUtil.cpp` GBuffer slot 推导：为 Custom Toon 调用 `SetStandardGBufferSlots(...)`，将 `GBS_CustomData` 标为 Written。

### 5.5.5 Opaque 与 Masked（已确认）

**没有** Toon 专用的 Opaque/Masked `FPermutationDomain`，也不需要分别写 Toon 分支。

* 两者由 `FMaterial::SetupMaterialEnvironment` 生成 `MATERIALBLENDING_SOLID` / `MATERIALBLENDING_MASKED`。
* 共用 `TBasePassPS`、`BasePassPixelShader.usf:MainPS` 与 `SetGBufferForShadingModel`。
* Masked 先执行 `GetMaterialCoverageAndClipping` 的 clip/coverage，存活像素再进入相同的 GBuffer 写入路径。

**Masked 验收陷阱**：`WritesEveryPixel()==true` 的 Masked 材质会被编译为 Solid，形成"Masked 已验证"的假象。验收必须使用非恒定 OpacityMask。

### 5.5.6 ShouldCompilePermutation 结论（已确认）

以下门控**无需**为 Custom Toon 增加特判（Custom Toon 是 Lit SM，自动满足 `IsLit()`）：

* `TBasePassVertexShaderBaseType / TBasePassVS / TBasePassMS / TBasePassPixelShaderBaseType / TBasePassPS::ShouldCompilePermutation`
* LightMap policy `ShouldCompilePermutation`
* `FDeferredLightPS::ShouldCompilePermutation`

**禁止**新增 Custom Toon 的 permutation bool——Deferred Lighting 是 Global Shader，在运行时按 GBuffer SMID 分支。错误新增会导致某些光源组合找不到 shader。

### 5.5.7 漏编译断点与检查方法

| 断点 | 表现 | 检查方法 |
| --- | --- | --- |
| Classic `GetMaterialEnvironment` 漏 define | 材质可保存但 BasePass 无 Toon 分支 | Opaque 与 Masked 预处理 shader 均须存在 `MATERIAL_SHADINGMODEL_CUSTOM_TOON=1` |
| MIR `GetShadingModelParameterName` 漏 case | MIR 翻译 unreachable / 失败 | 分别强制走 Classic 与 MIR，两者都必须生成同名 define |
| Classic 枚举 13 未映射到 14 | SMID 显示 13，命中原生 Substrate Toon | GBuffer debug 确认 `ShadingModelID==14` |
| MIR 固定 / 动态 SM 未映射 | MIR 材质或 Static Switch 分支写 13 | 分别编译固定 SM 与 Expression 材质并检查 SMID |
| `FShaderMaterialPropertyDefines` / FETCH 漏改 | define 存在但派生系统认为 false | 确认派生的 `WRITES_CUSTOMDATA_TO_GBUFFER=1` |
| 只改 `.ush` 漏 `ShaderMaterialDerivedHelpers.cpp` | Selective BasePass 不输出 GBufferD | 在 Selective BasePass Outputs 开 / 关配置下分别检查 |
| 漏 `ShaderGenerationUtil` GBuffer slot | RT layout / 输出签名不匹配 | 检查 BasePass shader 输出包含 GBufferD MRT |
| 漏 HLSL `WRITES_CUSTOMDATA` | Toon 分支写了但 MRT 被裁剪 | GBufferD 可视化在 Opaque 与 Masked 存活像素均非零 |
| 漏 `HasCustomGBufferData` | GBufferD 有值但 `FGBufferData.CustomData` 为零 | 临时 debug Deferred 解码值 |
| Deferred Global Shader 未重编译 | BasePass 写 14 但旧 Deferred 返回黑 | 确认 shader map 缓存键更新，日志无旧 map 回退 |
| 增量缓存掩盖问题 | 开发机正常、干净机器缺 shader | 在独立干净 DDC/CI 执行一次 ShaderCompile/Cook |

### 5.5.8 最小验收矩阵

| 维度 | 必测组合 |
| --- | --- |
| Translator | Classic；MIR |
| SM 来源 | 固定 `MSM_CustomToon`；`From Material Expression` / Static Switch |
| Blend | Opaque；真实 Masked（`WritesEveryPixel=false`，非恒定 OpacityMask） |
| Vertex Factory | Static Mesh LocalVertexFactory；Skeletal Mesh GPU Skin |
| Lighting policy | NoLightMap；项目实际启用的 LightMap policy |
| 光源 | Directional；Point；Spot |
| 必须观察 | `MATERIAL_SHADINGMODEL_CUSTOM_TOON=1`；`WRITES_CUSTOMDATA_TO_GBUFFER=1`；SMID=14；GBufferD 非零；Deferred 命中 `CustomToonBxDF` |
| 禁止出现 | ID 13；`MATERIAL_SHADINGMODEL_TOON`；原生 `ToonBxDF`；默认材质 fallback；`GetBasePassShaders` 失败 |

## 5.6 弃用路径警告（已确认）

传统非 Substrate 链仍可用，Deferred / Clustered 均进入 `GetDynamicLighting`，最终使用 `IntegrateBxDF`。但 `IntegrateBxDF` 已标记 `UE_DEPRECATED 5.7`，由 Substrate 替代。

含义：

* 固定 5.8 可以正常使用。
* 这是一条**被冻结的弃用路径**。
* 未来升级 UE 主版本时，该路径存在整体失效风险。

该风险再次支持固定 UE5.8 的决策（D-002）。

---

# 6. 数据进 Shader 的通道

本节规定数据如何从游戏线程进入 shader。通道一旦建成，之后只改内容不改结构。

## 6.1 KEY Light 数据链（已确认）

**事实部分（[源码]）**：UE5.8 不存在满足需求的"美术显式指定唯一 Key Light"机制。

不得复用的现有机制：

| 机制 | 为什么不能用作 Toon Key |
| --- | --- |
| `FScene::SimpleDirectionalLight` | 只是首个符合条件的动态方向光，存在加入顺序依赖；删除时直接清空，不会自动补选 |
| `FViewUniformShaderParameters::DirectionalLightDirection` | 来自 `SimpleDirectionalLight`，带原生语义，直接覆写会改变依赖它的原生逻辑 |
| Atmosphere Sun Light | 属于 Sky Atmosphere 体系，方向可能受代理影响 |
| Forward Selected Directional Light | 服务 Forward / Translucency 路径，选择时机与语义不适合 Deferred BasePass |

第一版采用独立 View 数据链：

1. 项目级 World Subsystem 保存美术指定 `UDirectionalLightComponent` 的弱引用。
2. `FToonSceneViewExtension::SetupView` 在 GameThread 对每个 View 读取 `-KeyLight->GetDirection()`。
3. 方向写入新增的 `FSceneView::ToonKeyLightDirectionAndValid`。
4. `FViewInfo(const FSceneView*)` 将其复制到 Renderer View。
5. `FViewInfo::SetupUniformBufferParameters` 写入 `FViewUniformShaderParameters::ToonKeyLightDirectionAndValid`。
6. BasePass shader 通过 `ResolvedView.ToonKeyLightDirectionAndValid.xyz` 访问，并检查 `.w`。

字段约定：

* `xyz`：归一化 surface-to-light 世界空间方向。
* `w = 1`：Key Light 有效；`w = 0`：无效，默认值写为 `(0,0,1,0)`。
* 每次 `SetupView` 必须**无条件初始化**字段，禁止只在光源有效时更新。
* 光源引用使用 `TWeakObjectPtr`；删除、卸载或替换后下一 View 自动写回无效状态。
* 数据按 View 保存。分屏与不同 ViewFamily 不共享可变静态状态；VR 双眼使用同一世界级 Key Light，但分别写入各自 View Uniform。
* shader 必须使用 `ResolvedView`，保证 Instanced Stereo 取到正确的 View 参数。

**实现期验证要求（待实验）**：本链路的"新增字段实现方案"整体为**架构推断**（[推断]）而非源码直证；"现有机制不能用"是 [源码]。第一次实现时必须用 Debug View 验证多 View 与删除回落行为，**不得假设一次写对**。

施工类型：增加字段与填充代码属 Engine rebuild；字段建成后仅调整 HLSL 使用逻辑才是 shader-only；项目侧 Subsystem / SVE 需项目 C++ 编译。

## 6.2 HeadBasis 通道（已确认）

UE5.8 原生提供 36 个 CPD float，HeadForward + HeadRight 需要 6 个，容量足够。现有 Proxy / Desc 已传输 CPD 数据，组件 Setter 调用 `Scene->UpdateCustomPrimitiveData`，Material HLSL 可通过 `GetPrimitiveData(...).CustomPrimitiveData` 读取。

第一版**不修改** `PrimitiveSceneProxy`、`PrimitiveSceneProxyDesc`、Skeletal / Static Mesh SceneProxy。

优先路径：Character Component / Blueprint → CPD → Material / BasePass → HeadForward / HeadRight。
备选：Character Material Parameter → BasePass。

**CPD 使用约束（已确认）**：

* CPD 运行时不序列化。
* CPD 数据属于每个 `UPrimitiveComponent`。

因此必须：预留明确的固定索引；多 Mesh 角色（头、发、身分离）逐组件同步；定义角色生成时的初始化规则与默认值。

具体索引表、坐标空间与归一化约定属**项目数据契约**（C-11 已降级为项目 Contract，Codex 仅做 sanity check）。

## 6.3 Face SDF 数据来源

| 数据 | 第一版方案 | 状态 |
| --- | --- | --- |
| KeyLightDirection | 新增自定义 View Uniform 字段 | 已确认 |
| HeadForward | Custom Primitive Data | 已确认 |
| HeadRight | Custom Primitive Data | 已确认 |
| FaceSDF | Material（**只传最终标量**，见 D-035） | 已确认 |
| FaceUV | Material | 已确认 |

第一版路径：BasePass 取得 KeyLightDirection 与 HeadBasis → 采样 FaceSDF → 计算最终标量 → 写入 CustomData.A。

## 6.4 全局参数通道

项目侧结构：

```text
UToonRendererSettings → Project Defaults
UToonRendererSubsystem → Runtime Global State
Camera / Sequence Override → FToonViewParameters 构建
```

Shader View 参数通道字段（**内容定义推迟到第二部分**）：

```text
FToonViewParameters
├─ ToonKeyLightDirectionAndValid
├─ GlobalGIWeight
├─ GlobalGIIntensity
├─ GlobalOutlineWeight
├─ GlobalOutlineLODBias
├─ GlobalOutlineColor
├─ GlobalRimWeight
├─ FillLightWeight
└─ DebugMode
```

后处理参数链：

```text
APostProcessVolume → Weighted Blendable → UToonPostBlendable : IBlendableInterface
→ FToonPostSettings → SceneViewExtension → RDG Pass Parameters
```

原则：**不修改 `FPostProcessSettings`。**

## 6.5 参数层级

```text
Settings → Subsystem / Global → Character → Material → Texture / Vertex / Pixel
```

* 项目默认 → Settings
* 世界 / 镜头一起变 → Subsystem
* 每角色不同 → Character / CPD
* 每材质不同 → Material
* 每像素不同 → Texture / GBuffer

---

# 7. 美术资产契约

本节属于数据语义。通道含义一旦变动，所有已产出资产作废，且 shader **不报错**，只是画面变丑。

## 7.1 第一版资产输入

| 数据 | 制作人 | Shader 用途 |
| --- | --- | --- |
| Base RGB | 美术 | 固有色 |
| ILM.R | 美术 | 高光 / 材质类型 |
| ILM.G | 美术 | 常驻 AO / 二级阴影 |
| ILM.B | 美术 | Specular 强度 |
| ILM.A | 美术 | Ramp Row |
| Ramp | TA | Toon 明暗颜色 |
| FaceSDF | TA / 美术 | 脸部阴影边界 |
| VertexColor.G | 美术 | Ramp / Shadow 微调 |
| VertexColor.A | 美术 | Outline Width（**契约未闭合**，见 D-036） |
| Smooth / Baked Normal | TA | 几何描边 |
| Hair Tangent | 模型 / Shader | Kajiya-Kay |

## 7.2 资产原则

* 贴图决定美术希望哪里亮、哪里暗。
* 灯光决定当前光从哪里来。
* Shader 决定如何组合贴图与灯光。

## 7.3 ILM 契约（第一版冻结后禁止改变）

* ILM.R = Material / Spec Type
* ILM.G = AO / Persistent Shadow
* ILM.B = Specular Strength
* ILM.A = Ramp Row

**ILM.G 是二游角色"像不像"的第一决定因素**——卡通渲染只有 2-3 个色阶，只靠 NdotL 会出现大片"一样亮"的平区域，ILM.G 是美术手绘的常驻暗（鼻侧、腋下、衣褶深处、裙底），给这些平区域加局部体积感。它不该随光变。

## 7.4 与美术对接要求

文档技术规范之外，还必须向美术交付：

* 每个通道的**参考图**，包含正确与错误样例。
* 常见错误示例：ILM.G 全白导致塑料感、过黑导致脏、边界过硬导致贴膏药感。
* 明确第一版技术限制：只有 2 到 3 阶；脸部阴影只能是干净一条；描边为屏幕空间且快速运动时会抖。
* **在 shader 完成前先用固定色 + NdotL 验证资产**，不要等全部做完再看——否则发现 ILM 画错，返工量巨大。

---

# 8. 功能范围与兼容性

## 8.1 功能状态表

| Feature | 第一版 | 优先级 | 备注 |
| --- | --- | --- | --- |
| Custom SM | 做 | P0 | E-1 |
| CustomData | 做 | P0 | E-2 / E-4 |
| Ramp | 做 | P0 | D-034 已定：首版单 Ramp（RowCount=1），绑定走 View 参数通道 |
| KEY / FILL | 做 | P0 | 依赖 E-5 |
| VSM | 做 | P0 | |
| Face SDF | 做 | P0 | D-035 已定：材质内采样，只传最终标量 |
| Hair Spec | 做 | P0 | |
| Eye Highlight | 做 | P0 | |
| Screen Hair Shadow | 做 | P0 | |
| Toon GI | 做 | P0 Research | 只改 Composite（D-026） |
| Screen Outline | 做 | P1 | |
| 逐顶点 Outline Width | **不做（降级）** | P1 | D-036 已定：走全局 + MaterialType 16 档宽度 |
| Rim | 做 | P1 | |
| Eyebrow | 做 | P1 | DepthBias 走材质侧（D-027） |
| Dither Fade | 做 | P1 | D-039 已定：接受阴影跳变 |
| **SSS 背光透光** | **不做** | P1 / B 组 | D-037 已定：后置。见 8.3，防遗忘见 13.14 |
| Geometry Outline | 不做 | P2 | |
| Multi-Layer | 不做 | P2 | |
| Bloom | 不做 | P2 | |
| ThickCoating | 不做 | P3 | |
| Kuwahara | 不做 | P3 | |
| SNN | 不做 | P3 | |
| Watercolor | 不做 | P3 | |
| Mobile | 不做 | Future | |
| Limited Animation | 不做 | Future | |

## 8.2 兼容性矩阵

| 系统 | 第一版 |
| --- | --- |
| Substrate | 不支持 |
| MegaLights | 不支持 |
| Forward | 不支持 |
| PathTracing | 不支持 |
| Mobile | 不支持 |
| Nanite Character | 待定 |
| Lumen | 支持 |
| TSR | 支持 |
| VSM | 支持 |

## 8.3 SSS 背光透光（缺口登记）

**现象**：逆光时耳廓、发梢、手指透出暖色。二游常见。

**当前状态**：文档在第一版范围外，属 B 组（后期补只改 shader）。

**为什么不进第一版**（按三种"无法分离"的情况判断）：

| 条件 | 是否满足 |
| --- | --- |
| 需要不存在的执行位置？ | 否，Deferred Lighting 即可 |
| 需要没有的通道？ | 否，厚度可走贴图材质输入 |
| 需要改上游计算？ | 否 |

三种均不成立 → **纯 shader 层 → 后期补零编译成本 → 现在不做**（B 组逻辑）。

**但这必须记下来**，否则会被遗忘。见 13.14。是否提前进入第一版由 D-037 拍板。

## 8.4 需求覆盖自检方法

判断"需求够不够"不能靠感觉，用**截图对照法**：

> 拿一张二游角色截图，逐个视觉现象找对应项。找不到对应的，就是缺口。

已跑过一遍的结果：二游角色最标志性的视觉特征（色块明暗、暗部偏冷、多材质配色、常驻暗、干净脸影、天使环、眼高光、边缘光、轮廓线、发影、主补光、GI 独立、眉毛透发）**第一版基本全覆盖**，唯一明显缺口即 SSS（8.3）。

**注意**：清单核对 ≠ 组合验证。每个效果都有对应项，不代表它们一起用画面是对的——后者只能跑起来看（见第 9 章 Golden Scene）。

---

# 9. 验收标准与 Debug 框架

## 9.1 Golden Test Scenes

| Scene | 验证什么 |
| --- | --- |
| A：Diffuse | Ramp 明暗、ILM、多材质行 |
| B：Face | Face SDF 干净弧线，光绕头 360° 不跳变；8bit 色阶检查 |
| C：Fill | Point / Spot 只补亮，不翻转造型 |
| D：GI | 角色 GI 独立可调，不被 Lumen 洗白 |
| E：Outline | 描边稳定性、宽度、颜色 LOD |
| F：Temporal | TSR 下相机运动无阻塞级闪烁（含描边、Ramp 硬边界、Dither） |

## 9.2 Debug View（12 个通道）

任何进入 GBuffer 的数据必须有可视化方式。通道：

1. ShadingModelID
2. CustomData.R（flags）
3. CustomData.G/B（oct tangent）
4. CustomData.A（FaceResult / RampRow + MaterialType）
5. RampRowID
6. MaterialType
7. NdotL
8. Face SDF raw
9. HeadBasis
10. KeyLightDirection
11. Toon Mask
12. GI 分流结果

原则：**禁止继续叠功能，先补 Debug**（见 12.8）。

## 9.3 第一阶段强 Gate（9 条）

1. SMID 必须等于 14，禁止出现 13
2. Opaque 与 Masked 行为一致
3. Masked 必须用非恒定 OpacityMask 验证
4. DefaultLit 行为不变（负对照）
5. CustomData 五处白名单全部生效
6. GBufferD 非零
7. Deferred 命中 `CustomToonBxDF`，不落入 default
8. 无默认材质 fallback、无 `GetBasePassShaders` 失败
9. 关键数据全部有 Debug 可视化

---

# 10. 分阶段施工计划

## 10.1 施工 DAG

```text
阶段 0 基线环境
  ↓
阶段 0.5 前置验证（已全部闭合）
  ↓
阶段 1 B0 粉色球  ←── 当前可开工
  ↓
阶段 2 GBuffer CustomData 与 Debug（B1 / B3）
  ↓
阶段 3 Ramp + VSM（B4）
  ↓
阶段 4 KEY Light 数据链 + FILL（B2）
  ↓
阶段 5 Face SDF
  ↓
阶段 6 Hair / Eye / Rim / 发影
  ↓
阶段 7 Toon GI Prototype（B5）
  ↓
阶段 8 Screen Outline（B6）
  ↓
阶段 9 Character MVP 验收
  ↓
阶段 10 Eyebrow / Dither / Material Layer
  ↓
阶段 11 ToonPost Framework
  ↓
阶段 12 风格化后处理
  ↓
阶段 13+ 后置项
```

## 10.2 阶段 0：UE5.8 基线环境

**目标**：拿到可编译的引擎源码，本地路径与主程一致。

任务：

1. 从主程处取得编译好的引擎（精简版：Source + Binaries + Shaders + Plugins，不含 PDB / Intermediate / DDC）
2. 建立本地 sparse-checkout 工作区
3. 确认编译责任人、push 分支与提交批次
4. 跑通一次完整编译，记录耗时

**验收**：本地能成功编译未修改的引擎。

**阻塞说明**：本阶段是全部工作的物理前提。`不解决，以上所有设计都是纸面。`

## 10.3 阶段 0.5：前置验证（已全部闭合）

| 验证项 | 结果 |
| --- | --- |
| KEY Light 身份与同步链 | 已闭合（D-021） |
| Opaque / Masked permutation | 已闭合（D-022，见 5.5） |
| GBufferD 分配、精度、Post 可读性 | 已闭合（见第 4 章） |

**阶段 1 硬阻塞已解除。**

## 10.4 阶段 1：最小 Toon ShadingModel 注册（B0）

**目标**：Opaque 与 Masked 球显示固定粉色，SMID=14。

任务（11 条）：

1. `EngineTypes.h` 加 `MSM_CustomToon`
2. `MaterialShader.cpp` 加名称分支
3. `HLSLMaterialTranslator.cpp` 加 define
4. `HLSLMaterialTranslator.cpp::ShadingModel` 显式映射 ID 14
5. `MaterialIRModuleBuilder.cpp` 固定 SM 映射 ID 14
6. `MaterialExpressionsToMIR.cpp` 动态 SM 映射 ID 14
7. `MaterialIRToHLSLTranslator.cpp` 加参数名
8. `ShaderMaterial.h` + `ShaderGenerationUtil.cpp` define 镜像与 FETCH
9. `ShadingCommon.ush` 定义 ID 14、NUM 15
10. E-2 五处 CustomData 白名单
11. `SetGBufferForShadingModel` 写 ID14 + 固定测试值；`IntegrateBxDF` 加 ID14 case 输出固定粉色

**验收**：Opaque 与真实 Masked 球为粉色；SMID=14；GBufferD 非零；DefaultLit 不变；未进入 ID13。

## 10.5-10.16 后续阶段（摘要）

| 阶段 | 目标 | 对应批次 |
| --- | --- | --- |
| 2 | 完整 CustomData 编解码与 Debug 通道 | B1 / B3 |
| 3 | Ramp 采样与 VSM 阴影 | B4 |
| 4 | KEY Light 数据链与 FILL 政策 | B2 |
| 5 | Face SDF | — |
| 6 | Hair / Eye / Rim / 屏幕发影 | — |
| 7 | Toon GI 最小原型 | B5 |
| 8 | Screen Outline | B6 |
| 9 | Character MVP 验收 | — |
| 10 | Eyebrow / Dither / Material Layer | — |
| 11 | ToonPost Framework | — |
| 12 | 风格化后处理 | — |
| 13+ | 后置项（几何描边、水彩、Multi-Layer 等） | — |

每个阶段完成后必须停下，提交验收结果，等待人工确认（Agent 总规则第 2 条）。

## 10.17 施工里程碑

| 里程碑 | 标志 |
| --- | --- |
| M0 | 引擎可编译 |
| M1 | 粉色球出现 |
| M2 | 一个完整卡通角色在 Lumen 场景中 |
| M3 | MVP 全项通过 Golden Scene |
| M4 | 风格化后处理上线 |

---

# 11. 源码确认事实速查

本节汇总所有经真实 5.8 源码验证的结论，**可直接作为施工事实使用**。设计内容在前文，本节只做速查。

| 验证项 | 结论 | 状态 |
| --- | --- | --- |
| GBuffer Layout / Format 文件 | `GBufferInfo.h/.cpp`、`SceneTexturesConfig.*`、`SceneTextures.cpp` 均存在，未改名 | 已确认 |
| CustomData 编解码主检查点 | `DeferredShadingCommon.ush::EncodeGBuffer` / `DecodeGBufferData`；解码受 `HasCustomGBufferData` 控制 | 已确认 |
| GBufferD 格式 | `GBT_Unorm_8_8_8_8` / `PF_B8G8R8A8`，每通道 8 位 UNorm | 已确认 |
| Toon SM 是否自动分配 CustomData | 不自动成立，必须加入显式白名单（5 处） | 已确认 |
| Opaque / Masked 基础写入条件 | `USES_GBUFFER` 同时覆盖 Solid 与 Masked，存活 Masked 像素进入相同编码 | 已确认 |
| **Opaque / Masked 完整 permutation** | **已闭合，无 Toon 专用 permutation 维度** | **已确认（D-022）** |
| SMID 位数 | `SHADINGMODELID_MASK=0xF`，仅 4 位；5.8 已用 `SUBSTRATE_TOON=13`，当前 `NUM=14` | 已确认 |
| Before / After Tonemap 可绑定 GBufferD | 可以，`FPostProcessMaterialInputs` 携带 `FSceneTextureShaderParameters` | 已确认 |
| SceneViewExtension RDG 可绑定 | 可以，回调可绑定 `FSceneTextureShaderParameters` | 已确认 |
| Post 时序坐标风险 | TSR / 上采样后 SceneColor 与 GBuffer 可能不同分辨率 | 已确认（描边放 TSR 前规避） |
| Deferred Lighting 五个文件 | `ShadingModels.ush`、`DeferredLightingCommon.ush`、`DeferredLightPixelShaders.usf`、`ClusteredDeferredShadingPixelShader.usf`、`DeferredShadingCommon.ush` 均存在未改名 | 已确认 |
| `IntegrateBxDF` 状态 | 仍可用，但已标记 `UE_DEPRECATED 5.7` | 已确认 |
| GI 三个文件 | `IndirectLightRendering.cpp`、`LumenScreenProbeGather.usf`、`DiffuseIndirectComposite.usf` 均存在且连通 | 已确认 |
| GI 最小分流点 | `DiffuseIndirectComposite.usf` 已能读 GBuffer 与 SMID，并对原生 Toon 应用 `DiffuseIndirectScale`，是独立按 SM 分流点 | 已确认 |
| CPD 容量 | 原生 36 个 float，HeadBasis 只需 6 个 | 已确认 |
| OverlayMaterial | 能力仍在，支持全局或按 Slot；会多一次 Draw；要求透明材质；`CastShadow=false` | 已确认 |
| Overlay DepthBias | 5.8 Overlay 接口只提供材质、Slot、最大绘制距离，**无专用 DepthBias 参数** | 已确认 |
| DitherOpacityMask | 仅对 Masked 生效；主 BasePass 使用时间抖动 | 已确认 |
| Dither 与 VSM | `GetMaterialClippingShadowDepth` 只执行普通 `GetMaterialMask`，不调用 `GetDitheredMaterialMask`，**阴影不跟随淡出** | 已确认 |
| Lighting Channels | 只影响 opaque material 的 direct lighting，对 masked / translucent 不适用 | 已确认 |
| SVE 签名 | 5.8 有效签名带 `const FSceneView&`；旧重载自 5.5 弃用且不会被调用 | 已确认 |
| 八面体误差 | oct 8:8 平均 `0.33709`、最大 `0.94424`（源码注释） | 已确认 |

## 11.1 UE5.8 Source Touch Map（文件级索引）

符号级施工图见第 2 章。本节只作文件级索引。

**P0：ShadingModel / Material** — `EngineTypes.h`、`MaterialShared.*`、`Material.h`、`HLSLMaterialTranslator.cpp`、`MaterialHLSLEmitter.*`、`MaterialAttributeDefinitionMap.cpp`、`MaterialExpressions.cpp`、`MaterialExpressionsIR.cpp`、`MaterialCachedData.cpp`、`MaterialTemplate.ush`、`ShaderMaterial.h`、`ShaderMaterialDerivedHelpers.cpp`

**P0：GBuffer / BasePass** — `GBufferInfo.h/.cpp`、`BasePassPixelShader.usf`、`BasePassCommon.ush`、`SceneTexturesConfig.h/.cpp`、`SceneTextures.cpp`、`SceneTexturesCommon.ush`、`ShaderGenerationUtil.cpp`。CustomData 编解码主检查点：`DeferredShadingCommon.ush`。重构路径检查点：`GBufferHelpers.ush`。

**P0：Direct Lighting** — `ShadingCommon.ush`、`ShadingModels.ush`、`Definitions.usf`、`DeferredLightingCommon.ush`、`DeferredLightPixelShaders.usf`、`ClusteredDeferredShadingPixelShader.usf`、`DeferredShadingCommon.ush`

**P0：View Uniform（KEY Light）** — `SceneView.h`、`SceneRendering.cpp`、`SceneViewExtension.h`、`LocalPlayer.cpp`

**P0：GI** — 必改：`DiffuseIndirectComposite.usf`。后置参考：`IndirectLightRendering.cpp`、`Lumen/LumenScreenProbeGather.usf`

**P1：Shadow** — `VirtualShadowMapProjection.usf`、`VirtualShadowMapProjectionSpot.ush`、`ShadowProjectionPixelShader.usf`、`DistanceFieldShadowing.usf`

**P1：SceneProxy（后置研究，第一版不依赖）** — `PrimitiveSceneProxy.*`、`PrimitiveSceneProxyDesc.*`、`SkeletalMeshSceneProxy.*`、`StaticMeshSceneProxy.*`、`SkinnedMeshSceneProxyDesc.*`、`PrimitiveComponent.*`、`MeshComponent.*`、`SceneView.*`

**P2：Anisotropy** — `AnisotropyPassShader.usf`、`AnisotropyRendering.cpp`

## 11.2 算法参考来源映射

本节只记录"需求来自谁"，不记录具体实现。

| 功能 | 主参考 | UE 侧关注 |
| --- | --- | --- |
| SM 注册 | Yumaoshier + 5.8 源码 | `EngineTypes.h`、`ShadingCommon.ush`、CustomData 白名单 |
| Toon 公共光照 | MooaToon 5.7 | 可选组织参考 |
| Toon BxDF | MooaToon 5.7 | 可选组织参考 |
| GBuffer / CustomData | UE_CelLit + 5.8 源码 | `DeferredShadingCommon.ush`、BasePass |
| Ramp / ILM | GenshinCelShaderURP | Direct Lighting |
| Face SDF | kaze-mio | BasePass / Lighting |
| Hair / Eye | UE_CelLit | Direct Lighting，复用原生八面体编码 |
| Screen Hair Shadow | UE_CelLit | Deferred Lighting |
| Multi-light Policy | HoyoToon | Direct Lighting |
| Stylized Ambient | MToon | Indirect |
| Toon GI | MooaToon 5.7 + 5.8 源码 | 首版仅 `DiffuseIndirectComposite.usf` |
| Geometry Outline | Yumaoshier | Renderer MeshPass |
| Outline UX | UTS / lilToon / HoyoToon | Width / Color / LOD |
| SceneViewExtension | A57R4L + 5.8 签名 | 必须用带 `FSceneView` 的新签名 |
| Kuwahara | noxtgm | RDG |
| SNN | kafues511 / t-takasaka | RDG |
| Bloom | kaze-mio | RDG |
| Watercolor | KinoAqua | RDG |
| Post 兼容性 | MooaToon 5.7 | 仅兼容性参考 |

---

# 12. 决策、风险与工程维护

## 12.1 最大风险

| 风险 | 说明 | 缓解 |
| --- | --- | --- |
| `IntegrateBxDF` 弃用路径 | 已标记 `UE_DEPRECATED 5.7`，升级主版本可能整体失效 | 固定 5.8（D-002） |
| GI 组合后果不可预测 | Lumen 输入不受我们控制，无法事先推演 | 最小原型试探（D-026） |
| CustomData 白名单静默失败 | 五处缺一即数据恒为 0，不报错 | 逐处 Buffer Visualization 检查 |
| ID 13 冲突 | C++ 枚举自然值即 13 | 四个位置强制显式映射 |
| 组合验证缺失 | 清单覆盖 ≠ 一起用是对的 | Golden Scene + 时序冲突登记 |
| 文档状态矛盾 | 教训：增量回填不回头扫 | Agent 总规则第 10 条三问巡检 |

## 12.2 MooaToon 5.7 在本项目中的角色

**定位：分三层用，不是"不用"。**

| 层面 | 依赖引擎版本吗 | 怎么用 |
| --- | --- | --- |
| 算法数学 | 不依赖 | **借鉴**。Ramp 采样、SDF、八面体编码、Kajiya-Kay 等纯数学，5.7 与 5.8 无区别 |
| 落点地图 | 部分依赖 | **参考未来**。它改了哪些文件，告诉我们以后加功能该动哪里 |
| 引擎改动代码 | 强依赖 | **不移植**。版本对不上、抄比写更容易出错、需求规模差一个量级 |

**为什么算法可以脱离它的引擎改动**（核心逻辑）：

> 算法 = 函数 f(输入)；引擎改动 = 给这个函数接两根线（输入从哪来、在哪执行）。
> 算法不关心线怎么接，只关心输入到位没有。

| 算法 | 它的管道 | 我们的管道 | 算法本身 |
| --- | --- | --- | --- |
| Ramp | CustomData | CustomData（自己的位分配） | 相同 |
| Face SDF | 68 个 Proxy 文件传 HeadBasis | CPD 传 HeadBasis | 相同 |
| Kajiya-Kay | CustomData + 八面体 | CustomData + 八面体 | 相同 |
| GI | 改 C++，计算阶段介入 | 改 shader，Composite 阶段介入 | 相同（时机不同） |

**边界——三种情况无法分离**（此时必须自建引擎改动）：

| 情况 | 例子 |
| --- | --- |
| 需要不存在的执行位置 | 几何描边要新增 MeshPass |
| 需要没有的通道 | CPD 36 个用完 |
| 需要改上游计算 | GI 必须在 Lumen 计算阶段排除角色 |

### 12.2.1 约束

* 所有 MooaToon 参考需 rebase 到 5.8。
* 不得直接移植其引擎改动代码。
* 不得复用其符号命名（避免与 5.8 原生冲突）。

### 12.2.2 已识别的适用范围差异

* MooaToon 是完整卡通管线（含水彩、几何描边、完整 GI 系统），我们第一版只要"角色在 Lumen 场景稳定闭环"。
* 它改 68 个 SceneProxy 文件是为传更丰富的每角色数据；我们用 CPD 已足够。
* 它的 GI 是完整风格化系统，需改 C++；我们最小分流只需改 Composite。

## 12.3 YivanLee Coverage

该来源（知乎 p/542384881）作为**功能覆盖度基准**使用，核对"二游管线需要哪些效果我们有没有覆盖"。

关键价值：四种勾线方式、ColorLOD、LineLOD、描边精细控制、PPVolume 参数传递。

**重要限制**：该来源本身只是功能罗列，且原文明说"方案还在持续优化开发中"。

**清单核对 ≠ 组合验证**——它能回答"有没有"，不能回答"一起用行不行"。组合验证只能靠 Golden Scene。

## 12.4 第一阶段禁止事项

* 禁止实现几何描边 / 几何发影
* 禁止实现水彩 / Kuwahara / SNN
* 禁止实现 Multi-Layer / ThickCoating
* 禁止为未来需求提前修改 SceneProxy
* 禁止为未验证需求提前新增 RT
* 禁止复用原生 Substrate Toon 符号
* 禁止依赖 `SUBSTRATE_ENABLED` 作为唯一隔离手段
* 禁止新增 Toon permutation bool
* 禁止提前建立空壳类假装预留（**但公共 shader helper 除外**，见 2.5.6）
* 禁止跨阶段提前实现
* 第一部分未冻结前禁止设计第二部分算法

## 12.5 最终 MVP 定义

一个卡通角色进入普通 UE5.8 Lumen 场景，具备：色块明暗、规整脸影、头发天使环、稳定眼高光、可调环境光、带距离衰减的描边、眉毛透发、平滑淡出，外加一整套 Debug 工具。TSR 下相机运动无阻塞级闪烁。

## 12.6 架构决策编号表

状态列使用 0.1 的四分类体系。

| ID | 决策 | 状态 | 直接含义 | 理由 |
| --- | --- | --- | --- | --- |
| D-001 | 自定义 Toon ShadingModel | 已确定 | 修改 UE5.8 Source | 原生实验路线不满足需求 |
| D-002 | 固定 UE5.8 | 已确定 | 不持续跟随主线 | 控制合并成本，集成路径已弃用 |
| D-003 | Substrate Off | 已确定 | 不实现 Substrate 路径 | 自定义 SM 路径不同 |
| D-004 | MegaLights Off | 已确定 | 使用标准 Deferred Local Light | 降低兼容风险 |
| D-005 | Deferred Renderer | 已确定 | GBuffer 为主数据契约 | 第一版核心路线 |
| D-006 | PC / DX12 / SM6 | 已确定 | 第一版不兼容移动 | 平台边界 |
| D-007 | CustomData.A 双语义 | 已确定 | FaceResult / RampRowID + MaterialType 复用 | 避免额外 RT |
| D-008 | KEY / FILL 分离 | 已确定 | Local Fill 不改变主造型 | 艺术政策 |
| D-009 | Face SDF 只响应 KEY | 已确定 | Local Light 不翻转 Face | D-007 的取舍 |
| D-010 | Screen Hair Shadow 首发 | 已确定 | Geometry 后置 | 降低 Renderer 工程量 |
| D-011 | Screen Outline 首发 | 已确定 | Geometry 后置 | MVP 优先 |
| D-012 | Global GI / Outline 参数不占 GBuffer | 已确定 | 使用 View / Global 参数 | 不属于逐像素数据 |
| D-013 | Eyebrow 使用 OverlayMaterial | 已确定 | 不重画 GBuffer，双 Draw，CastShadow=false | 第一版低侵入 |
| D-014 | Fade 优先 DitherOpacityMask | 已确定 | 仅 Masked 生效 | Deferred 兼容优先 |
| D-015 | Watercolor 后置 | 已确定 | P2/P3 | 不影响角色主闭环 |
| D-016 | Forward 关闭 | 已确定 | 首版无 Forward Toon | 边界控制 |
| D-017 | Mobile 关闭 | 已确定 | 远期独立设计 | 边界控制 |
| D-018 | Geometry Pass 不阻塞 MVP | 已确定 | Outline/Hair 均后置 | 风险控制 |
| D-019 | Multi-Layer 复用单 Toon SM | 已确定 | 少占 SMID，用 CustomData.A 高 4 位存 MaterialType | Hair/Eye/ClearCoat 为原生先例 |
| D-020 | MooaToon 只作 5.7 参考 | 已确定 | 分三层用：算法借鉴、落点参考、引擎改动不移植 | 版本不同 |
| D-021 | KEY 为美术指定唯一 Directional Light | 已确认 | 新增 `ToonKeyLightDirectionAndValid`，xyz=方向，w=有效位 | 5.8 无既有主光概念 |
| **D-022** | **Toon SM Opaque / Masked permutation 契约** | **已确认** | **无 Toon 专用 permutation 维度，共用 TBasePassPS** | **C-05 已闭合，见 5.5** |
| D-023 | Custom Toon 使用 GBuffer ID 14 | 已确认 | NUM 更新为 15，15 保留不用 | 4 位限制，15 等于 MASK 全 1 |
| D-024 | CustomData 采用 round-to-nearest 协议 | 已确认 | 先恢复完整 byte 再位运算，point/load 语义 | 避免 UNorm 插值与舍入误判 |
| D-025 | HeadBasis 使用原生 CPD | 已确认 | 不改 SceneProxy，36 float 足够 | 5.8 原生能力已足够 |
| D-026 | GI 首版只改 Composite | 已确认 | 不动 `IndirectLightRendering.cpp` 与 ScreenProbeGather | Composite 已是独立分流点 |
| D-027 | Eyebrow DepthBias 走材质侧 | 已确认 | Overlay 无 DepthBias 参数 | 5.8 接口只提供材质、Slot、距离 |
| D-028 | SVE 使用带 FSceneView 的新签名 | 已确认 | 旧重载自 5.5 弃用且不会被调用 | 避免后处理静默失效 |
| **D-029** | **Dither 与 VSM 阴影不同步** | **已确认（事实）/ 已确定（政策见 D-039）** | **事实：ShadowDepth 不走 Dither Mask。政策：首版接受跳变（D-039）** | **见 11 章速查表** |
| D-030 | 不依赖 SUBSTRATE_ENABLED 做隔离 | 已确认 | 所有判断显式比较 ID 14 | 5.8 仍有未受该宏保护的 ID 13 检查 |
| D-031 | 文档分为框架层与 Shader 实现层 | 已确定 | 第一部分未冻结前禁止设计第二部分算法 | 框架未定时细化算法必然返工 |
| **D-032** | `MP_ToonData` 四分量语义 = **选项 B：语义值** | **已确定** | 四分量承载 flag 掩码 / MaterialType / RampRowID / FaceResult，**BasePass 内集中编码** | 不让美术在材质图里做位打包（错误源）；符合 4.4"编解码集中为共享函数" |
| **D-033** | ILM.G / ILM.B 运输 = **选项 A：重解释标准槽** | **已确定** | **ILM.G → Specular 槽，ILM.B → Metallic 槽**；卡通材质下二者不再具备 PBR 语义 | 零成本；ILM.G 保留完整 8 bit；比 AO 槽稳（AO 受 `ALLOW_STATIC_LIGHTING` 干扰）。ILM.G 是"像不像"第一决定因素，不可丢（7.3） |
| **D-034** | Ramp atlas = **首版单 Ramp** | **已确定** | **RowCount = 1**；明暗 = `lerp(固有色, 暗部, α)`，α 由阶式 shade level 查 ramp 得到；全场共用一条曲线。绑定走 **View 参数通道（复用 E-5）**，不改 `FDeferredLightPS::FParameters` | 无美术、TA 为唯一决策人时简化优先；打光是全局 shader 拿不到材质贴图，只能全局绑 |
| **D-035** | FaceSDF 计算位置 = **材质内** | **已确定** | 材质 / BasePass 采样，只传最终标量；FaceSDF 只响应 KEY（D-009） | 二游脸影是风格化的，多光源翻转反而不是想要的效果 |
| **D-036** | 逐顶点描边宽度 = **降级** | **已确定** | 描边宽度走 **全局 + MaterialType 分档（16 档）**；不为逐顶点宽度改动 GBuffer | MaterialType 有 16 个值足够分档；为逐顶点改 GBuffer 属 S 级，不值 |
| **D-037** | SSS 背光透光 = **后置**（B 组） | **已确定** | 第一版不做；保留在 13.14 防遗忘 | 8.3 已论证三种"无法分离"全不成立 → 纯 shader，后期补零编译成本 |
| **D-038** | 描边 × 滤波 = **先滤波，后描边** | **已确定** | Post 链顺序改为 滤波 → 描边（见 3.3） | 滤波只动颜色不动深度/法线，先滤波不影响描边定位；线压在柔化色块上更锐利，符合二游观感；**零成本，swap 两行** |
| **D-039** | Dither × VSM 阴影 = **接受跳变** | **已确定** | 首版接受淡出时阴影整块消失 | 事实已确认（D-029）：ShadowDepth 不走 Dither Mask。单独做阴影淡出为 A 级且不保证干净；淡出多为瞬时演出 |

## 12.7 决策变更流程

1. 发现新证据 → 2. 建立 Decision Change → 3. 确认受影响 D 编号 → 4. 分析兼容影响 → 5. Golden / Perf Test → 6. 批准或拒绝 → 7. 更新架构文档 → 8. 实施迁移

修改记录：

| 日期 | 编号 | 原决定 | 新决定 | 新证据 | 影响 |
| --- | --- | --- | --- | --- | --- |
| 2026-09-14 | D-001 | 无 | 自定义 SM | Epic 原生 Toon 限制 | Fork UE |
| 2026-09-14 | D-020 | 无 | MooaToon 仅作 5.7 参考 | 实际 Diff | 需 5.8 rebase |
| 2026-09-15 | D-021 | 无 | KEY 身份与同步链待源码验证 | 架构审查 | FaceSDF 前置 |
| 2026-09-15 | D-022 | 无 | Opaque/Masked permutation 契约 | 架构审查 | SM 编译链前置 |
| 2026-09-15 | D-021 | 待查源码 | **已确认**：新增独立 View Uniform 字段 | 5.8 源码：无既有主光概念 | 新增 C++ 字段 |
| 2026-09-15 | D-023 | 无 | GBuffer ID 用 14，NUM=15 | 5.8 源码：MASK=0xF | SMID 分配 |
| 2026-09-15 | D-026 | GI C++ 必改 | GI 只改 Composite | 5.8 源码：Composite 已独立分流 | 减少引擎改动 |
| 2026-09-15 | D-025 | SceneProxy 待定 | 使用原生 CPD | 5.8 源码：36 float 足够 | 去掉一项引擎改动 |
| 2026-09-15 | D-019 | 暂定 | 锁定单 SM + MaterialType | 5.8 源码：Hair/Eye 先例 | 使用 A 高 4 位 |
| 2026-09-15 | D-031 | 无 | 文档分层 | 框架未定时细化算法必然返工 | 文档重构 |
| 2026-09-16 | **D-022** | **待验证** | **已确认** | **C-05 闭合：无 Toon 专用 permutation 维度** | **阶段 1 硬阻塞解除** |
| 2026-09-16 | **D-029** | **待验证** | **已确认（事实）+ 待拍板（政策）** | **源码已证实 ShadowDepth 不走 Dither Mask** | **政策改由 D-039 承载** |
| 2026-09-16 | **D-032** | 待拍板 | **已确定：选项 B（语义值）** | 不让美术在材质图做位打包 | B1 阻塞解除；MP_ToonData 四分量语义锁定 |
| 2026-09-16 | **D-033** | 待拍板 | **已确定：选项 A（ILM.G→Specular、ILM.B→Metallic）** | ILM.G 是“像不像”第一决定因素，须保留完整 8 bit；AO 槽受静态光照干扰 | 真实直接光阻塞解除 |
| 2026-09-16 | **D-034** | 待拍板 | **已确定：首版单 Ramp（RowCount=1）** | 无美术、TA 为唯一决策人，简化优先 | **新增 4.1.1 强制约束：RampRowID 位保留填 0** |
| 2026-09-16 | **D-035** | 待拍板 | **已确定：材质内采样，只传最终标量** | 二游脸影为风格化，多光源翻转非所需 | Face 光照阻塞解除 |
| 2026-09-16 | **D-036** | 待拍板 | **已确定：降级为全局 + MaterialType 16 档** | 为逐顶点宽度改 GBuffer 属 S 级，不值 | B6 阻塞解除 |
| 2026-09-16 | **D-037** | 待拍板 | **已确定：后置（B 组）** | 8.3 三种“无法分离”全不成立，纯 shader 零编译成本 | 防遗忘见 13.14 |
| 2026-09-16 | **D-038** | 待拍板 | **已确定：先滤波，后描边** | 滤波只动颜色不动深度/法线，先滤波不影响描边定位 | **3.3 Post 链顺序已改**；零成本 |
| 2026-09-16 | **D-039** | 待拍板 | **已确定：接受阴影跳变** | 单独做阴影淡出为 A 级且不保证干净；淡出多为瞬时演出 | Fade 阻塞解除 |

任何锁定决策改变都必须增加记录。

## 12.8 错误与失败策略

| 失败情况 | 首期处理 |
| --- | --- |
| CustomData 未分配 | 检查五处白名单 |
| CustomData 全 0 | 同上，白名单遗漏是首要嫌疑 |
| CustomData 精度不足 | 删除非 P0 数据，不立即扩 RT |
| Flags 位恢复错误 | 检查是否用了线性采样，改为 point / load |
| Opaque / Masked 行为不一致 | 停止实现，先修 permutation / BasePass |
| ShadingModelID 不够 | 单 Toon SM + MaterialType / Flags |
| 误入原生 Substrate Toon 分支 | 检查是否复用原生符号，改为独立命名与 ID 14 显式比较 |
| SMID 显示为 13 | 检查四个映射位置是否全部改到 14 |
| KEY Direction 无法进入 BasePass | 停止 FaceSDF，先完成 D-021 数据链 |
| KEY 方向残留旧值 | 检查 SetupView 是否无条件初始化，检查弱引用 |
| 多 View 串数据 | 检查是否用了全局静态变量，改为按 View 保存 |
| HeadBasis CPD 不够 | 退回 Material 参数 |
| CPD / Material 都不够 | 启动 SceneProxy 扩展评审 |
| 后处理完全没生效 | 检查 SVE 是否用了旧重载 |
| Post 无法读取 GBufferD | 已确认可读；若失败检查 SceneTexture 参数绑定 |
| Outline 与画面错位 | 检查 ViewRect / BufferSize / UV 映射；确认在 TSR 前 |
| 描边被后续滤波糊掉 | **不应发生**：D-038 已确定先滤波后描边（3.3）。若仍糊，检查 Post 链顺序是否未同步 |
| GI 分流失败 | GI 独立延期或退回 Stylized Ambient |
| GI 影响了场景 Lumen | 停止，检查 SMID 判断是否正确 |
| Lumen 5.8 接口变化过大 | 重做调用链定位 |
| TSR ghosting | 调整执行位置 / 降低历史依赖 |
| Ramp 硬边界在 TSR 下拖影 | 调整 feather 宽度，或降低历史权重 |
| 极窄头发高光闪烁 | 检查八面体量化误差，加宽高光或接受并回归验证 |
| FaceSDF 出现色阶 | 加宽 feather，做阈值区间重映射 |
| Hair Shadow 闪烁 | 降投影距离，Geometry 后置 |
| Screen Outline 断线 | 调整 Depth / Normal / Mask 组合 |
| Post 色彩空间错误 | 停止该 Pass，重定义输入空间 |
| Shader permutation 缺失 | 建 Opaque / Masked 编译矩阵 |
| Dither 后阴影突然消失 | **已知且首版接受**（D-039）。事实：ShadowDepth 不走 Dither Mask。若需修，走 12.7 变更流程，评估单独做阴影淡出（A 级） |
| VSM 异常 | 回退 UE 原生 Shadow Term |
| Debug 无法定位 | 禁止继续叠功能，先补 Debug |
| GPU 超预算 | 按 ΔA~ΔF 逐模块剥离 |

## 12.9 修改摘要

### 归档修订 1（2026-09-14）

1. 状态标记统一成纯文字。
2. 增加执行 Agent 总规则。
3. 施工计划扩成阶段化任务单。
4. 增加修改摘要制度。
5. YivanLee Coverage 增加施工阶段。
6. 明确阶段之间必须人工确认。

### 归档修订 2（2026-09-15，Codex 源码审查结果整合）

1. 闭合三个 P0：KEY Light 链、ShadingModelID、UNorm8 量化。
2. GI 范围缩减为只改 Composite。
3. HeadBasis 确认使用原生 CPD。
4. CustomData.A 非 Face 分支回收高 4 位存 MaterialType。
5. 确认 CustomData 白名单是硬性步骤。
6. 新发现五个坑：SVE 旧重载静默失效、Overlay 无 DepthBias、Dither 对 VSM 无效、TSR 后分辨率不一致、`IntegrateBxDF` 已弃用。

### 归档修订 3（2026-09-15，文档分层重构）

1. 按改动代价重新划分文档层次。
2. 引擎改动总清单提前为第 2 章。
3. 框架契约集中在数据契约 / Permutation / 数据通道三章。
4. Shader 算法与参数默认值剥离为第二部分的单章。
5. 新增 D-031，Agent 总规则新增第 9 条。
6. 新增 7.4 与美术对接要求。

### 归档修订 4（2026-09-16，C-05 Permutation 全链闭合）

1. D-022 锁定已确认，阶段 1 硬阻塞解除。
2. 新增 Permutation 全链、MIR Translator 必改点、漏编译断点表、最小验收矩阵。
3. 新增最高风险断点：C++ 枚举自然值 13 与原生 Substrate Toon 冲突，必须四处显式映射。
4. E-1 扩为 10 个文件，E-2 白名单扩为 5 处。
5. 确认 Opaque / Masked 无专用 permutation，Masked 验收须用非恒定 OpacityMask。

### 归档修订 5（2026-09-16，C-01 + C-07 + C-09 施工图闭合）

1. 新增符号级施工图，E-1 扩为 10 文件、E-2 白名单 5 处、E-3 八处、E-5 四处。
2. 新增硬约束：`MP_ToonData` 不得用 `FColorMaterialInput`。
3. 新增提交批次 B0–B6，明确 B0 为最快粉色球批次。
4. 新增 CustomData producer → consumer 全链与 16 条静默失败点。
5. **新发现：VertexColor.A 未进入 CustomData 契约。**

### 归档修订 6（2026-09-16，全局自洽性重构）

1. **重做状态标记体系（根因修复）**：把被滥用的"待验证"拆为 `待查源码` / `待拍板` / `待实验` 三类，并写入使用规则。
2. **新增 0.6 当前待办总表**：待拍板 / 待查源码 / 待实验 / 非技术前置四栏，作为全局进度单一入口。
3. **修复四处状态矛盾**：D-022 待验证→已确认；D-029 待验证→已确认（事实）+ 待拍板（政策，下沉为 D-039）；变更记录 D-021 同步；Opaque/Masked permutation 待验证→已确认。
4. **补 8 个决策编号**：D-032 ~ D-039，覆盖此前"P0 却无编号无归属"的孤儿项（原 4.8 节并入决策表）。
5. **补两个漏记项**：SSS 背光透光（8.3 + 13.14，含"为什么不进第一版"的三条件判断）；Ramp 明暗硬边界 × TSR（3.5 时序冲突登记，明确与"描边 × TSR"是两个独立问题）。
6. **新增 3.5 时序冲突登记**：把时序冲突从散落各处收拢为框架层一节，含 D-038（描边被滤波糊掉，文档先前顺序存在实质缺陷）。
7. **结构调整**：源码确认事实汇总与 Touch Map 后移为第 11 章速查（原第 8 章），设计内容前置；删除 12.10 中已完成的审查输出模板。
8. **Agent 总规则新增第 10 条三问巡检与第 11 条待办同步**，防止增量回填再次产生自相矛盾。
9. **新增 2.5.6**：明确"公共 shader helper"与"空壳类"的区别——建议第一版就建 helper，这不是提前实现功能。
10. 新增 11.2 算法参考来源映射保留、12.2 补"算法为什么能脱离引擎改动"的核心逻辑与三种无法分离的边界。

### **v0 定版（2026-09-16，第一版执行基线）**

1. **版本号统一为 v0**：此前 v0.3 ~ v0.8 全部为演进草稿，其技术内容合并归位到本版；历史修订记录降级为"归档修订 1~6"，仅作溯源用，不影响本版条款。
2. **关闭全部 8 个待拍板项 D-032 ~ D-039**（决策内容与理由写入 12.6，变更记录写入 12.7）：
   * D-032 = B（语义值，BasePass 集中编码）
   * D-033 = A（ILM.G→Specular，ILM.B→Metallic）
   * D-034 = 首版单 Ramp（RowCount=1），绑定走 View 参数通道
   * D-035 = 材质内采样，只传最终标量
   * D-036 = 降级为全局 + MaterialType 16 档宽度
   * D-037 = 后置（B 组，防遗忘见 13.14）
   * D-038 = **先滤波，后描边**
   * D-039 = 接受阴影跳变
3. **新增 4.1.1 强制约束**：首版单 Ramp 下，A 通道低 4 位 RampRowID **保留并固定填 0，禁止腾作他用**。理由：将来加多行色阶，保留是 C 级、删掉是 A 级，而首版选单 Ramp 只是因为无美术，并非论证过单行够用。
4. **3.3 Post 链顺序调整**（D-038）：Screen Outline 与 SNN/Kuwahara 对调，改为"先滤波后描边"，并写入顺序依据。零成本。
5. **3.5 时序冲突登记**：D-038 由待拍板改为已确定；D-039 状态同步。
6. **0.6 待办总表**：待拍板栏清空；新增 Q5（默认 material translator profile）与 Q6（GBuffer 各 MRT 分工与格式）两项待查源码；非技术前置新增"共享 DDC / 统一编译环境"与"里程碑必须干净 DDC 或真 cook 复验"。
7. **附录改写为 A.1~A.4**：明确 v0 为唯一有效版本，禁止引用旧版本号。

## 12.10 Codex 源码审查清单

### 已完成项

| 编号 | 审查任务 | 结果 |
| --- | --- | --- |
| C-03 | 验证关键 GI 符号 | 三文件均存在且连通 |
| C-04 | 验证 GBufferD Post 可读性 | 三种时机均可绑定 |
| C-06 | 追 KEY Direction 数据链 | 见 6.1 |
| C-08 | 追 Primitive 数据链，判断 CPD 是否足够 | CPD 足够 |
| C-05 | 追 Toon SM permutation 全链 | **D-022 已确认，阶段 1 硬阻塞解除** |
| C-01 | 文件级落点补成符号 / Struct + 插入位置 | 见第 2 章 |
| C-07 | 追 CustomData producer / consumer | 见 4.7 |
| C-09 | 标注施工类型与提交批次 | 见 2.6 |

### 待完成项

| 编号 | 审查任务 | 阻塞 | 时机 |
| --- | --- | --- | --- |
| C-10 | Screen Outline 时机与 UV 映射：TSR 前挂哪个 Pass 枚举；ViewRect / BufferSize / UV 换算写法 | B6 | Outline 开工前 |
| C-12 | Eyebrow DepthBias 材质侧方案：5.8 可用手段、Deferred+TSR 下最稳者、副作用 | Eyebrow | Eyebrow 开工前 |
| C-13 | Dither 与 VSM 阴影：有无可复用开关、两方案代价比较 | Fade | Fade 开工前 |
| C-02 | MooaToon 5.7 → Epic 5.8 rebase 对账，**收敛到 GI / GBuffer / Deferred 等真正参考点，不做全量** | 阶段 7 | 阶段 7 前 |
| C-11 | CPD 索引与初始化约定（**已降级为项目 Contract**，Codex 只做 sanity check） | 否 | 随时 |

**C-02 HIGH 判定标准**：只有满足以下之一才标 HIGH——MooaToon 和 Epic 5.8 同时修改同一函数、同一 Shader Entry、同一关键 Struct、同一 Uniform Parameter Struct、同一 GBuffer Encode / Decode 区域、同一 permutation gate。仅"同文件"不能直接判 HIGH。

### 六项容量调研（Q1–Q6）

| 编号 | 问题 | 为什么现在要想 |
| --- | --- | --- |
| Q1 | flags 8 位是否够用（已用 7，剩 1） | 扩位 = 改 GBuffer = 最贵 |
| Q2 | GBufferD 每通道 8 位是否长期方案 | 改格式属最贵一档 |
| Q3 | 原生 CPD 36 float 是否够用（现用 6） | 用超需改 SceneProxy |
| Q4 | GI 最小原型只改 Composite 的判据与失败信号 | 决定 B5 是否升级为 C++ 改动 |
| **Q5** | **5.8.2 默认 material translator profile（Classic / MIR）与开关名、默认值** | **决定"漏改 Classic"是否立刻暴露；CVar 名跨版本变过，必须源码直证。B0 验收矩阵可信度** |
| **Q6** | **GBuffer 各 MRT 的精确分工与格式（A/B/C/D/E 分别装什么）、Selective Outputs 在 5.8 的表现** | **服务于 4.6 的 8 处检查；Debug 与 Post 读 GBuffer 时会用到** |

> **Q5 特别说明**：任何 AI 报出的 translator 开关名都不可直接采信——该 CVar 在 UE5 各版本间更名过。必须由 Codex 在 `++UE5+Release-5.8` 分支源码中直接确认，并回填到此处与 11 章速查表。

## 12.11 项目代码目录

```text
Source/<Project>/Toon/
  Core/          ToonTypes.h, ToonSettings.h, ToonSubsystem.h
  ShadingModel/  Public, Private, Tests
  Material/      Public, Private, Tests
  Lighting/      Public, Private, Tests
  GI/            Public, Private, Tests
  Outline/       Public, Private, Tests
  Post/          Public, Private, Tests
  ViewExtension/ ToonSceneViewExtension
  Debug/         ToonDebugView, ToonShaderPrint, ToonConsoleCommands
  Tests/         Automation, GoldenScene, Performance
```

Shader：

```text
Shaders/Toon/
  Common/    ToonCommon.ush, ToonQuantization.ush
  Lighting/
  GI/
  Outline/
  Post/
  Debug/
```

原则：**Engine Fork 修改 ≠ Project Toon Runtime，两者分开维护。**

## 12.12 Engine Fork 维护规则

### 改动分层

```text
Engine Patch
├─ SM Registration
├─ CustomData Whitelist
├─ Material
├─ GBuffer
├─ View / Scene Parameters（KEY Light）
├─ Deferred Lighting
├─ GI（仅 Composite shader）
└─ Optional MeshPass
```

### 禁止

* 项目业务逻辑散落 Renderer
* 美术参数硬编码 Engine Shader
* Post 功能全部进入 Engine
* 为未来需求提前修改 SceneProxy
* 为未验证需求提前新增 RT
* 复用原生 Substrate Toon 符号
* 依赖 `SUBSTRATE_ENABLED` 作为唯一隔离手段

### 优先

* Engine：提供底层能力与数据通道
* Project Toon Module：提供项目策略
* Settings / Subsystem：提供全局配置
* CPD：提供轻量每角色数据
* Material：提供每材质参数
* Texture / Vertex：提供逐像素数据

---

# 第二部分：Shader 实现层

本部分是唯一允许自由迭代的区域。所有内容在框架层冻结后展开，改错只影响画面，可热重载。

**本部分当前只定义接口：每个模块需要什么输入、产出什么、参考谁的算法。具体公式、参数默认值、调参策略暂不设计，等到对应阶段实现时再展开。**

输入一律引用第一部分已冻结的契约字段，不得在本部分重新定义数据来源。

---

# 13. Shader 实现待办

## 13.1 Ramp 与 ILM 明暗

**效果目标**：把连续的 NdotL 明暗切成美术可控的若干块纯色，暗部是另一个颜色而不是亮部变暗。

**输入**：NdotL（来自 KEY）、ILM 四通道（见 7.3）、BaseColor、Ramp 贴图、RampRowID（来自 CustomData.A 低 4 位，见 4.3）。

**输出**：Toon 明暗颜色，供 Direct Lighting 累加。

**算法源**：GenshinCelShaderURP 为主，UTS 的 Step / Feather 为辅。

**第一版范围**：Ramp 采样、Step、Feather、ILM AO、ILM 高光强度。Ramp 行寻址必须遵循 4.4 的 `(RowID + 0.5) / RowCount` 行中心规则。

**待定**：阶数（2 阶还是 3 阶）、Step 阈值、Feather 宽度、Ramp 存相对系数还是最终颜色。

## 13.2 Face SDF 求值

**效果目标**：脸部阴影是干净规整的一条弧线，光源绕头转一圈不出现脏黑块或跳变。

**输入**：FaceUV、FaceSDF 贴图、HeadForward 与 HeadRight（来自 CPD，见 6.2）、KeyLightDirection（来自 View Uniform，见 6.1）。

**输出**：FaceResult，写入 CustomData.A 完整 8 bit（见 4.3）。

**算法源**：kaze-mio。

**第一版范围**：只需响应 KEY，Point / Spot 不翻转脸部阴影（D-009）。

**待定**：头空间转换的具体形式、SDF 阈值比较方式、feather 宽度（必须跨越数个量化级以避免色阶）。

## 13.3 Hair 与 Eye

**效果目标**：头发有沿发丝方向流动的带状高光（天使环），眼睛有稳定形状的高光点，逆光转正时不闪。

**输入**：八面体编码切向（来自 CustomData.G/B，见 4.1）、Hair Tangent、Eye 相关材质参数、KEY。

**输出**：Hair Spec、Eye Highlight，累加进 Direct Lighting。

**算法源**：UE_CelLit。八面体编解码复用 UE 原生函数（见 4.4）。

**第一版范围**：Kajiya-Kay 头发高光、虚拟眼球法线、固定形状眼高光。

**待定**：极窄高光在 8 bit 八面体精度下是否闪烁；如闪烁，是加宽高光还是改用其他编码（待实验）。

## 13.4 Rim 与 Screen Hair Shadow

**效果目标**：角色边缘浮出一圈光边；前额有刘海投下的影子。

**输入**：法线、视角方向、Rim 相关参数、屏幕空间深度与 Toon Mask。

**输出**：Rim 强度、刘海阴影，累加进 Direct Lighting。

**算法源**：Rim 参考 Genshin / kaze-mio；刘海阴影参考 UE_CelLit 的屏幕空间做法。

**第一版范围**：屏幕空间发影（D-010），几何发影后置。

**待定**：Rim 是只作用于剪影还是可按深度控制；屏幕空间发影在 TSR 下是否 ghosting。

## 13.5 KEY 与 FILL 光照政策

**效果目标**：主光决定造型，补光只提亮，不出现补光一打脸就变成另一套明暗。

**输入**：KEY 与 FILL 的 Deferred Lighting 结果、FillLightWeight（见 6.4）。

**输出**：最终 Direct Lighting。

**算法源**：HoyoToon 的多光源政策。

**第一版范围**：KEY 为美术指定唯一方向光，FILL 为 Point / Spot，FILL 只做加法并受 FillLightWeight 控制。

**待定**：FILL 的 clamp 上限；是否需要 Character-only Fill 的自定义 Light Tag（见 Lighting Channels 限制）。

## 13.6 Toon GI

**效果目标**：角色环境光可独立调节，不被 Lumen 的连续反弹光洗掉 Ramp 切出来的色块。

**输入**：Lumen Diffuse Indirect、Stylized Ambient、ShadingModelID、CharacterGIWeight。

**输出**：Character Indirect，替代原 Lumen 结果。

**算法源**：最小原型自研；完整版参考 MToon 的 Stylized SH 与 MooaToon 的 GI 思路。

**第一版范围**：只改 `DiffuseIndirectComposite.usf`，证明 Toon SM 可独立缩放 Indirect（D-026）。基础形式为在 Stylized 与 Lumen 之间按 CharacterGIWeight 混合。

**待定**：Stylized Ambient 的具体形式；Directionality、Saturation、Shadow Compression 等高级项是否进入第一版。

## 13.7 Screen Outline

**效果目标**：角色外有一圈轮廓线，粗细随距离变化，近处用材质色、远处统一淡出。

**输入**：SceneDepth、SceneNormal、Toon Mask（来自 GBufferD，见 4.1）、全局描边参数（见 6.4）。逐顶点宽度（VertexColor.A）**当前契约未支持**，见 D-036。

**输出**：叠加到 HDR SceneColor，执行位置固定在 TSR 之前（见 3.3、3.4）。

**算法源**：宽度与颜色的 UX 参考 UTS / lilToon / HoyoToon；边缘检测自研。

**第一版范围**：屏幕空间描边，含 Width、Color、LineLOD、ColorLOD、DPI / FOV 缩放。

**待定**：边缘检测算子；UV 换算写法（C-10 输出）。
**已定**：与滤波的顺序 —— **先滤波后描边（D-038 已确定）**；淡出时描边行为 —— **接受（D-039 已确定）**。

## 13.8 Eyebrow 与 Dither Fade

**效果目标**：眉毛透过头发显示在最前面；角色可平滑淡入淡出。

**输入**：Eyebrow 颜色与透明度、HairMask、DitherOpacityMask、材质参数。

**输出**：Eyebrow Overlay 绘制；角色淡出。

**算法源**：Eyebrow 走 UE 原生 OverlayMaterial（D-013）；Fade 走原生 DitherOpacityMask（D-014）。

**第一版范围**：Eyebrow 用 Overlay，接受双 Draw 与 `CastShadow=false`，DepthBias 必须走材质侧（D-027，方案由 C-12 输出）；Fade 仅对 Masked 生效。

**待定**：材质侧 DepthBias 的具体做法；VSM 阴影不跟随淡出的政策（D-039，由 C-13 输出）。

## 13.9 Multi-Layer 与 BxDF 内部层

**效果目标**：金属、宝石等部件保留写实反光，与卡通部分自然混合。

**输入**：MaterialType（来自 CustomData.A 高 4 位）、Flags（见 4.2）、材质参数。

**输出**：在单个 Toon BxDF 内按 MaterialType 分流的多种着色结果。

**算法源**：结构参考 MooaToon 的 BxDF 分层；类型分流参考 UE 原生 Hair / Eye / ClearCoat 的单 SMID 多类型模式。

**第一版范围**：实验项。只验证单 SM 多类型的数据通路，不做完整 Multi-Layer。

**待定**：BxDF 内部层的组合方式；PBR Spec 与 Toon Spec 如何混合不互相破坏。

## 13.10 Toon Post

**效果目标**：卡通泛光、油画笔触、保边降噪、水彩等整体画面风格化。

**输入**：HDR 或 Display 空间颜色，各 Pass 必须声明 Color Space（见 3.7）。

**输出**：风格化后的画面。

**算法源**：Bloom 参考 kaze-mio；Kuwahara 参考 noxtgm；SNN 参考 kafues511 / t-takasaka；Watercolor 参考 KinoAqua。

**第一版范围**：全部后置（D-015）。只建立 Post Framework 与参数链（见 6.4），不实现具体算法。

**待定**：各算法的 Color Space 与 History 属性；曝光依赖项的补偿方式。
**已定**：与描边的执行顺序 —— **滤波在前、描边在后（D-038 已确定）**。

## 13.11 Exposure 原则

算法需要 scene-referred value 时，在 Pass 内部自行做 Exposure Compensation，避免对整张 SceneColor 统一做 EyeAdaptationInverse。

必须单独确认：Bloom threshold、Pigment、Paper、Grain。

## 13.12 Debug View 实现

**效果目标**：一键把画面切成调试模式，单独查看进入 GBuffer 的每一项数据。

**输入**：9.2 节列出的 12 个 Debug 通道对应的数据字段。

**输出**：对应通道的可视化画面。

**原则**：任何进入 GBuffer 的数据必须有可视化方式。实现方式待定。

## 13.13 SSS 背光透光（后置，本版新增）

**效果目标**：逆光时耳廓、发梢、手指透出暖色。

**输入**：光方向、法线、视角、厚度 / 透射参数（走贴图材质输入，不占 GBuffer）。

**输出**：透射亮度，累加进 Direct Lighting。

**算法源**：待定（可参考 MooaToon / lilToon 的透射实现）。

**当前状态**：第一版不做（**D-037 已确定：后置**）。属 B 组——后期补只改 shader，零编译成本。

**记入文档的原因**：这是截图对照法查出的唯一明显需求缺口。不记下来就会被遗忘。

## 13.14 效果参数与仲裁权重

第一版所有美术可调参数暂不定义默认值与取值范围，实现时按以下分类补充：

| 类型 | 含义 | 设计要求 |
| --- | --- | --- |
| 效果开关 | 这个效果开多强 | 只需一端好看 |
| 仲裁权重 | 两个方案怎么混合 | **两端方案都必须完整可独立工作，中间过渡也要合理** |

已知属于仲裁权重的参数：`CharacterGIWeight`（Stylized 与 Lumen 之间）。

**防参数爆炸规则**：

1. 只有真正改变画面风格的才做成美术参数。
2. 技术性参数（feather 宽度、阈值、UV epsilon）内部定死或只给 TA。
3. 每个参数必须补充：默认值、0 与 1 分别长什么样、归属（美术还是 TA）。
4. 必须提供预设组合，让美术从预设出发微调，不从零开始。

**待定仲裁项**：

| 冲突 | 是否已定 |
| --- | --- |
| GI 污染色块 | 已有 `CharacterGIWeight` |
| 描边 × 后处理滤波 | **D-038 已确定：先滤波后描边** |
| Rim × 快速转镜头 | 待定，可能需要视角淡出参数 |
| 描边 × Dither Fade | **D-039 已确定：接受** |
| Ramp 硬边界 × TSR | 待实验，可能靠 feather 宽度解决而非权重 |

---

# 附录 A：v0 定版说明

## A.1 本版地位

v0 是**第一版执行基线**。此前的 v0.3 ~ v0.8 全部为演进中的草稿，其技术内容已合并归位到本版，历史修订记录保留在 12.9（标记为"归档修订"），仅作溯源用，**不影响本版条款**。

引用文档时一律以 v0 为准。任何 Agent 或人引用"v0.7 说……""v0.8 里写过……"均无效。

## A.2 本版相对草稿阶段解决的四类问题

1. **状态标记自相矛盾**：正文已闭合、标签仍写"待验证"（D-022、D-029、D-021、Opaque/Masked）。
2. **P0 项无编号无归属**：4 个 P0 未锁定项不在决策表、不在待办清单，必然被遗忘 → 补为 D-032~D-035。
3. **对话结论未落笔**：SSS 缺口、Ramp×TSR 冲突，只在对话里想清楚，文档一个字没有。
4. **"待验证"被滥用于三种情况**（根因）→ 拆为 `待查源码` / `待拍板` / `待实验`。

**"漏东西"和"文档自己跟自己打架"是两种性质的问题**，后者更糟：漏顶多是缺，打架会让人和 Agent 照着错的做。

## A.3 本版关掉的 8 个待拍板项

D-032 ~ D-039 在本次定版中全部拍板关闭（决策内容与约束见 12.6）。关闭后：

* 0.6 待办总表中"待拍板"栏为空。
* B1（材质数据契约）与真实直接光的阻塞解除。
* 剩余未决项全部为 `待查源码`（Codex 执行）与 `待实验`（实现后跑起来看）。

## A.4 全局结构约定

* **0.6 当前待办总表**：打开文档第一页就知道还差什么、归谁、阻塞哪个批次。
* **3.5 时序冲突登记**：时序属框架层——挪一个 Pass 可能比加参数更省事，也可能推翻方案。
* **Agent 总规则第 10 条三问巡检**：防止增量回填再次产生矛盾。
* **4.1 强制约束**：RampRowID 位保留填 0，禁止腾作他用。
