# Mirror酱服务集成指南

为你的应用接入 CDN 下载加速、版本检查和增量更新，让用户更顺畅地获取新版本。

Mirror酱（MirrorChyan）负责更新包的分发，你可以继续使用 GitHub Releases 等已有发布渠道。用户未填写 CDK 时，也能通过 API 检查最新版本；需要下载时，再使用项目原有下载源。

- **已有通用 GUI？** MXU、MFAAvalonia、MaaFwApp 等已内置 Mirror酱 更新能力，按官方资源信息配置即可。
- **有自己的客户端？** 通过一个 HTTP 接口获取版本、更新日志和下载地址，复用现有更新界面。
- **希望减少发版操作？** 官方技术支持会按项目提供自动化上传 CI/CD 方案。
- **资源更新频繁？** 支持增量包，减少更新时需要下载的内容；客户端也需兼容全量包。

[开始接入](./QuickStart.md) · [查看已接入项目](https://mirrorchyan.com/zh/projects) · [API 文档](https://apifox.com/apidoc/shared-ffdc8453-597d-4ba6-bd3c-5e375c10c789/253583257e0) · [联系项目接入](#联系我们)

## 选择接入方式

| 你的项目 | 需要做什么 | 从这里开始 |
| --- | --- | --- |
| 使用 MaaFramework 通用 GUI | 按官方信息配置 `interface.json`，验证 GUI 内的更新功能 | [快速开始：通用 GUI](./QuickStart.md#通用-gui-项目) |
| 使用自研客户端 | 接入版本检查、CDK 设置、下载校验和更新安装 | [快速开始：自研客户端](./QuickStart.md#自研客户端) → [客户端集成](./Integration.md) |
| 希望由 AI 辅助接入 | 提供官方 RID 和资源信息，安装接入技能 | [AI 接入](#ai-接入) |
| 已接入，正在排查问题 | 按现象定位参数、CDK 或上传配置问题 | [常见问题](./FAQ.md) · [错误代码表](./ErrorCode.md) |

## 开始接入

1. **联系项目接入 QQ：1751971337。** 提供项目地址、使用的 GUI 和发布平台，官方会确认资源 ID（RID，即 API 的 `res_id`）、资源包划分及上传方案。可直接使用下方的[沟通模板](#联系我们)。
2. **跑通一次版本检查。** 拿到官方 RID、确认首个版本已上传后，按[快速开始](./QuickStart.md)验证。首次请求不需要 CDK。
3. **验证下载与更新。** 通用 GUI 使用内置能力；自研客户端按[集成指南](./Integration.md)实现，并完成[验收清单](./Integration.md#上线前验收)。

RID 由官方分配，不能按项目名或仓库名自行填写。上传 CI/CD 也由官方按项目提供，无需自行设计 Mirror酱 上传工作流。

## AI 接入

本仓库提供 [Agent Skill](./.agents/skills/mirrorchyan-integration/SKILL.md)，可供 Claude Code、Codex 等 AI 编程助手使用：

```bash
npx skills add MirrorChyan/docs
```

也可以直接把技能文件链接发给 AI。请一并提供**官方技术支持给出的 RID、系统 / 架构 / 通道及资源包信息**，AI 会据此选择通用 GUI 配置或自研客户端集成路径。尚未取得这些信息时，先联系项目接入 QQ。

## API 接口

```text
GET https://mirrorchyan.com/api/resources/{res_id}/latest
```

成功响应包含最新版本号；有可用更新且 CDK 有效时，返回带时效的下载地址。**没有 `url` 不等于请求失败，也不能据此判断没有新版本。**

- [可执行的请求示例](./QuickStart.md#自研客户端)
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

官方技术支持会按项目提供定制 CI/CD 方案，在版本发布或提交推送时自动上传。请[联系我们](#联系我们)获取适合项目的方案，并按说明配置上传 Token。

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
