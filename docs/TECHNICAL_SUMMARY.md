# 米砂最终工程技术总结

**工程文件**：`0923_material_restored_v2.blend`  
**目标环境**：Blender 5.2.x / Cycles  
**审计方式**：直接解析 Blender 5.2 `.blend` 的 DNA/SDNA、Scene/Object/Material/NodeTree/Image/Text 等数据块，并读取工程内嵌的诊断脚本、报告和回退记录。当前分析环境没有 Blender 可执行程序，因此主体仍是**数据块级静态技术审计**。后续另结合最终渲染视频/静帧进行了**画面级结果验证**，但画面观察不能反向证明某个节点或算法是唯一因果来源。

**修订原则**：本文明确区分四种证据等级——①工程内事实；②最终成片观察；③公开技术资料支持；④对《蓝色星原》官方渲染的工作假说。除非有一手资料，否则不会把第④类写成官方实现事实。

---

## 1. 一句话结论

这不是一个“给恋活/提取模型套几个 PBR 节点”的工程，而是一套逐步演化出来的 **Cycles 动画角色 LookDev + NPR/PBR 混合面部 + 官方贴图语义适配 + 动态灯光/镜头 Rig + 程序化环境 + 非破坏式诊断/回滚** 管线。

最终工程的核心原则已经非常清楚：

> **以真实光照维持空间一致性，以有边界的 NPR/Art Direction 保护角色身份；贴图通道必须验证语义后再接；合成器不承担修复结构性错误的职责。**

可以进一步压缩成：

```text
Stylized Character Rendering
=
Physical Spatial Coherence
+
Bounded Identity Protection
```

这也是整个工程最值得沉淀成 Skill 的部分。

---

## 2. 工程规模与组成

直接读取 `.blend` 数据块得到：

| 数据类型 | 数量 | 说明 |
|---|---:|---|
| Scene | 3 | 原始角色、90mm 露台近景、最终动画场景 |
| Object | 10,228 | 数量很大，但绝大多数不是可见模型 |
| Mesh | 1,142 | 角色、环境、代理、辅助几何 |
| Material | 157 | 含历史版本、诊断副本、最终材质 |
| Image | 60 | 原角色、官方贴图、面部 mask、HDRI、环境纹理等 |
| Light | 50 | 历史灯组、场景灯、角色灯、动画 Rig 灯并存 |
| Camera | 15 | 静态宣发相机、材质检查相机、8 镜头动画 Rig |
| NodeTree | 11 | Shader/Compositor/Geometry Nodes 组 |
| Text | 73 | 大量 Python、报告、README、rollback 状态 |
| Armature | 2 | 角色骨架等 |

对象类型中约 **8,935 个为空对象/辅助对象**。`RigidBodyConstraints` Collection 单独就有约 **8,643 个对象**，因此“1 万多个 Object”并不等于场景有 1 万个可见模型；这是导入模型的物理/约束结构造成的。

### 工程本质上有四层

1. **源角色资产层**：角色 Mesh、Armature、原始贴图、物理/约束。
2. **LookDev 修复层**：脸、皮肤、眼睛、头发、薄纱、金属、法线、接缝修复。
3. **镜头/灯光/环境层**：露台、HDRI、体积光、动态灯光与镜头。
4. **工程化保障层**：副本、A/B 诊断材质、脚本、报告、回退、场景备份。

---

## 3. 三套 Scene：保留基准而不是覆盖

### 3.1 `原始角色 | FINAL_03 · 完整保留`

- Camera：`0922 CINEMA | 90mm 宣发近景`
- World：`米砂 · 低亮中性户外 HDRI`
- Renderer：Cycles
- 1920×1080 / 24 fps
- AgX / `Medium High Contrast`
- Exposure：+0.12
- 当前 Compositor NodeTree：`MSA_Compositor_52_TRUE_PASSTHROUGH`

用途：原始角色/近景审美基准、回退与对照。

### 3.2 `米砂 | 原90mm近景 · 露台`

- Camera：`CAM 02 | 保留原90mm近景`
- 1920×1080 / 24 fps
- 保留三套 View Layer：
  - `01 | 米砂 · 原始受光`
  - `02 | 露台 · 体积日光`
  - `00 | 完整场景预览`
- 曾使用 `米砂 · 近景角色保护合成`
- 当前根 Compositor 仍已切换为真正直通

用途：保留静态 90mm 宣发近景和早期“角色/环境分层合成”方案。

### 3.3 `米砂 | 晨光露台 · FINAL`

这是当前激活的最终 Scene。

- Active Camera：`米砂_LIGHT_RIG_V2_CAM_03`
- Camera 数据：65mm / f5.6 / 跟随 Focus Object
- World：`米砂_LIGHT_RIG_V2_WORLD`
- Renderer：Cycles
- 分辨率：**1800×1500**
- 帧率：**30 fps**
- 当前渲染范围：**480–1000**
- 当前时间游标：270（在渲染范围之外；批量渲染前应再次确认这是有意设置）
- AgX / `Medium High Contrast`
- Exposure：+0.12
- Film Transparent：关闭
- Final Compositor：`MSA_Compositor_52_TRUE_PASSTHROUGH`

### 关键架构结论

工程没有通过“覆盖旧版本”演化，而是通过 **Scene / Collection / View Layer / 材质副本**保留旧版本。这是一个很正确的技术美术工程结构：允许对照、允许回滚，也方便 AI 自动化脚本安全修改。

---

## 4. 最终渲染与合成：现在是零后期直通

最终活动合成树只有 3 个节点、2 条连接：

```text
Render Layers.Image
      ├────────→ Final Output
      └────────→ Viewer
```

工程内报告明确写明：

- Final Output 与 Viewer 吃同一个 `Render Layers.Image`
- 当前 Scene：`米砂 | 晨光露台 · FINAL`
- 当前 Layer：`00 | 完整场景预览`
- 没有 Exposure、Glare、Fog Glow、Color Grade、Alpha Over 等后期处理

### 这件事为什么重要

工程历史里曾经存在：

- `0922 CINEMA | 冰蓝辉光与高光`
- `米砂 · 近景角色保护合成`
- `MSA Project Finisher V2.5` 的 Exposure + Gentle Fog Glow
- 角色层 + 环境层 Alpha Over

这些都还保留在工程/脚本/报告里，但**不代表当前最终输出仍在使用**。

最终选择直通说明：

1. 画面主要由真实场景、材质和灯光完成；
2. 避免了此前 Viewport 与 F12 不一致的问题；
3. 避免合成器把角色皮肤、透明材质、体积光再次加工；
4. 最终结果更容易定位问题。

**未来维护时，不要因为看见旧报告就自动恢复 V2.5 Glow。**

这里的“直通”应理解为**当前项目的诊断/交付基准**，而不是“合成器永远不该参与最终画面”。真正属于 image-space 的轻量 Bloom、色彩 finishing 可以后加；但不能用后期去掩盖 UV、Normal、材质能量关系、透明或角色受光结构的问题。

---

## 5. World：HDRI 负责环境照明，相机看到的是受控背景

两个主要 World 都使用同一逻辑：

```text
forest.exr
  ↓
Hue/Saturation
  ↓
Background ─────────────┐
                        ├─ Mix Shader → World Output
Dark Grey Camera BG ────┘
        ↑
   Light Path / Is Camera Ray
```

技术意义：

- `forest.exr` 保留环境光与反射信息；
- 使用 `Light Path → Is Camera Ray`，让相机可以看到独立的暗灰/受控背景；
- “照明背景”和“相机背景”解耦；
- 不需要为了压背景而破坏 HDRI 对材质的照明贡献。

这是非常适合角色宣发渲染的做法，但它属于**有意的艺术化 ray separation**：相机看到的世界与 glossy/transmission ray 看到的世界可能不同。后续 QA 应检查金属、角膜、头发高光中是否出现与相机背景明显矛盾的环境信息。

---

## 6. View Layer / Collection：角色与环境曾经按职责拆分

最终工程仍保留：

- `00 | 米砂 · 原材质与原灯光保护`
- `01 | 露台场景 · TERRACE`
- `02 | 斜射日光 · 环境专用`
- `03 | Principled Volume 与 GN 微粒`
- `04 | 镜头与焦点`
- `05 | 角色接触阴影 · 渲染代理`
- 多组历史灯光、PROMO 灯组、Face Sculpt 灯组
- `米砂_LIGHT_RIG_V2_精调_1_1000`

旧方案中：

- `01 | 米砂 · 原始受光`：保护角色，不让露台体积与环境直接污染角色；
- `02 | 露台 · 体积日光`：渲染环境、体积、接触代理；
- `00 | 完整场景预览`：完整场景查看。

早期保真测试记录显示，64 samples 下角色不透明区域平均 RGB 差约 0.14%，脸/头发区域约 0.18%。这体现的是一种**角色 LookDev 锁定后，环境尽量不改变角色响应**的思路。

不过要注意：**最终活动 Compositor 已直接读取 `00 | 完整场景预览`，旧的 01+02 Alpha Over 已不再是最终输出方式。**

---

## 7. 面部：工程中最核心的 NPR + PBR 混合技术

当前脸部材质：

`FACE | 宣传图面部控制 v2__PBR_CENTER_FIX_FINAL`

节点数量约 **353**。这不是“优雅的小节点树”，而是多轮实验、诊断和最终修复累积的生产节点树。

### 7.1 设计目标

不是纯 PBR，也不是纯 Toon。

最终策略可以概括为：

- **PBR BSDF 保留真实主光、阴影、高光、SSS**；
- **NPR mask 只控制设计性的面部结构**；
- 面部不使用 Eevee-only 的 `Shader to RGB`；
- 设计阴影基于“头部空间中的主光方向 + 区域 Mask + 阈值”，而不是把一张死阴影贴在脸上。

### 7.2 面部材质的主要技术块

工程内存在：

- `FACE_HYBRID_COLOR`
- `FACE_HYBRID_PBR`
- `FACE_HYBRID_PBR_WEIGHT`
- `FACE_SHADOW_MASK`
- `FACE_REGION_MASK`
- `FACE_DETAIL_MASK_V2`
- `FACE_HAIR_SHADOW_MASK`
- `FACE V2 | Bounded local structure`
- `FACE | Shadow and Normal v1`
- `FACE_STYLIZED_NORMAL_FIX_FINAL`

其中：

- `FACE_REGION_MASK` 用不同通道区分 nose / cheek / chin 等区域；
- 左右主光方向由头部局部空间计算，不依赖简单 UV 镜像；
- 对鼻、脸颊、下巴的阴影阈值和强度做区域限制；
- hair shadow mask 被定义为**姿态相关的软接触引导**，而不是“实时刘海阴影替代品”。

### 7.3 当前面部 PBR 参数基准

当前节点中的关键值：

- Hybrid PBR Weight：约 **0.80**
- Skin Roughness override：约 **0.48**
- IOR：约 **1.40**
- Specular IOR Level override：约 **0.40**
- SSS Weight override：约 **0.10**
- SSS Scale：约 **0.0032**
- Coat：0
- 暖肤色全局 Tint 混合：约 **0.065**

这说明最终不是把脸做成强 SSS“蜡像”，而是以较保守的 SSS + 中等 Roughness + 设计性局部颜色/法线来获得肉感。

### 7.4 中央接缝的最终处理

工程最终没有继续用“疯狂调灯/贴一条遮罩”掩盖中央 PBR 分界，而是引入：

`FACE_STYLIZED_NORMAL_FIX_FINAL`

最终参数记录包括：

- face_normal_flatten_strength = 0.70
- face_center_normal_width = 0.24
- nose_normal_preserve = 0.50
- cheek_normal_flatten = 0.85
- chin_normal_preserve = 0.50
- face_specular_scale = 1.00
- face_roughness_add = 0

本质是：

> **只在脸中央需要的区域软化法线变化，同时保护鼻、下巴等必须保留立体感的位置。**

### 7.5 几何法线层也参与了脸部修复

脸对象还存在：

- Subdivision Surface：Viewport 1 / Render 2
- Data Transfer：来自 `FACE_NORMAL_SOURCE | follows head`
  - mix_factor ≈ 0.58
- Normal Edit：目标 `FACE_NORMAL_CENTER | follows head`
  - mix_factor ≈ 0.12
  - 受 Vertex Group 限制

这说明最终面部肉感是**Shader 法线 + Mesh Custom Normal + PBR 光照**一起完成，而不是单靠颜色节点。

---

## 8. 躯干皮肤：与脸分开处理，不强行“一套参数通吃”

当前躯干材质：

`米砂 | 暖肤 04_人物_躯干_B__TORSO_SKIN__BODY_SHAPING_A`

当前 Principled 关键参数：

- Roughness：**0.50**
- IOR：**1.42**
- Specular IOR Level：**0.28**
- SSS Weight：**0.12**
- SSS Scale：**0.0025**
- SSS Radius：约 `(1.0, 0.35, 0.20)`

最终脸和身体并没有做成完全相同的数值，而是维持视觉连续性、分别修正。

这比“脸和身体强制共用一个材质参数”更合理，因为两者原贴图、拓扑、法线、屏幕占比和光照需求不同。

---

## 9. 头发：各向异性 + 微法线，而不是靠强 Emission

当前主头发材质：

`米砂 | 头发 · 香槟金各向异性`

关键 Principled 参数：

- Roughness：0.40
- Specular IOR Level：0.30
- Anisotropic：**0.55**
- Sheen Weight：0.13
- Sheen Roughness：0.45
- 发丝微 Bump Strength：0.10
- Bump Distance：约 0.00007

节点还对原贴图高亮区域做压缩/暖香槟色重映射，并有 UV Tangent 方向参与。

技术方向正确：

> 头发的“丝感”来自切线方向的各向异性高光 + 很轻的微结构，而不是大面积自发光。

工程里曾尝试过专门的头发暖透光灯，但最终并没有把它当成唯一解决方案。

---

## 10. 眼睛：PBR 瞳孔 + 官方高光层 + 独立眼白/角膜

当前瞳孔材质：

`米砂 | 眼睛 · 琥珀棕瞳 0922__OFFICIAL_V1_1`

当前关键值：

- Roughness：0.28
- IOR：1.45
- Specular IOR Level：0.25
- Coat：0.04
- Coat Roughness：0.16
- Emission Strength：约 0.38
- 官方 H1 Strength：0.85
- 官方 H2 Strength：0.65

工程还新增：

- `PROMO_EYE_WHITE_CARD`：解决原模型“瞳孔/贴图覆盖眼白”的结构问题；
- `PROMO_CORNEA_CAPS`：独立角膜 cap；
- `PROMO_CORNEA_MAT`：角膜/透明控制。

所以最终眼睛不是“把原眼睛调亮”，而是拆成了**眼白、虹膜/瞳孔、角膜/高光层**的结构。

---

## 11. 官方贴图适配器：不是普通贴图导入，而是通道语义重建

工程内存在完整的 `Official Material Adapter V1 → V1.1 → V2 → V3` 演化链。

### 11.1 RG Normal 重建 Z

官方 Normal 并不是直接当普通 RGB Normal 使用，而是从 R/G 重建：

```text
Nx = 2R - 1
Ny = 2G - 1
Nz = sqrt(max(0, 1 - Nx² - Ny²))
```

然后重新编码为 RGB，进入 Blender Normal Map。

同时保留 Green Flip 控制，避免 DirectX/OpenGL 方向约定造成反转。

这个做法成立的前提是：R/G 已被验证为切线空间法线的 X/Y 数据、纹理以 Non-Color/Data 读取，并且 Z 采用可由单位长度约束恢复的半球约定。恢复后仍应检查归一化与切线基底是否匹配。

这是工程里非常重要的一项技术：**先理解源游戏贴图编码，再适配 Blender，而不是“看到 `_n` 就直接接 Normal”。**

### 11.2 ILM 通道

工程对 ILM 做了通道拆分，并测试：

- G：金边/材质分类 Mask
- B：Roughness / Specular 的调制候选
- R / A：保留为 Debug 输出验证语义

V3 报告特别强调：ILM-R/A Debug **不直接参与最终 Shader**。

### 11.3 Denier

`tex_05_cloth_014_denier` 被限定只用于：

- 明确属于 sheer 材质；
- family 与贴图匹配；
- 通过 master 控制影响透明/粗糙/透射。

V3 的 `DENIER_MASTER` 为 0.28。

这是一个很好的工程原则：

> **贴图文件名相似不代表可以跨材质族乱接。**

---

## 12. 薄纱与金边：两套物理响应在同一材质里共存

胸衣/裙摆等材质使用：

- 原始 painted detail
- Pearl fabric tint
- Alpha × sheer coverage
- Transmission
- Sheen
- 微织物 Bump
- 金色区域 Mask
- 独立 Metallic=1 的 `Gold trim PBR`
- `Mix Shader` 混合布料与真实金属

例如当前胸衣 B 的布料 Principled：

- Roughness：0.58
- Transmission：0.16
- Sheen：0.30
- SSS：0.12
- Emission Strength：0.035（非常轻）

金边则：

- Metallic：1.0
- Roughness：0.28

也就是“同一张服装贴图”没有被当成同一种材料，而是通过 Mask 拆成了**珍珠薄纱布料 + 真金属装饰**。

---

## 13. 胸衣 B：最终最典型的单变量诊断案例

当前对象：

`021_星原_30_服装_胸衣_B`

当前实际材质：

`米砂 | 30_服装_胸衣_B__OFFICIAL_V3__CHEST_FINAL_FIX__CHEST_DIAG_NO_NORMAL_ILM`

这很重要：**当前最终版停留在诊断副本，不是 Final Fix 副本。**

用户最终确认：`NO_NORMAL_ILM` 视觉正常。

这个模式做了：

1. 断开主布料 Principled 的 Normal 输入；
2. Micro Weave Bump Strength = **0**；
3. Official Normal Strength = **0**；
4. ILM-B Master = **0**；
5. 保留 Base Color / Alpha / Transmission / Sheen / Gold 等其它逻辑。

当前节点数据也验证了上述状态。

### 技术结论

导致胸衣“像贴图没对齐”的主因，不是 Base Color UV 真错位，而是 **Normal / Micro Bump / ILM 调制叠加后造成的视觉结构错位**。

这给出了整个工程最有价值的诊断模板之一：

```text
怀疑贴图错位
→ 不要先改 UV
→ 先做 TEXTURE_ONLY / NO_NORMAL_ILM A/B
→ 如果关闭 Normal/ILM 后恢复
→ 问题属于高阶通道/切线/语义，而不是 Base Color
```

未来不要机械地把该材质再“修复”为 Official Normal + ILM 全开。

---

## 14. 丝袜：区域恢复，而不是全材质回滚

当前腿部最终材质：

`...__OFFICIAL_V3__STOCKING_RESTORED`

使用 `AUDIT V2 | 原图袜区恢复` Shader Group，把官方原图/区域 A 用于白丝袜区域，同时保留当前皮肤区域和其它已调好的响应。

这是一种**区域级修复**：

- 不因一个区域错误而回滚整个材质；
- 通过 Mask 限定恢复范围；
- Roughness / SSS / Alpha / Sheen 也可在组内针对袜区独立修正。

---

## 15. 颈部与脸部接缝：Shader 之外还有几何级修复

颈部对象包含：

- `NECK_CENTER_MICRO_WELD`
- `NECK_SEAM_WELD_FIX`
- Armature
- Subdivision Surface（Viewport 1 / Render 2）

这说明工程最终接受了一个现实：

> 有些“材质接缝”实际来自几何、法线或顶点连续性，不能永远靠 Shader 掩盖。

这和脸部的 DataTransfer/NormalEdit 思路一致：先分类问题，再选择正确层级修复。

---

## 16. 灯光：从静态灯组演化为姿态感知的动画灯光 Rig

工程中保留多代灯光：

- 原模型灯
- 五灯柔光组
- `CINE 01/02/03`
- PROMO Key / Rim
- Face Sculpt L/R
- 露台 ENV 灯
- 六道丁达尔 Spot
- `米砂_LIGHT_RIG_V2_*` 动画灯光系统

### 动画 Rig 的核心不是“灯多”，而是灯会跟动作改变

`米砂_前1000帧精调_APPLY_RESTORE.py` 中实现了：

- 对角色头部方向、身体方向进行稀疏采样；
- 计算 head/body yaw、twist、风险值；
- 当头身扭转、轮廓灯进入正面风险区时降低 rim；
- 主光部分跟随 torso；
- 面补光跟随 head；
- 所有控制写入稀疏关键帧；
- 插值使用 Bezier + Auto Clamped；
- 不使用逐帧 Python handler。

控制灯包括：

- WARM_KEY
- COOL_RIM
- FACE_FILL
- 很弱的 BODY_FILL

### 为什么这是更成熟的动画灯光

固定三点光在角色转身时会出现：

- 轮廓灯跑到脸正面；
- 蓝光污染皮肤；
- 主光突然变背光；
- 头身扭转后脸和身体受光逻辑分裂。

本工程通过角色方向驱动灯位/能量，解决的是**动画中的光位稳定性**，不是单帧灯光好不好看。

---

## 17. MicroTune：最后阶段只做保守微调

工程内 `MSA_MICROTUNE_V4_REPORT` 显示最后阶段采用很克制的调整：

- Face Fill：约 +8%
- Area Size：约 +6%
- 蓝/青背景与 Rim：约 -16%
- 暖光：轻微 -3% 左右
- Sun angle：约 +10%
- 丁达尔灯：4500 → 4365

这说明最终阶段已经从“重做技术”转为**控制色彩污染、阴影硬度和脸部可读性**。

这是正确的收尾信号：最终版不应该还在大幅重写材质。

---

## 18. 相机：8 段镜头由动作风险与姿态完成点自动规划

动画 Rig 生成 8 个 Camera：

| Shot | 焦段 | 类型 |
|---|---:|---|
| 01 | 58mm | 全身 |
| 02 | 70mm | 半身 |
| 03 | 65mm | 半身 |
| 04 | 58mm | 全身 |
| 05 | 75mm | 中近景 |
| 06 | 65mm | 半身 |
| 07 | 78mm | 中近景 |
| 08 | 65mm | 半身 |

镜头切点不是均匀切，而是在预设窗口内寻找：

- pose completion
- turn transition
- 头部角速度较低
- 风险较低

每个 Camera：

- 36mm sensor
- 独立 Focus Empty
- 稀疏位置/旋转关键帧
- focus 跟踪头部/眼睛附近
- 避免 Euler 在 ±180° 处翻转

这属于一个简化版的**程序化动画摄影系统**。

当前 Final Scene 激活的是 CAM_03（65mm / f5.6）。另有 CAM_04 被后续设为约 f1.27，用于强景深镜头实验。

---

## 19. 露台环境：实体结构 + 程序材质 + 体积 + Geometry Nodes

露台不是单张背景图。

### 几何

- 复古木板，真实约 6mm 缝隙
- Bevel
- 两级大理石台阶
- 栏杆、扶手、香槟金嵌条
- 白花藤架
- 银绿色叶片
- 远景青绿空气层

### 材质

**旧橡木**：

- Roughness ≈ 0.49
- Bump Strength ≈ 0.19
- Bump Distance ≈ 0.0015
- 程序纹理与纹理贴图混合

**象牙白大理石**：

- Roughness ≈ 0.30
- Bump Strength ≈ 0.065
- Bump Distance ≈ 0.001
- 灰青细纹

### Volume

`TERRACE · Principled Volume | density .026 / anisotropy .45`

当前节点：

- Density：**0.026**
- Anisotropy：**0.45**
- 浅冷青体积色

另有 6 道窄 Spot 负责斜射丁达尔光束。

### Geometry Nodes

`米砂 · Dust / Fireflies | 体积分布与漂浮`

主要链路：

```text
Mesh → Mesh to Volume
     → Distribute Points in Volume
     → Scene Time + Noise → Set Position
     → Ico Sphere instance
     → Random Scale
     → Emissive Material
```

用于空气光尘/萤火点，漂移幅度约厘米级。

另有前景 Bokeh Geometry Nodes，负责镜头前散景。

---

## 20. 工程化方法：这是整个项目最值得保存的“技术”

工程内有 **73 个 Text datablock**，其中包含：

- apply / restore 脚本
- diagnosis 脚本
- changelog JSON
- backup state
- render/view layer diagnostics
- material adapter V1/V2/V3
- face seam diagnostics
- neck seam fixes
- chest material A/B tests
- compositor passthrough scripts
- light audit
- README / 使用说明

### 典型脚本设计思想

#### 1. Transactional

先 Snapshot，再修改，失败可回退。

#### 2. Idempotent

重复运行尽量不产生无限副本或重复连接。

#### 3. Never overwrite source

自动脚本新建副本或新文件，不直接覆盖基准工程。

#### 4. Single-variable diagnosis

例如胸衣依次测试：

- 原版
- bypass painted shadow
- NO_NORMAL_ILM
- texture only
- flat diffuse

从而判断到底是哪一层造成问题。

#### 5. Report everything

每个大改都有 Report / State JSON / Changelog。

这已经接近小型 Technical Art pipeline，而不是普通个人 Blender 工程。

---

## 21. 当前工程最重要的技术债：8 张活跃官方贴图没有 Pack

60 个 Image datablock 中约 40 个为 packed。

但当前角色使用的材质仍直接引用 **8 张未打包官方 TGA**：

1. `tex_05_cloth_014_denier__y0a3.tga`
2. `tex_05_cloth_014_ilm__y0a3.tga`
3. `tex_05_cloth_014_n__r0y0a3.tga`
4. `tex_05_clothtrans_014_ilm__y0a3.tga`
5. `tex_05_clothtrans_014_n__r0y0a3.tga`
6. `tex_05_hair_008_d__r0y0a3.tga`
7. `tex_05_eye_012_h1__y0.tga`
8. `tex_05_eye_012_h2__y0.tga`

它们当前路径在：

`D:\Azur Promilia ...\TEX\official\...`

### 影响

- 你本机正常；
- 换盘符/换机器可能丢贴图；
- 发工程给别人可能直接紫材质或丢高阶响应；
- 长期归档不够安全。

### 最终归档建议

在 Blender 本机完成：

1. `File → External Data → Pack Resources`
2. 保存为新的 archive 版本，不覆盖当前最终版
3. 关闭 Blender
4. 临时把原 `official` 贴图目录改名/断开
5. 重新打开 archive 文件
6. F12 测试角色全身 + 脸部 + 胸衣 + 眼睛
7. 确认无 Missing Files

这应该成为最终交付前最后一道检查。

---

## 22. 当前项目仍需注意的状态项

### 22.1 Timeline 状态

Final Scene 当前帧是 270，但 Render Range 为 480–1000。

这可能是故意只输出后半段，也可能只是最后一次操作遗留。批渲染前应确认。

### 22.2 历史说明与当前状态存在差异

工程内部分 README / Scene Property 仍写着：

- `00 完整场景预览仅用于视口`
- `01 + 02 通过 Alpha Over 合成`

但当前真正的 Final Compositor 已改为：

`00 | 完整场景预览 → TRUE PASSTHROUGH`

以后应以**实际节点连接**优先于旧说明。

### 22.3 节点树存在历史累积

脸部 353 nodes，大量历史节点/旁路逻辑仍存在。

它有利于追溯，但维护成本高。若未来准备“长期维护版”，建议另存一份并做 dead-node 清理，而不是在当前 final 上直接删。

---

## 23. 技术栈总览

### Renderer / Color

- Blender 5.2.x
- Cycles
- AgX
- Medium High Contrast
- Exposure +0.12

### Character LookDev

- Principled BSDF PBR
- SSS
- Anisotropic Hair
- Sheen / Transmission
- Metallic Gold separation
- Cornea / sclera reconstruction
- RG Normal reconstruction
- ILM channel decoding
- Denier map
- Custom Normal / Data Transfer / Normal Edit

### NPR Layer

- Head-space key direction
- Region masks
- Directional threshold shadow
- Pose-specific hair contact mask
- Local stylized normal correction
- PBR/NPR bounded mixing

### Environment

- HDRI + Camera-ray background separation
- Procedural wood
- Marble PBR
- Principled Volume
- Tyndall spot beams
- Geometry Nodes dust/fireflies
- Geometry Nodes foreground bokeh

### Animation / Cinematography

- Pose sampling
- Head/body yaw and twist risk analysis
- Dynamic light energy/position control
- Drivers
- Sparse keyframes
- AUTO_CLAMPED interpolation
- 8-shot procedural camera plan
- Focus empties / DOF

### Engineering

- Non-destructive Scene copies
- Material versioning
- View Layer isolation
- A/B diagnostic materials
- Snapshot / Restore
- Embedded reports and changelogs
- True-pass compositor validation

---


## 24. 技术路线验证：官方米砂渲染、Stylized Rendering 与当前工程的关系

本节不再讨论工程“现在有什么节点”，而是回答一个更重要的问题：

> **当前这套 NPR/PBR 混合路线在技术方向上是否成立？为什么官方米砂会出现“某些角度非常漂亮，但角度/光照一变就容易变怪”的现象？**

这里需要先区分两类结论：

- **工程内已确认事实**：来自当前 `.blend` 数据块、材质节点、脚本与诊断报告；
- **外部技术解释 / 对官方的推断**：来自 Stylized Rendering、Toon Rendering、Normal Editing、Face Shadow 等公开技术资料，以及对公开实机画面的视觉分析。

目前没有公开资料能够证明《蓝色星原》内部 Shader 的完整实现，因此对官方的具体技术结构只能做**画面级反推**，不能当作官方确认。

### 24.1 当前工程的核心方向是成立的

当前工程最终形成的原则：

> **真实光照负责体积与材质响应，NPR 只负责有限的设计性结构。**

这不是“既不 PBR、也不 Toon”的妥协，而是一类非常合理的 Stylized Rendering 思路。

对二次元角色而言，完全真实的几何受光并不总能得到视觉上正确的结果。Anime Face 的设计通常具有：

- 极浅的鼻部结构；
- 大面积干净面颊；
- 极弱的真实眼窝；
- 非真实比例的眼睛与五官；
- 强烈依赖二维轮廓与局部色块的可读性。

因此一个几何上正确的三维脸，并不意味着让 `N·L`、Specular、SSS 和真实遮挡完全自由工作后仍然会“画得像二次元”。

真正稳定的方案通常是：

```text
真实 / 连续 PBR 响应
        +
受限制的艺术化 Shading
        +
局部 Normal / Region 控制
```

这与当前工程中：

- PBR 保留主光、阴影、高光、SSS；
- NPR 只控制 nose / cheek / chin 等设计结构；
- Head-space light direction 驱动局部面部响应；
- 局部法线修正而非整脸替换；

的路线是一致的。

---

### 24.2 为什么二次元脸不能完全交给普通 PBR

普通漫反射可以简化为：

```text
D = max(0, N · L)
```

真实人脸中，鼻梁、眼窝、面颊、嘴唇、下巴的几何结构本身就应该生成相应阴影。

但 Anime Face 的问题在于：

```text
真实几何正确的阴影
≠
二维角色设计需要的阴影
```

例如一个在真实人脸上合理的鼻翼阴影，放在二次元角色上可能立刻产生：

- 鼻子过重；
- 眼窝脏；
- 脸颊“被打了一拳”；
- 嘴部周围形成不需要的灰阶；
- 正脸失去插画感。

因此 Stylized Character Rendering 的关键并不是“让光照更正确”，而是：

> **决定哪些视觉属性允许由真实光照自由变化，哪些属性必须被角色设计保护。**

---

### 24.3 为什么官方米砂在某些角度非常漂亮

目前公开画面中，米砂在接近正面、柔光、高明度、弱鼻影的镜头里可以非常漂亮。

这种情况下，画面的信息分工通常接近：

```text
脸
→ 大面积干净、柔和、低结构

眼睛
→ 高对比、高细节、负责表情

头发
→ 方向性高光、发束层次、负责空间感

服装 / 金属
→ 负责材质复杂度
```

这会产生一种非常典型的宣传图式 Stylized Look：

- 空灵；
- 圣洁；
- 柔软；
- 手办感；
- 接近二维插画但仍然保留空间体积。

其美感并不是偶然。

因为在正面 / 半正面：

- 角色原画比例最稳定；
- 双眼最接近二维设计；
- 鼻子几乎不需要承担真实空间结构；
- flatten / edited normal 最容易工作；
- 柔光可以主动隐藏眼窝、鼻翼等真实几何信息。

因此此类 Shader 的**单帧美学上限可以非常高**。

---

### 24.4 为什么 3/4 侧脸更容易暴露问题

当角色进入 3/4 侧脸，人类视觉开始要求更明确的三维结构：

```text
近侧 cheek
    ↓
nose ridge
    ↓
nose tip / wing
    ↓
远侧 cheek
```

同时：

- 近眼和远眼出现明显透视差；
- 鼻子开始突出轮廓；
- 面颊曲率开始承担空间感；
- 下巴和下颌线需要真正解释头部旋转。

如果此时 Shading Normal 仍然被强烈 Flatten，可能出现：

> **Geometry 已经明显转过去，但 Shading 仍然像在解释一张偏正面的二维脸。**

这可以称为：

### Geometry / Shading Normal Mismatch

即：

```text
可见轮廓描述 A 形状
受光却暗示 B 形状
```

这种错误通常不会表现成“某个节点坏了”，而是：

> **观众说不出哪里错，但会觉得脸突然很怪。**

---

### 24.5 Normal Editing 是正确技术，但不能无边界使用

Stylized Rendering 中常见的一项技术就是：

- 编辑 Vertex / Custom Normal；
- 控制 Shadow Line；
- Flatten 特定区域；
- 让 Toon / Anime 角色获得更整块的照明。

这项技术对 Anime Face 极其有效。

但它存在非常明确的权衡：

```text
Normal 越接近真实 Geometry
        ↓
三维体积越可靠
        ↓
鼻子 / 眼窝 / cheek 越容易产生不需要的真实阴影


Normal 越强烈 Flatten
        ↓
正脸越干净、越二次元
        ↓
侧脸越容易失去可信空间结构
```

这也是为什么当前工程最终没有采用“整张脸一起抹平”。

目前成功模式反而是：

- cheek flatten 较强；
- forehead 适度平滑；
- nose preserve；
- chin preserve；
- center seam 只做局部 Normal Correction。

也就是说：

> **只把应该平的区域做平，把必须承担三维空间信息的位置留下来。**

这是比 Uniform Face Flatten 更成熟的方案。

---

### 24.6 Face Shadow 为什么需要独立于普通受光

Anime Face 很难完全依赖真实 Normal 生成阴影。

成熟 Toon Rendering 常见的另一条路线，是让脸拥有单独的 Face Shadow System。

基本思想不是：

```text
Normal
  +
Light
  ↓
自动决定全部脸影
```

而是：

```text
Light Direction
      ↓
Head Local Space
      ↓
Face Shadow Controller
      ↓
Designed Shadow Shape
```

也就是说：

> **光负责告诉 Shader “现在光从哪里来”，但角色设计决定“这张脸应该怎么画”。**

因此会出现：

- Face Map；
- Face Shadow Mask；
- Head-space Direction；
- Threshold；
- Region Mask；
- SDF Face Shadow；

等技术。

当前工程已经拥有：

- Head-space key direction；
- FACE_REGION_MASK；
- directional threshold；
- cheek / nose / chin 局部控制；

因此已经进入这一技术家族。

它还不是一个完整的多方向 SDF Face Shader，但路线并没有错。

---

### 24.7 SDF Face Shadow 可以视为未来的进一步研究方向

SDF Face Shadow 的目标并不是模拟真实人脸。

而是：

> **让设计好的脸部阴影随着头部空间中的光照方向稳定变化。**

可以抽象成：

```text
Face Shadow = f(Face SDF, Head-space Light Direction)
```

相比单纯的 `N·L`：

- 阴影边界可设计；
- 鼻影不会由复杂拓扑随机决定；
- cheek 不容易出现破碎暗区；
- 左右转光时更稳定。

更进一步的系统还会考虑：

- 左 / 右方向；
- 上 / 下方向；
- Pose；
- Face Expression；
- Hair Contact。

因此如果未来要把当前 Blender Shader 从“米砂专用”继续抽象成通用 Anime Face Shader：

**Face SDF + Head-space Lighting** 是最值得补的一块理论与实现。

---

### 24.8 最终成片：从“理论可行”进入“结果支持”

后续验收阶段又加入了一层此前静态审计没有的证据：最终动画视频与多张成片静帧。

从结果层面可以观察到：

- 正脸与多个 3/4 角度下，cheek 保持干净，但 nose / chin 仍保留一定三维解释能力；
- Face 的结构变化受到约束，但 Body / Hair / Metal 仍持续接受更强的连续光照与材质响应；
- 冷色 Rim 大多数时间停留在头发、耳朵、肩、手臂、透明衣料边缘，没有频繁侵入脸中央；
- 头部倾斜、头身扭转、手臂遮挡、近景/半身切换时，没有观察到明显的 face-mask 翻转、shadow pop 或 normal 突跳；
- 透明衣料、金属、皮肤、头发在同一镜头中仍具有可区分的 material response。

这些现象与工程原先提出的“不同视觉属性分配不同物理自由度”是一致的。

但必须强调：

> **成片能够证明“这套组合在当前角色/场景中产生了预期结果”，不能单靠画面证明某一个节点、某一个 Normal Modifier 或某一段 Head-space 数学是唯一因果。**

因此这属于**结果支持（result-level support）**，不是严格的算法因果证明。

---

### 24.9 对三测官方实机的修正判断：不是“整个角色一起更 PBR”

结合三测实机画面，更准确的工作假说不再是：

```text
二测 = NPR 高
三测 = PBR 高
```

而应该改成：

```text
Body / Cloth / Hair
→ 更连续、更环境感知

Face
→ 仍维持很强的 Identity Protection
```

也就是说，三测更像是**Shading Domain 之间重新分配控制权**，而不是整个角色共享一个全局 PBR/NPR 权重。

这一点非常重要，因为它支持了当前工程的一个核心判断：

> **不要用一个全局 “PBR 80% / NPR 20%” 去描述整个角色。**

更合理的是：

```text
Stylization Weight
=
f(region, material, light, view, pose)
```

例如一个可操作的自由度表：

| Domain / 属性 | 建议物理光照自由度 |
|---|---|
| 金属反射 | 高 |
| 布料 Roughness / 大体明暗 | 高 |
| 头发方向高光 | 高 |
| 身体大尺度明暗 | 高 |
| 脸整体亮度 | 中 |
| 肤色环境染色 | 中低 |
| 鼻部阴影形状 | 低 |
| cheek 核心结构 | 低 |
| Anime Eye 可读性 | 很低 |
| 表情关键线索 | 很低 |

---

### 24.10 官方三测的正面启发：实时系统首先保护“最差帧”

从用户提供的三测实机截图可以观察到：即使环境明显偏蓝、背景复杂、镜头角度变化，官方脸部仍大体保持：

- 高明度；
- 暖肤色；
- 低频 cheek；
- 弱眼窝；
- 清晰眼睛；
- 稳定的表情阅读。

这说明一个非常值得借鉴的实时渲染目标：

### Worst-case Identity Preservation

开放世界角色 Shader 并不能假设：

- 摄像机永远在甜点角度；
- 环境永远是柔光；
- 玩家永远站在理想位置；
- 曝光永远受控。

因此实时系统往往不仅优化：

```text
Best-case beauty
```

还必须优化：

```text
Worst-case recognizability
```

这对当前 Blender/Cycles 工程也很有启发：

> **下一阶段不要只问“哪一帧最好看”，还要问“最差的一帧是否仍然像这个角色”。**

---

### 24.11 官方三测的负面启发：保护过度会造成 Shading Domain Inconsistency

强 Identity Protection 的代价同样明显。

在部分实机画面中，可以出现：

```text
Hair
→ 明显接受环境色与方向高光

Body
→ 连续明暗、空间体积明确

Face
→ 高明度、低结构、环境色进入很少
```

于是角色内部可能形成：

```text
Hair：3D / environment-aware
Body：3D / environment-aware
Face：2D-ish / identity-aware
```

这并不一定代表“脸单独做坏了”。

更准确的风险是：

### Shading Domain Inconsistency

也就是 Face / Hair / Body 对“这个世界的光是什么样”给出了不同程度的回答。

因此目标不是把脸简单做得更真实，而是：

> **允许不同 Domain 使用不同技术，但必须让它们说同一种视觉语言。**

---

### 24.12 Stylization × Exposure：一个容易被忽略的失真放大器

三测部分高曝光剧情镜头还揭示了另一个问题：

Stylized Face 本来就主动减少了：

- 眼窝暗部；
- 鼻翼阴影；
- cheek 高频明暗；
- 皮肤微结构。

如果此时曝光继续提高：

```text
Stylization ↑
+
Exposure ↑
↓
Remaining Form Cues ↓
```

就可能出现：

- 鼻部结构快速消失；
- cheek 与额头接近同一亮度；
- 五官只剩贴图/线条继续工作；
- Face 比 Body 更早进入“平面化”。

因此 Face Shader 的验证不能只测试灯光方向，还应该加入曝光矩阵。

建议至少：

```text
EV -2 / -1 / 0 / +1 / +2
```

并观察什么时候开始发生 `Face Structure Collapse`。

---

### 24.13 Environment Adaptation 应拆成两个轴，而不是一个 Skin Protection 滑杆

之前本文把问题抽象成 `Environment Adaptation`。结合成片与三测实机，更合理的设计是拆成两个相互独立的轴。

#### A. Chromatic Adaptation

控制“环境颜色进入角色多少”。

目标不是：

```text
Environment Affect = 0
```

也不是：

```text
Environment Affect = 1
```

而是允许：

- 冷环境对皮肤暗部/边缘产生有限 cyan/blue 影响；
- 暖环境对半影产生有限暖色影响；
- 角色核心肤色保持在 identity-safe envelope 内；
- Face / Body 的环境染色幅度不同，但不互相矛盾。

#### B. Luminance Adaptation

控制“角色亮度与结构如何跟随场景曝光”。

目标是：

- 场景变亮时，Face 可以变亮；
- 但 nose / cheek / chin / eye 的最小结构对比不能同时消失；
- 不通过简单 Emission 把脸“钉死”在固定亮度。

可以进一步加入：

#### C. Expression Protection

表情层可以比普通皮肤获得更强保护：

- eyelid；
- eyebrow；
- lip；
- blush；
- expression-specific marks。

这比把整个 Face 一起锁死更细，也更符合商业角色的阅读优先级。

---

### 24.14 重新定义最终目标：Physical Spatial Coherence + Bounded Identity Protection

经过工程、成片和官方实机三组材料交叉观察后，当前项目最稳定的理论表达不再只是：

```text
Physical Response
+
Bounded Artistic Correction
```

而可以进一步明确成：

```text
Stylized Character Rendering
=
Physical Spatial Coherence
+
Bounded Identity Protection
```

#### Physical Spatial Coherence 负责

- 角色确实属于当前环境；
- 大尺度明暗能解释角色朝向；
- hair / body / cloth / metal 有合理差异；
- 环境颜色能够有限进入角色；
- 相机运动与动画中三维结构连续。

#### Bounded Identity Protection 负责

- Anime Eye 始终可读；
- 鼻影不会突然变成真人式重结构；
- cheek 不被复杂几何阴影污染；
- 表情关键线索不被照明抹掉；
- 角色核心色彩设计不会被环境完全改写。

关键词是：

### Bounded

因为：

```text
保护不足
→ 角色设计被真实光照破坏

保护过度
→ 角色像贴纸 / Face 与世界脱节
```

真正成熟的系统不是选择其中一边，而是在两者之间寻找稳定区间。

---

### 24.15 Regional Normal Control：继续保留，但要从经验参数升级为可测方法

当前工程的成功经验仍然是：

- cheek flatten 较强；
- forehead 适度平滑；
- nose preserve；
- chin preserve；
- center seam 只做局部 correction。

未来若要从工程经验走向更一般的方法，可以形式化为：

```text
N'_i = normalize((1 - w_i) N_geo + w_i N_art)
```

其中：

```text
w_cheek > w_nose
```

并进一步研究：

```text
w_i = f(region, view, light, pose)
```

这比“整脸统一 Flatten”更有研究价值，因为它承认不同语义区域承担不同的空间信息责任。

但当前工程尚未证明最优 `w_i`，因此这里应视为**方法框架**，不是已经完成的普适算法。

---

### 24.16 Face SDF：是候选实现，不是终点，也不是官方实现事实

SDF Face Shadow 仍值得研究，因为它能把：

```text
Head-space Light Direction
→ Designed Shadow Boundary
```

稳定地连接起来。

但下一阶段更准确的目标应该写成：

> **研究多方向、角色空间驱动的 Designed Face Shadow；SDF 是成熟候选方案之一，而不是唯一正确答案。**

原因是当前 Cycles 管线并不一定需要照搬实时游戏的纹理编码方式。

可比较的实现至少包括：

- directional region masks；
- SDF face map；
- multi-direction face maps；
- analytic threshold/ramp；
- hybrid normal + face-map control。

只有 A/B 实验后，才能决定哪一个更适合本项目。

---

### 24.17 下一阶段验证：从“作品”升级为“研究”

如果希望把这套方法从 Technical Art Case Study 进一步提升为可复现研究，最关键的不是继续增加节点，而是建立测试矩阵。

#### 角度矩阵

```text
0° / 30° / 45° / 60° / 90°
```

左右两侧均测试。

#### 光照矩阵

- 正面柔光；
- 侧光；
- 顶光；
- 逆光；
- 侧逆光；
- 冷环境；
- 暖环境。

#### 曝光矩阵

```text
EV -2 / -1 / 0 / +1 / +2
```

#### 姿态矩阵

- head tilt；
- head/body twist；
- 头发遮脸；
- 手臂穿过脸附近；
- 表情变化；
- 中近景 / 特写。

#### Ablation

同一镜头、同一姿态、同一灯光依次比较：

```text
A — Pure PBR / Geometry Normal
B — Uniform Face Flatten
C — Regional Normal Control
D — Regional Normal + Designed Face Shadow
E — D + Environment Adaptation
```

这样才能回答：

> 到底是哪一个组件在改善稳定性？

而不是只回答：

> 最终版看起来很好。

---

### 24.18 建议记录的评价指标

不必把 Stylized Rendering 强行变成纯像素误差问题，但可以建立一组可重复的工程/研究指标。

#### 1. Cross-view Structure Stability

检查转头过程中：

- nose；
- cheek；
- chin；

是否持续给出互相一致的空间信息。

#### 2. Temporal Stability

记录是否出现：

- shadow pop；
- mask flip；
- specular jump；
- normal discontinuity；
- rim invasion。

#### 3. Normal–Geometry Deviation

可以记录 stylized normal 与 geometry normal 的夹角：

```text
D_N = arccos(N_geo · N_styled)
```

并比较不同 face region 的允许范围。

#### 4. Worst-case Identity Preservation

不是挑最好帧，而是取整个测试矩阵中最差帧，评价：

- 是否仍像同一角色；
- 眼睛/表情是否仍然清晰；
- 是否出现脸被环境“洗掉”的情况。

#### 5. Shading-domain Coherence

评价 Face / Hair / Body 是否像由同一个环境照亮，而不是三套互不相干的灯。

#### 6. Environment Integration

评价角色是否：

- 不像贴纸；
- 也没有被环境色完全吞没。

如果未来做正式研究，可加入 blinded user study，把“identity”“3D coherence”“environment integration”“anime fidelity”等拆成独立量表。

---

### 24.19 技术与学术定位

从当前状态看，这个项目最准确的定位是：

> **Blender/Cycles 上的 Stylized Character Rendering / Technical Art R&D Prototype。**

它已经超出普通 Blender LookDev 的原因，不是“节点很多”，而是已经形成：

- responsibility separation；
- single-variable diagnosis；
- reversible automation；
- region-aware normal control；
- domain-specific shading；
- pose-aware lighting；
- worst-case stability thinking。

从学术角度，它已经具备一个不错的应用型研究骨架：

```text
Problem
→ Anime identity 与 3D lighting 冲突

Hypothesis
→ Physical spatial coherence + bounded identity protection

Implementation
→ Regional Normal + bounded face control + material-domain separation + dynamic lighting

Observed Result
→ 多角度/动画中维持较稳定的 identity、材质分工和空间感
```

但它仍然主要是**高质量工程 case study / prototype**，不是已经完成的原创图形学算法论文。

要再向正式研究前进，最缺的是：

- baseline；
- ablation；
- stress-test matrix；
- metric；
- user study；
- cross-character reproduction。

其中最可能形成独立方法贡献的，不是单独“用了 SDF”或“用了 Custom Normal”，而是：

> **如何按语义区域和视觉属性分配 Physical Degrees of Freedom 与 Protected Design Constraints。**

---

### 24.20 与公开主流技术的关系：接轨，但不要反向证明官方

以下公开资料可以支持“这套思路属于成熟 Stylized Rendering / Technical Art 问题空间”，但**不能用于证明《蓝色星原》内部具体实现**。

#### Blender 5.2

- Normal Edit Modifier 支持生成/混合 Custom Normals，并明确提到可用于 toon-like shading：
  https://docs.blender.org/manual/en/5.2/modeling/modifiers/normals/normal_edit.html

- Data Transfer 支持传递 Custom Normals：
  https://docs.blender.org/manual/en/5.2/modeling/modifiers/modify/data_transfer.html

- Light Path / Is Camera Ray 支持按 ray type 做有意的非物理艺术控制：
  https://docs.blender.org/manual/en/5.2/render/shader_nodes/input/light_path.html

- Packed Data / Pack Resources 支持将符合条件的外部资源封装进 `.blend`：
  https://docs.blender.org/manual/en/5.2/files/blend/packed_data.html

#### Unity Toon Shader

公开文档提供了：

- Normal Map 对不同 shading 区域/Highlight/Rim 的影响控制；
- Shadow Control；
- Highlight；
- Rim；
- Scene Light Effectiveness；

说明主流 Toon 系统本身也不是“一个二值 Toon 开关”，而是对不同视觉属性进行独立控制。

参考：

- https://docs.unity3d.com/ja/Packages/com.unity.toonshader%400.8/manual/NormalMap.html
- https://docs.unity3d.com/ja/Packages/com.unity.toonshader%400.9/manual/Parameter-Settings.html

#### Arc System Works / Guilty Gear Xrd

Arc System Works 公开的 GDC 2015 技术分享展示了一个非常典型的产业问题：

> 在完整 3D 框架中，通过技术美术与程序 R&D 主动维护二维角色设计。

参考：

https://www.arcsystemworks.com/guilty-gear-xrds-art-style-the-x-factor-between-2d-and-3d-talk-from-gdc-2015-is-now-available-online/

#### DirectXTex

Microsoft DirectXTex 的 `texconv` 明确提供：

- `--reconstruct-z`：从 XY-only Normal 重建 Z；
- `--invert-y`：处理 OpenGL / Direct3D Green/Y 方向约定。

参考：

https://github.com/microsoft/DirectXTex/wiki/texconv

这为本工程的 RG Normal reconstruction / Green Flip 提供了标准图形资产处理上的外部依据，但仍然要求先验证源资产通道语义。

---

### 24.21 本节最终结论

经过：

1. `.blend` 数据块与脚本审计；
2. 最终动画/静帧结果观察；
3. 官方三测实机的正反案例；
4. Blender / Unity / Arc System Works / DirectXTex 等公开技术资料；

当前工程最值得继续发展的原则可以收敛为：

```text
Geometry
   ↓
Regional / Art-directed Normal
   ↓
Physical Material Response
   ↓
Designed Face Shading
   ↓
Chromatic + Luminance Environment Adaptation
   ↓
Bounded Identity Protection
   ↓
Final Stylized Character
```

它既不是：

```text
Pure PBR
```

也不是：

```text
Pure Toon
```

更不是：

```text
全角色统一 PBR/NPR 权重
```

真正的问题是：

> **每一个视觉属性应该拥有多少物理自由度，以及哪些 identity-critical 信息必须受到有边界的保护。**

因此最终原则可以写成：

> **让真实光照证明角色存在于空间里；让 Art Direction 阻止真实光照摧毁角色设计；同时限制 Art Direction，避免角色脱离这个空间。**

这比单纯讨论“PBR 还是 NPR”更接近当前工程真正形成的方法论。

---

## 25. 最终评价：这套工程真正形成了什么

从技术方向看，这个项目最后形成的不是某家游戏的“完全复刻 Shader”，而是一套更适合 Blender/Cycles 的**Stylized Character Rendering Translation Layer**：

- 保留原资产的二维设计信息；
- 用 Cycles 恢复真实空间光照与材质差异；
- 用 Regional Normal 与有限 NPR 控制脸部可读性；
- 用 eye / skin / hair / cloth / metal / sheer 等不同 Domain 分配不同物理自由度；
- 用动态灯光与镜头让单帧 LookDev 能进入动画；
- 用 Environment Adaptation 平衡“角色身份”与“场景融合”；
- 用工程化诊断、回滚与 A/B 避免自动化“改一处崩三处”；
- 用 worst-case / stress-test 思维把“甜点角度”升级为“跨角度、跨光照仍然稳定”。

最终最重要的不是某个 Roughness、SSS 或 NPR Weight 的数值，而是两条原则：

> **先证明问题属于哪一层，再修改那一层。**

以及：

> **角色设计决定哪些信息不能被破坏；物理渲染负责其余可以自由变化的部分；任何保护都必须有边界。**

因此，这套工程最有复用价值的并不是 353 个节点本身，而是已经形成的：

### Diagnosis → Responsibility → Minimal Intervention → Stress Test → Promotion

这才是应该沉淀进 Skill 的核心。

---

## 附录 A：后续案例补记——Jinshi 与 Shading Responsibility Allocation

本节是**独立于上述米砂工程审计的后续记录**，不追改米砂 `.blend` 的已确认状态。依据是本次提供的 [Jinshi 案例笔记](JINSHI_RESPONSIBILITY_CASE_STUDY.md)；本仓库此次未收到对应 `.blend`、脚本、状态报告或渲染输出进行独立复核。

案例记录中的保护不足使鼻侧、眼窝与面颊过度写实；保护过强或整脸压平又损害了 3/4 空间解释与脸身受光一致性。这在案例层面支持本文的 **Physical Spatial Coherence + Bounded Identity Protection**，但不把某一张图或某个节点提升为普遍因果证明。

新增加的诊断轴是 **Shading Responsibility Allocation**：在找到最早出错层后，继续问每个视觉线索由哪个子系统主要负责，其他子系统是否重复强化同一体积线索。案例将几何法线、局部法线、设计性脸影、PBR 与角色灯光重复塑造鼻、眼窝、面颊的情况称为 **Redundant Volume Encoding**。这扩展了已有的 `Diagnosis → Responsibility → Minimal Intervention → Stress Test → Promotion` 方法，不替换本文核心公式，也不把 Jinshi 的参数当作通用默认值。
