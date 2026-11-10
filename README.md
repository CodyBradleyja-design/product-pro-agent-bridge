# Product Pro Agent Bridge

ChatGPT 与本地 WorkBuddy 的异步任务中转仓库。

<aside>
本项目当前使用 GitHub 文件轮询与 Markdown 报告，不是实时 RPC。仓库文件创建成功不代表 WorkBuddy 已经执行；本地轮询代理仍需在 WorkBuddy 所在电脑完成配置和测试。
</aside>

## 目录

- [通信协议](docs/COMMUNICATION_PROTOCOL.md)
- [任务收件箱](tasks/inbox/README.md)
- [执行完成区](tasks/done/README.md)
- [运行报告](reports/README.md)
- [本地配置说明](config/README.md)
- [任务卡模板](templates/TASK_TEMPLATE.md)
- [执行报告模板](templates/REPORT_TEMPLATE.md)

## 通信流程

1. ChatGPT 将任务 Markdown 文件写入 `tasks/inbox/`。
2. 本地 WorkBuddy 代理按 10–30 分钟间隔轮询。
3. WorkBuddy 按任务范围执行，并将结果写入 `tasks/done/`。
4. 定时状态与同步摘要写入 `reports/`。
5. ChatGPT 读取报告并推进下一步。

## 当前状态

- [x] 私有仓库已建立。
- [x] 通信协议、目录说明和模板已提交。
- [ ] 本地 WorkBuddy 轮询代理尚未接入。
- [ ] 端到端测试尚未完成。

## 安全

绝不提交 Token、密码、Cookie、私钥或个人聊天记录。高影响操作必须先经用户明确批准。详见通信协议。
