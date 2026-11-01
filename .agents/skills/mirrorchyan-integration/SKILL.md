---
name: mirrorchyan-integration
description: 为应用接入 Mirror酱（MirrorChyan）更新分发服务。优先读取官方接入 PR 或交付说明，复用已支持 Mirror酱 的 GUI 更新能力，或为自研客户端实现版本检查、CDK 设置、下载校验及对应包型的安装流程。未取得官方 RID 时不开始集成，不猜测 RID 或资源包情况，不自行设计上传工作流，应引导联系项目接入 QQ 1751971337。当用户提到 Mirror酱、MirrorChyan、mirrorchyan.com、mirrorchyan_rid、CDK 下载更新，或要为应用添加检查更新 / 自动更新并使用 Mirror酱 时使用。Use when integrating MirrorChyan update checks, CDK-gated downloads or incremental updates into an app or a MaaFramework project. Read the official onboarding PR first; do not invent the RID, package mapping or upload workflows.
---

# Mirror酱 接入

权威文档：<https://github.com/MirrorChyan/docs>。先阅读 [快速开始](https://github.com/MirrorChyan/docs/blob/main/QuickStart.md)选择接入路径；自研客户端参考 [客户端集成](https://github.com/MirrorChyan/docs/blob/main/Integration.md)，错误和更新包格式分别参考 ErrorCode.md、Incremental.md、FAQ.md。本文与文档冲突时以文档为准。

## 硬性门槛：没有官方 RID 就停

先读取用户提供的**官方接入 PR 或交付说明**，提取 RID、发布包与平台的映射、GUI 配置及请求 URL 示例。信息已经给出时直接使用，不要求用户重复填写或证明，也不根据仓库名改写 RID。

尚未取得官方 RID 时，不修改客户端接入逻辑、不填写占位 RID，也不猜测资源包划分。引导用户把仓库地址及希望分发的 Release 包发给**项目接入 QQ：1751971337**，由官方确认资源信息并提供方案。

官方会适配上传、更新日志同步及触发方式，实际运行验证后提交接入 PR。开发者主要确认分发内容，在自己的仓库配置 `MirrorChyanUploadToken`，审阅并合并 PR；自研客户端再按下面的步骤接入 API。

**不要自行设计 Mirror酱 上传工作流或触发步骤。** 已有官方 PR 时以其改动为准，缺少信息时只询问缺失部分。API 请求用于客户端联调，不能以 `latest` 返回 404 为由否认已由官方提供的 RID。

## 先判断项目类型

| 迹象                                                                                                                                                                                                                     | 类型              |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------|
| 项目使用当前版本已支持 Mirror酱 的通用 GUI，如 [MXU](https://github.com/MistEO/MXU)、[MFAAvalonia](https://github.com/MaaXYZ/MFAAvalonia)；其他 GUI 先核对对应版本文档和接入 PR | **情况 A：通用 GUI** |
| 项目有自己的界面和代码，如 [MAA](https://github.com/MaaAssistantArknights/MaaAssistantArknights)                                                                                                                                    | **情况 B：自研界面**   |

无法判断时询问用户。

### 情况 A：通用 GUI —— 复用内置更新能力

通用 GUI 已内置 Mirror酱 的检查更新、CDK 设置和下载安装。**不要编写任何更新逻辑。**
还没拿到官方 RID 时，连 `interface.json` 也不要改，按 [硬性门槛](#硬性门槛没有官方-rid-就停) 停下来。
拿到官方信息后，先核对接入 PR 中已有的配置，避免重复修改。对于读取 `interface.json` 更新配置的 GUI，按 [ProjectInterface V2 协议](https://github.com/MaaXYZ/MaaFramework/blob/main/docs/zh_cn/3.3-ProjectInterfaceV2协议.md)检查以下字段；其他 GUI 按其自身文档和接入 PR 配置：

| 字段                          | 说明                                                                            |
|-----------------------------|-------------------------------------------------------------------------------|
| `mirrorchyan_rid`           | 资源 ID（即 `res_id`）。只能填官方技术支持提供的值，不要用仓库名、项目名或包名前缀代替。还没拿到就不要配置 |
| `mirrorchyan_multiplatform` | 对支持此字段的 GUI，上传指定了 `os` 或 `arch` 时为 `true`；两者都为空时为 `false` 或不填。以接入 PR 为准，不自行猜测 |
| `version`                   | 若 GUI 用此字段检查更新，需与 tag 及上传版本名一致；不能假设所有 GUI 都以此字段为版本来源 |
| `github`                    | 项目 GitHub 仓库地址，用于版本更新检查和问题反馈                                                  |

注意：

- GUI 的版本来源、平台参数和更新范围以实际使用版本为准，不能一律视为只更新资源。APK 项目需整体重新出包，不能套用桌面 ZIP 的解压覆盖流程，见 [MaaFwApp 接入文档](https://github.com/Aliothmoon/MaaFwApp/blob/main/INTEGRATION.md)。
- 同时发布多个通用 GUI 的版本时，每个 GUI 需要单独的资源
  ID，见 [FAQ.md](https://github.com/MirrorChyan/docs/blob/main/FAQ.md)。ZIP 与 APK / EXE 安装包也需使用不同 RID，由官方按包型及用途划分。
- `mirrorchyan_multiplatform` 必须与实际上传方式一致，否则检查更新会返回
  404，见 [返回 404（code 8001）时的排查](#返回-404code-8001时的排查)。
- 字段以协议文档和对应 GUI 的文档为准，如有出入以它们为准。

配置完成后，如需自动上传，继续看 [第 6 步](#第-6-步可选自动化上传)，其余步骤跳过。

### 情况 B：自研界面 —— 编写更新逻辑

按下面第 0 到 6 步实现，开始前必须先通过 [硬性门槛](#硬性门槛没有官方-rid-就停)。集成流程是推荐做法而非强制要求，具体文案和行为按项目实际调整。复用项目已有的 HTTP 客户端、JSON
库、设置存储和 UI 组件，遵循项目现有代码风格。

## 第 0 步：先确认，不要猜

[硬性门槛](#硬性门槛没有官方-rid-就停) 已经通过、用户已经拿出官方技术支持提供的 RID 和相关信息之后，再核对下面这些项。**不要编造 `res_id`、资源包的平台划分、CDK 或 API 字段。** 仓库里看不到的，问用户；和官方信息不一致的，以官方为准，不要改成仓库名或包名前缀。

| 信息              | 从哪里获取                                            | 用途                  |
|-----------------|--------------------------------------------------|---------------------|
| 资源 ID（`res_id`） | 只能使用官方技术支持提供的值。没有就回到硬性门槛，停止集成                  | 路径参数                |
| 当前版本号           | 项目内（程序集版本、`package.json`、`Cargo.toml`、Git tag 等） | `current_version`   |
| 发布的系统 / 架构      | 以官方技术支持提供的资源包信息为准。官方没说明时问用户，不要只按构建矩阵或 Release 产物自己定 | `os`、`arch`         |
| 更新通道            | 以官方说明为准。官方没说明时问用户是否区分正式版 / 测试版                  | `channel`           |
| 客户端标识           | 与用户商定，如 `MAA_WPF`                                | `user_agent`（签到源统计） |
| 现有更新机制          | 项目内（如 GitHub Releases 检查）                        | 作为回退下载源             |
| 安装包形态           | 按交付方案区分 ZIP 等归档资源与 APK / EXE 安装包；程序运行时能否替换自身 | 选择增量 / 全量和安装流程 |

确认后，优先使用交付的 URL 示例联调客户端。返回 404 时按 [返回 404（code 8001）时的排查](#返回-404code-8001时的排查)核对可用版本和参数，不把 API 响应当作官方 RID 是否成立的证明。

## 第 1 步：设置项

1. **CDK 输入框**：持久化保存；界面上掩码显示。**CDK 不得以明文出现在日志、崩溃报告、遥测或导出的配置中。**
2. **Mirror酱 跳转链接**：`https://mirrorchyan.com?source=<app名>_<链接位置>`，例如 `source=maa_app_settings`。`source`
   对应统计面板中的「付费源」。

## 第 2 步：检查更新

```text
GET https://mirrorchyan.com/api/resources/{res_id}/latest
```

| 参数                | 位置    | 说明                                                                            |
|-------------------|-------|-------------------------------------------------------------------------------|
| `res_id`          | path  | 资源 ID，必填                                                                      |
| `current_version` | query | ZIP 增量更新传本地版本，格式需与上传版本名一致；请求完整包时留空，APK / EXE 安装包留空。不保证一定返回增量包，以 `update_type` 为准 |
| `cdk`             | query | 用户 CDK，未填写时不传                                                                 |
| `os`              | query | `windows` / `linux` / `darwin` / `android`；也接受 `win`、`macos` 等别名。资源不区分系统时不传   |
| `arch`            | query | `386` / `amd64` / `arm` / `arm64`；也接受 `x64`、`x86_64`、`aarch64` 等别名。资源不区分架构时不传 |
| `channel`         | query | `stable`（默认）/ `beta` / `alpha`                                                |
| `user_agent`      | query | 客户端标识，对应统计面板中的「签到源」。`mirrorchyan_web` 为保留值，不要使用                               |

首次请求优先使用官方接入 PR 提供的请求 URL 示例，见 [快速开始](https://github.com/MirrorChyan/docs/blob/main/QuickStart.md#自研客户端)。没有示例时，使用用户提供的官方 RID 和平台信息构造请求；不携带 CDK 验证版本检查，不借用其他项目的 RID 或示例密钥。

要求：

- **未填写 CDK 时也调用**。Mirror酱 CDN 连通性好，可以更快获取更新信息，下载再回退到其他源。
- 异步执行，设置合理超时，**失败不得阻塞启动或导致崩溃**。
- 在启动时和用户手动点击时检查，不要高频轮询。
- CDK 在 URL 中，**不要记录完整请求 URL**。

## 第 3 步：处理响应

响应格式为 `{ "code": 0, "msg": "success", "data": { ... } }`。

### `code == 0`

| 字段                        | 说明                                          |
|---------------------------|---------------------------------------------|
| `version_name`            | 最新版本号，始终返回                                  |
| `release_note`            | 更新日志                                        |
| `url`                     | 带时效的下载地址，**仅在有新版本且 CDK 有效时返回**。拿到后尽快下载，不要缓存 |
| `sha256` / `filesize`     | 下载后校验完整性 / 显示进度                             |
| `update_type`             | `incremental`（增量）或 `full`（全量）               |
| `channel` / `os` / `arch` | 本次查询的通道 / 系统 / 架构                         |
| `custom_data`             | 开发者上传时附带的自定义数据                              |
| `cdk_expired_time`        | CDK 到期时间戳，可用于提示续费                           |

### `code != 0`

| code               | 含义                    | 建议处理                       |
|--------------------|-----------------------|----------------------------|
| 1001               | 参数不正确                 | 集成问题，检查请求参数                |
| 7001               | CDK 已过期               | 提示续费，附上付费链接                |
| 7002               | CDK 错误                | 提示检查 CDK                   |
| 7003               | CDK 今日下载次数已达上限        | 提示明日再试                     |
| 7004               | CDK 类型和待下载的资源不匹配      | 提示该 CDK 不适用于本应用            |
| 7005               | CDK 已被封禁              | 提示联系 Mirror酱 售后            |
| 8001               | 对应架构和系统下的资源不存在        | 见下方「返回 404（code 8001）时的排查」 |
| 8002 / 8003 / 8004 | 系统 / 架构 / 更新通道参数错误    | 集成问题，检查参数取值                |
| 8009               | 流程测试 CDK 不支持下载（下载时返回） | 提示使用正式 CDK                 |
| 1                  | 未区分的业务错误              | 显示 `msg`                   |
| 其他 > 0             | 业务错误                  | 显示 `msg`                   |
| < 0                | 意料之外的严重错误             | 提示联系 Mirror酱 技术支持          |

CDK 相关错误（7001–7005）不应影响用户通过其他渠道获取更新。网络失败时回退到项目已有的更新机制。

## 第 4 步：比较版本并下载

```text
code == 0 ?
├─ 否 → 按错误码提示
└─ 是 → version_name 比 current_version 新？
        ├─ 否 → 结束
        └─ 是 → 返回了 url？
                ├─ 是 → 从 url 下载 → 校验 sha256 → 安装更新
                └─ 否 → 从 GitHub 等其他源下载，并提示可填写 CDK 使用 Mirror酱 高速下载
```

- 按 [SemVer](https://semver.org/lang/zh-CN/) 比较，忽略前缀 `v`，正确处理预发布版本（如 `1.2.0-beta.1`）。项目不使用 SemVer
  时，与用户确认比较规则，**不要直接比较字符串**。
- 下载请求返回非 2xx 时，有 JSON 错误体则解析 `code` / `msg`，否则按下载失败处理，不能假设错误响应总是 JSON。
- `sha256` 校验失败时丢弃文件，不得安装。

## 第 5 步：安装更新

APK、EXE 等安装包按完整包分发，校验后交给对应安装流程，不解压或处理 `changes.json`。以下步骤仅适用于 ZIP 等归档资源：

1. 解压到临时目录，在写入每个条目前校验目标路径和链接，再校验 `changes.json`；通过后才准备退出主程序或由独立更新进程替换安装文件。
2. 若包内存在 `changes.json`，删除安装目录中 `deleted` 列出的文件和 `deleted_dir`
   列出的目录，目录按从深到浅的顺序处理。字段说明见 [Incremental.md](https://github.com/MirrorChyan/docs/blob/main/Incremental.md)，路径以 `/`
   分隔，相对于包根目录。缺失字段按空列表处理，删除失败时不能直接报告成功。
3. 用新文件覆盖安装目录。
4. 全部操作成功后再记录新版本。失败或中断后恢复完整旧版本，或停止增量更新并通过干净的全量安装恢复，不能仅保留旧版本号后继续叠加增量包。若采用重命名旧文件的方案，需处理重命名失败；能否重命名取决于系统和文件打开方式，不能假设一定不受占用限制。

**安全**：解压条目和 `changes.json` 中的每个路径都必须解析到对应目标目录之内，拒绝 `..`、绝对路径、盘符和 UNC 路径，并检查符号链接，避免越界写入或删除。

## 第 6 步（可选）：自动化上传

情况 A、B 都适用。**不要自己编写 Mirror酱 相关 workflow**，也不要在现有 CI 里自行添加上传或触发步骤。Mirror酱 官方会为项目定制 CI/CD，在发版时自动上传。需要自动化上传时，引导用户联系 **项目接入 QQ：1751971337**（见 [联系我们](https://github.com/MirrorChyan/docs#联系我们)），使用官方技术支持提供的方案。官方还没给方案时停在这一步，不要先写一版 workflow 占位。

上传 Token 用 `gh secret set MirrorChyanUploadToken` 存入仓库 secret，**不得写入仓库文件**。GitHub 会把 secret 名称显示为全大写
`MIRRORCHYANUPLOADTOKEN`
，属于正常现象，引用时不区分大小写。更多问题见 [FAQ.md](https://github.com/MirrorChyan/docs/blob/main/FAQ.md)。

## 返回 404（code 8001）时的排查

以下情况都会返回 HTTP 404 和 `code 8001`，**无法仅凭响应区分**：

- `res_id` 不存在：用户还没有接入 Mirror酱，或 ID 填错
- 已接入，但还没有上传过任何版本
- 已上传，但请求的 `channel` / `os` / `arch` 组合下没有包

匹配规则：

- `os` / `arch` 与上传时的值**精确匹配**。按平台分别上传的包，请求必须带上对应的 `os` 和 `arch`；不区分平台上传的包，请求不能带
  `os` / `arch`，带了反而返回 404。
- `channel=stable` 只查正式版；`beta` 取正式版和测试版中较新的一个；`alpha` 取三个通道中最新的一个。

接入时由 AI 排查：

1. 不带 CDK、使用 `channel=alpha`，分别请求「不带 `os` / `arch`」和「带官方提供的 `os` / `arch`」。官方没给出平台划分时，先问用户，不要按仓库构建产物自己假定一组平台再试。
2. 有组合返回 `code == 0`：说明已接入，是请求参数与上传的包不一致。按能返回的组合修正参数（情况 A 检查
   `mirrorchyan_multiplatform`），并确认目标通道确实上传过版本。
    - 两种组合都返回时，以 `version_name` 与项目最新发版一致的为准。从不区分平台改为按平台上传的项目，可能还留着旧的不分平台的包，不带
      `os` / `arch` 会拿到很旧的版本。
3. 全部返回 404：核对交付方案中的可用版本和平台 / 通道映射，必要时联系项目接入 QQ 1751971337。已经有官方 RID 时，不重复要求用户证明来源，不自行换 RID。尚未取得官方资源信息时，按 [硬性门槛](#硬性门槛没有官方-rid-就停)引导联系官方，不按仓库名或构建产物猜测。

运行时（情况 B 的应用内）：收到 8001 时提示「当前平台暂无可用更新」并回退到其他更新源，不要当作严重错误弹窗。日志中记录请求的
`os` / `arch` / `channel`（不含 CDK），便于排查。

## 验收清单

情况 A：按接入 PR 和对应 GUI 文档确认配置，并在 GUI 中实际检查一次更新。使用 `interface.json` 的项目还需通过 [schema](https://github.com/MaaXYZ/MaaFramework/blob/main/tools/interface.schema.json)校验。返回 404 时按上一节排查，APK 项目按对应版本的完整安装包流程验证。

情况 B：完成后逐项确认，并向用户报告结果：

- [ ] 不填 CDK：返回 `code == 0`，有 `version_name`，无 `url`，程序回退到其他下载源。
- [ ] 填入错误 CDK：返回 7002，界面有明确提示，程序不崩溃。
- [ ] 断网或超时：不阻塞启动，不崩溃。
- [ ] 当前平台 / 通道没有包（8001）：有明确提示，回退到其他更新源。
- [ ] 已是最新版本：不提示更新。
- [ ] 在日志、配置导出和崩溃报告中搜索 CDK，确认没有明文。
- [ ] `user_agent` 和跳转链接的 `source` 已按约定设置。
- [ ] ZIP 更新：`deleted` 中的文件被删除，路径越界的条目在写入前被拒绝；中断后可恢复完整版本。
- [ ] APK / EXE 更新：使用完整安装包和对应安装流程，不套用 ZIP 增量逻辑。
