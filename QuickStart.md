# 快速开始

[返回首页](./README.md) · [客户端集成](./Integration.md) · [常见问题](./FAQ.md)

本页帮助你选定接入路径，完成第一次版本检查。通用 GUI 项目可直接使用内置更新能力；自研客户端先验证 API，再接入应用。

## 开始前准备

联系**项目接入 QQ：1751971337**，提供[项目信息](./README.md#联系我们)，由官方确认：

| 信息 | 用途 |
| --- | --- |
| 官方分配的 RID | API 的 `res_id`，或 GUI 的 `mirrorchyan_rid` |
| 资源包是否区分系统、架构和更新通道 | 决定请求是否携带 `os`、`arch`、`channel` |
| 首个版本是否已上传并可用 | 没有可用版本时，正确的 RID 也可能返回 404 |
| 官方提供的自动化上传方案 | 后续发布时自动分发更新包 |

RID 不能根据仓库名、项目名或其他项目的示例猜测。资源包划分和上传 CI/CD 均以官方方案为准；还没拿到时，先完成这一步。

**验证版本检查不需要 CDK。** 验证实际下载时，再准备适用于该资源的有效 CDK。上传 Token 仅用于 CI，不能作为客户端 CDK 使用。

## 通用 GUI 项目

适用于基于 MaaFramework、使用 [MXU](https://github.com/MistEO/MXU)、[MFAAvalonia](https://github.com/MaaXYZ/MFAAvalonia)、[MaaFwApp](https://github.com/Aliothmoon/MaaFwApp) 等通用 GUI 的项目。

这些 GUI 已内置 Mirror酱 的 CDK 设置、版本检查和下载安装能力，通常只需在项目已有的 `interface.json` 中配置以下字段：

| 字段 | 填写方式 |
| --- | --- |
| `mirrorchyan_rid` | 官方提供的 RID |
| `mirrorchyan_multiplatform` | 按官方上传方案填写：系统 / 架构分别上传时为 `true`；共用一个包时为 `false` 或不填 |
| `version` | 当前资源版本，与发布 tag 和上传到 Mirror酱 的版本名保持一致 |
| `github` | 项目的 GitHub 仓库地址 |

字段定义以 [ProjectInterface V2 协议](https://github.com/MaaXYZ/MaaFramework/blob/main/docs/zh_cn/3.3-ProjectInterfaceV2协议.md)和对应 GUI 文档为准。请修改已有配置，保留项目其他字段。

配置后，在 GUI 中检查以下行为：

1. 使用一个真实的旧版本检查更新，能看到目标版本。
2. 使用适用的有效 CDK，能下载并完成更新。
3. 更新后再次检查，不重复提示同一版本。
4. 不填 CDK 或网络失败时，现有更新入口仍可使用。

返回 404 时，先核对 `mirrorchyan_multiplatform` 与实际上传方式，见[排查步骤](./FAQ.md#返回-404code-8001怎么排查)。

同时发布多个 GUI 时，每个 GUI 使用独立 RID，见[多 GUI 发布](./FAQ.md#同时发布多个-gui需要几个-rid)。按协议约定，通用 GUI 更新的是资源版本，UI 本体和 MaaFramework 版本由项目发版时打包。

完成验证后，按官方方案配置[自动化上传](./README.md#自动化上传)，无需再实现下面的自研客户端流程。

## 自研客户端

### 1. 发起第一次请求

取得官方资源信息、确认首个版本已可用后，在 Bash 中运行以下示例（Windows 可使用 Git Bash）。按提示输入官方提供的值；不区分系统或架构的资源，对应项留空。

```bash
read -r -p "官方分配的 RID: " MIRROR_RID
read -r -p "本地当前版本号: " MIRROR_VERSION
read -r -p "官方约定的 os（不区分则留空）: " MIRROR_OS
read -r -p "官方约定的 arch（不区分则留空）: " MIRROR_ARCH
read -r -p "更新通道（stable / beta / alpha，留空为 stable）: " MIRROR_CHANNEL

curl --get --silent --show-error --max-time 15 \
  "https://mirrorchyan.com/api/resources/${MIRROR_RID}/latest" \
  --data-urlencode "current_version=${MIRROR_VERSION}" \
  --data-urlencode "os=${MIRROR_OS}" \
  --data-urlencode "arch=${MIRROR_ARCH}" \
  --data-urlencode "channel=${MIRROR_CHANNEL:-stable}" \
  --data-urlencode "user_agent=myapp_desktop"
```

该请求不携带 CDK，也不会下载更新包。空的 `os` / `arch` 与不传对应参数等效；按平台上传的资源必须填写官方约定的系统和架构。

### 2. 确认请求成功

成功响应的主要字段示例（为便于阅读省略其他字段，实际版本号和日志以你的项目为准）：

```json
{
  "code": 0,
  "msg": "current resource latest version is v1.1.0",
  "data": {
    "version_name": "v1.1.0",
    "release_note": "新增功能与修复说明"
  }
}
```

| 看到的结果 | 下一步 |
| --- | --- |
| `code` 为 `0`，且有 `data.version_name` | 版本检查已跑通，与项目实际发布版本核对 |
| 没有 `url` | 本示例未填写 CDK，属于预期行为；不影响检查版本 |
| HTTP 404，`code` 为 `8001` | 核对 RID、首个包是否可用、平台及通道，见[排查步骤](./FAQ.md#返回-404code-8001怎么排查) |
| 其他错误或网络失败 | 查看[错误代码表](./ErrorCode.md)，记录不含密钥的请求参数 |

### 3. 接入应用并验证下载

第一次请求成功后，按[客户端集成](./Integration.md)完成 CDK 设置、版本比较、下载校验和更新安装。使用真实旧版本及适用的有效 CDK 验证下载，不要把 CDK 写进示例代码或提交到仓库。

先跑通更新流程，再按需添加[来源统计](./README.md#来源统计)。上线前完成[验收清单](./Integration.md#上线前验收)，后续发版使用官方提供的上传方案。
