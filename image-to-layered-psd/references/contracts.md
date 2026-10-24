# 案例、数据与 PSD 契约

## 目录

1. 目录规范
2. 语义清单与实例身份
3. 遮挡图
4. 分割报告
5. 完整对象补全报告
6. 背景报告
7. design.json 映射
8. 文字一致性
9. PSD 结构与命名
10. 来源与不确定性

## 1. 目录规范

代码、运行时中间结果和最终交付必须分开：

```text
PSD实现/
├─ PSD智能工作台/                  # 代码、模型、脚本、Schema、skill
│  └─ .agent/skills/image-to-layered-psd/
└─ 电商图/案例N/
   ├─ <原始图片>                   # 用户输入，不属于最终交付包，不得删除
   ├─ .runtime/<productionRunId>/  # 本轮临时工作区，验收或失败退出后整体删除
   │  ├─ source-reference.png
   │  ├─ object-integrity-plan.json
   │  ├─ semantic-inventory.json
   │  ├─ occlusion-graph.json
   │  ├─ background-reconstructed.png
   │  ├─ design.json
   │  ├─ layer-manifest.json
   │  ├─ canvas-reference.png
   │  ├─ final-layered.psd         # 验收前临时 PSD
   │  ├─ 拆分总览.png
   │  ├─ 拆分图/
   │  │  ├─ direct-extraction/     # 无遮挡实例的 SAM 原图完整透明层
   │  │  ├─ completion/            # 被遮挡实例的 Image 完整透明对象层
   │  │  ├─ masks-visible/
   │  │  └─ masks-completion/
   │  ├─ 中间证据/
   │  │  └─ sam-visible/           # completionTargets 的 SAM 可见证据，不进入 PSD
   │  └─ 验收/
   │     ├─ input-evidence-manifest.json
   │     ├─ run-initialization-report.json
   │     ├─ input-preflight.json
   │     ├─ instance-independence-report.json
   │     ├─ residual-ownership-report.json
   │     ├─ sam-revision-plan.json        # 仅语义计划在本轮 SAM 后修订时生成
   │     ├─ segmentation-report.json
   │     ├─ amodal-completion-report.json
   │     ├─ background-generation-report.json
   │     ├─ design-validation.json
   │     ├─ psd-capability-plan.json
   │     ├─ psd-export-report.json
   │     ├─ psd-roundtrip.json
   │     └─ psd-qa-report.json
   └─ 交付/                        # 最终交付白名单目录
      ├─ final-layered.psd
      └─ dependencies/             # 仅在确有外部依赖时存在
         ├─ fonts/*.{ttf,otf,ttc}
         ├─ linked-assets/*
         └─ color-profiles/*.{icc,icm}
```

`.runtime/<productionRunId>/` 中的文件是验收所需临时契约，不是最终交付物。成功验收、失败、阻断或异常退出后必须整体删除，不能保留兼容别名、历史报告或旧 PSD。

最终 `交付/` 目录只允许一份 `final-layered.psd` 和该 PSD 实际引用的外部依赖。没有依赖时只允许 PSD；生产工具依赖、源码、模型、检查点、Schema、报告、预览、拆分素材、`design.json`、日志和输入副本均禁止进入 `交付/`。

### 1.1 输入证据清单

在创建证据清单前必须先完成当前案例清场。只保留调用方明确指定的当前原图和用户真实参考；当前案例旧产物必须在未打开、未解析、未预览、未哈希、未备份的情况下删除。其他案例不得查看，也不得越权删除。

`run-initialization-report.json` 至少满足：

```json
{
  "schemaVersion":"1.0.0",
  "productionRunId":"<本次全新运行ID>",
  "preservedInputs":["<当前原图路径>"],
  "removedOldArtifactPaths":["<仅记录路径，不读取内容>"],
  "oldArtifactsOpened":false,
  "oldArtifactsRetained":false,
  "oldArtifactsBackedUpOrMoved":false,
  "otherCasesInspected":false,
  "processAuthority":[
    "SKILL.md",
    "references/pipeline.md",
    "references/contracts.md",
    "references/qa-gates.md",
    "references/project-implementation.md"
  ],
  "passed":true
}
```

除 `preservedInputs` 外，清场前存在于当前案例目录的内容都属于旧产物；如果无法确认某文件是否为明确输入，必须阻断并询问，不能打开它来判断。旧产物不得以改名、归档、临时文件、工作台缓存或跨案例复制的形式保留。

`input-evidence-manifest.json` 在任何视觉分析前生成：

```json
{
  "schemaVersion":"1.0.0",
  "currentSourceOnly":true,
  "allowedEvidence":[
    {"role":"current_source","path":"<当前原始图片>","sha256":"<sha256>"}
  ],
  "forbiddenEvidencePatterns":[
    "../案例*/**",
    "**/拆分总览*",
    "**/拆分图/**",
    "**/验收/**",
    "**/*.psd"
  ],
  "passed":true
}
```

`forbiddenEvidencePatterns` 表达默认禁用类别；路径同时出现在 `allowedEvidence` 时，仅当用户明确把它标记为当前任务真实参考才允许。代码、Schema、模型和命令不属于视觉证据。每次多模态、OCR、提示生成、SAM 和 Image 报告必须记录实际读取的证据哈希；存在清单外视觉哈希时 `passed` 必须为 `false`。

## 2. 语义清单

分割前必须先生成 `object-integrity-plan.json`：

```json
{
  "schemaVersion":"1.0.0",
  "sourceSha256":"<sha256>",
  "semanticPlanOrigin":"current_source_fresh_analysis",
  "semanticPlanVersion":1,
  "previousSemanticPlanHash":null,
  "copiedFromPriorCase":false,
  "evidenceSha256":["<sha256>"],
  "analysisStage":"pre_sam_multimodal",
  "backToFront":["worn_upper_outfit","worn_overall","overlay_brand_01"],
  "samVisibleTargets":["worn_upper_outfit","worn_overall"],
  "completionTargets":["worn_upper_outfit","worn_overall"],
  "directExtractionTargets":[],
  "instances":[
    {
      "key":"worn_upper_outfit",
      "name":"穿着状态的衬衫与领带整体",
      "state":{"wearState":"worn","displayState":"on_person","viewpoint":"front"},
      "components":["blouse","collar","ruffled_sleeves","plaid_tie","physical_label"],
      "outputRole":"complete_instance",
      "reusePolicy":"forbidden"
    },
    {
      "key":"worn_overall",
      "name":"穿着状态的牛仔背带裤整体",
      "state":{"wearState":"worn","displayState":"on_person","viewpoint":"front"},
      "components":["straps","bib","shorts","buttons"],
      "mutuallyExclusiveWith":["toy_cord","plush_toy"],
      "outputRole":"complete_instance",
      "reusePolicy":"forbidden"
    }
  ],
  "graphics":[],
  "unresolvedOcclusions":[],
  "brandCoverageCheck":{
    "wholeCanvasReviewed":true,
    "multiScaleTilesReviewed":true,
    "suspectedPlacementCount":1,
    "registeredPlacementCount":1,
    "unregisteredResiduals":[],
    "passed":true
  },
  "textGraphics":[
    {
      "key":"overlay_brand_01",
      "kind":"transparent_brand_overlay",
      "content":"活力宣言®",
      "secondaryContent":"HUOLIXUANYAN",
      "quad":[[0,0],[100,0],[100,50],[0,50]],
      "semiTransparent":true,
      "surface":"garment",
      "occludes":"worn_overall",
      "independentPlacement":true,
      "preserveSourceAppearance":true
    }
  ],
  "occlusionEdges":[
    {"front":"worn_overall","behind":"worn_upper_outfit","evidence":["轮廓终止与T形交界"]},
    {"front":"overlay_brand_01","behind":"worn_overall","evidence":["半透明品牌文字覆盖牛仔布"]}
  ],
  "passed":true
}
```

`instances[*].outputRole` 必须为 `complete_instance`。组成部件只能嵌套在所属实例中，不得在顶层重复登记为输出对象。`analysisStage` 必须为 `pre_sam_multimodal`；`backToFront`、`occlusionEdges` 和三个目标集合必须在任何对象级 SAM 前冻结。若本轮 SAM 证据要求修订语义计划，必须先生成新版本并再次冻结，再通过 `sam-revision-plan.json` 决定受影响闭包；禁止直接改写活动版本。`backToFront` 必须按全局从后到前顺序包含每个已登记的 `instances|graphics|textGraphics` 键且仅一次。最终合并后的 SAM 分割报告 `targetKeys` 必须与当前版本 `samVisibleTargets` 及全部 `instances[*].key` 完全一致；Image 补全报告的对象键必须与当前 `completionTargets` 完全一致。有遮挡实例同时拥有一份 `visible_only` SAM 结果和一份 Image 完整对象结果，这两个结果用途不同，不得互相替代。
首次计划 `semanticPlanVersion` 为 `1` 且 `previousSemanticPlanHash` 为 `null`；同一运行内每次合法修订必须递增版本，并把上一版本完整计划哈希写入 `previousSemanticPlanHash`。版本链断裂、回退覆盖旧版本或缺少上一版本哈希时不得执行增量复用。
`textGraphics` 必须由首轮多模态分析产生，并同时覆盖普通文字、透明文字内容与真实非文字符号；该兼容字段名不授权输出文字图形层。凡能读取为字符串者必须标记为原生文字交付，透明品牌文字的 `preserveSourceAppearance` 表示在同一原生文字层上尽量复现透明度、混合与效果，不能据此创建栅格字形层。只有不构成字符串的真实符号 Logo 才可标记为图形交付。

`brandCoverageCheck` 是阻断字段。`wholeCanvasReviewed`、`multiScaleTilesReviewed` 和 `passed` 必须为 `true`，`suspectedPlacementCount` 必须等于 `registeredPlacementCount`，`unregisteredResiduals` 必须为空。否则 `object-integrity-plan.passed` 必须为 `false`。

若发现疑似覆盖痕迹但无法解析为已登记前景实例，首轮计划必须包含非空 `unresolvedOcclusions` 并令 `passed: false`。不得为修复目标预造一个没有对应识别实例的遮挡边。

Image 候选注册表必须使用显式绑定：

```json
{
  "instanceKey":"worn_overall",
  "generationTaskId":"case5-worn-overall-01",
  "inputSourceSha256":"<source-sha256>",
  "allowedReferenceKinds":["original_source"],
  "result":"imagegen/worn-overall-01.png",
  "stateCheck":{"wearState":"worn","displayState":"on_person","passed":true},
  "accepted":true
}
```

不得用文件名排序、时间戳或人工观察缩略图替代以上绑定。派生 QA 图的哈希不得出现在 `allowedReferenceKinds` 或任何图像模型输入记录中。

`semantic-inventory.json` 必须在分割前生成：

```json
{
  "schemaVersion": "1.0.0",
  "sourceSha256": "<sha256>",
  "canvas": {"width": 941, "height": 1672},
  "objects": [
    {
      "key": "left_leg",
      "instanceId": "person_a_left_leg",
      "productFamily": "person_a_body",
      "name": "左腿完整对象",
      "class": "body_part",
      "editableRole": "semantic_object",
      "wearState": "not_applicable",
      "displayState": "on_person",
      "viewpoint": "front",
      "poseDeformation": true,
      "reusePolicy": "forbidden",
      "observedPixelHash": "<visible-pixel-sha256>",
      "hiddenEvidenceMask": "验收/hidden-evidence/person_a_left_leg.png",
      "coarseRegion": [120, 610, 400, 1600],
      "visibleParts": ["大腿", "小腿", "袜子", "鞋"],
      "likelyHiddenParts": ["被短裤遮住的上段"],
      "attachedComponents": [],
      "candidateOccluders": ["shorts"],
      "confidence": 0.88,
      "needsReview": []
    }
  ],
  "texts": [
    {
      "key": "headline",
      "content": "Dance Time!",
      "kind": "editable_text",
      "instanceId": "headline_placement_01",
      "independentPlacement": true,
      "language": "en",
      "lines": ["Dance Time!"],
      "quad": [[681,735],[838,735],[838,780],[681,780]],
      "confidence": 0.98,
      "preserveVerbatim": true
    }
  ]
}
```

对象键在同一案例内保持稳定。清单描述当前图片证据，不能演变为永久的具体形状特例库。

### 2.1 实例独立契约

每个实际出现位置都必须有唯一 `instanceId`。`productFamily` 只能表达“可能是同款/同类”，不能授权复用素材。以下差异任一存在时，两个实例必须分别生成和验收：

- 穿着与未穿着；
- 平铺、悬挂、人体、模特、包装或其他展示状态；
- 正面、背面、侧面、细节或不同透视；
- 姿态、褶皱、形变、光照、遮挡关系或可见范围。

`reusePolicy` 默认为 `forbidden`。不同实例不得共享可见素材 URI、素材哈希、Alpha 哈希、隐藏证据蒙版、完整补全素材、生成任务 ID 或候选 ID。只允许共享颜色、材质、款式和结构描述作为提示参考。

仅当源图证明确为像素级重复元素时，才允许 `reusePolicy: "proven_identical"`，并在 `验收/instance-independence-report.json` 中记录源区域、像素哈希、允许变换、差异图、审批状态和理由。无法证明时必须独立处理。

文字、艺术字、Logo、覆盖式透明品牌标签、注册标记、服装印花和实体标签都要登记 `kind` 与独立放置实例。相同内容出现在不同位置仍使用不同 `instanceId`。

实例独立性报告至少包含：

```json
{
  "schemaVersion":"1.0.0",
  "passed":true,
  "instances":[
    {
      "instanceId":"worn_overall_01",
      "productFamily":"denim_overall",
      "wearState":"worn",
      "displayState":"on_person",
      "viewpoint":"front",
      "reusePolicy":"forbidden",
      "sourceEvidencePixelHash":"<sha256>",
      "alphaHash":"<sha256>",
      "completionTaskId":"<unique-task-id>"
    }
  ],
  "pairChecks":[
    {
      "left":"worn_overall_01",
      "right":"flat_overall_01",
      "stateMismatch":true,
      "sharedAsset":false,
      "sharedMask":false,
      "sharedGeneration":false,
      "passed":true
    }
  ]
}
```

未归属像素报告至少记录未归属像素数、连通域、候选归属、证据和最终状态。`unassignedPixelCount > 0` 且无法取得可靠归属时，`passed` 必须为 `false`；不得用“最近对象”或“背景”作为无证据默认值。

## 3. 遮挡图

`occlusion-graph.json` 是 `object-integrity-plan.json` 中首轮层级结论的派生快照，明确描述前后深度：

```json
{
  "schemaVersion": "1.0.0",
  "reconstructionMode":"semantic_complete_layer",
  "nodes": [
    {"key":"background","kind":"background"},
    {"key":"left_leg","kind":"semantic_object"},
    {"key":"shorts","kind":"semantic_object"}
  ],
  "edges": [
    {
      "front":"shorts",
      "behind":"left_leg",
      "relation":"occludes",
      "confidence":0.95,
      "evidence":["T形交界", "腿部轮廓进入裤腰边缘"]
    }
  ],
  "samVisibleTargets":["left_leg","shorts"],
  "directExtractionTargets":["shorts"],
  "completionTargets":["left_leg"],
  "imageEditRequired":true,
  "backToFront":["background","left_leg","shorts"],
  "orderSource":"object-integrity-plan.json#backToFront",
  "acyclic":true,
  "ambiguousEdges":[]
}
```

每条边表示 `front` 在 PSD 中必须位于 `behind` 上方。除允许在派生图中添加底层 `background` 外，`occlusion-graph.json.backToFront` 必须与首轮 `object-integrity-plan.json.backToFront` 完全一致。拓扑顺序非法或二者不一致时必须阻断；不得在 SAM 后改写顺序。

`completionTargets` 只能由已通过的 `occludes` 边计算，值为所有 `behind` 节点的去重集合。`samVisibleTargets = instances`，`directExtractionTargets = instances - completionTargets`。SAM 可见像素提取覆盖全部实例；Image 只追加到 `completionTargets`。目标集合必须在任何对象级 SAM 或 Image 调用前冻结。`contains`、`attachedTo`、对象内部图案、文字排版、印花和边界接触不得进入 `completionTargets`。

每条边的 `front` 必须引用 `object-integrity-plan.instances|graphics|textGraphics` 中存在且已验收的键；`behind` 必须引用 `instances` 中的完整实例键。`unresolvedOcclusions` 非空、引用悬空或前景未验收时，遮挡图 `passed` 必须为 `false`，不得生成 `completionTargets`。补全报告中的对象键必须去重，且每个键只有一个 `accepted: true` 的任务结果。

没有对象间遮挡时，契约为：

```json
{
  "schemaVersion":"1.0.0",
  "nodes":[
    {"key":"graphic_top","kind":"semantic_object"},
    {"key":"graphic_bottom","kind":"semantic_object"}
  ],
  "edges":[],
  "samVisibleTargets":["graphic_top","graphic_bottom"],
  "directExtractionTargets":["graphic_top","graphic_bottom"],
  "completionTargets":[],
  "imageEditRequired":false,
  "backToFront":["graphic_top","graphic_bottom"],
  "acyclic":true,
  "ambiguousEdges":[]
}
```

`backToFront` 在无遮挡时只记录稳定显示顺序，不代表存在补全需求。

## 4. 分割报告

为 `samVisibleTargets` 中的全部完整对象记录 SAM/Matting 源图可见像素结果：

```json
{
  "key":"left_leg",
  "scope":"visible_only",
  "method":"sam2.1-promptable-multicandidate+pymatting",
  "segmentationStrategy":"tight_then_single_core_outer_recall",
  "semanticPlanHash":"<当前 object-integrity-plan sha256>",
  "instanceSemanticContractHash":"<当前实例语义切片 sha256>",
  "resultOriginSemanticPlanHash":"<实际执行本次SAM时的计划sha256>",
  "validatedAgainstSemanticPlanHash":"<当前计划sha256>",
  "acceptedFromRevision":1,
  "reusedInCurrentRevision":false,
  "revisionPlanRef":null,
  "candidateCount":8,
  "selectedCandidate":"candidate_05",
  "scores":{
    "model":0.94,
    "promptHitRate":1.0,
    "visiblePartRecall":1.0,
    "boundaryStrength":0.89,
    "sourceContinuationFit":0.96,
    "exclusionLeakage":0.0,
    "ensembleDisagreement":0.04
  },
  "integrity":{
    "contourContinuous":true,
    "noUnexplainedCropContact":true,
    "allVisiblePartsPresent":true,
    "noForeignPixels":true,
    "tightAlpha":true,
    "objectOmissionPixels":0,
    "foreignObjectLeakPixels":0,
    "backgroundHaloPixels":128,
    "backgroundHaloMaxWidthPx":2,
    "backgroundHaloOwnership":"verified_bottom_background_only"
  },
  "semanticInvariant":{
    "objectIdentityPreserved":true,
    "componentRecall":1.0,
    "componentOwnershipPreserved":true,
    "parentChildRelationsPreserved":true,
    "mutualExclusionPreserved":true,
    "occlusionAndZOrderPreserved":true
  },
  "tightPass":{"callCount":1,"omissionDetected":true},
  "recallPass":{
    "required":true,
    "callCount":1,
    "promptMode":"subject_core_plus_complete_outer_boundary",
    "usedOriginalFullResolutionSource":true,
    "postSamPixelRepair":false
  },
  "passed":true
}
```

`directExtractionTargets` 的 `scope` 为 `complete_from_source`；`completionTargets` 的 `scope` 必须为 `visible_only`。字段名属于程序契约，保留英文；报告的说明、证据和错误信息尽量使用中文。顶层报告必须记录是否通过、模型/检查点哈希、源图哈希、提示计划哈希和全部 `samVisibleTargets` 结果。

分割完整性使用非对称契约：`objectOmissionPixels` 与 `foreignObjectLeakPixels` 必须同时为零，`semanticInvariant` 各项必须全部通过。允许任意数量的 `backgroundHaloPixels > 0`，但只能来自已验证的最底层背景，并必须记录数量、最大宽度和所有权证据；背景余量不触发失败或再次收紧。若余量触及或包含任何其他前景实例、文字或装饰，立即失败。粗 ROI 只记录为搜索范围，不能作为最终 Alpha 的硬裁剪依据。

每个实例在同一语义版本中必须先有且只有一次 `tightPass`。紧候选无缺失时 `recallPass.required == false` 且 `callCount == 0`；只有检测到对象缺失时才允许 `recallPass.required == true`，此时 `callCount` 必须恰好为 1。补齐仍失败时 `passed == false`，禁止在同一版本内第三次 SAM、循环重试和事后像素补丁。紧/宽两次调用的 `semanticPlanHash` 与 `instanceSemanticContractHash` 必须分别相同。若语义计划需要更改，必须产生新计划版本和哈希，并按下面的增量重启契约处理；不能把新版本的 tight pass 伪装成旧版本的第三次调用。

### 4.1 语义修订后的 SAM 增量重启契约

只有同一生产运行中，SAM 证据暴露遗漏对象、新遮挡关系或对象归属/层级错误时才允许生成 `验收/sam-revision-plan.json`；它不能用于复用旧运行或其他案例产物：

```json
{
  "schemaVersion":"1.0.0",
  "productionRunId":"<同一本轮运行ID>",
  "sourceSha256":"<sha256>",
  "previousSemanticPlanHash":"<旧计划sha256>",
  "currentSemanticPlanHash":"<新计划sha256>",
  "encoderCacheKey":"<source+normalization+model+checkpoint+encoder-config sha256>",
  "changedKeys":["new_foreground","covered_object"],
  "affectedSamTargets":["new_foreground","covered_object","competing_neighbor"],
  "reusedSamTargets":["unrelated_object"],
  "removedTargetKeys":[],
  "invalidationReasons":{
    "covered_object":["target_scope_changed","occlusion_edge_changed"],
    "competing_neighbor":["exclusion_inputs_changed"]
  },
  "reuseProofs":[
    {
      "key":"unrelated_object",
      "priorResultSha256":"<sha256>",
      "sourceAndNormalizationUnchanged":true,
      "modelCheckpointAndEncoderConfigUnchanged":true,
      "instanceSemanticContractHashUnchanged":true,
      "promptPlanHashUnchanged":true,
      "competitorSetHashUnchanged":true,
      "exclusionInputsHashUnchanged":true,
      "scopeUnchanged":true,
      "noPixelOwnershipIntersection":true,
      "revalidatedAgainstCurrentPlan":true,
      "passed":true
    }
  ],
  "downstreamInvalidations":{
    "imageTargets":["covered_object"],
    "background":true,
    "design":true,
    "psd":true,
    "globalQa":true
  },
  "fallbackToAllSamTargets":false,
  "passed":true
}
```

`affectedSamTargets` 是变化的传递闭包，不是手工挑选列表：必须覆盖新增/变化实例、删除实例影响的存续竞争者、变化遮挡边的存续端点、目标 `scope` 变化者，以及所有提示、ROI、组件、互斥、竞争、排除 Alpha、像素所有权或遮挡支持输入发生变化的实例，并沿互斥、空间重叠和遮挡支持依赖扩展至稳定。`reusedSamTargets` 与 `affectedSamTargets` 必须互斥，二者并集必须等于当前 `samVisibleTargets`；删除键不得出现在二者中，其旧结果必须作废。

复用证明任一字段不为 `true`、哈希无法取得、影响闭包无法证明完整，或合并后全局零缺失/零前景泄漏/互斥/目标集合门禁失败时，`fallbackToAllSamTargets` 必须为 `true`，并从新计划对全部当前 `samVisibleTargets` 执行 SAM。受影响对象仍使用原始全分辨率源图，ROI 只能限定搜索范围。只有 `encoderCacheKey` 完全相同时才能复用 SAM2 全图编码。

最终 `segmentation-report.json` 必须覆盖当前全部目标：重跑对象记录新修订号，且 `resultOriginSemanticPlanHash == validatedAgainstSemanticPlanHash == currentSemanticPlanHash`；复用对象必须令 `reusedInCurrentRevision: true`，保留原 `resultOriginSemanticPlanHash`，把 `validatedAgainstSemanticPlanHash` 更新为当前计划哈希，并记录原接受修订号、原结果哈希、当前 `instanceSemanticContractHash` 及 `sam-revision-plan.json` 的复验引用。任何目标缺失或陈旧结果未被证明时报告失败。

## 5. 完整对象补全报告

即使图片没有对象间遮挡，也要生成该报告；对象列表为空、图像编辑调用数为零且 `passed: true`：

```json
{
  "schemaVersion":"1.0.0",
  "passed":true,
  "imageEditRequired":false,
  "imageEditCallCount":0,
  "skippedReason":"no_inter_object_occlusion",
  "objects":[]
}
```

只有 `completionTargets` 非空时才使用以下有补全对象的形式：

```json
{
  "schemaVersion":"1.0.0",
  "passed":true,
  "objects":[
    {
      "key":"left_leg",
      "instanceId":"person_a_left_leg",
      "occludedBy":["shorts"],
      "generatedCompleteAsset":"拆分图/completion/left_leg-generated-complete.png",
      "candidateAlpha":"拆分图/masks-completion/left_leg-candidate-alpha.png",
      "samVisibleEvidenceAlpha":"中间证据/sam-visible/left_leg.png",
      "frontOccluderSupport":"拆分图/masks-visible/shorts.png",
      "hiddenEvidenceMask":"验收/hidden-evidence/left_leg.png",
      "generator":{
        "provider":"configured-image-edit-provider",
        "model":"image-2.0",
        "modelVersion":"记录实际版本",
        "promptVersion":"amodal-v1",
        "seed":null
      },
      "constraints":{
        "sourceVisibleSamCount":1,
        "sourceVisibleSamScope":"visible_only",
        "alphaConversionCount":1,
        "semanticSamCallCountAfterCandidateAccepted":0,
        "candidateAlphaIsFullContourAuthority":true,
        "visibleSamCannotClipCandidateAlpha":true,
        "generatedCompleteObjectLayer":true,
        "generatedAlphaEqualsCandidateAlpha":true,
        "generatedRgbEqualsAcceptedCandidate":true,
        "generatedOnlyInsideHiddenSupport":false,
        "deliveredPixelLayerCountForInstance":1,
        "sourceVisibleLayerIncludedInPsd":false,
        "layerOrderSource":"object-integrity-plan.json#backToFront",
        "generatedCompleteFormula":"generatedCandidate ∩ candidateAlpha",
        "reusedAcrossInstances":false
      },
      "critic":{
        "geometryContinuous":true,
        "materialContinuous":true,
        "structurePlausible":true,
        "matchesAcceptedCandidate":true,
        "singleLayerDelivery":true
      },
      "confidence":0.86,
      "generationTaskId":"<unique-per-instance-task-id>",
      "needsReview":["generated_complete_object"],
      "passed":true
    }
  ]
}
```

生成补全的来源绝不能写成 `original` 或 `extracted`。

图像模型必须输出当前实例的完整对象候选。每个补全目标在 Image 前必须已有且只有一份 `visible_only` SAM/Matting 中间证据，用于提示与验收但不得进入 PSD。候选通过状态与组成检查后只允许一次确定性 Alpha 转换/规范化；此后不得再次运行第二次 SAM，也不得用源图可见残片、遮挡者蒙版、`hiddenEvidenceMask` 或任何洞口蒙版裁剪、覆盖或拼贴候选。`generatedCompleteAsset.alpha` 必须与规范化 `candidateAlpha` 一致，RGB 除键色边缘去色外必须与接受候选一致。每个 `completionTarget` 在 PSD 中只能有这一张交付像素层，并按首轮 `backToFront` 放置。中位色、纯灰、纯背景色、模糊色块或手绘占位不得作为补全素材通过。

## 6. 背景报告

记录恢复范围、方法、模型/版本、已知像素保护、残影检测和通过状态：

```json
{
  "passed":true,
  "recoveryMask":"验收/background-recovery-mask.png",
  "result":"background-reconstructed.png",
  "methods":["确定性渐变重建","生成式局部修复"],
  "knownPixelsUnchanged":true,
  "unassignedPixelsForcedIntoBackground":false,
  "foregroundGhostsDetected":false,
  "movedStateChecksPassed":true,
  "generatedRegionMarked":true
}
```

## 7. design.json 映射

当前仓库 Schema 是直接约束。被遮挡对象通过现有图层组表达：

```text
对象组_左腿完整对象
└─ 对象_左腿_Image完整透明对象   source.type=generated，完整 candidateAlpha 与候选 RGB
```

组顺序直接来自首轮 `object-integrity-plan.json#backToFront`，`occlusion-graph.json` 仅作一致性证明。被遮挡实例只允许一个生成像素层；无遮挡实例只允许一个原图完整提取像素层。

如果需要把 `occludedBy`、`visibleMask` 或 `completion` 直接写进 `design.json`，必须先把 `packages/design-schema/design.schema.json` 迁移到新版本，并同步更新校验器与导出器。不能绕过 `additionalProperties: false`。

设计节点必须包含稳定 ID、来源、素材哈希、明确的位置和变换、可编辑等级、置信度和复核标记。图片的 `frame` 只是变换框；真实可见形状由 Alpha 决定。

## 8. 文字一致性

文字执行严格身份契约：

```json
{
  "sourceContent":"Dance Time!",
  "designContent":"Dance Time!",
  "psdContent":"Dance Time!",
  "preserveVerbatim":true,
  "translationAllowed":false
}
```

除非用户明确修改文案，三个字符串必须 Unicode 完全一致。用户编辑后保存其准确新字符串，不得裁掉空格、自动规范化、翻译、改变大小写、修改标点或替换换行。

记录字体 PostScript Name 和字体文件哈希。字体缺失时标记字体警告，不得静默修改文字。

每个字符串和实际放置位置必须有且只有一个原生文本框节点。只要能识别为可编辑字符串，就不得额外创建或交付包含该字形的栅格层、图片层、图形层或效果子层。源外观像素只允许作为同一个原生文字层的内部显示缓存，不能拥有独立元素 ID、图层 ID、素材节点、PSD 图层或可选交互对象。不得把标题、副标题、中文、英文、Logo 和装饰字合并成一个大栅格块。只有无法作为字符串编辑的真实 Logo 图形、服装印花或实体标签可以按正确类别使用独立栅格层，并记录 OCR、位置、层级、混合模式和来源；蝴蝶、插画、图标等非文字图形单独登记，若被覆盖则走完整图形补全。

## 9. PSD 结构与命名

根图层组使用中文编号名称：

```text
00_参考（默认隐藏）
01_背景恢复
02_完整语义对象
   ├─ 对象组_...
   │  └─ ..._原图完整提取 或 ..._Image完整透明对象（二选一）
03_原生文字与形状
04_前景装饰与效果
```

`02_完整语义对象` 内部从底到顶必须符合遮挡图。生成层必须明确标注，不能伪装为原图提取层。

能力计划中的策略值可保留程序枚举 `native`、`approximate`、`rasterize`、`unsupported`，但面向人的说明使用“原生、近似、栅格化、不支持”。透明像素层降级可以接受，静默丢失或整图扁平化不能接受。

## 10. 来源与不确定性

记录以下内容的哈希与版本：

- 原图和规范化源图；
- 语义清单和遮挡图；
- 提示计划及模型/检查点；
- 每个可见素材、补全素材和背景素材；
- 每个实例的状态字段、独立生成任务、复用审计、未归属像素和附着部件核对结果；
- `design.json`；
- 导出器、解析器和渲染器版本；
- 最终 PSD。

置信度用于安排复核优先级，不能把不确定内容变成事实。无论置信度多高，生成的隐藏内容始终属于推测。阻断式歧义必须保持失败，直到用户或更可靠的真实素材解决。
