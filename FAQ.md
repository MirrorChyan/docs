# 常见问题

## 如何设置仓库的 MirrorChyanUploadToken？

在 GitHub 仓库页面依次进入 `Settings` → `Secrets and variables` → `Actions`，点击 `New repository secret`，名称填写 `MirrorChyanUploadToken`，值填写从 Mirror酱 获得的 Token。

也可以使用 [GitHub CLI](https://cli.github.com/)：`gh secret set MirrorChyanUploadToken`

不要把 Token 写入仓库文件，也不要提交到 workflow 里。

> [!NOTE]
> GitHub 会将 secret 名称统一保存为大写，保存后列表中显示为 `MIRRORCHYANUPLOADTOKEN` 属于正常现象。工作流中通过 `secrets.MirrorChyanUploadToken` 引用时不区分大小写，可以正常读取。

> [!WARNING]
> 出于安全设计，编辑已有 secret 时输入框为空，点击 `Update secret` 会直接覆盖原值，请自行妥善保存 Token。

## 已有 MIRRORCHYANUPLOADTOKEN，无法再创建 MirrorChyanUploadToken？

两者是同一个 secret（名称不区分大小写），无需重新创建。如需更换 Token，直接编辑已有的 `MIRRORCHYANUPLOADTOKEN` 即可。

## 在 `mirrorchyan_release.yml` 工作流中新增架构后，Action 显示已上传到 Mirror酱，但官网上没有出现新架构？

新的系统或架构要先在 Mirror酱 侧为该资源登记。workflow 里增加架构并且 Action 显示上传成功，并不表示官网已经提供这个平台。请 [联系我们](./README.md#联系我们) 添加。上传前的约定见 [自动化上传](./README.md#自动化上传)。

## 同一项目想同时发布 [MaaFwApp](https://github.com/Aliothmoon/MaaFwApp)、[MXU](https://github.com/MistEO/MXU)、[MFAAvalonia](https://github.com/MaaXYZ/MFAAvalonia) 等不同 UI 的版本，直接上传后出现增量更新问题？

不同 GUI 的文件树不一样，共用一个资源 ID 时，增量包会按错误的差异去增删文件。每个 GUI 使用独立的资源 ID（`res_id`）。请 [联系我们](./README.md#联系我们) 获取新的资源 ID。

MaaFwApp 可在 `pi-profile.yaml` 的 `update.mirrorchyanRid` 中单独指定，覆盖 `interface.json` 里的值。
