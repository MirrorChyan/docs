# 增量包介绍

`current_version` 与上传时的版本名格式一致，且服务端能算出差异时，返回增量包（响应字段 `update_type` 为 `incremental`）。包内包含 `changes.json`。版本字符串对不上，或无法计算差异时，返回全量包，包内没有该文件。

## 客户端如何安装

1. 将更新包解压到临时目录，不要直接解压到安装目录。
2. 若临时目录中存在 `changes.json`，先在安装目录删除 `deleted` 列出的文件，以及 `deleted_dir` 列出的目录（含其中的文件）。
3. 再用临时目录中的文件覆盖安装目录。
4. 程序本身正在运行时，先将旧文件重命名，写入新文件，下次启动再删除旧文件。

压缩包条目和 `changes.json` 中的每个路径都必须解析到安装目录之内。拒绝 `..` 和绝对路径。某个路径在安装目录中已经不存在时，跳过即可。

## `changes.json`

Mirror酱会在增量包中提供 `changes.json`，描述两个版本之间的差异。

### 字段说明

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `added` | `string[]` | 新增的文件列表 |
| `modified` | `string[]` | 内容发生变更的文件列表 |
| `deleted` | `string[]` | 被删除的文件列表 |
| `added_dir` | `string[]` | 新增的目录列表 |
| `deleted_dir` | `string[]` | 被删除的目录列表 |

所有路径均使用 `/` 作为分隔符，为相对于包根目录的相对路径。

当某个类别没有变更时，对应字段不会出现在 JSON 中。`added`、`modified`、`added_dir` 描述差异；这些路径上的新内容在压缩包内，覆盖安装目录即可。客户端必须处理的是 `deleted` 和 `deleted_dir`。

### 示例

```json
{
  "added": [
    "foo/a.png",
    "foo/b.png"
  ],
  "deleted": [
    "bar/c.png"
  ],
  "modified": [
    "resource/config.json"
  ],
  "added_dir": [
    "foo"
  ],
  "deleted_dir": [
    "bar"
  ]
}
```
