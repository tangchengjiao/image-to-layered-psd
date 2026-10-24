# 图片转完整分层 PSD · Image to Layered PSD

把电商图、海报、营销图、商品图和包装图的拆解方法整理成可复用的 AI Skill：识别对象与文字、确定遮挡关系、提取透明素材、补全被遮挡对象、重建背景，并指导分层 PSD 导出和验收。

**当前发布：v1.0.0 文档型 Skill。** 包含完整流程规范和参考文档，供具备图像工具与 PSD 实现的 Agent 使用。原始压缩包的 `scripts/` 为空，本仓库没有附带模型、执行器、PSD 导出代码或可直接运行的工作台；安装 Skill 后仍须接入这些实现。

[下载 Skill ZIP](https://github.com/tangchengjiao/image-to-layered-psd/releases/latest/download/image-to-layered-psd-v1.0.0.zip) · [版本发布页](https://github.com/tangchengjiao/image-to-layered-psd/releases) · [安装指南](docs/INSTALL.md) · [使用教程](docs/USAGE.md) · [实现接入](docs/IMPLEMENTATION.md)

## 这套 Skill 规定了什么

| 环节 | 目标 |
| --- | --- |
| 语义分析 | 按可独立操作的完整对象拆分，识别文字、Logo、附着部件和前后关系 |
| 可见像素提取 | 使用 SAM2 / Matting 提取原图实际可见部分，细化透明边缘 |
| 遮挡对象补全 | 仅对存在对象间遮挡的目标生成完整透明对象，区分生成内容与原图证据 |
| 文字重建 | 每条可识别字符串对应独立原生文字层，保留原文、位置和样式信息 |
| 背景与组装 | 重建移开对象后露出的背景，按确定的前后顺序组织图层 |
| PSD 验收 | 检查图层结构、文字、透明边缘、移动后的效果与导出回读结果 |

这些是生产目标与验收规范，实际能力取决于接入的工具。仅有单张扁平图无法找回被遮挡部分的真实原始像素；AI 补全属于推测生成。艺术字、字体和复杂效果的还原也取决于字体资源及导出器能力。

## 快速安装与调用

在 Codex 中发送：

```text
$skill-installer 从 https://github.com/tangchengjiao/image-to-layered-psd 安装 image-to-layered-psd 目录中的 Skill。
```

也可以下载上方 ZIP，将里面的 `image-to-layered-psd/` 文件夹放入项目的 `.agents/skills/` 或个人的 `~/.agents/skills/`。安装目录、Windows 命令与验证方法见[安装指南](docs/INSTALL.md)。本次发布不会自动安装到本机。

配置好实现环境后，调用示例：

```text
使用 $image-to-layered-psd 处理这张电商海报。
源图：<源图绝对路径>
案例目录：<本次专用案例目录>
实现目录：<PSD 实现项目目录>
保留原文，重建独立对象、原生文字层和背景，导出分层 PSD。
请先核对实现、模型与导出器是否可用，再按 Skill 执行。
```

原 Skill 包含重跑前清理案例旧产物及完成后清理中间文件的规则。请使用专用案例目录并明确源图路径；需要长期保留的原始材料放在该目录之外。首次接入可先让 Agent 只输出环境缺项与实现映射。

## 仓库结构

```text
image-to-layered-psd/
├── SKILL.md                            Skill 主入口
├── agents/openai.yaml                  显示名称与默认调用提示
└── references/
    ├── pipeline.md                     生产流程
    ├── contracts.md                    数据与 PSD 结构契约
    ├── qa-gates.md                     质量验收
    └── project-implementation.md       原实现项目映射
docs/
├── INSTALL.md                          安装与发现
├── USAGE.md                            输入、调用、交付与常见问题
└── IMPLEMENTATION.md                   依赖、接口与迁移边界
CHANGELOG.md                            发布记录
```

原包中的 6 份 Skill 文件保持原内容。空 `scripts/` 目录没有执行文件，Git 不追踪该空目录；ZIP 中保留此目录。原实现映射中的本地路径和案例命令属于原项目背景，本仓库未包含这些外部文件，不能在下载后直接执行。接入方式见[实现说明](docs/IMPLEMENTATION.md)。

## 交付与验证范围

生产流程的预期输出是一份 `final-layered.psd`，存在实际必需外部依赖时附带 `dependencies/`。对象通常是透明像素层；原生文字层可修改文案，具备可靠几何信息的形状才适合重建为原生形状层。分层不代表所有内容都自动成为矢量或参数化对象。

本版本核对了压缩包文件完整性、UTF-8 文本、Skill 元数据、文档相对链接和发布包结构。**没有执行图片转 PSD 或 Photoshop 实测，也没有附带已验收 PSD 示例。**

## 文档导航

- [安装指南](docs/INSTALL.md)：下载、目录放置和 Skill 发现。
- [使用教程](docs/USAGE.md)：准备输入、环境检查、调用模板和交付检查。
- [实现接入](docs/IMPLEMENTATION.md)：外部程序、模型职责及原项目迁移。
- [更新记录](CHANGELOG.md)：本次发布内容与范围。
- [Skill 主规范](image-to-layered-psd/SKILL.md)：进入 PSD 生产时阅读的完整要求。

<!-- JIANZHOU-PUBLIC-INFO:START -->
## 企业服务与联系

**华链 AI** · 企业 AI 业务落地与内生能力建设

培训与 FDE 培养 · 场景验证 · 智能体定制 · 企业实施 · 产品应用支持

[华链 AI 官网](https://hualian-ai.pages.dev) · [企业服务](https://hualian-ai.pages.dev/services/) · [服务参考价](https://hualian-ai.pages.dev/pricing/) · [企业宣传册](https://hualian-ai.pages.dev/downloads/company-brochure.pdf) · [合作咨询](https://hualian-ai.pages.dev/contact/)

咨询联系人：**见舟** · 微信：**`DST10209`**

<img src="https://hualian-ai.pages.dev/assets/contact/wechat-qr.png" alt="见舟微信二维码" width="200">

抖音：`74683161410` · 视频号：见舟AI赋能
<!-- JIANZHOU-PUBLIC-INFO:END -->
