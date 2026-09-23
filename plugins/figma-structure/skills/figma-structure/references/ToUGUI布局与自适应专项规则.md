# ToUGUI 布局与自适应专项审计规则

本参考补充“每层结构如何获得尺寸、如何定位以及父级变化时如何响应”的读取和判读方法。与既有 [几何布局与 UGUI 注意事项](Figma复合组件几何布局与UGUI注意事项.md)、[复合结构整理规则](Figma复合结构转UGUI整理分析.md)、[批量界面审计规则](批量ToUGUI界面结构审计补充规则.md) 配合使用；不替代原有角色、几何、点击区域和导出限制。

## 1. 学习范围与证据边界

- 布局学习默认读取目标的完整层级及必要父级上下文，将 Figma 当前状态、布局意图的静态推演、已明确的导出行为分别记录。
- 用户要求 Figma 只读时，不得临时 resize、clone、改 Constraints、改 Auto Layout、切换 Variant 或调整组件实例属性进行试验。通过读取属性和几何做静态推演，不能把推演称为实际适配测试通过。
- 当前尺寸下对齐、宽度碰巧相同、名字带 `stretch` 或实例带 `@size`，均不足以单独证明响应式布局成立。
- 不根据现有文件的出现频次改变导出规范。异常、特殊业务布局、字段不可读和导出支持未知项分别报告。

## 2. 布局字段与父子责任链

每个节点至少记录以下信息；按 API 能力读取，不存在或无法读取的属性标记为 unavailable，不默认为 FIXED、NONE 或零值：

```text
id, parentId, ordered child index, node type, visible/inherited visibility,
x, y, width, height, rotation/transform, parent width/height,
constraints.horizontal, constraints.vertical,
layoutMode, layoutPositioning, layoutSizingHorizontal, layoutSizingVertical,
primaryAxisSizingMode, counterAxisSizingMode, layoutGrow, layoutAlign,
primaryAxisAlignItems, counterAxisAlignItems,
paddingLeft, paddingRight, paddingTop, paddingBottom,
itemSpacing, counterAxisSpacing, layoutWrap,
minWidth, maxWidth, minHeight, maxHeight, clipsContent,
textAutoResize and text alignment when applicable,
Instance source identity, source size, target size, @size when applicable
```

遇到 GRID 时，额外读取当前 API 暴露的行列数、轨道尺寸策略、行列间距、单元对齐、跨行跨列与放置位置；保存原字段名和值，不能把无法读取的轨道信息推定为等宽等高。布局中的描边占位、反向绘制顺序等属性可读时也应记录，不能把绘制顺序直接等同于布局顺序。

按 `UIRoot → 顶/底栏或面板 → 内容容器/List → Item → 控件 → Text/Image` 建立责任链；实际不存在的层不补造。对每层水平与垂直轴分别记录：

| 责任 | 需要回答的问题 |
| --- | --- |
| 尺寸来源 | 固定尺寸、子内容撑开、父级分配空间、Constraints 拉伸、实例引用缩放，还是运行时控件驱动？ |
| 位置来源 | 父 Auto Layout 流、普通 Constraints、Absolute 定位，还是控件运行时驱动？ |
| 边界 | 哪一层提供 padding、点击 Rect、可视裁剪、最小/最大尺寸？ |
| 依赖 | 父级变化是否能沿该轴传递到当前节点；哪一层 FIXED 会终止传递？ |

同一节点可以在内部排列子项，同时由上级分配自身尺寸。将“节点自身的 Auto Layout”与“作为父布局子项的 sizing”分开记录；不能仅因节点 `layoutMode=HORIZONTAL` 就称其宽度自适应。

## 3. Constraints 逐轴判读

下面描述普通非流式布局中 Figma Constraints 的含义。Auto Layout 普通流子项先按第 4 节判读；Slider/Scrollbar 等运行时驱动节点继续按既有几何参考处理。

| API 值 | 水平方向 UI 名称 | 垂直方向 UI 名称 | 父尺寸变化时保持的量 | 变化的量 |
| --- | --- | --- | --- | --- |
| `MIN` | Left | Top | 起始边距、该轴尺寸 | 末端边距 |
| `MAX` | Right | Bottom | 末端边距、该轴尺寸 | 起始边距 |
| `CENTER` | Center | Center | 相对父中心的偏移、该轴尺寸 | 两侧边距 |
| `STRETCH` | Left & Right | Top & Bottom | 两端 inset | 该轴尺寸 |
| `SCALE` | Scale | Scale | Figma 的比例约束意图 | 位置与尺寸随父级比例变化；导出受下述限制 |

注意：

- 水平与垂直约束独立判读。Center 表示保持原中心偏移，不保证偏移为 0。
- Stretch 不等于铺满；只有两端 inset 都为 0 时才表示铺满。不要为统一布局清零有意保留的边距。
- `SCALE` 的导出处理沿用现有几何参考第 3.2 节：当前不转换为比例 Anchor，水平回退 Left、垂直回退 Top 并产生警告。不能把 Figma 比例行为承诺为 Unity 比例行为。
- 节点存在旋转、非平凡变换或不明确的父边界时，保留变换证据；不能只用屏幕轴对齐 bounds 推断局部 inset。

### 用户示例：L ↔ R + Center

横向 `Left & Right`、纵向 `Center` 表示：左右边距固定，宽度随父容器变化；高度固定，纵向保持相对父中心的偏移。

下式只用于父边界稳定、父子局部坐标轴平行且无旋转/斜切的普通约束分支；子节点不受父 Auto Layout 流或运行时控件驱动。父级应是已确认的约束容器，不直接把由后代包围盒决定边界的 Group 当作可独立改尺寸的 Frame。Auto Layout 中的 Absolute 子项，只有确认其局部坐标和约束参照边界后才可套用，不能忽略 padding 或导出重建层级带来的参照差异。

在上述条件下，设父宽高为 `W/H`，子节点为 `x/y/w/h`：

```text
left = x
right = W - x - w
verticalCenterOffset = y + h/2 - H/2

父级变化到 W2/H2 后的静态预期：
x2 = left
w2 = W2 - left - right
h2 = h
y2 = H2/2 + verticalCenterOffset - h/2
```

这是 Constraints 意图的静态推演。最小/最大尺寸、实际布局控制器、变换和运行时驱动可能限制结果；必须单列。推演出现负尺寸或交叠时记录冲突，不自动修改 Figma。

## 4. Auto Layout 与尺寸依赖

| 尺寸策略 | 判读 | 审计重点 |
| --- | --- | --- |
| FIXED | 该轴有明确目标尺寸 | 是否有意固定；父级变化时是否形成溢出或终止拉伸链 |
| HUG | 该轴由参与流式布局的内容及 padding/spacing 决定 | 哪些子项实际参与流；文本变化、隐藏项与浮层不能混为同一种占位 |
| FILL | 该轴由父级可分配空间决定 | 父级是否有可解析尺寸、兄弟项如何分配空间、最小/最大尺寸是否限制 |

- 宽高分别判读，结合现代 sizing 字段与现有 `layoutGrow/layoutAlign` 等字段；不要只看其中一个字段就覆盖其它证据。
- 先判断字段适用路径。普通非流式节点即使返回 `layoutSizingHorizontal/Vertical=FIXED`，仍可能通过 Constraints 的 STRETCH 随父级变化；不要用 FIXED 覆盖其约束语义。反之，普通 Auto Layout 流子项的原始 Constraints 值不替代父级布局规则。对 Text 还需独立看 `textAutoResize`；返回 FIXED/FIXED 与 HEIGHT 自动扩高可以同时出现。
- 父 HUG 与子 FILL 同轴组合应核实 Figma 当前实际解析后的模式和尺寸。不能声称父子两者可同时独立决定同一轴；不机械更改为 Fixed。
- 普通流子项的位置由父排版决定，当前 x/y 是排版结果。Absolute 子项退出普通流式排版，但仍需要检查 Constraints、所在父级几何、裁剪和层级。
- 固定 `itemSpacing` 与 `SPACE_BETWEEN` 等分布方式分别记录。当前子项之间的像素距离不一定是固定间距。
- 非对称 padding、负间距、基线对齐、隐藏项、跨轴对齐及最小/最大宽高均可改变适配结果。字段存在不等于当前导出器已完整支持；旧参考未明确的映射列为待验证。
- 父 LayoutGroup、ContentSizeFitter 和运行时代码在同轴的职责冲突，继续执行既有几何参考第 3.4 节。行为根是否可 HUG 也遵循原有限制。

## 5. 各结构的布局决策表

以下是读取和诊断入口；详细控件几何、目标 Graphic、初始值和运行时行为以既有几何参考相应章节为准。

| 结构 | 需要学习的布局关系 | 不可由外观直接推断的结论 | 原规则入口 |
| --- | --- | --- | --- |
| UIRoot / 全屏背景 | 根边界、背景覆盖、内部顶/底/居中区域的约束 | 全屏设计不代表所有内部节点都要四向 Stretch | Page/Section 参考；几何 §3 |
| 普通 Container / 面板 | 自身 sizing、子项排布、padding、固定和拉伸轴 | Auto Layout 不等于运行时 List | 几何 §3.2–3.4；Role 总览 |
| Button | 行为 Rect、背景/HitArea 覆盖、文本安全区、背景是否退出流 | 背景图尺寸不天然等于点击范围 | 几何 §4 |
| Toggle / ToggleGroup | 图标和 checked 重叠、文字间距、整行点击区、状态尺寸一致性 | Toggle 子项集合不自动意味着互斥组 | 几何 §5 |
| Input | Text Area inset、placeholder 与实际文本 Rect/对齐、图标预留区 | 同处容器不保证两个 Text 完全重叠 | 几何 §6 |
| ProgressBar | 完整轨道、Fill 同起点与中心、适配时共同拉伸 | 当前短 Fill 不可直接当完整进度素材 | 几何 §7 |
| Slider | Track/Fill Area/Handle Area 共轴、Handle 固定视觉尺寸与移动范围 | 普通 Constraints 不决定运行时 Handle 运动 | 几何 §8 |
| List / ScrollView | 视口尺寸、主轴、Item sizing、padding/spacing、模板与浮层 | 裁剪边界与内容尺寸是不同职责 | 几何 §9 |
| Scrollbar | 完整轨道、Handle 比例、交叉轴 inset、父级与 Absolute 状态 | Figma Handle 长度不一定是绑定 ScrollRect 后的永久长度 | 几何 §10 |
| Mask / Clip | 裁剪边界与后代归属、边界外装饰和阴影 | native mask、Clip、UiMask 不可互换 | 几何 §9.3；批量审计 §5 |
| Component / Instance / Variant | 源与实例尺寸、内部拉伸链、状态布局差异 | 单个状态不证明所有状态响应一致 | 几何 §3.6、§11 |

List/ScrollView 的宽高沿用几何参考第 9 节：对确认走该生成路径且开启 Clip content 的 List，根 Rect 定义可见 Viewport，Content 尺寸另由 Item、padding/spacing 与视口下限决定。不能用全部 Item 的包围盒替换视口尺寸，也不能把未开启 Clip 的普通排版容器直接套入这条生成规则。ScrollBar 是否覆盖内容或需预留空间继续按旧规则单列。

## 6. GRID 与 Wrap 的支持边界

- 继续执行批量审计参考的二维顺序、首个正式模板、示例项保留和静态 Container/运行时 List 分类规则。
- 分开记录原生 `layoutMode=GRID` 与横向/纵向 Auto Layout 的 Wrap；不能仅因视觉都像网格就认为同一机制。
- 对二维结构读取行列轨道、单元尺寸、间距、padding、跨度、对齐及尺寸策略，判明列数固定、尺寸固定或可分配空间的证据。不得以静态截图推断自动换列阈值。
- 既有参考确认的 Grid/Wrap 与后端 `LoopGridView` 关系继续有效，但这不证明原生 GRID 的全部轨道类型、跨行跨列、特殊对齐或可变列数均有等价导出支持。未在参考中明确支持的特性列入待验证，不新造导出 Role 或参数。

## 7. 文本与组件实例

### 文本

- 分别记录 Text 宽度、高度、`textAutoResize`、换行/截断状态与文本对齐。Label Rect 的位置与字形在 Rect 内的对齐是不同职责。
- 短文本目前能容纳，不等于长文本、本地化或字体 fallback 可适配。只读学习中记录可用空间和依赖；没有实际运行测试时不能声称长文本已验证。
- 固定宽度配自动扩高会影响父 HUG、列表 Item 高度与裁剪；自动扩宽会影响横向兄弟项和可用空间。文本尺寸变化应沿父子责任链检查。

### 组件实例与 Variant

- 分别读取源组件身份、源根宽高、实例目标宽高、比例差、精确 `@size` 参数及内部 Constraints/Auto Layout，保留真正的 Instance 身份。
- 组件身份以真实 Instance 的 source component ID/key 为证据，不能由同名、同尺寸、相似布局或 `@size` 推定。布局专项记录不替代 Common 身份匹配流程，也不触发 detach、替换或迁移；来源不可读时保留 unresolved，不宣称无匹配。节点类型、显式导出 Role 与是否被 PNG 烘焙分别记录，遵循旧 skill 的优先级。
- 默认引用缩放与 `@size` 直接使用实例尺寸的规则沿用几何参考第 3.6 节。`@size` 不承诺任意拉伸安全，内部固定坐标或固定尺寸仍可能阻断响应。
- 按真实 Variant 状态读取结构和对应节点；状态根尺寸、布局模式、文本基线或内部约束不一致时报告差异。用户要求只读时，不通过切换现有实例状态实施试验。

## 8. 结果与验证等级

布局专项结果至少应包含：

```text
layoutEvidence:
  nodeId / parentId / path / role
  horizontal: sizeSource / positionSource / constraints / sizing / limits
  vertical: sizeSource / positionSource / constraints / sizing / limits
  flowOrAbsolute / padding / spacing / alignment / clipBoundary
  sourceComponentSize / instanceSize / sizeDirective
  observedFields / unavailableFields
assessment:
  currentGeometry
  staticResponsivePrediction
  existingExporterRule
  unknownExporterSupport
verification:
  figmaReadOnly
  actualResizeTestPerformed
  unityExportTestPerformed
warnings: []
```

“已读完整布局属性”“静态推演具备某轴适配条件”“实际尺寸变化已测试”“Unity 导出已验证”是不同结论。仅完成前两项时，后两项应明确为未执行。无需为学习而调整文件；后续执行用户授权的结构修改时，仍按原 skill 的计划、指纹、布局与视觉复核要求进行。

## 9. 从批量布局观察中提取规则

- 顶层居中/贴顶/贴底、内部背景拉伸、内容区固定或HUG、列表Item FILL、文本HEIGHT可在同一界面共存；按层级和轴组合，不把单一模式推广到全部子层。
- 看到 `STRETCH/CENTER + FIXED/FIXED` 时，先判明它是普通约束分支；看到 `FILL/FIXED` 时先查父布局方向与可分配宽度。相同当前尺寸不能代替这些字段。
- `GRID` 中 HUG 行轨道与 FLEX 列轨道可以并存；必须记录两维轨道，不能把它简化为相等的固定格子。规则只确认观察方法，具体导出支持沿用第6节边界。
- `ABSOLUTE + STRETCH/STRETCH` 常可表达随容器包围内容的底图；仍需逐项核实角色、边距与Clip。Absolute是退出流，Stretch才表达双边约束，二者承担不同职责。
- 总体频率包含实例内部、九宫格切片、PNG后代、隐藏和排除树。汇总时同时给出全节点范围与用于学习的可见结构候选范围；候选范围本身不是实际导出结果。不因SCALE出现多就推荐SCALE，不因Absolute出现少就省略控件重叠规则。
