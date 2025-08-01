# Error Code

## 定义

| code | description |
| --- | --- |
| = 0 | 请求成功 |
| < 0 | 意料之外的严重错误，请及时联系 Mirror 酱的技术支持处理 |
| > 0 | 业务逻辑错误，根据具体错误码判断 |

## 业务逻辑错误码



| status | code | details                  | remark                    |
| ------ | ---- | ------------------------ | ------------------------- |
| 400    | 1001 | INVALID_PARAMS           | 参数不正确，请参考集成文档             |
| 403    | 7001 | KEY_EXPIRED              | CDK 已过期                    |
| 403    | 7002 | KEY_INVALID              | CDK 错误                     |
| 403    | 7003 | RESOURCE_QUOTA_EXHAUSTED | CDK 今日下载次数已达上限            |
| 403    | 7004 | KEY_MISMATCHED           | CDK 类型和待下载的资源不匹配          |
| 403    | 7005 | KEY_BLOCKED              | CDK 已被封禁                    |
| 404    | 8001 | RESOURCE_NOT_FOUND       | 对应架构和系统下的资源不存在            |
| 400    | 8002 | INVALID_OS               | 错误的系统参数                   |
| 400    | 8003 | INVALID_ARCH             | 错误的架构参数                   |
| 400    | 8004 | INVALID_CHANNEL          | 错误的更新通道参数                 |
| -      | 1    | UNDIVIDED                | 未区分的业务错误，以响应体 JSON 的 msg 为准 |



> 若遇到了不在表上的错误码，请及时联系 Mirror 酱的技术支持。
