# Stylized Character Rendering 验证框架

[English](validation-framework_EN.md) · [项目首页](../README.md) · [方法论](methodology.md)

目标是找出**最差条件下仍成立**的角色渲染方案。以下是后续可复现研究的测试协议，不表示这些矩阵已在原工程中全部完成。已发生的工程审计与成片观察见[技术总结](../docs/TECHNICAL_SUMMARY.md)。

## 固定基准与记录

每轮比较固定角色姿态、相机、灯光、环境、渲染器、色彩管理、分辨率和随机种子，只改变待测因素。保存 `.blend` 版本、材质/Modifier 开关、灯光与曝光值、镜头编号、帧号、输出图、观察者判断和回滚路径。使用同样的 crop 对比脸、颈、头发与身体。区分“尚未测试”“通过”“失败”，不把主观印象写成工程事实。

## 压力测试矩阵

| 轴 | 最小采样 | 重点观察 |
| --- | --- | --- |
| 视角 | 0°、30°、45°、60°、90°；有不对称时左右两侧都测 | 眼、面颊、鼻、下巴与轮廓能否连续解释空间 |
| 光照 | front、side、top、back、rim、mixed lighting | 脸影翻转、rim 侵入、材质域受光是否冲突 |
| 环境 | neutral、warm、cyan/blue、green、red、dark、bright | 环境色和亮度能否有限进入脸部；肤色是否失真 |
| 曝光 | EV −2、−1、0、+1、+2 | 高亮时 nose/cheek/forehead 结构是否消失 |
| 姿态与动画 | head tilt、head/body twist、头发局部遮脸、手臂接近脸、表情变化；关键转身段逐帧或密采样 | shadow pop、mask flip、normal pop、rim intrusion、表情可读性 |

先做每个轴的单变量扫描，记录风险组合；再将高风险视角、光向、环境和曝光交叉测试。不要声称仅靠逐轴测试覆盖了所有组合。保持左右转向与时间序列的连续性检查。

## 面部消融实验

同一姿态、相机、灯光、环境与曝光下比较：

| 版本 | 条件 | 检验问题 |
| --- | --- | --- |
| A | Pure PBR / geometry normals | 未保护的基线如何失效？ |
| B | Uniform Normal Flatten | 整脸压平是否改善正脸却破坏空间信息？ |
| C | Regional Normal Control | 局部法线能否同时保留干净面颊和鼻/下巴体积？ |
| D | C + Designed Face Shading | 设计性脸影是否改善身份与跨视角稳定性？ |
| E | D + Environment Adaptation | 场景融合是否改善且不抹掉身份？ |

每个版本记录启用的节点、Modifier 和 mask。若 D 优于 C，只能将差异归于两版的受控变化；不应从最终图像推断某单一节点是唯一原因。

## 评价指标

建议每项以 **0（失效）—4（稳定）** 评分，同时记录截图和具体失败描述。先按主观锚点给分，再由第二名观察者盲评高风险样本；不要把这些分数误称为客观像素真值。

| 指标 | 观察点 |
| --- | --- |
| Character Identity Preservation | 眼形、脸形、表达和核心配色是否仍像同一角色 |
| Spatial Coherence | 鼻、下巴、头部方向与几何空间是否相符 |
| Face/Body/Hair Shading-domain Coherence | 各域是否像受同一环境和灯光影响 |
| Cross-view Stability | 左右、正脸到侧脸的结构是否平稳过渡 |
| Temporal Stability | 是否有 shadow/mask/normal 跳变、rim 突然侵入 |
| Worst-case Identity Preservation | 全矩阵最低分样本仍否可辨认 |
| Exposure Robustness | EV +1/+2 下五官与低频结构是否保留 |
| Environment Integration | 色彩与亮度有场景响应，同时未失去设计身份 |

至少报告每项**最低分**、对应条件与失败帧，再报告中位数；不要只挑最漂亮的一帧。若修正改善平均值却降低最差帧的身份或时序稳定性，应保留为实验，不直接推广。

## “惨白脸”假说的检查

可能的技术链：

```text
Stylized face → reduced geometry-driven shading → low local contrast
→ high face/scene luminance → exposure → tone-mapping highlight compression
→ remaining nose/cheek/forehead cues collapse → pale or flat-looking face
```

`Pale Face ≠ necessarily hard RGB clipping`。在自有可控场景中，保持贴图、法线、灯光和相机不变，扫描曝光与 tone mapping；比较渲染前线性亮度、输出图的局部对比及是否真正达到通道上限。再单独改变面部设计性保护与环境亮度。若没有像素饱和但结构对比仍显著下降，就支持“低对比 + 高亮度 + tone-mapping 压缩”这一解释；若存在硬裁切，也要记录为另一机制。它是对公开画面的**视觉工作假说**，不能据此确认《蓝色星原》内部算法。

## 报告结论的格式

每项结论标注来源：**工程内事实 / 成片观察 / 公开技术先例 / 工作假说**。报告测试版本、条件、最差帧、负面结果与尚未覆盖的组合。只有受控 A/B 能支持组件层因果判断；成片可支持系统在当前案例中的整体结果。
