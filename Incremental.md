# 增量包介绍

## `changes.json`

### 介绍

Mirror 酱会为增量包提供不同版本之间的 `changes.json` 文件，用于描述两个版本之间的差异。

### 字段说明

| 字段 | 类型 | 说明 |
|------|------|------|
| `added` | `string[]` | 新增的文件列表 |
| `modified` | `string[]` | 内容发生变更的文件列表 |
| `deleted` | `string[]` | 被删除的文件列表 |
| `added_dir` | `string[]` | 新增的目录列表 |
| `deleted_dir` | `string[]` | 被删除的目录列表 |

所有路径均使用 `/` 作为分隔符，为相对于包根目录的相对路径。

当某个类别没有变更时，对应字段不会出现在 JSON 中。

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
