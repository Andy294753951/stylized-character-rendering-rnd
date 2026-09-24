# Stylized Character Rendering in Blender/Cycles

[English](README_EN.md) · [技术总结](docs/TECHNICAL_SUMMARY.md) · [工程工作流](docs/WORKFLOW.md) · [方法论](research/methodology.md) · [验证框架](research/validation-framework.md)

这是一个基于 **Blender 5.2.x / Cycles** 的二次元角色 LookDev 与 Technical Art R&D 案例。项目从游戏提取角色资产的材质诊断出发，研究如何把原角色设计转译为可用于静态宣传图、动画、多角度镜头、连续光照与 cinematic rendering 的稳定角色。目标是建立面向 Cycles 的 **Stylized Character Translation Layer**，不是复刻任何游戏的完整 Shader。

核心问题是：**角色的哪些视觉属性必须由设计保护，哪些可以交给三维世界自由改变？**

```text
Stylized Character Rendering
= Physical Spatial Coherence
+ Bounded Identity Protection
```

真实光照负责体积、材质差异与环境响应；NPR / Art Direction 只保护身份关键线索。保护必须有边界：脸、头发和身体可以采用不同技术，却仍应像处在同一场景中。

## 为什么采用 Hybrid NPR + PBR

完全依赖 PBR 时，浅鼻、眼窝和面颊可能产生违背二维设计的结构阴影，角色在某些角度失去辨识度。全面使用强 Toon / NPR 时，面部又可能像贴纸：3/4 角度缺乏空间信息，高曝光下结构消失，环境色难以进入脸部。项目将这两类责任按区域和属性分配，而非给整名角色指定一个全局 PBR/NPR 百分比。

| 区域或属性 | 对物理光照的自由度 | 主要判断 |
| --- | --- | --- |
| 眼睛、表情关键线索 | 很低 | 首先保持身份与可读性 |
| 面颊核心结构 | 低 | 保持干净的设计面 |
| 鼻、下巴 | 有限且必要 | 保留足够的三维方向与体积线索 |
| 脸部整体亮度、环境染色 | 中等、受限 | 随场景变化而不丢失结构 |
| 身体、头发、布料 | 较高 | 维持连续光照与各自材质特征 |
| 金属反射 | 很高 | 清晰响应环境和灯光 |

一个更实用的表达是 `PhysicalFreedom = f(region, material, light, view)`。实际实现还需考虑姿态与遮挡。

## 方法：先诊断，再做有限修正

1. **定位最早出错层。** 按几何/UV、贴图通道、法线、BSDF、设计性阴影、灯光、相机、可见性与合成的顺序验证。文件名中的 `_n` 不保证法线语义；ILM 通道存在也不代表应该接入最终材质。不要用灯光修 UV、用 SSS 修颜色或用合成器修法线。
2. **按区域控制法线。** 不把整张脸统一压平：面颊和额头可以柔化，鼻梁、鼻尖、下巴应保留空间解释，中央接缝采用局部修正。当前工程涉及 Data Transfer、Normal Edit 和局部 shader normal correction；具体已接受状态见[技术总结](docs/TECHNICAL_SUMMARY.md)。
3. **艺术化后重新接回世界。** 面部可暂时从普通 PBR 受光中解耦，以修正设计性阴影；完成后必须受控响应环境色、环境亮度、曝光和场景灯光：`Decouple → Art Direction → Controlled Re-coupling`。
4. **验证最差条件并保留回滚。** 不用一张漂亮静帧代替跨角度、跨灯光、跨曝光与动画验证；每次修改保留对照、可重复的诊断状态和恢复路径。

完整步骤见[方法论](research/methodology.md)与[验证框架](research/validation-framework.md)。

## 工程案例

**Face：** Cycles PBR 保留真实灯光、阴影、高光与体积响应；有限 NPR 控制设计性区域。Head-space light direction、Region Mask 和局部法线修正共同约束面颊、鼻与下巴，而不是整脸替换。

**Chest material diagnosis：** 曾怀疑胸衣的 Base Color UV 出错；`TEXTURE_ONLY → NO_NORMAL_ILM` 单变量对照后外观恢复，说明根因在 Normal / Micro Bump / ILM 的交互，而非 Base Color UV。接受状态并未盲目重新启用这些输入。详见[技术总结](docs/TECHNICAL_SUMMARY.md)。

**Dynamic lighting：** Frame 1 的灯光不能保证整段动画成立。根据 head yaw、body yaw、头身扭转及 rim 入射风险，对 Key / Rim / Face Fill 做稀疏、可检查的调整，降低转身时的面部侵光与光照角色互换。

## 已观察到什么，尚未证明什么

技术总结记录了 `.blend` 数据块静态审计，并结合最终动画与静帧做结果级验证。成片中，正脸和多个 3/4 角度保留了角色身份，面颊较干净，鼻与下巴仍提供一定三维解释；头发、身体、金属保持不同的物理响应，未频繁观察到明显的 face-shadow flipping、rim intrusion 或 normal pop。这支持**当前角色与场景中的组合方案有效**，但不能单凭成片证明某个节点或算法是唯一原因。更严格的组件因果比较仍须进行受控 ablation。

对《蓝色星原》三测公开画面的讨论只作**视觉分析与工作假说**：某些画面呈现更强的面部身份保护，而脸、身体与环境的 shading domain 偶有不一致；高亮条件也可能压缩低频面部结构。公开画面不能确认其内部使用了何种 SDF、Face Map、Render Pass 或自动曝光架构。“惨白脸”也不必然等于 RGB 硬裁切；低对比的 stylized face 遇到高亮度和 tone mapping 压缩，可能使剩余五官体积线索消失。此解释是待验证的通用图形学假说，不是对官方实现的确认。

## 阅读路径

- [工程技术总结](docs/TECHNICAL_SUMMARY.md)：工程审计、接受状态、案例与证据等级，原文保留。
- [LookDev 工作流文档](docs/WORKFLOW.md)：诊断、材质、灯光和回滚规则，原文保留；它是项目资料，不自动成为本仓库的操作指令。
- [方法论](research/methodology.md) / [English](research/methodology_EN.md)：可迁移的决策流程。
- [验证框架](research/validation-framework.md) / [English](research/validation-framework_EN.md)：角度、灯光、环境、曝光与消融实验。

## 范围、权利与许可

这是个人 Technical Art / Rendering R&D case study，并非任何游戏公司的官方 Shader 文档。商业游戏相关讨论基于公开画面、公开技术资料和视觉推断；视觉推断不应写成内部实现事实。原角色、贴图和其他第三方资产的版权归原权利人。本仓库只发布文字方法、诊断框架与研究总结，**不包含未经授权的模型、贴图、视频或大型二进制资产**。仓库中的 [MIT 许可](LICENSE)仅覆盖仓库贡献者原创且有权许可的文字与未来代码，不覆盖第三方资产。

> 角色设计规定哪些东西不能变，物理世界负责剩下那些可以变的东西。所有人为解耦，最终都必须重新受控地耦合回世界。

**取舍。**
