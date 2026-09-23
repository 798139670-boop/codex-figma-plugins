# 批量 ToGUI/ToUGUI 界面结构审计补充规则

本参考用于“从目录超链接学习公共组件库/多个 Figma 文件”的只读审计模式。它补充单一 URL 的结构整理规则；不改变单一节点任务的范围，也不把观察到的历史异常提升为规范。

## 1. 目录链接与页面发现

- 从目录文字的 hyperlink 目标建立输入清单，保留目录序号、显示名、源 URL、file key 和源节点 ID。不要只依赖目录文字或文件名猜测目标。
- 每个链接文件都扫描全部 Page 名称，匹配 `ToGUI`、`ToUGUI`（允许全角/半角括号和模块后缀）。同一文件可能有多个匹配页面，例如同一模块的主页面、子页面、ServerList 或 Confirm 页面；每页分别审计，不能只取第一个。
- 对每个匹配页面只认 Page 直属、名称为 `界面` 的 `SECTION`。没有匹配页、没有直属 `界面` Section、存在多个候选 Section 或页面读取受限时，保留原始证据并列入报告，不猜测、迁移或合并内容。
- `ToUGUI` 页面本身不是 UIRoot。页面下 `界面` Section 的直属子节点才是 View 候选；`组件`、`图片`、说明文字和页面级样例不计入 View。

## 2. `界面` 直属节点的分类

先分类，再决定是否可作为正式导出单元。每个直属节点应记录 `id/name/type/visible/bounds/exportSettings/childCount`：

1. **正式 View 候选**：通常为可见的 Frame，具有唯一导出名，根尺寸通常为 `1080×2400`；其完整后代树作为一个审计单元。
2. **辅助或排除根**：名字符合 `NotExport`、`NoExport`、`Ignore`、`Placeholder` 等约定，或明确是说明/备份/运行时插槽；不把它们自动升级为 UIRoot。
3. **隐藏草稿、资源或孤立节点**：保留并报告可见性、父级和用途证据；不要因为它们位于 Section 下就删除或重命名。
4. **非 Frame 根或尺寸异常根**：作为异常保留。`RECTANGLE`、`INSTANCE`、空 Frame 和非标准尺寸不能被批量规则自动改造成标准 View。

正式 View 的判定需要结合命名唯一性、节点类型、尺寸和后代结构；可见性用于区分草稿和正式候选，不单独决定导出资格。单一条件（例如“直属子节点”或“名字含 View”）不足以证明作者意图。直属正式 View 可使用现有 Page/Section 规则中的 UIRoot 推断。内部 `Root/root` 的业务用途可能只是容器，但精确 Role token 仍可能优先识别为 UIRoot；必须记录作者意图与实际 Role 推断的冲突，不能按普通 Container 忽略它。只读审计保持原名，修改任务按已有 Role 规则和授权范围处理。

## 3. 完整树读取与可审计性

- 扫描每个 View 的全部后代，不能只读取根和前两层，也不能在 Instance/Component 边界停止。至少记录层级、parent、顺序、几何、可见性、layout、clip、native mask、Export、Instance identity、Text 字符和 Role 线索。
- 大树必须分段读取。每个分段记录 `fileKey/pageId/viewId/offset/declaredTotal/readRows`，用 `fileKey + viewId + offset` 识别分块，节点/父级索引使用 `fileKey + nodeId`。Figma node ID 不是跨文件唯一值，不能用裸 ID 合并不同文件。
- 合并时保持原遍历顺序，验证所有位置连续、节点 ID 唯一、后代 parent 存在、声明总数一致；重叠分段的节点 ID/次序必须一致。树在分段之间变化时，仅重读受影响的树，不能拼接两个版本凑足数量。
- 只有唯一 `readRows == declaredTotal` 且没有尚未解决的 failed offset、截断或读取错误，才能标记为“节点结构枚举 complete”。后续获准的成功补读可消除当前缺段，历史错误另存；历史失败不永久否定完整性。部分读取不能据此推导缺失节点的规范。
- 节点枚举完整不等于所有检查通过。分别记录已采集属性、未采集属性、视觉验证、Common 身份匹配和 Unity 导出验证状态。只根据实际完整取得的字段归纳对应规则。
- 不同批次的 Export/Instance/Text 序列化格式可能不同，先规范化再比较并保留原始证据。不要把一个模糊的 `excluded` 布尔值当作完整导出语义；分别记录命名排除边界、UiMask、祖先 PNG、Instance 边界与继承可见性，最终有效输出仍按既有 Role/Render Policy 优先级判断。
- 报告至少提供：目录链接数、匹配页面数、含页面的文件数、无 ToGUI 页面文件数、缺失 `界面` Section 数、发现 View 数、完整 View 数、完整节点总数、受限 View 及失败 offset。

## 4. GRID/Wrap 与重复项

- `GRID` Auto Layout 是二维排版线索，不等于普通横向/纵向 List。记录行优先/列优先、实际 child order、列数或行数、item 尺寸、横纵间距、padding、absolute overlay 和首个运行时模板。
- 只保留第一个运行时模板时，结合实际 child order、Auto Layout 方向/reverse/wrap 与视觉顺序判断（通常先按行、再按列），不要按 Layer ID、名称或分块响应到达顺序重排。保留后续可见示例的 authoring 布局；只有明确的冗余备份或排除内容才使用隐藏 `NotExport`。
- GRID 只是静态排版时显式使用 `Container_*`；确认是运行时数据列表时，Figma 端采用既有 `List_*` Role/推断规则，Grid/Wrap 对应后端的 `LoopGridView`。`LoopGridView` 是生成的 Unity 组件，不是 Figma Role 前缀。不要因发现 GRID 就机械改名、移动或删除节点。
- GRID/Wrap 审计应与现有普通 List/ScrollView/ScrollBar 规则一起报告：模板、点击区、独立 overlay、滚动条父级和排除树边界分别列出。

## 5. 原生 Mask 与导出 Role 分离

- `isMask=true` 是 Figma 视觉遮罩证据，不等同于导出 Role `Mask_*` 或 `UiMask_*`。批量数据中九宫格、Instance 内部切片和静态合成会产生大量 native mask，不能逐个重命名为 Mask。
- 对 `Mask`，检查明确的裁剪职责、裁剪图形和被裁剪内容的关系，以及与既有 Mask 模板一致的层级。对 `UiMask`，检查单一可见纯色 Fill 及归属的最近导出父级；它只提供元数据，不生成 GameObject，不承担裁剪，也不是遮罩视觉层。两者仍按现有显式 Role 和 Render Policy 判定。
- 完整的静态 mask+artwork 组合可按现有 PNG 规则选择整体视觉源；已有 NineSlice 按专用规则处理。native mask 本身不证明该组合适合九宫格，也不授权新增或改造 NineSlice；需要独立动态内容时保持对应结构节点。
- 报告分别统计 native mask 节点、显式 Mask/UiMask Role 和位于 `NotExport`/Instance/PNG 祖先下的 mask；不要把三者混为一项。

## 6. Export、排除树与动态内容

- 正式 PNG、排除树内既有 PNG、专用 NineSlice/mask 源、运行时 image slot 和 placeholder 分开统计。`NotExport` 后代已有 PNG 不代表它是正式资源，也不应在无明确范围时清除。
- 若排除根自身仍有 PNG，单独报告其可能覆盖排除 Role 的冲突；不要仅凭名称自动移除。继续遵守现有 PNG 规则：行为根、UIRoot、列表/控件根、Text、UiMask、NotExport、placeholder 和交互 HitArea 不作为普通 PNG 源。
- 动态文本、奖励数量、进度、预览 tint、空 Image 插槽和隐藏预览不能因外观完整或有名字就静态 PNG 化。静态视觉源应覆盖完整不可分割的 mask+artwork 组合，动态节点保持结构身份。
- 某正式 View 内可能有多个与独立 SubView 同名的空 Frame，用作业务装配位置。只凭同名不能认定它已建立 Reference，也不能自动克隆独立界面填进去；区分“空 Frame 插槽”“明确 Reference Role”“真实 Instance 身份”，按可见证据记录，绑定关系未知时保持结构。

## 7. 异常与结论边界

以下项目应分类记录，不因出现频繁就成为可复制的设计规范：没有明确模块例外的非 `1080×2400` 根、空 View、用途未确认的隐藏或非 Frame 根、输出侧中文 layer name、直属 `NotExport`、行为根带 Export、排除树 PNG、发生 Role 冲突的内部 `Root/root`、工具读取失败和页面无 `界面` Section。其中排除树 PNG、空运行时插槽、明确的模块尺寸例外可能合法；应报告依据，不统一判错。中文 Text characters 和已排除的说明文字不属于中文输出名错误。只有与既有 exporter 规则一致且有完整证据支持的模式，才可写入正向规则。

建议每个 View 使用一行机器可读摘要：

```text
file | page | sectionId | viewId | viewName | rootType | width×height | visible | nodeCount/readCount | maxDepth | roles | gridCount | nativeMaskCount | pngCount | excludedCount | status | warnings
```

受限数据必须保留失败原因（例如自动审批超时、stream disconnected 或服务容量不足）和已读 offset；不得以“未发现”替代“未读取”。

## 8. 与现有规则的关系

本参考只新增批量发现、完整性核对、二维 GRID、native mask 证据分层和审计报告边界。现有关于 UIRoot、Page/Section、Container/List、ScrollView、Toggle、UiMask、NotExport、PNG 选源、Common Instance、命名和几何的规则继续有效，冲突时按 `SKILL.md` 的优先级执行。
