# ToUGUI 初步 Export 勾选规则

## 1. 规则定位

本文件补充 `Figma节点PNG-Export设置规则分析.md` 第 13 节，用于在完整结构审计后建立第一轮、保守、可复核、可重复执行的 PNG Export 候选。它不替代角色、复合控件、布局、Common、Mask、NineSlice 或既有 Export 规则。

“初步勾选”表示：已经读取完整目标树，并只为职责明确、能独立复现静态外观、无需结构调整的图片源建立首轮候选。它不等同于给所有叶子节点勾选，也不代表所有视觉问题已经解决，更不代表 Unity Sprite、Prefab 或运行时行为已经验证。

当前任务若要求 Figma 只读，只能学习和报告这些候选，不得写入 `exportSettings`。后续用户授权实际勾选时，确定项应沿用 PNG 参考第 13 节的执行、批次、回读和幂等要求，不把“初步”变成只输出建议。

## 2. 安全候选的必要条件

一个节点或最小静态子树只有同时满足以下条件，才可进入 `safeInitialCandidates`：

1. 目标节点及其所有后代已经递归读取，包括非 PNG Instance 的内部节点；读取不完整时只能列入 `needsReview`。
2. 节点有可观察的完整静态视觉，包括可见 Fill、Stroke、Effect、渐变、Image Fill、Boolean 或静态遮罩合成；不能只凭节点名、叶子类型、当前截图或 `Image_*` 前缀判定。
3. 节点本身不是 UIRoot、行为根、布局根、动态内容根、运行时驱动区域、有效 `NotExport`/`UiMask`/`placeholder`/`PlaceLimiter` 排除节点，且后代没有需要保留为独立结构的职责。显式、已确认的复杂 `Mask` 根是专用例外：可按既有 Mask 规则在 Mask 根出图，同时保留其 Mask 语义；不能把它按普通布局根排除。
4. 该节点能独立复现完整外观，或其子树是最小且不可再拆的静态视觉源；它不是只包含一部分边框、阴影、遮罩或当前进度的切片。
5. 没有有效 PNG 祖先已经覆盖同一视觉，也不会与子节点形成父子重复 Export。语义容器和真正视觉源二选一。
6. 不包含动态、本地化、数据绑定、运行时数值或需要独立替换的内容，也不把多状态行为根烘成一张图。单个已经确认的 Toggle checked/unchecked 静态状态视觉、确认不会变化的装饰文字，可以在各自状态视觉源上候选；普通 Text 不能因当前文案完整就自动勾选。
7. 只需改变目标节点的 `exportSettings` 即可完成，不需要移动、拆分、克隆、Detach、改名或改变组件身份；支持的 Instance Export override 可作为目标节点设置。

先确认是否已经有权威图片来源。已从 Common Sprite/Prefab 获得资源的消费实例不重复生成本地 PNG；图片 Section 的 ForcedSectionRender 等来源继续按原规则判断。已有合法 PNG Instance 的优先规则仍有效，不因本条反向强制 Common 匹配。

上面的装饰文字例外沿用旧第 6.1 节全部条件：完全静态、确定不需要修改，而且必须与复杂视觉一起烘图；仅作为外层 PNG 视觉源的后代合并，不把普通 Text 本身列为自动勾选候选。

标准设置沿用既有政策：

```js
{
  format: "PNG",
  constraint: { type: "SCALE", value: 1 },
  contentsOnly: true,
  suffix: ""
}
```

用户或项目已经明确指定其他倍率时，以该政策为准；不能通过反复追加数组条目实现补勾。

## 3. 布局与复合结构边界

布局字段是判断“能否保持结构”的证据，不是 Export 开关。`FIXED`、`HUG`、`FILL`、`STRETCH`、`CENTER`、`ABSOLUTE`、`@size` 需要沿父子责任链解释；当前尺寸相同、节点名称带 `stretch` 或实例带 `@size` 都不足以证明可以烘成图片。

- 普通 Auto Layout 容器、List、ScrollView、Item、GRID、Wrap、Content、Viewport 和布局根不得为了省事整体 PNG 化。静态背景可以单独成为候选，布局和运行时 Item 必须保留。
- `ABSOLUTE + STRETCH` 的底图可能是安全候选，但必须确认其不包含需要独立保留的文本、按钮、状态或动态内容，并检查 Clip、边距和祖先覆盖关系。
- Button、Toggle、Input、ProgressBar、Slider、ScrollBar 根及 `Fill Area`、`Handle Slide Area`、`Sliding Area` 保持既有结构规则；只检查其内部最终视觉源。
- ProgressBar 只选择完整 Track/Fill 素材，不能把当前百分比的短 Fill 烘成 Sprite。Slider 的 Track、Fill、Handle 分别判断，保留运行时驱动关系。ScrollBar 的轨道和 Handle 也分别判断。
- GRID 的行列轨道、Wrap 的换行和 FILL/HUG 关系必须保留；不能将整个 GRID/Wrap 当作一张图，也不能把单元格当前排列当作运行时布局已验证。
- Text 的 `WIDTH_AND_HEIGHT`、`HEIGHT` 或 `TRUNCATE` 只说明当前文字布局状态，不证明文字可以导出。动态、可本地化、可变数量、名称、状态和倒计时文本保持真实 Text。

## 4. 既有例外与待复核边界

### 4.1 可保留或进入候选

- 已有单条、参数合规的 PNG，且完整静态外观未烘入动态内容：归入 `existingValidExports`，保留并复核，不重复写入。
- 确认的隐藏状态图、未显示但属于正式状态的静态视觉：按实际职责判断，不能仅因 `visible=false` 排除。隐藏备份、设计稿样例和排除树仍按排除语义处理。
- 完整 Boolean 根、可见 Stroke-only 图形、复杂 Effect、Image Fill，以及 mask 与 tint/Fill 共同组成的静态视觉：在能完整复现外观的共同视觉根判断，不逐个勾内部切片。
- 带有已经证明不会变化的装饰文字的静态合成：可由共同视觉根统一导出；证据不足时列入 `needsReview`。
- 已有合法 PNG 的 Instance：保留真实 Instance 身份，不因初步勾选强制 Common 匹配、替换、Detach 或迁移。
- 显式复杂 `Mask` 根采用专用策略：PNG 作为遮罩 Graphic 的 Sprite 来源，Mask 和被裁剪内容仍按既有 Mask 规则处理，不套用普通 PNG 压平全部子树的假设。原生 `isMask` 或 `UiMask` 不能据此直接当作显式 Mask；边界或职责不清时待复核。

### 4.2 有意不勾选

归入 `intentionallyUnexported`，并记录节点 ID、边界和原因：

- UIRoot、行为根、布局根、普通布局容器和必须保留的结构节点；
- 多状态 Toggle/Variant 行为根；但其中已经确认的单个 checked/unchecked 静态视觉源仍按 4.1 判断；
- 动态/本地化 Text、Placeholder、Input Text、数量或状态文字；
- 透明 HitArea、运行时 Image 插槽及其预览 tint；
- 有效 `NotExport`、`UiMask`、`placeholder`、`PlaceLimiter` 排除树；
- 九宫格已由专用结构或插件数据识别的源及内部切片，按 NineSlice 专用规则处理，不机械统一为 PNG；
- 普通矩形 Mask 的裁剪结构；显式复杂 Mask 根不在此列，按既有 Mask 专用规则判断；
- 被合法 PNG 祖先覆盖、且不需要独立资源的后代候选。

这些分类不表示清空已有设置。NineSlice 的既有 PNG 不机械增删；`NotExport` 备份内部的既有 PNG 保持不动并单列。`NotExport`/`UiMask` 等排除节点自身若已误勾 PNG，会覆盖排除语义，必须单列错误，按旧第 13 节判断安全纠正或待复核，不能作为正常排除树静默跳过。

### 4.3 只报告、不自动勾选

归入 `needsReview`：

- Instance 的身份、Export override 支持或静态/行为职责不确定；
- Mask/Clip 的完整视觉边界不明确，或需要拆分才能同时保留动态内容；
- GRID/Wrap/List 的根与静态背景边界无法分开；
- 非平凡旋转、复杂变换、裁剪越界、混合模式或字段不可读影响最终出图；
- 可能包含运行时数值、交互状态、预览样例或组件内部切片；
- 修正错误根 Export 需要改结构、Detach、移动节点、改名或改变组件身份；
- 任何祖先/后代关系尚未完整读取。

低置信度不等于永久禁止。后续补齐结构、职责或 API 证据后，再按第 13 节的确定性迁移条件处理。

## 5. 首轮判定顺序

按下列顺序形成清单，保证重复运行不会因扫描顺序产生不同结果：

1. 解析精确 URL 和目标范围，递归读取所有后代及必要祖先；保存父子关系、布局、可见性、组件身份和已有 Export。
2. 先标记 `NotExport`、`UiMask`、`placeholder`、`PlaceLimiter`、NineSlice、有效 PNG 祖先和其他专用边界。
3. 识别 UIRoot、行为根、布局根、文本、运行时驱动区域、动态内容和交互 HitArea。
4. 在剩余树中选择能完整复现外观的最小静态视觉源；优先共同 Boolean/mask/tint 视觉根，避免无意义切碎。
5. 消除候选之间的父子重复和语义容器/视觉源重复；记录被压制的后代，不把压制误报为删除。
6. 保留 `existingValidExports`，为确定候选生成唯一 PNG 参数；错误根 Export 按旧规则判断是否能安全迁移。
7. 用户授权写入时，按约十项逻辑修改一批执行；每批前复读指纹，写后回读 `exportSettings`，失败即停止该批并报告不确定状态。
8. 最后重新展开完整目标树，复核父子重复、动态内容、专用边界及非 Export 属性；输出分类和验证等级。

## 6. 审计分类与验收

初步审计至少输出以下分类：

```text
safeInitialCandidates       # 可安全首轮新增/修正的节点与证据
existingValidExports        # 已合规、无需写入的节点
intentionallyUnexported     # 按职责或专用策略有意不勾的节点
needsReview                 # 证据不足、结构冲突或 API 阻塞
suppressedDescendantExports # 已有但被合法祖先 PNG/排除边界覆盖的后代设置
```

实际写入结果继续使用既有字段 `added`、`corrected`、`removed`、`movedExport`、`unchanged`、`unresolved`；审计分类不是修改数量。`suppressedDescendantExports` 只说明覆盖关系，尤其不能把 `NotExport` 备份内部既有 PNG 误报成已删除。

验收至少确认：完整遍历计数一致；每个有绘制内容的非 Text 节点都有合法图片来源、专用策略、祖先覆盖或明确待复核原因；正式源的 PNG、倍率、contentsOnly、suffix 和设置条数正确；没有动态文字、占位预览或独立控件被烘入；名称、层级、几何、布局、可见性、绘制属性和组件身份未因 Export-only 操作改变。Figma 读审、Export 设置核验和 Unity 运行时验证分别报告，不能互相替代。

## 7. 与既有规则的关系

本规则只增加首轮筛选和报告分类。实际 Export 写入、参数、实例处理、父子压制、错误根迁移、批次限制、回读和停止条件，以 `Figma节点PNG-Export设置规则分析.md` 第 13 节为准；发生冲突时保留更具体的既有例外，并报告 unresolved。
