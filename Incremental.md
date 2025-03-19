# 增量包介绍

## `changes.json`

### 介绍

Mirror 酱会为增量包提供不同版本之间的 `changes.json` 文件，用于描述两个版本之间的差异。

### 示例

```json
{
    "added": {
        "foo/a.png",
        "foo/b.png"
    },
    "deleted": {
        "foo/c.png"
    },
    "modified": {
        "resource/config.json"
    }
}

```
