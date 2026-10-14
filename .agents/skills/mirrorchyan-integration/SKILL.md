---
name: mirrorchyan-integration
description: 为应用接入 Mirror酱（MirrorChyan）更新分发服务。通用 GUI（MXU、MFAAvalonia、MaaFwApp）由官方提 PR 写好 interface.json 和上传 workflow，不要自行修改仓库。自研界面在拿到官方 RID 后按 README 实现检查更新、下载校验、增量更新和错误码处理，仍然不要自行编写 Mirror酱 workflow。未拿到官方 RID 时禁止开始集成，禁止假设 RID 或资源包情况。联系项目接入 QQ 1751971337。当用户提到 Mirror酱、MirrorChyan、mirrorchyan.com、mirrorchyan_rid、CDK 下载更新，或要为应用添加检查更新 / 自动更新并使用 Mirror酱 时使用。Use when integrating MirrorChyan update checks, CDK-gated downloads or incremental updates into an app or a MaaFramework project. If an official RID has not been provided, do not start, do not assume the RID or package layout, and do not write MirrorChyan workflows. For generic GUI projects, do not edit the repository.
---

# Mirror酱 接入

实现前先读本仓库的 [README.md](https://github.com/MirrorChyan/docs/blob/main/README.md)、[ErrorCode.md](https://github.com/MirrorChyan/docs/blob/main/ErrorCode.md)、[Incremental.md](https://github.com/MirrorChyan/docs/blob/main/Incremental.md)、[FAQ.md](https://github.com/MirrorChyan/docs/blob/main/FAQ.md)。请求参数、响应字段、错误码、增量安装和验收清单以这些文档为准。本文与文档冲突时以文档为准。

本文只规定 AI 什么时候必须停、什么时候不要改仓库。

## 硬性门槛：没有官方 RID 就停

在判断项目类型、修改代码、修改 `interface.json`、请求接口或编写任何 Mirror酱 相关逻辑之前，先用询问的语气确认 RID 是不是官方给的。例如：

> 官方提供的 RID 你拿到了吗？还是目前只是自己按仓库名、项目名假设的，或者还没跟官方对过？

下面任一情况都算**还没拿到 RID**，必须立刻停止，**禁止开始集成**：

- 用户没有给出 RID
- 用户说「应该就是仓库名 / 项目名 / 包名前缀」，或让你看着仓库自己定
- RID 是你根据仓库、历史项目或文档示例（如 `M9A`）猜的
- 用户说先写上占位，以后再改

停止之后告诉用户：请先联系 Mirror酱官方，**项目接入 QQ：1751971337**。等官方提供 RID 和相关信息后，再把这些信息发过来。

还没拿到时，同时禁止：

- **自行假设 RID**
- **自行假设资源包情况**（有没有上传过包、按不按 `os` / `arch` / `channel` 区分、是不是多平台、要不要多个资源 ID 等）
- **自己编写任何 Mirror酱 相关 workflow**（不要新建 `mirrorchyan_release.yml`、`mirrorchyan_release_note.yml`，不要加 uploading action，也不要在现有 workflow 里加触发步骤）
- **修改 `interface.json`**

只有用户明确给出**官方技术支持提供的 RID 和相关信息**之后，才继续下面的步骤。RID、资源包的平台 / 架构 / 通道划分，都以官方给的为准。官方没提到的部分要问用户，或让用户联系项目接入 QQ 1751971337 确认，不要自己补。

自动化上传的 CI 由官方提 PR 提供。即使用户已经拿到 RID，也不要自己设计 Mirror酱 workflow。官方还没给方案时，引导用户联系项目接入 QQ。

## 先判断项目类型

| 迹象 | 类型 |
| --- | --- |
| 项目基于 MaaFramework，有带 `interface_version` 的 `interface.json`，发版时打包 [MXU](https://github.com/MistEO/MXU)、[MFAAvalonia](https://github.com/MaaXYZ/MFAAvalonia)、[MaaFwApp](https://github.com/Aliothmoon/MaaFwApp) 等通用 GUI | **情况 A：通用 GUI** |
| 项目有自己的界面和代码，如 [MAA](https://github.com/MaaAssistantArknights/MaaAssistantArknights) | **情况 B：自研界面** |

无法判断时询问用户。

### 情况 A：通用 GUI —— 不要改仓库

通用 GUI 已内置检查更新、CDK 设置和下载安装。**不要编写更新逻辑，不要修改 `interface.json`，不要编写 workflow。**

告诉用户：把仓库地址发给**项目接入 QQ：1751971337**。官方会提 PR，写入 `mirrorchyan_rid`、`mirrorchyan_multiplatform`，并加上传 workflow。merge 后按 [FAQ.md](https://github.com/MirrorChyan/docs/blob/main/FAQ.md) 设置 `MirrorChyanUploadToken`。字段含义见 [README 的通用 GUI 一节](https://github.com/MirrorChyan/docs/blob/main/README.md#通用-gui)。

用户已经合并官方 PR、只是来排错时：只读核对 `interface.json` 和 README，不要改 RID，不要新写 workflow。返回 404 时按下面的排查用 curl 确认，把结论告诉用户，平台划分不清楚时让用户回项目接入 QQ。

### 情况 B：自研界面 —— 按 README 实现客户端

通过 [硬性门槛](#硬性门槛没有官方-rid-就停) 之后，按 README 实现检查更新、下载、校验和安装。复用项目已有的 HTTP 客户端、JSON 库、设置存储和 UI 组件，遵循项目现有代码风格。

不要编写上传 workflow。需要自动上传时，让用户联系项目接入 QQ。

开始写代码前，对照 README「API」核对官方已提供的信息：资源 ID、版本号格式、`os` / `arch`、`channel`。官方没说明的不要从构建产物推断。然后不带 CDK 用 curl 请求一次接口。返回 404 时按下方排查，确认已接入且参数正确后再继续。

完成后逐项执行 README「验收」，并向用户报告结果。

## 返回 404（code 8001）时的排查

以下情况都会返回 HTTP 404 和 `code 8001`，**无法仅凭响应区分**。匹配规则见 README「请求参数」。

- `res_id` 不存在：用户还没有接入 Mirror酱，或 ID 填错
- 已接入，但还没有上传过任何版本
- 已上传，但请求的 `channel` / `os` / `arch` 组合下没有包

接入时由 AI 排查：

1. 不带 CDK、使用 `channel=alpha`，分别请求「不带 `os` / `arch`」和「带官方提供的 `os` / `arch`」。官方没给出平台划分时，先问用户，不要按仓库构建产物自己假定一组平台再试。
2. 有组合返回 `code == 0`：说明已接入，是请求参数与上传的包不一致。按能返回的组合修正参数（情况 A 只核对 `mirrorchyan_multiplatform`，不要改 RID），并确认目标通道确实上传过版本。
    - 两种组合都返回时，以 `version_name` 与项目最新发版一致的为准。从不区分平台改为按平台上传的项目，可能还留着旧的不分平台的包，不带 `os` / `arch` 会拿到很旧的版本。
3. 全部返回 404：**停下来向用户确认**，不要自行猜测或修改 `res_id`，也不要换一个 ID 再试：
    - 用询问的语气问：官方提供的 RID 你拿到了吗，还是自己假设的？
    - 用户拿不出官方技术支持提供的 RID 时，按 [硬性门槛](#硬性门槛没有官方-rid-就停) 停止集成，让用户先联系项目接入 QQ 1751971337
    - 是否已经上传过至少一个版本，上传时使用的 `channel`、`os`、`arch` 是什么。这些以官方提供的资源包信息为准，不要根据仓库自己推断

运行时（情况 B 的应用内）：收到 8001 时按 README 提示「当前平台暂无可用更新」，并回退到其他更新源。
