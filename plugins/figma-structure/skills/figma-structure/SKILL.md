---
name: figma-structure
description: Inspect and normalize a Figma design URL's target node into a structure that the project's FigmaToUGUI plugin can export to Unity UGUI. Use when asked to整理、转换、修正、检查 or prepare a Figma node, view, component, list, control, image, mask, or page/section hierarchy for UGUI export, including automatic PNG Export selection (自动勾选、补勾、复查漏勾), Common component replacement, Variant/Toggle, ScrollView/ScrollBar, UiMask/NotExport, naming, and geometry rules.
---

# Figma to UGUI Structure

Turn the exact node referenced by a Figma URL into an exportable FigmaToUGUI authoring structure. Use this file, its bundled `references/` directory, and `config/common-module.json` as the complete source of truth. Do not load workspace-level docs, static Common registries, exporter source, web documentation, or old conversations. Preserve the design while making the smallest safe changes.

## Operating contract

- Require a Figma design URL containing a file key and `node-id`. Convert URL node IDs such as `23-62` to Plugin API IDs such as `23:62`.
- Use the URL target even when the current Figma selection differs. Ask for a URL only when no unambiguous target ID is available.
- If the user asks to modify,整理, convert, or prepare the node, perform safe in-scope edits after the audit. Do not impose a mechanical second confirmation.
- Pause before a detach, destructive replacement, ambiguous Common match, uncertain interaction role, or any operation that cannot preserve layout and visuals. Explain the exact choice needed.
- Never delete source content. Preserve uncertain or replaced originals inside an invisible `NotExport` Frame when the rules call for a backup.
- Do not silently fix missing fonts. Before moving or cloning Text across parents, load its existing fonts. If they are unavailable, use the documented visual-copy split strategy or stop.
- Keep every exported Prefab, GameObject, Sprite, module, Variant property/value, and Export suffix name free of Chinese characters. Chinese Text characters are allowed; exported layer names are not. When the user limits the task to structure and asks to preserve names, report existing naming issues instead of performing a naming cleanup; follow the narrow directive exception in section 4.1.
- Return all inspected and modified node IDs, warnings, unresolved items, and verification results.

### 中文层名统一改为英文（新增优先规则）

- 对本次目标及其完整后代，凡 `node.name` 含任一汉字，都必须改为语义明确的英文名，包括中英混合名。检查可使用 JavaScript `/\p{Script=Han}/u`，不能只检查纯中文名或 `Image_*`。
- 该命名标准覆盖 Frame、Group、Text 图层名、形状、Component、Instance、Variant member、Section，以及隐藏层、备份层、NotExport 后代和 PNG 后代；不因节点不导出或不可见而免除。Page 在任务范围内时也按此标准处理。不得扩大到目标之外的页面、文件或远程主组件。
- Text 的 `characters` 是显示文案，可以保留中文；只修改图层 `name`，不能为满足命名规则翻译或改写界面文案。
- 英文名按业务含义与职责翻译，保留已有英文业务词、编号及有效 Role/参数语法；不以拼音、删除中文后留下空名，或批量 `Frame1`/`Image1` 代替翻译。例如 `Button_领取奖励` → `Button_ClaimReward`、`Label_剩余时间` → `Label_RemainingTime`、`背景_01` → `Background_01`。不得为了改英文名给原节点错误增加或移除行为 Role。
- 对范围内的 Section，`界面` → `Views`、`组件` → `Coms`、`图片` → `Res`；带业务后缀时翻译后缀并保留识别前缀，例如 `界面-弹窗` → `Views_Popups`。直接复用原 Section，不创建替代 Section、不迁移子节点。审计发现阶段同时识别中文旧名与英文兼容名，不能改名后漏读目标。Page 模块名使用英文并保留 `ToUGUI(ModuleName)` 格式。
- 本节优先于本 skill 及 bundled references 中一般性的“保留中文层名”“仅输出节点禁中文”“非导出说明层可留中文”“Section 必须中文”和默认“保留名称”约定；旧条文保留，其命名冲突按本节处理。对已授权的结构整理或命名整理任务，中文层名翻译属于默认必要步骤。
- 当前明确的只读、不修改 Figma、不改名或仅修改 `exportSettings` 限制仍优先：此时生成完整 `nodeId / oldName / proposedEnglishName / reason` 清单并报告未执行，不据此擅自写入。仅要求“向 skill 追加规则”不构成修改 Figma 的授权。
- 改名前读取完整目标树，形成确定的旧名到新名映射并检查同级/产物名及 Common 规范化名称冲突。冲突用英文业务限定词或稳定编号消歧；已有合法英文名不作无关改写。Variant 的 `Property=Value`、`@` 参数与依赖名称的引用须保持一致，不能只翻译字符串而破坏绑定。

### 泛化图层名必须语义化（新增规则）

- 在已授权的结构整理或命名整理范围内，`Frame`、`Frame_数字`、`Rectangle`、`Rectangle_数字`、`Vector`、`Vector_数字`、`Group`、`Ellipse`、`Layer_数字`、`Part_数字`、`Structure` 及仅由这些通用词与编号组成的中英混合名，都视为未完成命名；不能只把它们改成 `Frame_1`、`Image_1` 或删除编号。
- 读取完整目标树并结合父节点职责、同级顺序、尺寸、图像/Fill hash、文字和截图推断用途，改成稳定的英文业务名；优先保留现有 Role 前缀（例如 `Container_`, `Image_`, `Button_`, `Reference_`, `Component_`），为同类重复节点使用明确的业务后缀和稳定编号。名称必须让不了解画布的导出维护者能区分背景、台座、徽章、星级、图标、装饰和状态内容。
- `Image_*_Part_1` 仅在其对应视觉职责已确认时改成 `Image_<businessVisual>`；不得把父级语义容器、行为根或动态文字改成图片名，也不得因为名称清理改变 Figma 原生类型、层级、几何、可见性、布局、Export 或组件身份。
- Component Set 的 `Property=Value` 变体名、真实 Component/Instance 身份、Role 前缀和绑定引用必须保留；变体成员内部语义节点在各状态中使用一致名称。真实 Instance 的远程内部节点只能在 Figma 支持的本地 name override 范围内修改；不为命名而 Detach、修改远程主组件或伪造组件。
- 不能从尺寸或通用名称唯一推断业务职责时，保留原名并列入 `needsReview`，同时继续完成有截图、同源 hash、父职责或同状态对照支持的命名；报告每个未解决节点及证据缺口。
- 保留 Instance 的真实组件身份；仅在目标内支持的名称 override 上操作。不能为改名 Detach、替换组件或写入远程主组件。若依赖名称的引用、Variant 定义或实例内部名称不能安全同步，列出具体节点及阻塞原因，继续完成其他确定项，不把该节点当作已完成。
- 改后重新扫描完整目标树并回读改名结果，报告 `renamed` 映射、剩余中文层名及原因；验证层级、几何、布局、文字内容、可见性、Export 和组件身份未改变。只读任务报告候选数；写入任务只有范围内中文层名清零且相关引用验证通过，才可声称命名整理完成。

## Bundled references and execution boundary

Read each required bundled reference completely before the first related Figma read or mutation. Resolve paths relative to this `SKILL.md`; never substitute similarly named workspace docs.

| Concern | Required bundled reference |
|---|---|
| Role inference, prefix omission, Render Policy, Mask/UiMask/NotExport | `references/FigmaToUGUI导出节点类型总览.md` |
| Page, module, Section, UIRoot, output naming | `references/FigmaToUGUI模块与Page-Section导出规范.md` |
| Compound controls, structure/visual separation, List/ScrollView/ScrollBar | `references/Figma复合结构转UGUI整理分析.md` |
| Rects, anchors, Auto Layout, hit areas, runtime-driven geometry | `references/Figma复合组件几何布局与UGUI注意事项.md` |
| Layout learning, per-axis Constraints, responsive sizing, nested Auto Layout, native GRID tracks, and read-only adaptation analysis | `references/ToUGUI布局与自适应专项规则.md` |
| Automatic PNG Export selection, placement, full-tree recheck, suppression, scale, suffix, Sprite policy | `references/Figma节点PNG-Export设置规则分析.md` (automatic execution: section 13) |
| Conservative first-pass PNG Export candidates, layout boundaries, initial audit categories, and read-only candidate learning | `references/ToUGUI初步Export勾选规则.md` (read before planning an initial Export pass) |
| Component Set, Variant, Toggle, ToggleGroup, optional prefixes | `references/公共组件库new-Variant与Toggle导出规则分析.md` |
| Variant authoring mindset, semantic state axes, real combinations, baseline/fallback branches, per-state layout review, and Variant Toggle construction | `references/ToUGUI变体制作与状态轴专项规则.md` (read for Variant authoring or learning) |
| Reference-driven structural normalization while preserving existing names | `references/参考驱动结构整理.md` |
| Directory-driven batch discovery, complete `界面` View audits, GRID/Wrap, native mask evidence, and anomaly reporting | `references/批量ToUGUI界面结构审计补充规则.md` |
| Server-list example and target-specific lessons only | `references/测试skill-FigmaToUGUI结构整理记录.md` |

- Always read the Role overview. Read every other reference whose concern appears anywhere in the target subtree or requested change; multiple concerns require multiple references.
- Treat the server-list record only as a worked example. General rules and current observable target data override it; never copy its node IDs, sizes, names, or decisions into another target mechanically.
- Treat a behavior not stated in this file or a required bundled reference as unknown. Preserve the current structure and report the uncertainty instead of inferring exporter behavior from memory.
- Do not use or create a bundled Common component/image snapshot or path registry. Read only the default Common URL from `config/common-module.json`, then fetch current Common identities dynamically on every task as specified below.
- When the request is directory-driven or asks to learn every linked ToGUI/ToUGUI interface, also read `references/批量ToUGUI界面结构审计补充规则.md`. In that mode, enumerate every matching page and classify every `界面` Section child before treating any node as a formal View; this does not widen a single-node task.
- Use the available Figma Plugin API execution mechanism directly. Send standard JavaScript, await every asynchronous API operation, and never assume browser DOM, Node.js, filesystem, or network APIs are available inside Figma.
- Resolve behavior from observable Figma data: node type, name, hierarchy, layout, geometry, export settings, component identity, and variant metadata.
- Use this precedence for conflicts: explicit user instruction, safety and preservation rules, explicit rules in this `SKILL.md`, current observable target data, focused bundled reference, general bundled reference, worked example.
- Do not inspect or modify exporter code. A request to change exporter code is a separate task outside this skill.

## Workflow

### 1. Resolve the target and intent

Extract:

- file key;
- target node ID;
- effective Common module URL, file key, and node ID;
- requested scope: audit-only or modify;
- any explicit target role or component type;
- whether Page/Section organization is in scope.

Do not expand a single-node request into unrelated Page cleanup. Inspect ancestors, siblings, source components, Common definitions, and Sections only as needed to make the target correct.

### 2. Perform a read-only audit

Read the target, its complete descendant tree, and the minimum necessary context.

For PNG Export work, enumerate every descendant, including descendants of non-PNG
Instances. A component/reference boundary is not an audit stopping point. Record
ancestor PNG and exclusion/special-policy boundaries separately from enumeration so
their descendants can still be checked for duplicate Export or baked dynamic content.

Capture at least:

```text
id, name, type, parentId, sibling index, visible, locked,
x, y, width, height, absolute bounds, rotation,
layoutMode, layoutPositioning, sizing modes, constraints,
padding, spacing, alignment, clipsContent,
fills, strokes, strokeWeight and per-side weights,
strokeAlign/cap/join/miter/dash, cornerRadius and per-corner radii,
cornerSmoothing, effects, opacity, blendMode, exportSettings,
componentId/componentKey/mainComponent/variant properties,
text characters and fonts without returning image bytes
```

Also inspect:

- ancestors through Page and Section;
- siblings that affect z-order, clipping, popup background detection, list ordering, or ScrollBar placement;
- the dynamically fetched Common module subtree when an identity/name match or image-name collision is considered;
- source Component/Component Set metadata for Instances and Variants.

For a batch interface audit, read the complete descendant tree of every formal View. If the tool returns a large tree in chunks, record declared and read counts and failed offsets, merge by offset, and mark the View incomplete until the counts agree. Do not use incomplete or approval-rejected trees to infer positive exporter rules; report them as limitations.

For layout learning or adaptive/responsive layout work, also read `references/ToUGUI布局与自适应专项规则.md`. Trace each axis from the parent through containers and controls to text/images, distinguishing Constraints, flow sizing, absolute overlays, clipping, and runtime drivers. Node-tree coverage alone does not prove layout-property coverage. Under a read-only instruction, infer from observed fields without resizing, cloning, switching Variant states, or changing the Figma design; report actual resize and Unity tests separately.

For Variant authoring or Variant-learning requests, also read `references/ToUGUI变体制作与状态轴专项规则.md`. Read every Component Set's direct Component members, property definitions, real combinations, default member, internal structure, per-state layout, Instance identity, and Export boundaries. Do not switch states or modify Figma during read-only learning; a single visible member never proves the Set is structurally or responsively complete.

Record a compact before snapshot: target tree, absolute geometry, visibility, Export settings, and a deterministic structural fingerprint. Use IDs, ordered parent-child relations, names, types, geometry, layout fields, and Export settings. Recheck it before each write batch.

### 3. Classify before changing

Classify each node using plugin precedence:

1. Explicit Role name.
2. Instance -> `Reference`.
3. Auto Layout Frame -> `List`.
4. Text -> `Label`.
5. Rectangle/Image -> `Image`.
6. Frame/Group/Component/Component Set -> `Container`.

Explicit behavior and directive prefixes are required for `Button`, `Toggle`, `ToggleGroup`, `Input`, `ProgressBar`, `Slider`, `ScrollBar`, `Mask`, `UiMask`, and `NotExport`. A plain Frame reference also requires `Reference`. A static Auto Layout container must use `Container_*` or it will become a List.

These prefixes may normally be omitted only when inference is stable:

- Text -> Label;
- Rectangle/Image -> Image;
- true Instance -> Reference;
- non-Auto-Layout Frame/Group/Component -> Container;
- direct View Frame under `界面` -> UIRoot;
- true runtime Auto Layout list -> List;
- Variant member names under a correctly typed Component Set.

Inside compound controls, retain recognized responsibility names such as background, checkmark, fill, handle, label, content, and viewport even when the primitive type could otherwise infer a Role.

### 4. Build a deterministic plan

List every rename, create, move, clone, detach, instance replacement, visibility change, Export change, fill extraction, and geometry adjustment before writing. For each operation include source ID, destination parent/index, before/after name, absolute bounds, and reason.

Reject or pause on:

- duplicate candidates for a required role;
- ambiguous Toggle versus list semantics;
- multiple Common entries after identity/name normalization;
- nonzero transforms or Auto Layout moves that cannot preserve layout;
- a missing popup boundary that makes `UiMask` z-order grouping unsafe;
- required Text movement with unavailable fonts;
- a stale fingerprint.

### 4.1 Reference-driven structure (preserve names)

When the user supplies a correct-structure reference and asks to keep names, use the
reference as a hierarchy and responsibility model, not as a rename template. Read the
reference subtree, compare ordered parent-child edges, component identity, visibility,
layout, and bounds, then plan the smallest structural additions or moves that make the
target export semantics equivalent.

- Preserve existing target node names, artwork, text, and business suffixes unless a
  name is itself an explicit exporter directive that is invalid. Add a semantic wrapper
  when a missing role cannot be expressed without renaming the existing node. New
  wrappers may use the required directives (`Btn_click`, `NotExport`, `Image_bg`,
  `Image_fill`, `Fill Area`, `Handle Slide Area`, or an equivalent role name).
- Do not copy reference-only names, IDs, dimensions, sample counts, or a nested `Root`
  mechanically. A nested `Root` in a reference can conflict with UIRoot inference;
  retain the target's one valid UIRoot and express the inner responsibility with a
  wrapper only when the exporter requires it.
- Preserve visible design by moving existing nodes or using same-bounds structural
  wrappers. Do not invent copy, icons, colors, or decorative layers merely to match a
  reference tree. A placeholder may be an empty structural node only when the plugin
  needs a binding slot and the visual is supplied by an existing node or a verified
  Common Instance.
- For repeated items, keep the target's first flow item as the template. Put a full
  size `Btn_click` hit area inside the template/item structure when the reference has
  one, and verify its actual bindable Graphic or the reference's documented hit-area
  mechanism; an empty Frame with matching bounds alone does not prove clickability.
  Do not turn later demo copies into runtime items. Preserve their visible authoring
  layout when adding `NotExport`; hide only redundant replacement backups. Keep
  per-item red dots or badges under the item instead of leaving an unbound page level
  overlay, and suppress them only when the reference or requested scope establishes
  that they are design-only content.
- A ProgressBar fill source must cover the complete track and remain suitable for
  runtime fill/amount. Never use a cropped sample percentage as the source Sprite and
  never bake dynamic values, reward counts, or localized text into a static PNG.
- Record Figma node type and exporter Role separately. An `INSTANCE` keeps its component
  identity when named `Image_*`, but an explicit Role still precedes inferred Reference
  according to section 3. Supported PNG Export remains an image-source policy. Neither
  the name nor visual similarity alone justifies detaching an Instance or replacing it
  with Common. Resolve relevant component identities against the current Common index.
  If the user explicitly requires reuse for a same-looking component, insert a verified
  Common Instance and preserve the old node as an invisible `NotExport` backup. Missing,
  inaccessible, or ambiguous identity evidence remains unresolved; it is not proof that
  no Common match exists and does not trigger Clone + Detach.
- When the reference uses a Common component for a visual/control, prefer a real
  Instance with its source identity and preserve its original target name. Do not
  flatten that Instance into a local PNG, and do not replace a local node solely from
  screenshots or bounds. Respect source aspect ratio unless the Common definition is
  explicitly responsive (`@size`/Stretch) as described in the geometry reference.

For reference-driven edits, the plan must include a structural diff (missing wrapper,
wrong parent, wrong sibling order, missing hit area, missing placeholder, or incorrect
export suppression) and a visual-preservation check for every moved or wrapped branch.
After mutations, perform both a second tree/fingerprint audit and a screenshot or
render comparison of the complete target and the key compound controls. Report
structural differences that remain even when the visual comparison passes.

### 5. Apply in small verified batches

Limit each Figma script call to about ten logical mutations. Use one Page switch at most per call. Await asynchronous Plugin API operations.

Before a batch, re-read affected IDs and verify their parent, order, bounds, name, and relevant properties. For a visual extraction or repair, compare the complete static appearance bundle: fills; strokes and all stroke geometry; uniform and per-corner radii; corner smoothing; effects; opacity; and blend mode. After a batch, re-read them and compare against the plan. Stop after the first API error; report the partial state and do not immediately retry.

When cloning or detaching an Instance, assume descendant and sometimes ancestor identities can change. Re-discover the resulting subtree from the returned node and parent instead of reusing stale IDs.

### 6. Verify the exported meaning

After all edits, inspect the complete target tree and affected Page/Section context again. Verify:

- exact names and parent-child edges;
- absolute positions, sizes, sibling order, visibility, constraints, and Auto Layout behavior;
- Clip content and ScrollView routing;
- PNG Export format/scale/suffix and ancestor suppression;
- Component identity or detach result;
- Common image/component canonical-name collisions;
- absence of Chinese in every output-facing name, or explicit reporting of existing naming issues preserved by the user's scope;
- no hidden original lacking a formal replacement;
- no unexpected node outside the intended Section.

Capture a concise after snapshot and final fingerprint. Do not report success when only names changed but the export semantics remain wrong.

## Page and Section organization

- Derive the module from the first parentheses in the Page name. Prefer `ToUGUI（ModuleName）`; keep `ModuleName` ASCII and stable.
- Use real Page-direct `SECTION` nodes named in Chinese: `界面`, `组件`, `图片`. English `Views/Coms/Res` are compatibility names, not authoring names.
- For ordinary modules, create only `界面` by default. Create `组件` or `图片` only when their outputs are needed.
- If the target is already under `界面`, reuse it. Never create a duplicate `界面` or rename it to English.
- Do not nest the three Sections below a wrapper Section. Their direct children are independent export units.
- Treat each direct View Frame as a UIRoot. Standard size is `1080 x 2400` unless the bundled module rules or current target establish a module exception.
- Keep UIRoot free of Color Fill. Put the visual background in the lowest English-named Image/Rectangle child.
- Do not relocate unrelated Page-level samples into export Sections. Keep formal output units unique after name cleaning.

## Compound structures

Use plugin-recognized roles and edges. Preserve meaningful business suffixes and existing artwork.

```text
Button_*
├─ Image_bg
└─ Label_*

Toggle_*
├─ Image_bg
├─ Image_checked / Image_checkmark
└─ Label_*

ToggleGroup_*
└─ two or more descendant Toggle roots

Input_*
├─ Image_bg
└─ Text Area
   ├─ Label_input_holder
   └─ Label_text

ProgressBar_*
├─ Image_bg
└─ Image_fill

Slider_*
├─ Image_bg
├─ Fill Area
│  └─ Image_fill
└─ Handle Slide Area
   └─ Image_handle

ScrollBar_*_h|v
├─ Image_bg
└─ Image_handle
```

- Do not invent visuals, placeholder copy, or controls merely to fill a template.
- Keep behavior roots and Area nodes free of PNG Export.
- Make Button/Input hit graphics cover the intended clickable Rect. Keep overlapping background, checkmark, fill, handle, and text layers absolute or constraint-driven rather than flow siblings.
- Align Toggle checkmark with its background center. Ensure the intended label region is clickable.
- Overlap Input placeholder and input text with matching Rect/alignment.
- Align ProgressBar fill to the complete track; do not use a shortened source Sprite to encode a runtime percentage.
- Give Slider track, Fill Area, and Handle Slide Area the same axis and center; keep the handle's runtime travel range valid.
- Variant state names describe state, not Role. A Component Set that must behave as Toggle still needs a Toggle root Role. Preserve member `Property=Value` names and prefer one `checked/unchecked` axis.
- Add `ToggleGroup_*` only for real mutually exclusive choices. A container of Toggle children does not imply a ToggleGroup.

## List and ScrollView rules (project v2 standard)

### List identification (v2.1 strict)

- Audit lists only from the complete descendant tree of the requested target.
  The current expanded state of Figma layers is not audit scope. Expand every
  collapsed container or traverse it through the available Plugin API before
  reporting that a target has no list candidate. Declaring "no list" while a
  descendant container remains unexpanded is an invalid audit.
- **Mandatory candidate gate (the anti-omission rule):** first enumerate every
  descendant Frame and native GRID container without using its name, visual
  appearance, business meaning, or current selection as a filter. A node is a
  list candidate as soon as both hard conditions are true:
  1. it is an Auto Layout Frame or native GRID layout container; and
  2. it has at least two **direct** children whose item signatures repeat.
  Do not require a `list_*` name before detection. Do not reject a candidate
  because it looks like a rating row, star row, state row, card strip, reward
  row, or other apparently static visual group. `container_rating_stars`
  (`HORIZONTAL`, five equal direct star children, each with off/on states) is
  the regression example and must be reported as a candidate under this gate.
- After the hard gate, record confidence evidence rather than using it to hide
  the candidate: same component/source, same child responsibility tree, equal
  or stable item bounds, regular primary-axis spacing/padding, and runtime data
  semantics. If business intent is uncertain, keep the candidate in the list
  report and mark only the intent field `needsReview`; do not silently omit it.
- The deepest direct-child container that owns the repeated children is the
  processing root. A parent ScrollView/page area is not automatically the list
  root, but it must still be scanned for its own direct repeated children.
- Explicit exclusions must be evidence-based and recorded with an ID and
  reason. Alignment alone is not an exclusion. Button internals, a single
  repeated row, unrelated freeform siblings, and Toggle/Variant states may be
  excluded only when they fail the hard gate or the user explicitly says that
  the container is not a runtime list. A stateful repeated row that passes the
  hard gate remains a candidate and must be surfaced before mutation.
- Prefer the **deepest direct-child container** that actually owns the repeated
  items as the list root. A larger parent may be the ScrollView/page area, but
  it is not automatically the list root.
- Record for each candidate: root ID/name/bounds, layout mode/axis, sizing,
  gap/padding, item IDs, item source/structure signature, item count, and
  evidence for why the candidate passes or fails the rule.

### List scan completeness gate (must pass before any write)

- Run a dedicated candidate pass before renaming, Export changes, image
  processing, or deletion. The pass must return `treeNodeCount`,
  `layoutContainerCount`, `candidateCount`, `excludedCount`, and one record for
  **every** Auto Layout/GRID container, including containers whose names do not
  contain `list`, `item`, `star`, `card`, or any other expected keyword.
- For every layout container, compute direct-child signatures from ordered
  child types, relative sizes, names/roles, visible state branches, and source
  component keys. Never infer repetition from a screenshot or from a name-only
  search. Nested repetition does not count for the parent, but the nested
  container must be scanned independently.
- Before claiming completion, run the same full pass again. The after-pass must
  show no unreviewed hard-gate candidate. A candidate may be `processed`,
  `explicitlyExcluded` with an ID/reason, or `needsReview`; it may not be
  absent from the report. This second pass is mandatory even when an earlier
  candidate was already named `list_*`.
- Keep a regression record for missed candidates. When a candidate was omitted
  in an earlier run, add its ID and the detection evidence to the audit report
  and use it as a test case in the next scan. For this project the regression
  case is `232:484 container_rating_stars` with five direct repeated children.

### List normalization (v2.1 strict)

- `Clip content` is not a prerequisite: automatically enable `Clip content` on
  every confirmed list root and process it as a List/ScrollView.
- Before deleting later repeated items, change the confirmed list root's both
  axes from `Hug`/`Fill`-driven shrink behavior to fixed size equal to the
  original total outer bounds. Never let the root shrink after deletion.
- Naming enforcement: whenever a confirmed list root or any of its
  export-facing descendants still carries a Chinese or generic name
  (`列表`, `Frame 3`, `item1`, etc.), renaming is a mandatory part of the
  same list-normalization pass, not a deferred task. A confirmed list may
  not be reported as `complete` while its root is not `list_*` named. Text
  `characters` (visible copy such as question text) stays unchanged; only
  layer names are renamed.
- In naming-cleanup scope, rename the list root to a `list_*` name and its
  export-facing descendants to English while preserving List/ScrollView
  semantics. In structure-only, preserve-names scope, report existing name
  issues and apply section 4.1.
- Item template rule: use the first repeated child as the only export
  template. The template Item's outer bounds must equal the **union bounds
  of all observed states of the same item** (for example the normal/selected
  overlays of item1), not the union bounds of the entire list. The list root
  is the outer runtime range; the surviving item keeps or wraps only its own
  state range. Delete all later repeated items. Preserve the list root's
  original total size, alignment direction and padding. Do not wrap later
  items in `NotExport_*`; this project deletes them instead of keeping
  visible demo copies.
- If the first repeated child is a Component/Instance and enlarging it to
  its own state union would stretch, crop, or mutate its internal component
  visuals, do not resize the Instance directly. Create an outer Item
  template container with that state union size, place the original
  Instance inside without changing its component identity or original size,
  and preserve the Instance's design position within that container unless
  the user explicitly defines a runtime slot rule.
- If an outer wrapper was already created against the old whole-list rule
  and it is larger than the item's own state range, correct it: keep the
  list root at its original size, restore the inner item to its own state
  bounds, and resize the wrapper to the item's state union (not the list
  bounds).
- Separate the Item Auto Layout container from independent overlays. A
  ScrollBar must not be placed inside the repeated-item container such as
  `ServerItems`.
- Place ScrollBar as a sibling of the item container under the nearest
  List/ScrollView root that can bind it. If that root is Auto Layout, set
  the ScrollBar to absolute positioning so it does not become Content.
- Use the ScrollBar root as the full track. Keep `Image_handle` size and
  endpoint position meaningful; Unity creates `Sliding Area`.
- Preserve viewport space or explicit overlap intentionally; the generated
  Viewport does not automatically shrink for a ScrollBar.
- Export and structure validation: list roots and behavior roots do not
  carry PNG Export; only their static visual sources inside the surviving
  template item may carry the standard PNG 1x Export.
- When a reference example defines a correct List/Grid structure, use the
  example to validate hierarchy and responsibilities, but the project v2
  list rule above controls template selection, item deletion, root sizing,
  and `Clip content`.
## Fill, clipping, and background extraction

- UIRoot: remove direct Color Fill and create/reuse a lowest-layer English background visual of equal intended bounds.
- Treat background extraction as a complete static-appearance transfer, not a Fill-only copy. Copy the source's visible fills, strokes, uniform and per-side stroke weights, stroke align/cap/join/miter/dash, uniform and per-corner radii, corner smoothing, effects, opacity, and blend mode when those properties contribute to the rendered background. Preserve the destination node's name, parent/index, geometry, constraints, and intended PNG Export unless the plan explicitly changes them.
- When the background node already exists but its appearance is incomplete, recover the missing values from the closest untouched equivalent: prefer the original pre-extraction node or preserved backup, then a same-state repeated item/Variant member with matching geometry and Fill. Do not infer stroke or corner values from visual similarity alone when more than one candidate differs.
- Keep the existing rule to clear only the root Fill unless the plan explicitly transfers another root property. A behavior root may still carry Stroke or corners for authoring/hit-area purposes, while the PNG background must independently contain every property needed in the baked Sprite. After copying, screenshot both the background node in isolation and the complete compound root; reject missing corners/strokes and visible double-stroke artifacts.
- If a non-Image node has Fill plus additional static visual properties such as stroke, effect, gradient, image fill, or complex clipped decoration, prefer a dedicated PNG visual source rather than reconstructing it as a plain color.
- Rectangle nodes in formal View/Component trees normally require one `PNG 1x`, contents-only, empty-suffix Export unless excluded by UiMask, NotExport, ancestor PNG, or another specialized policy.
- For Clip content with clipped visual descendants, first extract recognizable non-image roles such as Text and controls so they remain real UGUI nodes. Then export only the remaining static clipped visual root.
- When Text cannot be safely moved because fonts are unavailable, duplicate the clipped visual as a same-size image source, hide dynamic roles in the visual copy, and keep the original structural branch for active roles.
- Preserve sibling order and absolute bounds when splitting visual and structural branches.

## PNG Export decisions

When the user asks to check or configure image Export, scope the work to Export
settings on the target's actual image sources. Preserve names, hierarchy, geometry,
component ownership, and artwork unless a structural blocker is separately in scope.
Report added, corrected, removed, and intentionally unexported nodes. A Figma Export
setting audit is distinct from downloading PNG files or validating Unity output.

### Automatic selection and recheck

Requests to set, automatically check, supplement, or correct image Export, including
rechecks of that work, mean execute the certain in-scope changes after inspection.
Do not stop at a recommendation or require another confirmation for those Export
settings. Explicit audit-only / 仅检查不修改 requests remain read-only. Follow section 13
of the PNG reference for the decision table and execution procedure.

- Default each accepted source to exactly one `PNG / SCALE 1 / contentsOnly:true /
  suffix:""` setting unless the user or project explicitly specifies another policy.
  Apply it through the Figma API; a report or selection highlight is not a checked Export.
- Inspect static bitmap fills, visible strokes even with Fill disabled, effects,
  Boolean Operation roots, and complete mask-plus-tint compositions inside Instances.
  Do not select every leaf or use the node name alone as the classifier.
- Preserve valid Instance identity; change only supported Export overrides within the
  target. Do not detach, migrate gameplay-local components, or edit remote definitions
  merely to mark images. An unsupported override remains unresolved.
- Treat effective NotExport/UiMask/placeholder trees, transparent interaction-only
  HitAreas, runtime image slots (including their preview tint), and recognized NineSlice
  branches according to their own policy. Do not count them as ordinary missed PNGs.
- For a genuine control root with mistaken PNG, move only the Export setting to its
  existing complete static source(s) when every newly exposed static branch is covered
  and independent dynamic/control nodes remain intact. If this needs restructuring or
  has ambiguous roles, leave it unresolved in Export-only scope and finish other items.
- Keep the operation idempotent: skip settings already matching the intended policy;
  do not accumulate additional Export entries. Remove redundant descendant PNG only
  when a valid ancestor already represents the same indivisible static visual.
- After writing, re-enumerate the complete tree, verify settings, parent-child PNG
  duplication, and dynamic descendants, and compare non-Export properties and component
  keys with the before snapshot. Report formal sources separately from PNGs suppressed
  by backups or other exclusions. A moved Export is one move, not an extra missed image.

### Initial Export pass

When the request asks for an initial, conservative, or preliminary Export pass, read
`references/ToUGUI初步Export勾选规则.md` in addition to the PNG reference. Use its
`safeInitialCandidates`, `existingValidExports`, `intentionallyUnexported`,
`needsReview`, and `suppressedDescendantExports` categories to make the first selection
auditable. Initial selection is not a leaf-node sweep: preserve layout roots, behavior
roots, dynamic content, GRID/Wrap/List structure, runtime-driven control parts, valid
exclusion trees, and unresolved boundaries. When the user authorizes actual Export
changes, apply only the certain candidates through the existing section 13 workflow and
report Unity verification separately.

### Source placement

Treat Figma Export as "bake this visible subtree into one Sprite," not as a generic inclusion switch.

Set a single PNG Export on the final indivisible static visual source when needed. Prefer `1x`, contents-only, no suffix unless the project explicitly requires otherwise. Never duplicate PNG Export on both a semantic Image container and its visual child.

- Do not add blank PNG assets to fully transparent, interaction-only HitArea nodes.
- Keep runtime `Image_*` slots and their `placeholder` / `PlaceLimiter` authoring
  previews without PNG; those preview names carry exclusion semantics which PNG can
  override.
- For a static masked visual, include the mask and the masked color/artwork in the
  same final PNG source. Do not export only the mask or only its tint rectangle.
  Preserve separate sources where parts must change independently; never assume that
  an explicit Image container automatically merges several nested PNG sources.

Do not set PNG Export on:

- UIRoot;
- Button, Toggle, ToggleGroup, Input, ProgressBar, Slider, ScrollBar, List/ScrollView roots;
- Text Area, Fill Area, Handle Slide Area, Sliding Area;
- Component Set or Variant member roots;
- dynamic/localized Text;
- UiMask or NotExport;
- a layout root whose children must remain independent.

An Instance with a supported PNG Export is an image source first. Keep it in place and skip Common matching, backup wrapping, cloning, and detach.

Before naming any PNG source, compute its canonical basename: remove the recognized `Image_`/`Img_` Role prefix, remove Export suffix text, strip `@` parameters, trim separators, and compare case-insensitively. Compare against the image/component names dynamically fetched from the effective Common module URL. If naming changes are in scope and a local PNG would collide, rename the Figma visual source to a unique English business name. In Export-only or preserve-names scope, preserve existing names and report collisions instead. Avoid generic names for new sources such as `bg`, `icon`, `fill`, `handle`, `image_bg`, and `Rectangle 1`; in particular, treat `image_bg` and `bg` as the same canonical name. If collision checking is required but no valid Common URL is available, report it as unresolved instead of assuming no collision.

## Dynamic Common module resolution

Determine component ownership before Common matching. A component confirmed by the
user or current module context as gameplay/module-local is not a Common candidate.
Preserve its real Instance and verify its source identity against the local definition
or supplied reference. Absence from Common does not justify migration, replacement,
or detach. Report unresolved Unity Prefab availability separately; matching the Figma
source alone does not prove the Prefab can be resolved after export.

Never hardcode a Common file key, node ID, component list, image list, or URL in the instructions or generated scripts. Resolve the URL through the bundled config on every task:

1. Read `config/common-module.json`.
2. If the user supplies a Common module URL for the current task, use it as a run-scoped override. Otherwise use `defaultCommonModuleUrl`.
3. Never persist a run-scoped override or edit the config unless the user explicitly asks to change the default Common URL.
4. Parse the effective Common URL's file key and `node-id` independently from the target URL.
5. Fetch that exact Common target afresh before planning target mutations, even when a previous task already fetched it. Do not reuse a component/image list cached in chat, files, or memory.
6. Read the minimum Page/Section context needed to identify the Common module. Do not modify the Common document unless the user explicitly asks.
7. If fetching Common changes the active Figma file or Page, reopen the exact target URL afterward and recheck the target fingerprint before any write. Never apply target mutations while the Common document is active.
8. Index the current Common module dynamically:
   - under the real Page-direct `组件` Section, collect direct export units and descendant Component/Component Set identities;
   - under the real Page-direct `图片` Section, collect direct image export units and their canonical names;
   - record node ID, component key, component ID, normalized name, source Section, Variant metadata, visibility, and export settings.
9. Record whether the URL came from config or a run-scoped override, plus the URL and fetched node IDs, in the audit snapshot.
10. If the config, URL, or Common node is absent, inaccessible, stale, or ambiguous, continue only with changes proven independent of Common. Do not replace or detach an affected Instance, do not assume a local PNG name is collision-free, and report the failure in `warnings`/`needsUserDecision`.

Match Common components using the dynamically fetched index in this order:

1. `componentKey`;
2. component ID;
3. localized node ID;
4. unique normalized name.

Normalize names case-insensitively, trim whitespace and separators, and strip `@size` and other `@` parameters. Never match by visual similarity or bounds alone. Skip name fallback if more than one Common definition has the same normalized name. If a required component identity is absent from the fetched Common module, leave the node unresolved; do not search a static registry or another source.

For a node that should use a Common component but is not its actual Instance:

1. Wrap the original in a Frame named `NotExport` and set the wrapper invisible.
2. Create a real Instance from the matched Common component at the original sibling position.
3. Restore the original absolute position, bounds, layout sizing, and intended scale/`@size` behavior.
4. Verify that the formal Instance and hidden backup are both present.

For an Instance without PNG Export, use the following fallback only when a complete,
fresh Common index establishes that no match exists and Clone + Detach is within the
authorized plan. Missing or ambiguous identity evidence is unresolved, not a confirmed
non-match. If the task requires Common reuse, preserve unresolved content and report it
instead of silently substituting this fallback. Apply the operating contract for any
detach decision, reusing existing user authorization where it covers the operation.

1. Clone it at the same position and sibling order.
2. Detach the clone.
3. Wrap the original Instance in an invisible `NotExport` Frame.
4. Process the detached copy recursively as ordinary nodes.
5. Verify parent, order, bounds, layout sizing, and the formal replacement.

Never leave only a hidden `NotExport` reference when the content is meant to export. A backup needs a formal Common Instance or detached replacement. No replacement is required only when the entire subtree is intentionally excluded, such as design notes, demo duplicates, or covered scene background.

## Project-specific questionnaire conventions (v2, validation draft)

These conventions are authorized project rules for the current questionnaire/UI
migration workflow. They refine naming, visual-text handling, page layering,
list behavior, and responsive constraints. The list standard in this section is
the project v2 list rule and overrides older generic list guidance when they
conflict.

### Three-layer page organization

- When the target is a full questionnaire/view, first classify its direct page
  children into three responsibilities: upper/background visual layer, middle
  content layer, and lower/page-action layer. Prefer this order and separation:

  ```text
  ViewRoot
  ├─ container_*   # upper: background and decorative visuals
  ├─ ContentRoot   # middle: content, controls, and pending list area
  └─ Button_*      # lower: page-level actions such as back/close
  ```

- This is a responsibility model, not a command to create empty wrappers. Reuse
  existing containers and preserve sibling order, artwork, geometry, and clipping.
  Create or move nodes only when the layer boundary is evidenced by current
  hierarchy, z-order, geometry, or the user's explicit structure.
- Classify the list area with the project v2 list standard in the List and
  ScrollView rules section: enable `Clip content`, name the root `list_*`, keep
  only the first repeated child as the template, delete later repeated items,
  and keep the list root's original total size.

### Short visual-resource naming

- For this project, static visual/image sources may use concise lowercase prefixes:
  `img_*` for image/art resources, `container_*` for structural visual groups,
  and `list_*` for any list root that satisfies the project v2 list standard. Preserve full semantic behavior prefixes such as `Button_*`, `Toggle_*`,
  `Input_*`, and `Label_*` for runtime-facing nodes.
- Do not expand a valid short project name into a long generic name merely for
  stylistic consistency. Names must still communicate responsibility; `img_bg`,
  `img_biaoti`, and `img_wancheng` are valid project names when their visual role
  is evidenced.
- Same-level duplicate short names are not silently accepted for new or renamed
  export sources. Add a stable semantic suffix such as `_01`, `_02`, `_left`, or
  `_right`, chosen from geometry/role evidence. Existing duplicates are reported
  in `needsReview` until a deterministic distinction is available.
- Do not use this convention to rename remote Instance internals, alter Variant
  property syntax, or replace a real component identity.

### Font-based artistic-text policy

- `FZCuYuan-M03S` is the project's approved ordinary UGUI text font. Text using
  this family must be named exactly `txt` (not `Label_*`) and remains a live
  runtime Text node; preserve its Chinese `characters`, never add a PNG Export
  to it, and inspect dynamic/localized behavior before finalizing any runtime
  behavior binding. Generic compound-control labels (`Label_input_holder`,
  `Label_text`, and equivalent input placeholders) keep their existing
  compound-control names and are not affected by this rule.
- Text using any other font family is an artistic-text/image candidate by default.
  Keep the source text and characters unchanged, place or retain it inside a
  semantically named `img_*` visual source when the visual grouping is evidenced,
  and set exactly one standard `PNG 1x`, `contentsOnly:true`, empty-suffix Export
  on the final indivisible artistic visual source.
- This policy does not authorize changing fonts, rewriting text, flattening a
  live label, or detaching an Instance. If the artistic Text must remain a live
  node for structure or interaction, preserve it and report the unresolved export
  boundary instead of guessing.
- The source Text is not automatically hidden, deleted, or wrapped in `NotExport`.
  Only suppress it after verifying that the selected `img_*` source is the sole
  formal visual output and that no duplicate Text GameObject or dynamic behavior
  is required. Parent/child PNG duplication is a `needsReview` condition unless
  the sources are intentionally independent runtime visuals.
- Font family alone is not enough to prove that a Text is static: inspect text
  role, localization/runtime evidence, parent visual source, visibility, and
  screenshot before finalizing the Export boundary.

### Nine/Three-slice (九宫/三宫) image section policy

- **Local plugin source (mandatory):** For every nine-slice or three-slice
  operation, first inspect and use the user's local Figma plugin at
  `G:\a_项目\new插件`. Its entry manifest is
  `G:\a_项目\new插件\manifest.json`; verify that `main` resolves to
  `G:\a_项目\new插件\dist\code.js` and that the UI resolves to
  `G:\a_项目\new插件\dist\ui.html` before running it. Do not substitute
  another similarly named community plugin unless the user explicitly asks.
- In Figma Desktop, load/run this local plugin through
  `Plugins → Development → Import plugin from manifest` (only when it is not
  already installed), then run the installed entry named `九宫格插件` on the
  selected source image. Run it separately for each eligible asset and choose
  three-slice or nine-slice according to the proven stretch axes. If the
  manifest/build output is missing or the plugin reports an error, stop the
  structural mutation and report the exact blocker instead of manually faking
  slice output.
- The plugin-generated slice tree is the source of truth. After execution,
  read back the generated nodes and verify their bounds, names, constraints,
  clipping, and export settings before componentization and replacement.
- Before any local-plugin operation, read the complete behavior contract in
  `references/本地九宫格插件逻辑.md`. It is authoritative for this plugin's
  selection rules, 10% defaults, 0-based `slice-row-col` names, zero-orthogonal
  inset three-slice mode, `stretch/tile` behavior, shared plugin data, source
  preservation, and the fact that the plugin itself does not write Figma
  `exportSettings`.

- When an image resource can be used as a nine- or three-slice asset, the
  agent must:
  1. Reuse an existing Page-direct `SECTION` named `图片`, or create one when
     absent.
  2. Treat the user's previously applied Figma nine-slice plugin output as the
     source of truth for the processed asset.
  3. Name the processed image with the project short convention `img_*` and
     set exactly one `PNG 1x / contentsOnly:true / empty suffix` Export on the
     processed unit.
  4. Replace the on-screen visual with an image reference pointing back to the
     `图片` Section source while preserving on-screen geometry, constraints,
     and clipping.
- Never duplicate the processed nine-slice inside the view tree; the view
  references the `图片` Section source. Parent/child duplicate Export of the
  nine-slice is prohibited.
- Do not alter behavior roots, text nodes, or Instance identities when
  inserting a nine-slice reference.

### Constraints and adaptive layout policy

- For full views and their major modules, inspect and preserve per-axis Figma
  `constraints` as first-class layout evidence. Do not treat a screenshot or fixed
  coordinates alone as proof of adaptive behavior.
- Preferred evidence patterns for this project are:
  - background container: `CENTER/CENTER`;
  - centered content/list: `CENTER` on the relevant axis;
  - bottom-centered primary action: `CENTER/MAX`;
  - bottom-left page action: `MIN/MAX`;
  - scaleable decorative artwork: `SCALE` only when its visual role and bounds
    support scaling.
- These are evidence-based defaults, not blind assignments. Preserve existing
  constraints unless the user requests normalization and the before/after geometry
  can be verified. Record each changed axis and its reason.
- Validate constraints together with parent size, local/absolute bounds,
  `clipsContent`, Auto Layout mode, positioning mode, and sibling overlap. A
  constraint change must not be reported as a successful responsive solution
  without actual resize/Unity verification; report that verification separately.

### Export and visual-boundary conventions

- Static `img_*` sources normally use one `PNG 1x / contentsOnly / empty suffix`
  setting. Behavior roots (`Button_*`, `list_*`, `ContentRoot`, and the UIRoot)
  remain unexported unless an explicit project exception is evidenced.
- For a visual group such as `img_wancheng`, choose either one complete parent
  Sprite or intentionally independent child Sprites. Do not leave parent and child
  PNG settings duplicated without recording the independent-runtime reason.
- A newly visible state is an observed state change, not proof that it is the
  default runtime state. Record `designState`, `visibility`, and `exportState`
  separately and leave runtime-state interpretation unresolved unless supported by
  prototype/variant evidence or user instruction.

### Validation and list boundary

- After applying these conventions, re-read the complete target tree and report:
  direct layer classification, short-name duplicates, all Text font families,
  artistic-text candidates, PNG parent/child duplication, constraints, geometry,
  visibility, and component identity.
- List validation uses the project v2.1 strict standard. First report every
  list candidate and the identification evidence required by `List
  identification`. Then report list-root name, whether `Clip content` was
  enabled, surviving template-item name and outer bounds (must equal the union
  bounds of that item's observed states, not the whole-list bounds),
  deleted-item count, kept list-root size, fixed
  root sizing, preserved Auto Layout direction/alignment/padding, and unchanged
  Component/Instance identity when an outer Item wrapper was needed. A list
  that cannot be verified against these points goes to `needsReview`, not
  `complete`.

## UiMask and NotExport

- Use exact `uimask` or an explicit `UiMask_*` name only for a single visible solid Fill whose metadata belongs to the nearest exported parent.
- Remove PNG Export from UiMask. It produces no GameObject and suppresses its subtree; it is not a visible overlay or a clipping Mask.
- For a non-fullscreen popup, identify the popup background above the mask using z-order, bounds, and business structure. It may be a smaller image node or Component Instance.
- Put only the covered scene/background nodes below the mask into a new `NotExport` Frame. Do not group popup labels, controls, lists, or other formal content solely because their bounds intersect.
- If a node or ancestor already has `NotExport`, `NoExport`, or `Ignore` semantics, do not wrap it again.
- A `NotExport` node whose descendant itself has PNG Export is an intentional suppressed image subtree; do not run Common replacement/detach logic inside it unless the user explicitly wants that subtree exported.

## Result format

Report the outcome concisely with:

```text
target: { url, fileKey, nodeId, beforeName, afterName }
commonSource: { source: config|override, url, fileKey, nodeId, fetchedNodeIds, fresh }
classification: { module, section, role, compoundType, confidence, evidence }
changes:
  renamed / created / moved / cloned / detached /
  visibilityChanged / exportChanged / fillsExtracted / commonReplaced
verification:
  hierarchy / geometry / autoLayout / clipContent /
  exports / commonReferences / names / fingerprints
untouched: [{ nodeId, reason }]
warnings: []
needsUserDecision: []
```

When the request says to keep names, explicitly report `renamed: []` (or list only
names that had to change because an exporter directive was invalid); structural wrapper
creation and reparenting belong in `created`/`moved` instead.

For audit-only requests, return the same structure with a proposed deterministic plan and make no edits. For failed or partial operations, identify the last verified batch and every node whose final state is uncertain.




### 范例驱动的图片与美术组件分区规则（新增规则）

- 参考文件中“界面”“样式状态-美术组件”“图片”是 Page 的直属同级 SECTION；外部总 Frame 只可作为画布背板。整理副本时使用英文兼容前缀，例如 Views_<Module>、Coms_<Module>Art、Res_<Module>Images，不得把三个产物区再包进一个总 Section 或总导出 Frame。
- Views Section 的直属子节点才是正式 View 根。保持每个 View 根 1080×2400、可见、唯一命名；可复用背景、图片和组件必须作为 View 内真实 Instance 消费，不能只在旁边摆一张视觉相同的 Frame。
- Res Section 的直属子节点使用真实 COMPONENT 作为图片源。可拉伸图片沿用九宫格/三宫格：角使用 MIN/MAX 固定，边按单轴 STRETCH，中心按双轴 STRETCH；图片源根通常设 PNG 1x / contentsOnly / 空 suffix，slice 子层不导出。组合阴影或遮罩若不是同源九宫格，保留其原构图并单独建源，不能凭尺寸直接替换。
- Coms Section 放主题和状态组件集。证书、场景背景、弹窗背景等按真实语义轴建立 Category=Mining/Farming/Fishing/Insect 等完整组合；奖励卡按 State=Locked/Claimed/Selected 等真实状态轴保留已有状态。Component Set 与 variant 根不导出，静态装饰或图片边界按实际资源规则导出。
- 重复的本地视觉 Frame 可在不改变其子层、几何、填充、可见性和效果的前提下转换为本地 Component，并以真实 Instance 回连 View；需要保留的原 Frame 放入隐藏 NotExport_* 备份。不得为了整理 Detach 远程/Common Instance、改写远程主组件或用外观相似但来源未验证的资产替换它。
- 变体只在存在真实消费差异时创建；没有现成弹窗消费者时可以保留完整 PopupBackground 变体源，但不要给界面凭空增加弹窗实例。重复场景若只有一份视觉构图，优先一个可复用主组件并保留页面级效果 override（例如奖励页的模糊）。
- 自适应布局必须同时检查根尺寸、子节点尺寸、Constraints/Auto Layout 和运行时拉伸关系。九宫格纹理源不能直接代替动态进度条；进度条应保留轨道、mask、fill 与动态尺寸，图片源只负责可拉伸纹理。
- 调整完成后重新扫描完整副本：确认 Section 直属角色、真实 Instance→mainComponent 链、变体轴组合、PNG 设置和中文/泛化层名；对原稿与只读范例保存结构指纹，并对 View 做至少一次导出或截图复核。字体缺失时不替换字体、不重排文字，记录限制。



### 历史 ToUGUI 页面三分区回顾（新增规则）

- 当任务要求整理完整 ToUGUI Page 或回顾页面制作规律时，先读取 references/历史ToUGUI页面三分区与资源规律.md，再扫描该 Page 的全部直属 children。不要把只枚举“界面”的历史批量审计当成“其他 Section 不存在”的证据；旧快照须标记为 coverage-limited。
- 完整模块的组织目标是三个同级、Page-direct 的职责区：Views_<Module>、Res_<Module>Images、Coms_Art_<Module>。历史中文旧区“界面”“图片”“样式状态-美术组件”分别按 Views、Res、Coms_Art 识别；在用户已授权的英文命名任务中直接改名并复用原 Section，不能创建中英重复区。不得在总 Frame/总 Section 中嵌套三个区。
- 对完整 Page 重建任务默认准备并检查三类区；若反向审计确实没有对应资源，空区可以保留但要记录 empty/needsReview，不能伪造 Component、图片或状态。只读审计、单节点整理或用户限制不扩展 Page 时，不创建空区。若页面另有“组件”“色值”“动效”等直属 Section，保持其独立角色。
- 先遍历所有 View 内真实 INSTANCE，反向汇总 mainComponent、component key、variant properties、消费尺寸和出现次数，再按 source key 去重建立 Res/Coms canonical 清单。相同来源只保留一个定义；不能按名称、外观或尺寸重复造资源。
- Res 只放独立、无行为、可复用的视觉源：真实 COMPONENT/图片 COMPONENT_SET、背景、标题条、边框、装饰和九宫格/三宫格 Sprite。源根按 PNG 规则设置 1x / contentsOnly / 空 suffix，slice 子层不独立导出；View 内保留真实 Instance。包含动态 Text、Input、List、Toggle、进度逻辑或行为根的结构留在 View/Coms。
- Coms_Art 只放有主题、状态、等级或复用语义的 COMPONENT/COMPONENT_SET。保留观察到的 Theme/Category、State、Property 等轴和值，只建真实存在或有明确消费证据的组合，不补造笛卡尔变体。Set 根和 variant 根通常不 PNG，成员中的独立静态图按 Res/PNG 规则处理。
- 背景和美术必须保持可独立变化的分层：九宫格底图、底色、阴影、前景装饰、标题条、遮罩可分别作为源和实例；不能为了填满图片区把多层视觉合并为一张静态 PNG。实例尺寸、源尺寸、MIN/MAX/STRETCH 约束和运行时动态尺寸要一起复核。
- 若将重复的本地视觉 Frame 转为组件，先保存原层级、几何、约束、填充、效果和可见性，并在需要时移入隐藏 NotExport_* 备份后用真实 Instance 回连；不得为了分区整理 Detach Common/远程实例或改写远程主组件。
- 每个 Page 完成后输出三类区是否存在、是否 Page-direct、direct-child 类型、空区状态、Instance→source 链、变体轴组合、根/子层 Export、中文/泛化命名及 View 尺寸。正式 View 通常为 1080×2400，但以当前页面实际规范和证据为准。
- 本新增条文只扩大完整 Page 的审计和资源归属覆盖，不取消旧有的只读边界、最小安全修改、字体保护、PNG 复查和“不删除原内容”规则。



- Section inventory must recognize all observed aliases before naming changes: Views/界面, Res/图片, Coms/组件, and historical 样式状态-美术组件 as an ArtComs candidate. Preserve each Section's original index/order and empty-child state in the audit; Section order is not a dependency. The historical ArtComs alias is not a default Coms export prefix, so an authorized production rename to Coms_Art_<Module> is required for unambiguous export classification.



- Resource ownership is proven by the source's ancestry, not by a matching name or screenshot: for every View Instance, follow getMainComponentAsync() to its source Component/Component Set and verify that the source parent chain reaches a confirmed Res, Coms, or Coms_Art Section in the same module; the source may be on another Page of that module, which must be recorded. Common/remote sources remain external dependencies and are reported separately; unused local sources are reported as orphaned instead of silently exported.



- 历史兼容例外：Res 直属资源优先使用 COMPONENT/COMPONENT_SET，但如果已有静态 FRAME/GROUP 具备明确独立边界、PNG 设置和可复现视觉（例如历史 CollectFish 的 img_xlnamebg），应保留其原生类型，不为统一类型而转换或 Detach。只有资源需要真实复用身份、Instance 链或组件变体时，才要求建立 COMPONENT/COMPONENT_SET。



- 全量历史直属 Section 复核（79 个 Page，全部成功）：54 个含“图片/Res”，22 个含“样式状态-美术组件/美术组件”，18 个含普通“组件”；组合中有 19 个仅“图片+界面”，13 个“图片+美术状态+界面”，7 个“图片+界面+组件”，7 个仅“界面”，另有 17 个辅助/未组织页面没有直属 Section。该统计说明三类 Section 是完整页面的检查模板，不是每个页面都必须机械创建的空结构。
- “组件”与“样式状态-美术组件/美术组件”可以并存：前者承载业务或交互组件，后者承载主题、状态和样式轴。历史还出现“时装”等业务美术 Section，须按内容分类并报告，不要把未知 Section 自动当作 Coms 或 Res；无意义的“Section 1”应列入命名待处理。
- 当页面没有真实静态源或状态组件时，保留已有空 Section 并报告即可；页面没有该 Section 时，只有完整页面标准化任务、用户明确要求补齐模板，或资源反向索引提供证据时才创建。不要把“多数页面有图片区”升级成所有页面强制新增资源。

### 历史三分区规则的优先级澄清（追加覆盖）

- 本节优先于本文件及 `references/历史ToUGUI页面三分区与资源规律.md` 前文“正式结构使用三个”“默认准备三个”“Res 必须 Component”及同 Page 限制等宽泛表述。三类 Section 是完整 Page 审计的检查项，不是机械创建要求；即使任务为完整 Page 标准化，没有真实资源且用户未明确要求模板时，也不新建空 Section。已有空 Section 可保留并记录 `empty/needsReview`；符合独立静态资源边界的已有 FRAME/GROUP 保留原生类型，不为统一类型强制组件化。
- `样式状态-美术组件`、`美术组件`、`时装` 等历史创作资产区先按内容分类；它们不是默认正式导出前缀。只有明确需要组件 Prefab 产物、且导出标准化在授权范围内时，才将相应区映射为 `Coms_Art_<Module>`；仅英文命名时可用 `ArtComponents_<Module>` 保持非导出用途。该条件覆盖前文无条件映射 Coms_Art 的示例；不能因翻译名称让原先不导出的创作资产自动成为正式产物。
- View Instance 的资源归属应追溯到已确认的同模块 Page 的 `Res`、`Coms` 或 `Coms_Art`；跨 Page 时记录模块关系和来源 Page。Common/remote 来源单独记录为外部依赖，不复制、Detach 或改写远程定义。

### 新增节点的简短语义命名（追加规则）

- 只对本次新建的节点、Section、资源源和组件定义启用简短命名；已有合法名称不因“简化”批量改写。新名称优先使用 1–3 个英文语义词，省略已经由 Page、Section、父节点或组件集表达的重复模块名、场景名和实现细节。
- 仍须保留导出器需要的 Role/指令前缀、Variant 属性语法和唯一性：例如 `Image_Background`、`Button_Claim`、`Label_Title`、`RewardCard`；不要把 `Button`、`Toggle`、`List`、`UiMask`、`NotExport` 等行为或排除标记缩短成含义不明的别名。短名应使用常见完整英文词，不用只有缩写、拼音或坐标编号的名称。
- 同一父级下只有发生职责或视觉差异时才加后缀，如 `Icon_Left`、`Icon_Right`、`Background_Popup`；同职责重复项用稳定编号消歧。不要重复拼接 `Module_Page_Section_Component_Image` 这类祖先路径，也不要把 `Frame`、`Rectangle`、`Component` 当作新增业务语义。
- 简短不等于含义过泛：跨同级可能重复或无法判断职责时，不单独使用 `Bg`、`Icon`、`Item`、`Card`、`State`、`Part`、`Group`、`Btn`、`Img`；补上最短必要限定词，例如 `PopupBackground`、`RewardCard`、`CurrencyIcon`。必须保留 `Toggle_*`、`ToggleGroup_*`、`ProgressBar_*`、`UiMask_*`、`NotExport_*` 及控件内部 `fill`、`handle`、`checkmark` 等有效责任标记。
- 新建 Page 直属 Section 时，Page 名已经提供模块作用域，英文命名优先使用 `Views`、`Res`、`Coms` 或 `Coms_Art`；只有同一 Page 存在同类多个区、跨 Page 需要区分或导出路径确实要求时，才追加最短业务后缀，例如 `Views_Popup`、`Res_Images`。已有 Section 直接复用，不能为简化而创建中英重复区。
- 新增名称简化后仍须通过中文层名、泛化命名、同级冲突、PNG canonical basename、Component/Variant 引用和导出路径检查；名称短不等于可以省略必要的唯一限定词。已有合法长名只在用户明确要求重命名、存在冲突或导出规则要求时处理。

### 图片 Export 覆盖完整性（追加规则）

- “应该导出”必须按完整树的静态视觉职责判定，不能只搜索 `Image_*`、Image Fill 或当前截图。展开非 PNG Instance 的内部节点，检查 Fill、Stroke-only、Effect、Boolean、Image Fill、复杂 Mask+tint、静态装饰和被裁剪的完整视觉根。
- 对每个独立静态视觉源建立唯一覆盖结论：`exported`（自身有合规 PNG）、`coveredByAncestor`（合规祖先完整覆盖）、`specialPolicy`（NineSlice、ForcedSectionRender、UiMask/NotExport/placeholder 等专用规则）、`intentionallyUnexported`（动态或结构职责）或 `needsReview`（证据不足）。有绘制内容但没有任何结论的节点计入 `missingCoverage`，不能把它静默当作已完成。
- 补勾后必须重新遍历完整目标树并回读设置，确认每个静态源都有合法来源，父子 PNG 不重复，动态文字/控件/运行时插槽没有被烘入；`missingCoverage` 必须为 0 才能声称 Export 完整。若需要移动、拆分、Detach、改主组件或结构证据不足，保留 Figma 原状并将节点及原因列入 `needsReview/unresolved`。
- 报告正式图片源、祖先覆盖、专用策略、主动不导出、漏导出和待复核节点的 ID 与原因；不能只报告“新增了几个 PNG”。Export 设置核验、截图检查和 Unity 运行时验证分别记录。
- 覆盖闭环要从可见绘制分支开始判断：包括 `paint.visible && opacity > 0` 的 Rectangle/Vector/Boolean/Line/Ellipse、Fill 关闭但 Stroke 可见的节点、Effect 以及 Mask+tint 合成。`coveredByAncestor` 只有在祖先自身可见且 `opacity > 0`、其所有可见静态后代都被完整覆盖，且没有动态 Text、控件、运行时插槽、独立替换分支或排除边界时才成立；祖先仅有 Export 设置不够。
- Component Set、Variant member 或普通 Component 根不 PNG，不等于其内部静态像素可以漏掉；若成员根承载完整像素，必须找到可导出的静态子源，或将其列入 `needsReview/unresolved`。View Instance 也不能凭实例名推断已有图片，需查 canonical source 和实际 Export。
- `ForcedSectionRender` 只有在节点确实位于 `Res/图片` Section、插件策略可读且来源 Section 已记录时，才可归入 `specialPolicy`；否则仍是 `missingCoverage` 或 `needsReview`。
- Export 复查分为 safe-candidate pass 与 coverage-closure pass。最终报告增加 `coverageTotals={painted,exported,coveredByAncestor,specialPolicy,intentionallyUnexported,needsReview,missingCoverage}`；`missingCoverage > 0` 或仍有未解释的 `needsReview/unresolved` 时，只能报告“Export 覆盖未闭环”，不能声称全部正确导出。

### 公共组件库复合结构参考（结构用例 13369:1253）

当目标中出现与公共组件库参考页一致的复合控件时，优先复用下列结构骨架。参考来源为
`https://www.figma.com/design/qXI6CRnXkfo8FjGZxWu9Ic/公共组件库?node-id=13369-1253`。
参考页只作为结构和职责依据；不得复制示例文案、尺寸或视觉资源，也不得因为结构相似而
Detach、替换或修改 Common/远程 Instance。

- `Button_*`：根节点直接包含 `Image_bg` 与 `Label_text`。`Image_bg` 是静态背景职责，内部
  可放背景图形并设置唯一的 `PNG / Scale 1 / contentsOnly / 空 suffix`；Button 根和文本不导出。
- `ToggleGroup_*`：根节点直接包含一个 `Label_title` 和两个或以上 `Toggle_*` 子项。每个
  `Toggle_*` 直接包含 `Image_bg`、`Image_checked`、`Label_text`；选项背景和勾选状态保持独立，
  不把多个 Toggle 合并成一张图片。只有真实互斥选项集合才使用 `ToggleGroup_*`。
- `Toggle_*`：直接包含 `Image_bg`、`Image_checked`、`Label_text`。`Image_checked` 是状态视觉
  槽位；即使当前状态不可见，也保留该职责和状态边界，不用截图替代状态结构。
- `Input_*`：直接包含 `Image_bg` 与 `Text Area`。`Text Area` 必须 `clipsContent=true`，其下
  同级保留 `Label_input_holder`（占位文案）和 `Label_text`（输入文案），二者使用相同文本区域
  和对齐基准以支持运行时切换；Input 根、Text Area 和文本不设置 PNG。
- `ProgressBar_*`：直接包含 `Image_bg` 与 `Image_fill` 两个同级职责。背景和填充源保持完整
  轨道尺寸；填充源可以有静态 PNG 视觉子层，但不能把某个运行时百分比裁切结果当成唯一 Sprite，
  也不能将动态数值或文案烘入图片。进度条根不导出。
- `Slider_*`：直接包含 `Image_bg`、`Fill Area`、`Handle Slide Area`。`Fill Area` 下放
  `Image_fill`，`Handle Slide Area` 下放 `Image_handle`。三个行为/区域根不导出，只有完整静态
  视觉子层按 PNG 规则处理；轨道、Fill Area、Handle Slide Area 必须共享同一滑动轴和有效范围。
- `NotExport_*`：表示明确排除的分支。其内容不参与普通 PNG、Prefab 或 GameObject 导出；不为
  排除分支补建 Common 替代物，也不在其内部继续执行普通资源覆盖补勾，除非用户明确要求恢复导出。

复合结构的通用约束：

- 行为根、区域根、文本和布局容器不设置 PNG；静态视觉源设置单一合规 PNG，避免父子重复导出。
- 保留 `Image_bg`、`Image_checked`、`Image_fill`、`Image_handle`、`Text Area`、`Fill Area`、
  `Handle Slide Area` 等职责名称；业务名称只能作为后缀，例如 `Toggle_Category`、`Button_Claim`。
- 以参考层级为准检查直接父子关系、同级顺序、重叠几何、裁剪、动态槽位和运行时区域，不能只
  依据截图或节点名称判定结构正确。
- 参考页中的示例字体为 `FZCuYuan-M03S Regular`；字体不可用时不得替换字体或重排文字，只
  记录字体限制并继续进行不依赖字体的结构检查。
- 复合控件完成整理后，必须回读完整子树并验证：Role 前缀、父子边、视觉/行为分离、PNG 边界、
  动态内容未被烘入、Common 身份未改变、几何和截图保持一致。

### 正确结构范例：九宫格资源、List 与状态组件（112:2966）

> **优先级说明**：本节是参考范例，用于验证结构与职责。当本节内容与本文件
> List and ScrollView rules (project v2 standard) 或
> Nine/Three-slice (九宫/三宫) image section policy 冲突时，以项目 v2 规则为准。
> 特别地：v2 规则要求删除后续重复 Item 并保持 list 根原始总尺寸，而不是用
> NotExport_* 保留演示副本；九宫/三宫处理以项目 v2 的图片 Section 流程为准。

以下范例来自用户提供的正确结构参考：
`https://www.figma.com/design/DcVoTRs1WzJGS5DhscJMGL/Untitled?node-id=112-2966`。
公共组件和复合结构的权威来源为用户提供的 Common 页面：
`https://www.figma.com/design/qXI6CRnXkfo8FjGZxWu9Ic/公共组件库?node-id=47-505`。
两条链接必须分别读取：`112:2966` 只定义目标界面的层级、图片资源分区、九宫格/三宫格、List/Grid、
ScrollView/ProgressBar、动态槽位和视觉还原；`47:505` 只用于解析 Common Instance、Variant、
Button/Toggle/ProgressBar 等复合结构的真实身份。不得用旧 Common 文件、聊天记录或相似名称替代该路径。
它是后续完整界面结构整理的优先参考。复用其职责和父子关系时，使用当前目标的业务名称、
尺寸和真实资源；不得机械复制范例节点 ID、文案、数量或视觉内容，也不得为了匹配范例而修改
远程/Common 主组件。

#### 参考读取与落地顺序（强制）

1. 先读取目标 URL 对应的 Page、Section 和完整后代树，记录 `界面/图片/组件` 的直属关系、
   View 尺寸、资源 Component、Instance→mainComponent/key 链、List/Grid 参数及截图。
2. 再读取 Common URL 的 `47:505` 页面和所需组件身份；只把当前 Common 中确认存在的组件作为外部依赖，
   不复制、Detach 或修改 Common 主组件。若 Common 读取改变了当前文件/Page，必须重新打开目标 URL，
   回读目标指纹后才能写入。
3. 对目标进行结构整理时，先建立或复用 `图片` Section 的静态资源 Component，再把界面中的原始大图替换为
   对应真实 Instance；随后按范例修正 List/Grid、动态槽位和复合控件的直接父子关系。每个写入批次后都要
   回读树和截图，确认几何、可见性、裁剪、文本和视觉没有改变。
4. 最终报告必须分别列出：参考文件读取结果、Common 外部依赖、资源 Component 与消费 Instance、
   九宫格切片约束、List 模板、动态槽位、PNG 边界、未解决的 `needsReview`，不能只报告“已整理”。

#### Page/Section 三分区

- 完整界面模块可按三个 Page-direct 同级 Section 组织：`界面`、`图片`、`组件`。
- `界面` 直属子节点是正式 View；每个正式 View 通常为可见的 `1080×2400` Frame，并保持
  独立导出边界。
- `图片` 直属子节点优先使用真实 `COMPONENT` 作为可复用静态视觉源；界面中必须通过真实
  Instance 消费这些源，并保留 Instance→mainComponent/key 链。
- `组件` 直属子节点放真实 `COMPONENT` 或 `COMPONENT_SET`，用于状态、主题、奖励节点等
  可复用结构；Component Set 与 Variant 根通常不 PNG。
- 不因范例存在三类 Section 就为没有真实资源的页面机械创建空 Section；若页面已有这些区，
  保留其 Page-direct 顺序和空状态并记录审计结果。

#### 九宫格/三宫格图片组件源

- 大图需要拉伸时，先在 `图片` Section 建立真实 `COMPONENT`，再在 `界面` 中放置 Instance；
  不要把大图留在界面里作为普通 Rectangle/Frame，也不要复制一份外观相同的本地 Frame。
- “先切片、后组件化、再引用”是不可省略的顺序：原始大图只作为资源 Component 的视觉源；九宫格/三宫格
  切片必须是该 Component 的直接子节点，界面不能继续保留同源原始大图作为第二份正式视觉源。
- 组件根设置唯一合规 PNG：`PNG / Scale 1 / contentsOnly:true / suffix:""`；切片子层不单独
  导出，避免父子 PNG 重复。
- 九宫格切片使用插件实际的 **0-based** `slice-row-col` 语义
  （`slice-0-0` 至 `slice-2-2`）和 Constraints：
  - 左/中/右列：`MIN / STRETCH / MAX`；上/中/下行：`MIN / STRETCH / MAX`。
  - 四角固定：`MIN|MAX` 组合；边沿单轴 `STRETCH`；中心双轴 `STRETCH`。
  - 横向三段使用中间段水平 `STRETCH`；纵向三段使用中间段垂直 `STRETCH`。
- 资源根负责导出和复用身份，切片只负责拉伸约束；切片不能含动态文字、控件行为或运行时
  数值。创建或检查资源时同时验证源尺寸、Instance 消费尺寸、Constraints 和截图结果。
- 每个资源必须验证“源尺寸→切片边界→消费 Instance 尺寸”的闭环：Instance 可以改变宽高，
  但四角/边沿的固定与拉伸职责不得因缩放丢失；若无法从节点约束证明切片边界，保留原结构并标记
  `needsReview`，不得凭截图猜测九宫格。
- `Res/图片` 中已有独立静态 `FRAME/GROUP` 且具备明确边界、PNG 和可复现视觉时可保留原生类型；
  只有需要真实组件身份、Instance 链或变体时才转换为 Component。

#### 正确 List/Grid 结构

- 列表根使用项目 v2 标准的 `list_*` 命名；仅在纯参考对齐场景中可沿用范例的 `List_*`
  语义。不设置 PNG；若是网格，使用原生
- 参考 `112:2050` 的 List 结构时，根是固定尺寸的 GRID 容器，直接子级中的第一个 `Item` 是唯一正式
  运行时模板；其余可见示例不能自动成为运行时数据项。项目 v2 list 标准范围内，
  后续重复 Item 按 v2 规则删除并保持 list 根总尺寸；不适用 v2 标准的场景
  （如纯参考对齐、非项目界面）可使用 `NotExport_*` 备份语义，保持原有位置、尺寸和可见性证据。
- 正确的 GRID List 至少检查：`clipsContent=true`、固定根尺寸、`layoutWrap`、列/行数量、
  `gridColumnGap`、`gridRowGap`、`itemSpacing`、`constraints`、主/交叉轴对齐和固定 sizing。
- List 背景应是 List 的同级视觉 Instance（参考 `112:2049`），不能并入 Item 或烘成 List 根 PNG；
  Item 内按直接子级分离选中状态 Instance、静态背景、`img_icon → placeholder` 动态槽位和角标 Instance。
- 还原设计样式时，先保持 Item 的实际 234×240（或目标业务等价尺寸）与 Grid 根的实际列/行/间距，
  再调整资源 Instance 的消费尺寸；不得通过复制图片或绝对定位副本掩盖 GRID 参数错误。
- 使用一个真实 `Item` 作为运行时模板时，保留其完整职责边界；后续演示副本不能自动当成正式
  运行时 Item。项目 v2 list 标准范围内后续演示副本按 v2 规则删除；仅在不适用
  v2 标准的场景中才按 `NotExport`/备份规则处理，不改变可见布局证据。
- Item 可以按业务包含：选中状态 Instance、静态背景图片、运行时图片槽位、placeholder 和
  角标 Instance。运行时图像槽位及其 placeholder 不设置普通 PNG；静态背景源可以单独 PNG。
- 列表背景、装饰条或遮罩是独立图片 Instance 时，放在 List 外作为同级层；不要把背景合并进
  Item 或把 List 根烘成一张图片。

#### ScrollView 与 ProgressBar

- 当进度区域具有裁剪视口、可横向滚动或状态节点叠加时，使用 `ScrollView_*`/等价容器包住
  `ProgressBar_*`；视口、轨道、填充和状态节点保持分层。
- `ProgressBar_*` 直接包含同级 `Image_bg` 与 `Image_fill`；根和行为/视口容器不 PNG。
  完整轨道/完整填充源应作为 `图片` Section 的 Component Instance 使用，不能用某个百分比
  截图代替完整 Sprite。
- 动态里程碑、奖励文字和状态节点放在独立 overlay 或 `NotExport` 分支，不得烘入轨道或填充
  图片。只对确定的静态轨道/填充源设置 PNG。
- 进度背景、进度轨道、填充源和状态节点要分别检查运行时尺寸；不能以当前画布上显示的填充
  百分比推断 Sprite 源尺寸。

#### Variant 状态组件

- 真实状态差异使用 `COMPONENT_SET` 和明确的 Variant 属性轴，例如 `Property 1=...`；保留
  观察到的状态值，不补造不存在的笛卡尔组合。
- Variant 根不 PNG；其内部可变的静态装饰或图片源按实际职责单独导出。
- 界面消费状态时使用真实 Instance；不把多个状态复制成普通 Frame，也不为了命名而 Detach
  Common/远程 Instance。

#### Popup、UiMask 与 NotExport

- 弹窗场景可采用 `NotExport` 背景、单独的 `uimask` 和外置 Popup/Tips 内容：遮罩负责覆盖
  场景，弹窗标题、按钮、列表和控件不能因为与遮罩相交而被错误包进 NotExport。
- Popup 中的 Button 仍按 `Button_* → Image_bg + Label_*` 结构；Popup 内 List 仍独立使用
  List/Grid 规则。
- `NotExport` 可以是覆盖场景的可见排除分支，也可以是隐藏备份；两者都不能参与普通 PNG
  覆盖补勾。UiMask 不设置 PNG，且只用于明确的单一遮罩职责。

#### 范例驱动复核清单

- 写入前记录三类 Section、View 尺寸、资源 Component、Instance→source 链、List/Grid 参数、
  ProgressBar/ScrollView 层级、Variant 轴和 Popup 排除边界。
- 写入后重新扫描完整树，确认：图片源位于 `图片`、组件源位于 `组件`、界面只消费 Instance；
  九宫格切片约束正确；List 不被烘平；动态文字/填充未进入 PNG；父子 PNG 不重复；几何、可见性、
  组件身份和截图保持不变。
- 若目标无法证明某个大图的切片边界、List 的运行时模板或资源归属，保留原结构并列入
  `needsReview`，不要凭截图、尺寸或名称猜测。

#### 动态资源槽位命名规则

- 所有运行时替换的图片、图标、鱼图、道具图或其他动态视觉槽位，统一使用以下结构：

  ```text
  img_icon
  └─ placeholder
  ```

- 品质色、品阶色、品质边框、稀有度色带和其他随品质/等级/状态运行时替换的视觉，也属于动态
  资源槽位，必须遵循同一结构：

  ```text
  img_icon
  └─ placeholder
  ```

  不得把品质色作为静态 `Image_*`、普通 `Rectangle` 或已烘焙的固定颜色 PNG 留在 Item 外部，
  也不得把多个品质状态合并成一张不可替换的静态图片。若存在多种品质状态，应保留状态轴、
  Variant 或运行时替换槽位；当前展示的某一种品质只作为 placeholder 预览。

- `img_icon` 是动态资源外层槽位；`placeholder` 是其直接子节点，表示运行时图片的占位/预览
  槽位。不得只使用 `Image_*`、`Container_*`、`Image_FishSlot`、`Image_ItemIconSlot`、
  `Container_FishIcon` 或 `Container_FishArtwork` 代替该结构。
- 正式 List `Item`、Grid Item、卡牌图标、角色头像和主界面动态图标都必须检查这一层级。若
  动态资源没有外层 `img_icon`，补建同尺寸同位置的 Frame 并将原预览节点放入；不得改变绝对
  边界、裁剪、可见性和层级顺序。
- `placeholder` 可以是运行时预览图层或占位容器，但其本身及后代不得设置普通 PNG Export；
  动态预览不可被当作正式静态 Sprite 导出。若预览来自 Common/本地 Instance，保留组件身份，
  不为命名而 Detach。
- 动态槽位的 `img_icon` 和 `placeholder` 不承担 Button、Toggle、List、ProgressBar 或
  ScrollView 行为；行为根应保持独立，动态槽位只作为视觉绑定位置。
- List 的第一个正式 `Item` 模板必须包含 `img_icon → placeholder`；后续演示副本也应保持相同
  子结构，并用 `NotExport_Item_*` 或等价排除语义标记，不能因为排除导出而删除动态槽位。
- 动态槽位预览和静态资源源要分开：`图片/Res` 中的正式静态 Component 可以有 PNG，但界面
  中消费动态资源的 `img_icon → placeholder` Instance 不得继承或新增 PNG。父级已有 PNG 时，
  也要检查是否会把动态预览烘入父级，并将其列入 `needsReview`。
- 品质色/品阶色节点同样不得设置 PNG；若品质色与卡牌背景、角标或边框重叠，必须把可替换的
  品质分支放在对应 `img_icon → placeholder` 槽位下，静态底图与品质动态层分离，不能仅凭当前
  截图把品质颜色归类为静态背景。
- 检查结果必须报告动态槽位数量、缺失 `img_icon` 的节点、缺失 `placeholder` 的节点、槽位内
  PNG 数量和仍保留的预览 Instance 身份；只有这些项目闭环后，才能声称动态资源结构正确。



