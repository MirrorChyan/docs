# docs

Mirror酱集成文档

## 接口介绍

请参考 [API 详情](https://apifox.com/apidoc/shared-ffdc8453-597d-4ba6-bd3c-5e375c10c789/253583257e0)。

举例来说，您需要 `GET` 请求 `https://mirrorchyan.com/api/resources/{res_id}/latest?current_version=v0.0.1&cdk=xxxx&user_agent=xxxx`，其中：

- `{res_id}` 设置为您要更新的资源 ID，必选。例如 `M9A`, `MaaResource` 等，具体请联系 Mirror 酱技术支持人员。
- `current_version` 设置为当前用户本地的资源版本号，推荐必选。
- `cdk` 设置为用户提供的 cdk。可选。若用户未提供 cdk，也可使用该 API 来检查更新，但无法进一步进行下载。即单独作为检查更新 API 来使用是免费的。
- `user_agent` 设置为 UI 的名称。例如 `MFA`, `MAA_WPF` 等，由您自行决定，后续保持不变即可。

响应举例：

```json
{
  "code": 0,
  "msg": "success",
  "data": {
    "version_name": "v1.1.0",
    "url": "https://mirrorchyan.com/resources/download/xxxxxx"
  }
}
```

在请求时，若 cdk 正确，我们会在响应中提供一次性下载链接，`data.url` 字段，直接调用下载文件即可。

若未提供 cdk，则 `data.url` 字段为空，但 `version.name` 始终有值，您可借此来提示用户有更新，即上文提到的单独作为检查更新 API 来使用是免费的。  
由于用户侧的 github 访问可能受限，我们推荐您优先使用 Mirror 酱来检查是否有更新。至于后续是自动从 github 下载、跳转 github 网页、亦或是提示用户购买 Mirror 酱，可根据您的项目实际情况选择。

详细的请求及响应字段请参考：[API 详情](https://apifox.com/apidoc/shared-ffdc8453-597d-4ba6-bd3c-5e375c10c789/253583257e0)。

响应中 `code` 为 0 表示成功，其他情况可参照 [ErrorCode.md](./ErrorCode.md) 显示错误信息，或直接显示一下 `msg` 字段信息。  

下载的增量包中，会额外包含一个 `changes.json` 文件，包含文件变动信息。详情请参考 [Incremental.md](./Incremental.md)，推荐关注其中 `deleted` 字段，删除新版本不再需要的文件。

## 经典流程

1. 在您的软件设置中加入输入框，可输入 cdk；旁边添加 `Mirror酱` 网页跳转链接：`https://mirrorchyan.com`。
2. 若用户未输入 cdk，仍可用 Mirror 酱检查是否有更新。若有更新，提示用户：“检测到更新，正在使用 github 源下载中，可进入设置切换 Mirror酱 源以高速下载”。具体文案和后续行为由您自行决定。
3. 若用户已输入 cdk，则使用响应中的链接下载增量更新包，并完成更新。
4. 我们会每月根据用户使用情况，向 res_id 方（资源作者）及 user_agent 方（UI 作者）分别共享收益，具体请进群详聊！

*请考虑日志、配置文件等中，尽量不要出现 CDK 明文，避免意外泄漏。*

## 上传资源

现阶段由 Mirror酱 开发组为您提供技术支持，我们将为您的项目 PR 一个自动资源上传 CI，会在每次版本发布的时候全自动上传，也可根据您的需求改为每次有新提交则自动上传或其他方式~

## 联系我们

QQ 群 1026040805，您有任何集成开发问题，或合作意向，欢迎加群详谈！
