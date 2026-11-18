# Git Pack 源码热更新接入

[返回首页](./README.md) · [客户端集成](./Integration.md) · [普通 ZIP 增量更新](./Incremental.md)

**这是面向少数项目的特殊更新方式，集成复杂，一般不推荐使用。绝大多数项目应继续使用普通 ZIP 增量更新或完整安装包分发。** 只有用户安装目录本身是 Git 仓库、更新器需要沿分支拉取源码并维护 Git 状态时，才考虑 Git Pack。

本指南面向已收到 Mirror 酱接入 PR 或交付说明的项目开发者，说明客户端如何检查、下载和应用源码更新。

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

## 开始前：使用官方交付的信息

从接入 PR 或交付说明取得源码 RID、跟踪分支、通道和请求示例，同时确认项目现有更新器使用的本地仓库与更新流程：

| 信息 | 客户端用途 |
| --- | --- |
| 源码 RID | 请求源码更新；它与启动器 / 运行时发行包的 RID 分开 |
| 跟踪分支与通道 | 确定要更新的本地分支及 `channel`；不根据仓库默认分支自行猜测 |
| 请求 URL 示例 | 核对 RID 和通道；源码不区分平台，不传 `os` / `arch` |
| 本地源码仓库与现有更新流程 | 确定 HEAD 来源、对象导入位置、检出、依赖同步和失败恢复方式 |

源码更新包包含一个提交及其完整目录树、文件，不含父提交历史。它不会升级启动器或运行时；这一层继续使用项目已有的发版更新方式。客户端不能把发行包的 RID 或 SemVer 版本号用作源码更新参数。

开发者需要在应用中提供更新源选择和 CDK 设置，并实现下文的版本检查、Git 对象导入、浅克隆边界维护及检出。CDK 应掩码显示并按项目现有方式保存，日志、配置导出和崩溃报告中需脱敏；上传 Token 不放入客户端。通用 GUI 对普通 ZIP 的支持不代表支持 `update_type=pack`。

## 客户端版本与请求

### 从 HEAD 计算当前版本

客户端从当前源码仓库 HEAD 读取 **committer 时间**的 Unix 时间戳，转换为 **UTC**，格式化为 `YYYYMMDDHHmmss`，例如 `20261116083000`。这应与同一提交已发布的版本名一致，不能使用 author 时间、本地时区、Release tag 或短 SHA。

使用 git 命令行的 Python 示例，应在待更新的源码仓库目录中执行：

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

### 请求最新版本

复用现有 `/latest` 接口；下例中的 RID、通道以官方交付为准，CDK 替换为用户自己的值，不传 `os` / `arch`：

```http
GET https://mirrorchyan.com/api/resources/YOUR_PACK_RID/latest?current_version=20261116083000&channel=stable&cdk=YOUR_CDK
```

当前还没有仓库 / HEAD，或需要完整快照时，省略或留空 `current_version`。未填写 CDK 时仍可检查版本；下载需要适用于该资源的有效 CDK。

同时检查 HTTP 状态和 JSON 的 `code`；成功时 `code` 为 `0`，从 `data` 读取：

| 字段 | 客户端处理 |
| --- | --- |
| `version_name` | 按下文规则与本地版本比较 |
| `release_note` | 展示本次更新日志 |
| `url` | 有更新且 CDK 有效时取得带时效的下载地址；支持重定向，过期后重新请求 |
| `sha256` | 校验本次返回的整个下载包，失败时丢弃，不能安装 |
| `filesize` | 下载大小（字节），可用于进度展示 |
| `update_type` | 本指南处理 `pack`，其他类型不进入 Git 对象应用流程 |

没有 `url` 不代表没有更新，应先比较 `version_name`；需要下载但没有地址时，提示填写有效 CDK，或按项目约定使用原 Git 源。请求应异步执行并设置超时；非 2xx 响应若是 JSON，也读取业务码，网络错误或解析失败不能当作“已是最新版本”。

**pack 只比较版本名是否相等，不使用“大于当前版本才更新”的规则。** 分支回退后，旧时间戳也可能是最新发布版本；客户端应允许跟随这个回退，不能按时间戳大小推断服务端当前跟踪的分支状态。

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

下列 Git 命令展示调用 git 命令行的实现路径。使用 pygit2 的客户端可按下一节通过对象 API 和原有导入流程完成等价操作，无需因此引入 git.exe。

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
- **发版后可继续热更新。** 使用发版所带源码的 HEAD 计算当前版本并请求后续更新；服务端不认识该版本时会返回完整快照。
- **错误应清楚可见。** 尤其要区分 CDK 错误 7001–7005、下载失败、基准不匹配和 Git 导入 / 检出失败，见[错误代码表](./ErrorCode.md)。是否自动退回原 Git 源由项目决定；回退时明确显示实际更新源。
- **更新记录需适配浅历史。** 不能假设 `git log` 能跨过所有热更新版本；需要连续更新记录时，另行保存更新前后的提交信息或专用引用，并设置保留上限。
- **不支持子模块的自动更新。** 子模块对象不在该仓库的快照中。Git LFS 的外置内容及仓库外运行依赖也需单独安排分发，不能认为它们会随 pack 一起下载。

## 上线前验证

- [ ] 客户端从源码 HEAD 算出的 UTC 版本名与该提交已发布的版本一致，不把 Release 版本作为 pack 的当前版本。
- [ ] 完整快照和匹配基准的增量包均能应用，包含新增、修改、删除文件的更新结果正确。
- [ ] `meta.base` 与 HEAD 不匹配时改取完整快照，应用前 HEAD 改变时不会误用旧增量包。
- [ ] 缺少父提交时能维护 shallow，后续 Git 检查、更新及项目需要的历史展示仍可用。
- [ ] 分支回退后能跟随旧时间戳的目标提交；更新成功后不重复提示。
- [ ] 本地修改、运行时不兼容、依赖同步失败、下载损坏和更新中断都有明确处理，能够恢复可运行版本。
- [ ] 不填 CDK、错误 CDK 和网络失败按项目约定处理，用户能看出实际使用的更新源；日志不含密钥或完整下载链接。

使用真实旧版本的源码仓库完成上述验证，确认更新后的 HEAD、工作区和运行依赖一致，应用可正常启动。
