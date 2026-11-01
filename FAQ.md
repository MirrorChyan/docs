# 常见问题

[返回首页](./README.md) · [快速开始](./QuickStart.md) · [错误代码表](./ErrorCode.md)

## 接入与使用

### 还没拿到 RID，怎么开始？

把仓库地址和希望分发的 Release 包发给**项目接入 QQ：1751971337** 即可，可使用[沟通模板](./README.md#联系我们)。我们会确认 RID、资源包划分，适配并验证上传工作流后提交接入 PR，无需先自己填写 RID 或编写 CI。

RID 是官方分配的资源 ID，不一定等于项目名或仓库名。拿到官方信息后，再配置客户端或请求接口。

### 使用通用 GUI，还需要自己写更新器吗？

使用已支持 Mirror酱 的 MXU、MFAAvalonia 等 GUI 时，直接复用其内置能力，我们会在接入 PR 中补充[适用的配置](./QuickStart.md#通用-gui-项目)。具体支持范围、配置入口和更新包形态以使用的 GUI 版本及其文档为准；自研客户端才需要实现[版本检查与下载安装](./Integration.md)。

### 不填 CDK，也能检查更新吗？

可以。不带 `cdk` 请求版本接口，仍可获得最新版本信息；有更新时，可通过项目原有下载源下载。Mirror酱 的加速下载需要适用于该资源的有效 CDK。

CDK 是用户下载凭证；`MirrorChyanUploadToken` 是开发者上传凭证，两者不能混用。接入不会要求你移除 GitHub 等已有下载渠道。

### 接入费用和合作方式如何确认？

项目接入与合作方式请联系项目接入 QQ，根据项目实际情况确认。用户侧的 CDK 方案以 [Mirror酱官网](https://mirrorchyan.com)当前说明为准。

## 版本检查与下载

### 返回 404（code 8001）怎么排查？

以下情况都可能返回同一个错误，不能仅凭 404 认定 RID 错了：

1. RID 不存在或填写有误。
2. 已接入，但尚无可用版本。
3. 已上传版本，但请求的 `os` / `arch` / `channel` 组合下没有包。

按以下顺序核对：

1. 确认使用的是官方提供的 RID，且首个版本已上传并可用。
2. 对照接入 PR 或交付的 URL 示例检查 `os` / `arch`：按平台上传的包需要对应参数；不区分平台的包应不传或留空。使用 `mirrorchyan_multiplatform` 的 GUI 同时检查这一字段。
3. 确认请求通道有可用版本。默认 `stable` 只查正式版，测试版可能只在 `beta` / `alpha` 可见。
4. 若需诊断，可在不带 CDK 的情况下临时用 `channel=alpha` 检查，并仅对照官方提供的平台配置；不要把这个诊断设置直接用于正式用户。
5. 仍无法定位时，向项目接入 QQ 提供 RID、系统、架构、通道及上传记录，请官方核对，不要尝试猜测其他 RID。

从共用包迁移到按平台上传后，可能仍留有旧的共用包。如果不带平台参数能够成功但版本很旧，也需要核对当前的上传划分，而不是直接采用这一组参数。

### `code == 0`，为什么没有 `url`？

未填写 CDK、或当前版本与最新版本相同时，都可能不返回下载地址。先比较 `version_name`，再决定是否下载。没有 `url` 不表示没有更新，详见[响应处理](./Integration.md#成功响应)。

### 为什么传了 `current_version`，仍然拿到全量包？

对于支持增量更新的 ZIP 资源，该参数用于选择可用增量包，不保证一定返回增量包。核对本地版本名是否与上传时一致（包括 `v` 前缀），然后以响应中的 `update_type` 为准；没有可用增量包时应能正常安装全量包。

APK、EXE 等安装包按完整包分发，不使用 `current_version` 请求增量包，也不需要处理 `changes.json`。具体包型和 RID 以接入方案为准。

### 下载链接失效或下载失败怎么办？

下载地址有时效，不应持久缓存。先检查 HTTP 状态和可用的 JSON 错误体，按[错误码](./ErrorCode.md)处理；链接失效时重新请求版本接口。重复请求仍失败时，保留项目原有下载入口，并提供脱敏后的错误信息。

## 上传与多平台发布

### 收到接入 PR 后，需要做什么？

确认要分发的包及平台，在**自己的项目仓库**配置下面的上传 Token，再审阅并合并 PR。我们会在提交 PR 前实际运行上传工作流验证，你无需再手动上传首个包或重新设计 CI。

合并后按原有方式发版即可，上传包和同步更新日志的触发时机以 PR 说明为准。使用自研客户端时，API 集成仍由项目方完成；请求示例会随接入方案提供。

### 如何设置仓库的 MirrorChyanUploadToken？

收到接入 PR 和官方提供的上传 Token 后，在**你自己的项目仓库**中配置。无需将 Token 发到 PR 或交给 AI。

在 GitHub 仓库页面依次进入 `Settings` → `Secrets and variables` → `Actions`，点击 `New repository secret`，名称填写 `MirrorChyanUploadToken`，值填写从 Mirror酱 获得的 Token。

也可以在目标仓库目录中使用 [GitHub CLI](https://cli.github.com/)交互式设置，避免将 Token 放入命令参数：

```bash
gh secret set MirrorChyanUploadToken
```

GitHub 会将 secret 名称统一保存为大写，列表中显示 `MIRRORCHYANUPLOADTOKEN` 属于正常现象。工作流通过 `secrets.MirrorChyanUploadToken` 引用时不区分大小写，可以正常读取。

编辑已有 secret 时输入框不会显示原值，点击 `Update secret` 会覆盖原值。请妥善保存 Token，不要将其提交到仓库或粘贴到公开日志中。

### 已有 MIRRORCHYANUPLOADTOKEN，无法再创建 MirrorChyanUploadToken？

两者是同一个 secret，名称不区分大小写，无需重新创建。如需更换 Token，直接编辑已有的 `MIRRORCHYANUPLOADTOKEN` 即可。

### Action 上传成功，但新架构未出现？

在 `mirrorchyan_release.yml` 中增加架构，还需要技术支持在 Mirror酱 侧为资源添加对应架构。请[联系项目接入](./README.md#联系我们)，并核对上传和客户端请求的系统、架构是否一致。

### 同时发布多个 GUI，需要几个 RID？

同一项目同时发布 [MaaFwApp](https://github.com/Aliothmoon/MaaFwApp)、[MXU](https://github.com/MistEO/MXU)、[MFAAvalonia](https://github.com/MaaXYZ/MFAAvalonia) 等不同 GUI 的版本时，**每个 GUI 需要独立 RID**，避免不同包混用导致增量更新问题。请联系官方获取，不要自行命名后上传。

同一产品、同一用途的不同系统 / 架构可以共用 RID，通过 `os` / `arch` 区分；ZIP 与 APK、EXE 等安装包需要拆分 RID，GUI / CLI 等需独立选择的包也应分开。我们会按实际发布产物确定划分，并随接入方案说明对应关系。

### 接入后会自动加入官网项目列表吗？

加入或更新[官网项目一览](https://mirrorchyan.com/zh/projects)是可选的。希望展示时，我们会与你确认项目名称、简介、链接、支持平台和 Logo；接入分发服务不等于默认同意上架。

## 提问时需要提供什么？

开发接入问题联系**项目接入 QQ：1751971337**；用户下载、CDK 等售后问题可加入 **QQ 群：995458883**。

为便于定位，请提供项目地址、RID、客户端 / GUI 版本、本地与目标资源版本、`os` / `arch` / `channel`、HTTP 状态和 `code` / `msg`。上传问题可附上脱敏后的 Action 日志。

请去掉 CDK、上传 Token、完整请求 URL 和带时效的下载链接后再分享。
