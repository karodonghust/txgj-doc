# Schools

返回[唯一事实源索引](README.md)。

## BR-051 学校目录与数据变更

状态：`confirmed`

- 学校不在目录时，Advisor 可建立 provisional School，并标记未验证。
- 学校资料变更通过 ChangeRequest 提交，不能直接改写不可变爬虫快照。
- Release 1 的普通字段、学校身份、合并、拆分、停用和主要官网等所有学校资料变更均由 Founder 审核。
- 提交者不能审核自己。
- 批准修改以 overlay revision 生效，错误修改通过停用新 revision 回滚，不改写历史。
- 候选发布存在 warning 时不能自动接受或发布，必须由 Founder 对精确 manifest 一次性批准。
- 爬虫发布、复制快照、Git 发布和应用部署是独立动作，不能相互推导授权。
