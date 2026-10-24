# 图生 PSD 项目实现映射

## 1. 仓库边界

```text
E:\AI_opc\work\产出\PSD实现\PSD智能工作台   # 实现代码
E:\AI_opc\work\产出\PSD实现\电商图\案例N   # 输入与结果
```

修改前必须遵守最近的 `AGENTS.md` 所规定的安全、权限、文件边界和执行环境约束，但案例生产流程只能由本 skill 文档集定义。案例结果不能复制进代码目录或 skill 目录。

每次生产案例 N 前，只保留调用方明确指定的案例 N 当前原图与用户真实参考，并在不打开内容的前提下删除案例 N 全部旧派生产物。禁止备份、迁移或缓存旧产物；禁止查看其他案例。若输入身份不明确则阻断询问，不能通过预览文件来判断。清场后生成全新的 `验收/run-initialization-report.json`。

## 2. 现有代码仅作为实现骨架

仓库中的案例适配代码只提供“可见对象拆分”的实现骨架。处理当前任务时不得打开其旧结果进行视觉分析，也不得把其语义配置当作当前任务的事实：

- `configs/case3-semantic-plan.json`：仅用于兼容旧适配器的配置示例；其中任何对象键、数量、坐标、框、多边形、提示词和排除证据均禁止复制到新任务。
- `scripts/generate_case3_sam2_masks.py`：SAM 2.1 图像编码、多候选提示、评分、完整性门禁、候选接受与 PyMatting 精修。
- `.models/sam2.1_hiera_tiny.pt` 与 `.vendor/sam2`：固定版本的分割模型实现和检查点。
- `scripts/decompose_case3.py`：透明素材生成、OCR/文字组装、AI 背景候选净化与恢复、拆分证明图和 `design.json` 生成。
- `scripts/validate_case_design.py`：Schema、素材、引用、语义元素、原生文字和 Alpha 检查。
- `scripts/export_layered_psd.mjs`：把 `design.json` 导出成带图层组、原生文字和透明像素层的 PSD。
- `scripts/validate_psd_roundtrip.py`：PSD 回读、独立 `psd-tools` 解析、结构检查和像素对比。

当前一键基础命令：

```powershell
npm run rebuild:case3:ai
```

执行顺序：

```text
SAM2 可见对象分割
→ 语义与完整性接受
→ Alpha 精修
→ 背景重建
→ design.json
→ PSD 导出
→ PSD 回读验收
```

不能把案例3当前结果说成已经完成“遮挡对象完整补全”。完整 skill 必须在设计组装前增加通用语义清单、SAM 前冻结的遮挡有向无环图、独立完整透明补全素材及补全验收。

## 3. 当前项目实现接口（非流程权威）

以下文件仅是执行本 skill 的代码/接口，不得定义、补充或覆盖生产流程：

- `packages/design-schema/design.schema.json`
- `requirements.txt`
- `requirements-ai.txt`
- `package.json`

当前 Schema 设置了 `additionalProperties: false`。在正式迁移 Schema 前，新的遮挡与补全信息放进标准旁路 JSON；在 `design.json` 中使用图层组和“提取/生成”子层表达完整对象。

## 4. 扩展新案例

处理 `案例N` 时：

1. 在工作台相邻位置创建案例目录，并保留原始源图。
2. 在任何视觉读取前，只保留调用方明确指定的当前源图和用户真实参考，在不打开内容的前提下删除当前案例全部旧产物；不得归档、迁移、缓存或查看旧产物，也不得查看其他案例。生成并通过 `验收/run-initialization-report.json` 后，再生成全新的 `验收/input-evidence-manifest.json`。
3. 基于当前源图全新生成 `object-integrity-plan.json`。先确定完整实例及其嵌套组成部件，再为每个实际位置独立判断 `wearState`、`displayState`、`viewpoint`、`poseDeformation` 与 `reusePolicy`；不能复制其他案例的对象数量、名称、结构、坐标或提示。
4. 首轮多模态分析同时产出完整实例、组成部件、穿着/展示状态、全部普通文字、艺术字、文字型 Logo、低对比/半透明品牌文字和真实非文字符号，以及不同完整对象之间的 `occlusionEdges`、完整 `backToFront` 和三个目标集合；全部写入 `object-integrity-plan.json` 并通过后才能启动 SAM。不得先分割后补做对象整体性判断或重算层级。
5. 根据完整实例计划生成 `semantic-inventory.json` 和 `验收/instance-independence-report.json`，确认不同实例不会共享素材 URI、哈希、Alpha、蒙版、隐藏证据范围、完整补全素材、生成任务或候选结果。
6. 校验首轮计划中已经冻结的 `samVisibleTargets = instances`、`completionTargets = unique(occludes.behind)`、`directExtractionTargets = instances - completionTargets` 和 `backToFront`。`occlusion-graph.json` 只复制这些结论作为派生快照。每条 `front` 必须引用首轮已登记并验收的实例/图形/透明文字键。对象内部排版、印花和 `contains`/`attachedTo` 不触发补全；疑似覆盖层身份未决时写入 `unresolvedOcclusions` 并停止。
7. 首次 SAM 让执行器遍历全部 `samVisibleTargets`。内部组件提示只能参与当前实例候选的组合与召回检查，不得写成拆分素材、图层或展示预览；每个实例先执行一次原收紧逻辑，只有检测到缺失才以同一语义契约执行一次“主体核心 + 完整外沿”补齐。补齐只多出经验证的底层背景时允许通过；带入其他对象或仍有缺失时阻断，不得在同一语义版本中第三次 SAM。若 SAM 证据暴露遗漏对象、新遮挡或归属/层级错误，返回首轮分析生成新版本，并通过 `sam-revision-plan.json` 计算受影响闭包；只复用同一运行中具备完整哈希证明并针对新计划复验通过的未受影响对象，无法证明则重跑全部对象。受影响对象仍读取原始全分辨率源图，不能改用局部裁图。无遮挡对象标记 `complete_from_source`，有遮挡对象标记 `visible_only`。
8. 对所有可识别字符串使用 OCR 与原生文字分支；每个字符串只交付一个原生文字层，源外观仅可作为同层内部缓存，禁止建立独立文字图形层。艺术字、文字型 Logo、透明品牌文字和注册标记不得因效果复杂而降级为图形。只有不构成字符串的真实符号 Logo、非文字服装印花或实体图形标签才进入图形/Alpha 专用分支；不要把文字当作通用对象 SAM 目标。
9. 验收全部对象的源图可见像素、原生文字层与真实非文字图形层后冻结覆盖蒙版，按遮挡图从前到后处理更深对象。对未归属像素生成 `验收/residual-ownership-report.json`；禁止按距离强制分配给最近对象或背景。
10. 对 `completionTargets` 直接执行完整对象 Image 分支：
   - 目标为空时禁止调用图像编辑模型，并生成零调用空列表通过报告；
   - 每个目标以独立 `instanceKey + generationTaskId` 调用配置的 Image 模型，生成符合该穿着/展示状态、视角和姿态的完整对象；
   - 任务按去重后的目标实例创建，而不是按遮挡边创建；多个透明文字共同覆盖同一上衣时先合并前景 Alpha，同一实例只允许一个活动任务和一个最终接受结果；
   - 复用该目标在步骤 6 已冻结的唯一 `visible_only` SAM/Matting 结果作为 Image 提示与候选验收证据；该结果不得进入 PSD；
   - 候选通过状态与组成检查后，只执行一次透明/键色背景转 Alpha 与坐标对齐，此后不得再次运行第二次 SAM 或语义抠图；
   - 隐藏证据只用于 Image 提示和 QA，不得裁剪候选；
   - 保存 `generatedCompleteAsset = generatedCandidate ∩ candidateAlpha`，并验证其 Alpha 与完整候选 Alpha 一致，RGB 除键色边缘去色外与接受候选一致；
   - 每个完成实例在 PSD 中只写入这一张 `generatedCompleteAsset`，禁止同时写入源图可见层或补丁层；
   - 生成 `验收/amodal-completion-report.json`。
11. 明确全部前景后重建完整背景，并逐个移开对象检查残影和污染。
12. 为每个字符串、文字型 Logo、透明品牌文字和注册标记独立建立唯一原生文字层；纯符号 Logo 独立建立图形层。不得输出整片排版大图块，不得输出任何可识别字符串的图形层。
13. 严格读取首轮 `object-integrity-plan.json#backToFront` 生成 design；`occlusion-graph.json` 只用于一致性校验，不能依赖人工猜测或 SAM 后生成的平面列表。
14. 为整个案例增加一个可重复执行的一键命令。各阶段只执行契约允许的修复；SAM 缺失补齐严格限一次，仍失败时立即停止，不能发布只完成一部分的 PSD。

通用执行器完成后的推荐命令形式：

```powershell
.venv-ai\Scripts\python.exe scripts/reconstruct_image_to_psd.py --case-dir ..\电商图\案例N
```

在通用执行器尚未实现时，可以使用案例适配脚本编排同样步骤，但必须生成 `contracts.md` 规定的标准文件。

## 5. 模型职责

- 视觉语言/大语言推理：语义清单、OCR 逐字复核、关系与遮挡推断、隐藏结构规划、候选检查和失败位置分析。
- SAM2：为全部 `samVisibleTargets` 提取源图实际可见像素并生成多组候选；无遮挡对象得到完整源图 Alpha，有遮挡对象只得到 `visible_only` Alpha。
- Alpha Matting：恢复原图支持的柔边。
- Image 2.0 或配置的图像编辑模型：为 `completionTargets` 独立生成完整对象候选，并生成缺失背景像素。
- 确定性代码：实例复用审计、哈希、单次 Alpha 转换/规范化、候选对齐、可见/隐藏支持计算、可见像素零差异、未归属像素、附着部件、图有效性、素材紧边、Schema、PSD 写入、回读解析和像素对比。

不能要求语言模型直接生成像素，也不能让图像模型修改受保护的原图可见商品/对象像素。
图像模型必须返回完整候选；候选 Alpha 与 RGB 是被遮挡对象最终交付像素的唯一依据。此前唯一一次 `visible_only` SAM/Matting Alpha 只作为提示与验收证据。确定性代码必须把当前实例完整候选转换为一张连续透明 `generatedCompleteAsset`，不得按隐藏支撑、遮挡者或源图可见残片再次裁剪、覆盖或拼贴；不得再保存源图可见 PSD 子层。候选通过后禁止再次运行第二次 SAM。商品类别、材质和结构知识可以共享，实际像素、几何、蒙版和生成结果不能跨实例共享。

## 6. 使用当前 Schema 生成完整对象层

被遮挡对象使用一个语义父组：

```text
对象组_完整短裤
└─ 对象_短裤_Image完整透明对象  (source.type = generated，唯一像素层，完整 candidateAlpha 与候选 RGB)
```

父组在全局 PSD 中必须位于遮挡者下方；组内不得再出现第二张源图残片层。

真实前后交错场景可以在一个逻辑父对象内建立最少的前/后渲染子层，并在遮挡报告中解释原因。

## 7. 文字行为

提取、design 和 PSD 导出必须逐字保留文字。

可编辑文案映射为唯一原生文字层；不得另外建立包含同一字形的栅格、图片或图形层。源外观只能作为该原生文字层的内部显示缓存。无法作为字符串编辑的真实 Logo 图形、服装印花和包装艺术内容默认保留为栅格图形；只有任务提供可靠可编辑源文件并明确要求时才重建。

每个实际放置位置必须是独立节点。半透明覆盖式品牌标签和注册标记不能因为低对比而遗漏；标题、副标题、中文、英文、Logo 和效果不能合成一个不可编辑的大块。

## 8. 背景行为

继续采用案例3的方式，分别保存原始生成候选、净化候选、恢复蒙版、最终重建背景和报告。改进点是：必须根据最终前景并集和遮挡计划计算已知背景保护区与缺失区。

生成只能发生在恢复区域内。移动对象后不得留下残影；本来就应独立移动的装饰必须保留为前景层。

## 9. 导出与验收命令

现有构建模块：

```powershell
python scripts/validate_case_design.py --case-dir <案例目录>
node scripts/export_layered_psd.mjs <案例目录>
python scripts/validate_psd_roundtrip.py --case-dir <案例目录>
```

运行这些命令前，必须先通过语义、蒙版、完整对象补全和背景门禁。PSD 校验器只能证明文件结构及 design/PSD 一致性，不能证明原始拆分在语义上完整。

通用执行器必须在 PSD 导出前至少提供以下阻断验证：

- `validate_instance_independence`：检测跨实例素材、蒙版、哈希和生成任务复用及状态错配；
- `validate_completion_eligibility`：确认补全目标只来自 `occludes.behind`，无对象间遮挡时图像编辑调用数为零；
- `validate_observed_pixel_lock`：检查受保护可见像素零差异；
- `validate_residual_ownership`：拒绝强制残余归属；
- `validate_attached_components`：核对附着部件与透明品牌内容；
- `validate_sam_revision_scope`：语义修订后验证受影响对象传递闭包、同运行复用证明、全图编码缓存键、删除结果作废和全量回退条件；
- `validate_downstream_invalidation`：验证变化 SAM/遮挡支持导致的 Image、背景、design、PSD 与全局 QA 均已正确失效并重建；
- `validate_moved_states`：逐对象移开检查补全和背景。

这些检查失败时不得调用导出器。修复后只可从失败阶段继续，且必须校验冻结的上游输入哈希，避免使用陈旧产物。

## 10. 最终发布与临时文件清理

以 `contracts.md` 为标准。规范化参考图、语义清单、遮挡图、可见/补全素材及蒙版、背景、拆分总览、`design.json`、图层清单、设计参考渲染图和全部 QA 报告只存在于本轮临时工作区，用完必须删除。

PSD 创建与阻断式验收完成后还必须执行：

1. 原子发布 `交付/final-layered.psd`；
2. 解析最终 PSD 外部引用，只复制确实必需的字体、链接素材、ICC/ICM、LUT 等依赖到 `交付/dependencies/`；
3. 删除整个本轮临时工作区；
4. 运行最终目录白名单检查，确认除 PSD 和真实依赖外无任何文件；
5. 失败或异常退出同样清理本轮中间结果和半成品。

只有最终白名单门禁通过后，本 skill 的生产任务才结束。
