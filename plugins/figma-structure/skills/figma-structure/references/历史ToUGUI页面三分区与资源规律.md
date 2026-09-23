# 历史 ToUGUI 页面三分区与资源规律（只读回顾）

## 证据范围

本回顾使用 2026-09-17 的目录与界面审计产物，以及对历史页面直属节点的只读元数据复核。批量界面审计共记录 79 个 ToUGUI Page、145 个界面根和 27,141 个节点，但其原始扫描主要枚举“界面”Section；不能仅凭该快照断言其他 Section 缺失。随后对历史页面元数据抽样复核，确认多种页面存在同级“图片”和“样式状态-美术组件”Section。

已复核的页面组合包括：

| 页面 | 直属 Section 组合 |
| --- | --- |
| FirstCharge | 界面、图片、样式状态-美术组件 |
| LoginReward | 图片、样式状态-美术组件、界面、组件 |
| RechargeView（两份） | 界面、图片 |
| NPCDesign | 图片、样式状态-美术组件、界面、组件 |
| NPCAIChat | 界面、图片、组件 |
| IslandCircle | 样式状态-美术组件、界面、图片 |
| FishingPulling | 界面、图片、组件 |
| CollectFish / InteractResult | 界面、图片 |
| Login | 界面、色值、动效、组件、图片 |
| Launch / Loading / SceneTransform / Guide / FreeRename | 界面、图片（部分另有动效） |
| Dialogue / FurnitureBag | 界面、图片、样式状态-美术组件 |
| Main / Craft / Bag | 界面、图片、组件（部分另有动效） |
| RewardFull | 界面、图片、样式状态-美术组件 |

典型 FirstCharge 页面 6WpliqFpmtiN3WB5pYSqbA / 19:24 的直属节点证据：

- 19:64 界面：FirstChargeView、FashionPreView，均 1080×2400。
- 19:549 图片：3 个独立 COMPONENT 图片源，根有 PNG 1x / contentsOnly / 空 suffix，子层不作为独立资源。
- 61:127 样式状态-美术组件：2 个 COMPONENT_SET，每个 2 个状态成员，Set 根无 Export。

历史界面树还显示：ServerList、Certificate、GachaShop、Achievement、NPCMemory、DriftBottle、IslandBuild 等页面反复使用可复用图片 Component、九宫格 slice、主题/状态 Instance；静态装饰与背景常由多个独立层组合，而不是一张扁平大图。

## 归纳规则

1. 完整 ToUGUI 模块整理先扫描 Page 的直属 children，分类 界面/Views、图片/Res、样式状态-美术组件/Coms_Art、组件/Coms 和其他非产物 Section。三类是完整 Page 的检查项；实际产物区必须保持同级、Page-direct，不得在总 Frame 或总 Section 下嵌套它们。
2. 只有完整 Page 标准化、用户明确要求补齐模板，或资源反向索引提供真实证据时，才准备 Views_<Module>、Res_<Module>Images、Coms_Art_<Module>；没有真实资源且用户未要求模板时不创建空 Section。已有空区可以保留并标记 empty/needsReview，不能伪造图片或变体；只读审计、单节点任务或用户限制不扩 Page 时不创建。
3. 先从全部 View 的真实 Instance 反向收集 mainComponent、component key、variant properties、使用尺寸和出现次数，再去重建立 Res/Coms canonical 清单。历史树有约 3,024 条 source 记录、约 1,320 个 source identity；同一 source key 只保留一个 canonical 定义，实例通过真实链路消费。
4. Res 只收独立、可复用、无行为的视觉源：COMPONENT 或图片 COMPONENT_SET，包括静态背景、边框、标题条、装饰和九宫格/三宫格 Sprite。图片源根可设置 PNG 1x / contentsOnly / 空 suffix；slice 子层通常不导出。含动态 Text、Input、List、Toggle、Mask 行为根或进度逻辑的结构留在 View/Coms，不能把动态控件烘成图片。
5. Coms_Art 只收具有主题、状态、等级或复用语义的 COMPONENT / COMPONENT_SET。保留真实轴和值，如 Theme/Category、State、Property 1/Property 2；只建立已观察到的组合，不做无证据的笛卡尔补全。Component Set 根与 variant 根通常不 PNG；静态成员内部的独立图片按 Res/PNG 规则审计。
6. View 内保留真实 INSTANCE，并验证其 mainComponent / key / variant metadata 指向 Res 或 Coms canonical。不要只把一张视觉相同的 Frame 摆入图片区，也不要复制 Common/远程组件后 Detach。若要把重复的本地视觉 Frame 组件化，先保留原 Frame 的层级、几何、约束、填充、可见性和效果，必要时放入隐藏 NotExport_* 备份，再用 Instance 回连。
7. 复合美术要保持分层：背景九宫格、底色、阴影、前景装饰、标题条、遮罩可以是独立层并分别约束/导出；不能为了“图片 Section”把会独立变化或需要自适应的装饰合成一张 PNG。消费尺寸和 Constraints 必须从源与实例同时复核。
8. 同一 Page 还可以有 组件、色值、动效 等其他直属 Section；不要把它们误并入 Res 或 Coms_Art。页面顺序不表示依赖关系。
9. 完成后逐 Page 输出：三类 Section 是否存在及直属性、各区 direct-child 类型、Res/Coms 是否为空、Instance→source 反向索引、变体轴/组合完整性、根/子层 PNG、中文/泛化命名、View 1080×2400 与自适应约束。对历史“只扫描界面”的快照，标记为 coverage-limited，不把未扫描当作不存在。

## 证据文件

- outputs/togui-interface-structure-summary-20260917.json
- outputs/togui-interface-structure-audit-20260917.md
- outputs/togui-interface-full-trees-20260917.md
- outputs/togui-layout-learning-20260917.md
- outputs/togui-variant-learning-20260918.md



### 静态 Frame 资源兼容例外

历史页面的图片区并非所有资源都是 Component。若静态 Frame/Group 已有明确独立边界、PNG 1x/contentsOnly 设置且被界面稳定消费，可以保留其原生类型；不要为统一类型强制转换或 Detach。只有需要 Instance→source 身份、组件变体或跨页面复用时，才建立真实 Component/Component Set。



## 全量 Section 复核结果（2026-09-18）

对历史摘要中的 79 个 ToUGUI Page 做了直属 Section 元数据复核，79/79 成功：

- 54 个页面含 图片/Res；
- 22 个页面含 样式状态-美术组件 或 美术组件；
- 18 个页面含普通 组件；
- 常见组合为 图片+界面（19）、图片+美术状态+界面（13）、图片+界面+组件（7）、仅界面（7）；17 个页面没有直属 Section，属于辅助或未组织页面。
- 真实页面还出现 时装 等业务美术 Section 和 Section 1 等待命名 Section。它们必须先分类/报告，不能按名称直接归入正式导出区。
- LoginReward 证明空的样式状态-美术组件和组件 Section 可以作为预留存在；RechargeView 证明图片 Section 也可能为空。空区不应伪造内容。

结论：三类 Section 是完整 Page 整理时的检查模板；资源区的创建和填充仍由真实消费、source ancestry、状态轴和资源边界决定。

## 最终判定优先级（追加澄清）

本节与 SKILL.md 的“历史三分区规则的优先级澄清”覆盖前文固定三分区、仅 Component 或同 Page 限制等表述。即使是完整 Page 标准化，没有真实资源且用户未明确要求模板时，也不新建空区；已有空区可保留。稳定静态 FRAME/GROUP 不强制组件化。资源归属通过 mainComponent 的祖先链确认，可来自已确认同模块的其他 Page 的 Res、Coms 或 Coms_Art，跨 Page 记录模块与来源；Common/remote 单列，不复制或 Detach，结构归属不等于已经验证 Unity Prefab 可解析。

历史美术区属于内容分类线索，不自动等于正式 Coms 导出语义。仅英文命名可用 ArtComponents_<Module> 保留非导出用途；只有明确需要组件 Prefab 且导出标准化在授权范围内，才采用 Coms_Art_<Module>。保留普通 Coms 与美术创作区的区别。

