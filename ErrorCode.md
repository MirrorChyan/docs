# Error Code

## 定义

| code | description |
| --- | --- |
| = 0 | 请求成功 |
| < 0 | 意料之外的严重错误，请及时联系 Mirror 酱的技术支持处理 |
| > 0 | 业务逻辑错误，根据具体错误码判断 |

## 业务逻辑错误码

| status | code | details |
| -------- | ------- | ------- |
| 400 | 1001 | INVALID_PARAMS |
| 400 | 7001 | KEY_EXPIRED |
| 403 | 7002 | KEY_INVALID |
| 403 | 7003 | RESOURCE_QUOTA_EXHAUSTED |
| 403 | 7004 | KEY_MISMATCHED |
| 404 | 8001 | RESOURCE_NOT_FOUND |
| 400 | 8002 | INVALID_OS |
| 400 | 8003 | INVALID_ARCH |
| 400 | 8004 | INVALID_CHANNEL |

> 若遇到了不在表上的错误码，请及时联系请联系 Mirror 酱的技术支持。
