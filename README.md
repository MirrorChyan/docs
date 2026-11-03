# Mirror酱服务集成指南

为你的应用接入 CDN 下载加速、版本检查和增量更新，让用户更顺畅地获取新版本。

Mirror酱（MirrorChyan）负责更新包的分发，你可以继续使用 GitHub Releases 等已有发布渠道。用户未填写 CDK 时，也能通过 API 检查最新版本；需要下载时，再使用项目原有下载源。

- **已有通用 GUI？** 使用已支持 Mirror酱 的 MXU、MFAAvalonia 等 GUI 时，通常无需另写更新器。
- **有自己的客户端？** 通过一个 HTTP 接口获取版本、更新日志和下载地址，复用现有更新界面。
- **希望减少发版操作？** 我们会提交接入 PR，适配现有发布流程，配置更新包上传与更新日志同步。
- **资源更新频繁？** ZIP 资源包可使用增量更新，减少下载内容；APK、EXE 等安装包按完整包分发。

[开始接入](./QuickStart.md) · [查看已接入项目](https://mirrorchyan.com/zh/projects) · [API 文档](https://apifox.com/apidoc/shared-ffdc8453-597d-4ba6-bd3c-5e375c10c789/253583257e0) · [联系项目接入](#联系我们)

## 选择接入方式

| 你的项目 | 需要做什么 | 从这里开始 |
| --- | --- | --- |
| 使用已支持 Mirror酱 的通用 GUI | 按接入 PR 完成配置，验证 GUI 内的更新功能 | [快速开始：通用 GUI](./QuickStart.md#通用-gui-项目) |
| 使用自研客户端 | 接入版本检查、CDK 设置、下载校验和更新安装 | [快速开始：自研客户端](./QuickStart.md#自研客户端) → [客户端集成](./Integration.md) |
| 希望由 AI 辅助接入 | 将接入 PR 和客户端需求交给 AI，安装接入技能 | [AI 接入](#ai-接入) |
| 已接入，正在排查问题 | 按现象定位参数、CDK 或上传配置问题 | [常见问题](./FAQ.md) · [错误代码表](./ErrorCode.md) |

## 开始接入

1. **把仓库地址发给我们。** 联系项目接入 QQ：1751971337，准备好至少一个带发布包的 Release，告诉我们希望分发哪些包即可。
2. **我们适配并提交接入 PR。** 根据发布产物确认资源 ID（RID，即 API 的 `res_id`）和平台划分，配置上传、更新日志同步及适用的 GUI 设置，实际运行上传工作流验证，并提供请求 URL 示例。
3. **你配置 Token，审阅并合并 PR。** 在自己的仓库设置 `MirrorChyanUploadToken`。合并后，后续版本按 PR 约定自动上传，继续沿用项目已有的发版方式。
4. **验证客户端更新。** 支持 Mirror酱 的 GUI 复用内置能力；自研客户端按[集成指南](./Integration.md)接入。具体步骤见[快速开始](./QuickStart.md)。

RID、平台配置和上传工作流会随接入方案一起提供，你无需先自行设计这些配置。首次沟通也可使用下方的[项目信息模板](#联系我们)。

## AI 接入

本仓库提供 [Agent Skill](./.agents/skills/mirrorchyan-integration/SKILL.md)，可供 Claude Code、Codex 等 AI 编程助手使用：

```bash
npx skills add MirrorChyan/docs
```

也可以直接把技能文件链接发给 AI，并附上**官方接入 PR 或交付说明**及你希望完成的客户端功能。AI 会使用其中的 RID、平台配置和请求示例，复用现有 GUI 或接入自研客户端。还没有接入方案时，先把仓库地址发给项目接入 QQ。

## API 接口

```text
GET https://mirrorchyan.com/api/resources/{res_id}/latest
```

成功响应包含最新版本号；有可用更新且 CDK 有效时，返回带时效的下载地址。**没有 `url` 不等于请求失败，也不能据此判断没有新版本。**

- [首次请求指引](./QuickStart.md#自研客户端)
- [请求参数、响应字段和客户端处理](./Integration.md#api-接口)
- [完整 API 文档](https://apifox.com/apidoc/shared-ffdc8453-597d-4ba6-bd3c-5e375c10c789/253583257e0)
- [错误代码表](./ErrorCode.md)

## 集成流程指南

集成流程为推荐做法，具体界面和行为可按项目调整。

### 检查更新流程

未填写 CDK 时也可使用 Mirror酱 API 检查版本。先判断响应是否成功，再比较版本，最后选择下载源；CDK 错误或网络失败时，保留项目原有更新入口。详见[客户端集成](./Integration.md#检查更新流程)。

### 增量更新流程

增量包包含新增、修改的文件及 `changes.json`。客户端需要先处理删除项，再覆盖新文件，并兼容服务端返回全量包的情况。格式和安装步骤见[增量更新指南](./Incremental.md)。

## 自动化上传

我们会根据项目现有的构建和 Release 流程，提交包含更新包上传、更新日志同步及触发配置的接入 PR，并在提交 PR 前实际运行验证。你主要需要在自己的仓库配置上传 Token，审阅并合并 PR。

之后按项目原有方式发布版本即可，具体触发时机以接入 PR 为准，无需每次手动上传到 Mirror酱。

Token 配置、增加架构和多个 GUI 的资源划分见[常见问题](./FAQ.md#如何设置仓库的-mirrorchyanuploadtoken)。

## 日活统计

Mirror酱 提供按 IP 去重的近似日活统计，以及检查更新量、下载量等数据，可在统计面板中查看。此数据用于观察分发使用情况，不代表精确的独立用户数。

## 来源统计

来源统计是可选功能，建议在基本更新流程跑通后配置。

### 签到源

在版本检查请求中添加 query 参数 `user_agent`，标识用户从哪个客户端使用 CDK，对应统计面板中的「签到源」。例如使用 `user_agent=myapp_desktop`，项目内保持命名一致即可。

从 [Mirror酱项目一览](https://mirrorchyan.com/zh/projects)下载应用时，该值为 `mirrorchyan_web`，客户端请勿使用这一保留值。这里的 `user_agent` 是 API 参数，不是 HTTP `User-Agent` 请求头。

### 付费源

在跳转到 Mirror酱 的网页链接中添加 `source`，标识用户从哪个入口打开网站并付费，对应统计面板中的「付费源」。建议包含应用名和链接位置，例如：

```text
https://mirrorchyan.com?source=myapp_settings
https://mirrorchyan.com?source=myapp_github_release
```

## 常见问题

- [不填 CDK，也能检查更新吗？](./FAQ.md#不填-cdk也能检查更新吗)
- [返回 404 / code 8001 怎么排查？](./FAQ.md#返回-404code-8001怎么排查)
- [Action 上传成功，但新架构未出现？](./FAQ.md#action-上传成功但新架构未出现)
- [更多问题](./FAQ.md)

## 联系我们

- 项目接入：**QQ 1751971337**
- 用户售后：**QQ 群 995458883**

首次沟通可复制以下模板，方便确认接入路径；暂不确定的项可以留空：

```text
项目名称与仓库地址：
使用的 GUI / 自研客户端：
发布的系统与架构：
是否区分正式版 / 测试版：
现有发布方式（如 GitHub Releases）：
希望解决的问题（如下载慢、增量更新、自动上传）：
```

请勿在公开 Issue、截图或日志中附上 CDK、上传 Token 或完整下载链接。
