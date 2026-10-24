# 安装指南

安装会让 Agent 读取这套工作流程。执行图片分层还需要[实现接入](IMPLEMENTATION.md)所列工具。

## 方法一：使用 Codex Skill Installer

在 Codex 中发送：

```text
$skill-installer 从 https://github.com/tangchengjiao/image-to-layered-psd 安装 image-to-layered-psd 目录中的 Skill。
```

仓库：`tangchengjiao/image-to-layered-psd`；Skill 子目录：`image-to-layered-psd`；默认分支：`main`。只需安装该子目录，无须把整个仓库当作一个 Skill。

## 方法二：下载 ZIP 手动放置

从 [Release 下载](https://github.com/tangchengjiao/image-to-layered-psd/releases/latest) `image-to-layered-psd-v1.0.0.zip`。解压后，找到直接包含 `SKILL.md` 的 `image-to-layered-psd/` 文件夹：

| 使用范围 | 放置位置 |
| --- | --- |
| 当前项目 | `<项目根目录>/.agents/skills/image-to-layered-psd/` |
| 当前用户 | `~/.agents/skills/image-to-layered-psd/` |

以下以当前项目安装为例。将第一行替换为解压后的 Skill 文件夹路径，在目标项目根目录运行 PowerShell：

```powershell
$skillSource = 'C:\Downloads\image-to-layered-psd'
$skillParent = Join-Path (Get-Location) '.agents\skills'
New-Item -ItemType Directory -Path $skillParent -Force | Out-Null
Copy-Item -LiteralPath $skillSource -Destination $skillParent -Recurse
```

预期结构：

```text
<项目>/.agents/skills/image-to-layered-psd/
├── SKILL.md
├── agents/openai.yaml
└── references/（4 份 Markdown 文档）
```

已有同名 Skill 时先核对安装位置与版本，避免重复嵌套目录。使用 Skill Installer 时，让它按本机配置选择安装目录。

目录发现与显式调用方式依据 [OpenAI 官方 Skill 文档](https://learn.chatgpt.com/docs/build-skills)，核对日期：2026-10-24。Codex 通常会自动发现变化；未出现时重新启动或开启新聊天再检查。

## 验证安装

在新一轮对话输入：

```text
读取 $image-to-layered-psd，概述它需要的输入、外部工具和最终交付。此次只检查安装，不运行图片处理，也不清理案例目录。
```

确认 Agent 能找到主文件及四份参考文档，再按[使用教程](USAGE.md)准备首次任务。Skill 标识为 `image-to-layered-psd`，显示名称为“图片转完整分层 PSD”。

## 更新或移除

更新时从新 Release 取得同名目录，替换之前安装的 Skill 文件，并保留项目自己的实现代码和配置。移除时删除安装的 `image-to-layered-psd/` 目录即可；这不应涉及图片输入或 PSD 交付目录。
