# Git Pack 源码热更新接入

[返回首页](./README.md) · [客户端集成](./Integration.md) · [普通 ZIP 增量更新](./Incremental.md)

**这是面向少数项目的特殊更新方式，集成复杂，一般不推荐使用。绝大多数项目应继续使用普通 ZIP 增量更新或完整安装包分发。** 只有用户安装目录本身是 Git 仓库、更新器需要沿分支拉取源码并维护 Git 状态时，才考虑 Git Pack。

## 我的项目需要这种方式吗？

| 项目的实际更新方式 | 应选择的方式 |
| --- | --- |
| 代码托管在 GitHub，开发时用 Git，用户下载 Release ZIP / 安装包 | 普通接入；使用 Git 开发不代表客户端需要 Git Pack |
| 通过 MXU、MFAAvalonia 等 GUI 更新资源包 | 按 GUI 的现有接入方案，通常使用普通 ZIP 增量更新 |
| 更新时下载 ZIP，解压覆盖程序或资源 | [普通 ZIP 增量更新](./Incremental.md) |
| 更新时下载安装 APK、EXE 等完整安装包 | [完整包下载安装](./Integration.md#下载与安装) |
| 用户本地保留源码 Git 仓库，更新器调用 `git fetch` / pygit2，再按分支检出或重置源码 | 可评估本页方案；需要开发者改造更新器 |

**不确定时，先按普通接入沟通即可，不需要为了接入 Mirror 酱把项目改成 Git 仓库更新。** 将仓库地址和现有更新方式发给[项目接入](./README.md#联系我们)，我们会一起判断。

这类项目如果直接用普通 ZIP 增量包覆盖工作区，文件虽变了，Git 的 HEAD 和分支引用却没有更新，后续可能被判定为本地修改，或又被原来的 Git 更新流程覆盖。Git Pack 分发的是 Git 对象，客户端导入后还需更新引用、执行原有检出流程，才能保持源码和 Git 状态一致。

## 接入前需要确认什么？

源码热更新与启动器 / 运行时发版分开：

| 对比项 | 启动器 / 运行时发版 | Git Pack 源码热更新 |
| --- | --- | --- |
| 内容 | 启动器、Python 等运行时、完整安装产物 | 一个提交及其完整目录树、文件，不含父提交历史 |
| 触发 | 沿用 tag / Release 发版流程 | 跟踪分支的提交更新 |
| 资源 | 现有的发版 RID | 独立的 pack RID，不与发版包混用 |
| 版本名 | 项目原有版本，如 `v2.4.6` | 提交的 committer 时间，UTC 格式 `YYYYMMDDHHmmss` |

- **Mirror 酱接入方**确认 pack RID、跟踪分支及通道，适配并验证上传工作流，随接入 PR 提供这些信息。
- **项目开发者**配置上传 Token，改造客户端的版本计算、检查更新、对象导入、浅克隆边界和检出流程，并验证失败恢复。仅合并上传工作流不能完成客户端接入。
- **项目开发者**继续负责源码与运行时的兼容性判断、依赖安装、本地修改的处理策略及 Git 更新源回退策略。通用 GUI 对普通 ZIP 的支持不代表支持 `update_type=pack`。

## CI 上传

使用专用的 [git-pack-uploading-action](https://github.com/MirrorChyan/git-pack-uploading-action)，将跟踪分支当前提交上传为快照。它会收集这个提交及其完整目录树和文件，生成非 thin 的 `.pack`，不携带历史；浅检出 `fetch-depth: 1` 即可。

下面是跟踪 `main`、发布到 `stable` 的示例。实际 RID、分支、通道及触发方式以官方接入 PR 为准；将 `OWNER/REPO` 替换为上游仓库，`YOUR_PACK_RID` 替换为官方提供的 pack RID。Token 按[说明](./FAQ.md#如何设置仓库的-mirrorchyanuploadtoken)存入 `MirrorChyanUploadToken` secret。

```yaml
name: MirrorChyan Git Pack

on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read

concurrency:
  group: mirrorchyan-pack-stable
  cancel-in-progress: false

jobs:
  upload:
    if: ${{ github.repository == 'OWNER/REPO' }}
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          ref: main
          fetch-depth: 1
      - uses: MirrorChyan/git-pack-uploading-action@v1
        with:
          mirrorchyan_rid: YOUR_PACK_RID
          upload_token: ${{ secrets.MirrorChyanUploadToken }}
          branch: main
          channel: stable
```

- pack 源码资源不区分平台，上传和客户端请求均不填写 `os` / `arch`。RID 对应的更新类型需为 `pack`，由接入方案确认。
- `actions/checkout` 的 `ref` 和 Action 的 `branch` 必须一致。非 `push` 触发时也应明确这两项，避免手动运行上传了默认分支的源码。
- 保留 checkout 默认的 `persist-credentials: true`，私有仓库检查远程分支最新提交时会用到这份凭据。
- `concurrency` 串行处理同一上传流程。Action 只上传仍是 `origin` 上该分支最新提交的 HEAD，旧任务重跑时会跳过过期提交。
- 用 `GITHUB_TOKEN` 推送或合并产生的提交不会再次触发 `push`；有这类自动提交时，由接入方案补充 `workflow_run` 或 `schedule`。
- Action 默认自动生成更新日志；其他参数和输出见 [Action 文档](https://github.com/MirrorChyan/git-pack-uploading-action#输入)。

分支与 `stable` / `beta` / `alpha` 的映射在 CI 中设置。同一个版本只属于一个通道：再次上传到更稳定的通道时会移动过去，上传到更不稳定的通道时保持原通道。不要依靠把同一提交重复上传到多个通道来维护互相独立的版本。

## 客户端版本与请求

### 从 HEAD 计算当前版本

CI、服务端和客户端使用同一规则：读取 **committer 时间**的 Unix 时间戳，转换为 **UTC**，格式化为 `YYYYMMDDHHmmss`，例如 `20261116083000`。不能使用 author 时间、本地时区、Release tag 或短 SHA。

使用 git 命令行的 Python 示例：

```python
import subprocess
from datetime import datetime, timezone

raw = subprocess.run(
    ["git", "cat-file", "commit", "HEAD"],
    check=True, capture_output=True, text=True, encoding="utf-8",
).stdout
header = raw.split("\n\n", 1)[0]
committer = next(line for line in header.splitlines() if line.startswith("committer "))
timestamp = int(committer.rsplit(" ", 2)[1])
current_version = datetime.fromtimestamp(timestamp, timezone.utc).strftime("%Y%m%d%H%M%S")
```

使用 pygit2 时，从 HEAD 的 commit 对象读取 `commit_time`，同样用 `datetime.fromtimestamp(commit.commit_time, timezone.utc).strftime("%Y%m%d%H%M%S")`。

读取提交信息使用 `git cat-file commit`，避免用 `git log` / `git show` 计算版本：快照不带父提交，在写入 shallow 边界前，遍历历史的命令可能失败。

版本名精确到秒；同一秒的不同提交会重名，服务端会拒绝用另一个提交覆盖已有版本。遇到这种上传冲突应检查提交时间，不能随意改上传版本名绕过校验。

### 请求最新版本

复用现有 `/latest` 接口；下例中的 RID 和 CDK 需替换为实际值，不传 `os` / `arch`：

```http
GET https://mirrorchyan.com/api/resources/YOUR_PACK_RID/latest?current_version=20261116083000&channel=stable&cdk=YOUR_CDK
```

当前还没有仓库 / HEAD，或需要完整快照时，省略或留空 `current_version`。未填写 CDK 时仍可检查版本；下载需要适用于该资源的有效 CDK。HTTP 状态、业务码、下载地址时效和 SHA-256 校验沿用[客户端集成](./Integration.md#api-接口)。

**pack 只比较版本名是否相等，不使用“大于当前版本才更新”的规则。** 分支 force-push 回到旧提交并重新上传后，旧时间戳也可能是最新发布版本；客户端应允许跟随这个回退。同一提交可重新发布，不能按时间戳大小推断服务端当前跟踪的分支状态。

pack 资源的全量和增量响应都使用 `update_type: "pack"`，目标提交在下载包的 `meta.json` 中，不在 API 中另加 commit 字段。当前版本未知、没有可用增量包或增量包较大时会返回完整快照，客户端必须兼容。

## 下载包格式与应用步骤

下载结果是一个 ZIP，包含：

```text
pack-<sha>.pack
pack-<sha>.idx
meta.json
```

`meta.json` 的 `commit` 是目标提交的完整对象 ID；`base` 是增量包对应的基准提交对象 ID，完整快照则为空字符串。完整快照示例：

```json
{
  "commit": "<目标提交的完整对象 ID>",
  "base": ""
}
```

这里的 ZIP 是 Git 对象的容器，**不能解压到程序目录覆盖文件，也不使用 `changes.json`**。增量包省略本地基准提交已具备的对象；pack 为非 thin pack，但增量包仍需配合对应基准仓库使用。

可以直接从当前版本更新到目标版本，无需逐个下载中间版本；完整快照和增量包使用相同的客户端应用流程。

### 通用应用流程

1. **检查本地状态。** 按项目原有策略处理本地修改，记录更新前 HEAD；让自动检查和实际更新共用同一套更新源设置，避免选择 Mirror 酱后仍定时执行上游 `git fetch`。
2. **下载并校验。** 使用本次响应中的 `sha256` 校验整个 ZIP，解压到临时目录，写入前校验归档路径，读取并校验 `meta.json` 的对象 ID。
3. **核对基准。** `meta.base` 非空时，必须等于当前 HEAD 的完整对象 ID；应用前再次确认 HEAD 没有变化。不匹配时丢弃该增量包，不带 `current_version` 重新请求完整快照。
4. **导入对象。** 没有仓库时先 `git init`。用 `git index-pack --fsck-objects --stdin` 将 `.pack` 的二进制内容通过标准输入导入，它会生成索引；不要使用要求父提交连通的 `--strict`。也可将成对的 `.pack` / `.idx` 导入 Git 对象目录。
5. **验证目标。** 用 `git rev-list --objects --no-walk <meta.commit>` 确认目标提交的树和文件完整；用目标提交计算的 UTC 版本名应与 API 的 `version_name` 一致。校验失败时停止，保留现有引用和工作区，通过完整快照重试。
6. **维护浅克隆边界。** 读取 `git cat-file commit <meta.commit>` 的 `parent` 行；有任一父提交本地缺失时，将目标提交加入 shallow 文件。路径通过 `git rev-parse --git-path shallow` 获取，保留已有边界并安全写回；父提交都在时无需新增边界。快照更新可能跳过中间提交，本地历史因此会截断。
7. **接回项目的检出流程。** 先完成源码与当前运行时的兼容性检查，再更新跟踪引用（例如 `git update-ref refs/remotes/origin/main <meta.commit>`），通过已有的 checkout / reset 流程更新工作区，并按需要同步依赖。`reset --hard` 会丢弃本地修改，必须遵守第 1 步确定的项目策略。
8. **确认成功。** HEAD、工作区和依赖均符合目标版本后，再向用户报告完成；再次检查应不再提示同一版本。中断时按项目已有的备份 / 回滚机制恢复，不能仅写入一个新版本号就当作更新成功。

### 使用 pygit2 的客户端

可以沿用项目现有的临时 bare 仓库导入流程：将 `.pack` / `.idx` 放入临时仓库的 `objects/pack/`，通过 `objects/info/alternates` 指向正式仓库的对象目录，以便读取增量包省略的已有对象；创建指向 `meta.commit` 的分支引用，复制已有 shallow 边界，父提交不全时再加入目标提交。

校验后交给项目原有的本地仓库导入逻辑，导入对象并同步 shallow，再执行兼容性检查、检出和依赖同步；失败时回滚。必须确认目标对象已进入正式仓库，再删除临时仓库，不能留下依赖已删除临时目录的 alternates。

## 与现有发布和失败处理配合

- **运行时兼容性由客户端负责。** 源码更新不包含启动器或运行时升级；不兼容时保留当前代码，提示先更新发版资源。依赖是否需要重新安装由项目现有机制决定。
- **首次安装仍需单独设计。** 完整快照可初始化源码仓库，但 `/latest` 只提供最新版本；若首次启动必须检出与内置运行时匹配的指定 tag，仍需保留原有获取方式。
- **发版后可继续热更新。** 发版所带 HEAD 若已作为 pack 版本上传，可用它计算当前版本并请求后续更新；服务端不认识该版本时会返回完整快照。
- **错误应清楚可见。** 尤其要区分 CDK 错误 7001–7005、下载失败、基准不匹配和 Git 导入 / 检出失败，见[错误代码表](./ErrorCode.md)。是否自动退回原 Git 源由项目决定；回退时明确显示实际更新源。
- **更新记录需适配浅历史。** 不能假设 `git log` 能跨过所有热更新版本；需要连续更新记录时，另行保存更新前后的提交信息或专用引用，并设置保留上限。
- **不支持子模块的自动更新。** 子模块对象不在该仓库的快照中。Git LFS 的外置内容及仓库外运行依赖也需单独安排分发，不能认为它们会随 pack 一起下载。

## 上线前验证

- [ ] CI 和客户端从同一提交算出相同的 UTC 版本名；不把 Release 版本作为 pack 的当前版本。
- [ ] 普通 push 能上传；旧任务重跑会跳过过期提交；自动提交触发方式符合项目配置。
- [ ] 完整快照和匹配基准的增量包均能应用，包含新增、修改、删除文件的更新结果正确。
- [ ] `meta.base` 与 HEAD 不匹配时改取完整快照，应用前 HEAD 改变时不会误用旧增量包。
- [ ] 缺少父提交时能维护 shallow，后续 Git 检查、更新及项目需要的历史展示仍可用。
- [ ] 分支回退后能跟随旧时间戳的目标提交；更新成功后不重复提示。
- [ ] 本地修改、运行时不兼容、依赖同步失败、下载损坏和更新中断都有明确处理，能够恢复可运行版本。
- [ ] 不填 CDK、错误 CDK 和网络失败按项目约定处理，用户能看出实际使用的更新源；日志不含密钥或完整下载链接。

上传工作流验证通过只代表快照已分发；上线前仍需开发者完成客户端应用和恢复验证。
