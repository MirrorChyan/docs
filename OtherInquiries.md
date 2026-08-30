# 更多问题

## 注意事项

> [!WARNING]
> 设置仓库的MirrorChyanUploadToken时,请注意显示空白是正常的安全设计.  
> 点击 `Update secret` 按钮后会替换值,请保存好获得的MirrorChyanUploadToken. 

## 问题

### Q&A 1

**Q：** 在 `mirrorchyan_release.yml` 工作流增加新架构时,仓库 Action 显示工作流已经上传到 Mirror酱,但是 Mirror酱官网架构没有增加

**A：** 这个需要联系技术支持增加上传的架构

### Q&A 2

**Q：** 想要增加MaaFwApp,mxu,mfa等UI,直接上传显示增量更新问题

**A：** 这个需要联系技术支持获取一个新的rid,用于上传不同UI项目

### Q&A 3

**Q：** 怎么设置仓库的MirrorChyanUploadToken

**A：** 可以在github网页仓库上栏 `Settings` ->左栏 `Security and quality` -> `Actions secrets and variables` -> `Repository secrets`编辑或者使用命令行工具

### Q&A 4

**Q：** 设置仓库的MirrorChyanUploadToken时,已经有MIRRORCHYANUPLOADTOKEN没法创建MirrorChyanUploadToken

**A：** 因为防混淆机制,无法创建只有大小写不一样的secrets
