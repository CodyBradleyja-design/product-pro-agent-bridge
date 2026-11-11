# T20261111-002 轮询速度与即时触发可行性核查

- 任务编号：T20261111-002
- 核查时间：2026-11-11 09:53–09:58 (GMT+8, Asia/Shanghai)
- 执行方：WorkBuddy 本地代理（agent-bridge）
- 性质：只读核查，未修改配置、未启动新服务、未安装软件、未读取客户数据

## 一、实际运行状态（配置 vs 证据）

| 项目 | 配置值 | 实测证据 | 结论 |
|---|---|---|---|
| 轮询间隔 | 10 分钟（`bridge_config.json: poll_interval_minutes`） | `logs/poller-20261111.log` 无 10 分钟节律记录；无 `state/poller.pid`；`tasklist` 无 python.exe | **配置未生效**（常驻未运行） |
| 执行模式 | `queue`（只入队不执行） | 日志 `poller start mode=queue` | 已验证 |
| 实际驱动源 | — | WorkBuddy 定时任务「agent-bridge 任务执行循环」，`FREQ=HOURLY;INTERVAL=1` | 实际有效延迟 **≤60 分钟** |
| 最近一次成功领取 | — | 09:52:36 起轮，09:52:37–09:52:38 领取 3 个新任务（002 / 003 / 004） | 已验证 |
| 最近一次成功回写 | — | 09:52:43、09:52:46、09:52:48 三条 `tasks/done/*` 推送成功；心跳 `reports/heartbeat-20261111-0952.md` | 已验证 |
| 此前成功回写 | — | 08:32:24（T20261110-001）、08:38:07 与 08:38:46（T20261111-001） | 已验证 |
| 自启注册 | 启动夹 `agent-bridge-autostart.vbs` 存在 | 文件存在（2026-11-11 00:03），但本次登录会话内 poller 未存活 | 注册存在、**未复活** |

关键判断：**当前链路由「每小时一次的 WorkBuddy 定时任务」驱动，不是由 10 分钟常驻轮询驱动**。10 分钟是纸面配置。

## 二、缩短间隔的可行性

| 途径 | 最小粒度 | 可行性 | 成本 |
|---|---|---|---|
| 调小 `poll_interval_minutes` | 任意 | 仅对常驻 poller 有效 | 无 |
| WorkBuddy 定时任务 | **1 小时**（`FREQ=HOURLY;INTERVAL=n`，不支持分钟级） | 已到下限 | 每轮起一次模型会话，NO_TASK 也要付一次上下文 token |
| 常驻 `poller.py`（`start.bat` / 自启） | 秒级 | 可行，但受「AI 会话拉起的进程会被进程树回收」限制，必须人工双击或登录自启 | 脚本成本≈0 |
| Windows 计划任务 | 分钟级 | **不可用**：本机 `schtasks.exe` 在程序黑名单、`Register-ScheduledTask` 拒绝访问 | — |

成本口径需分开看：
- **脚本轮询**只打 GitHub Contents API：10 分钟一轮 = 144 次/h 的上限下约 300 请求/h，fine-grained PAT 限额 5000/h，可忽略；不消耗模型 token。
- **模型调用**只在真正执行任务卡时发生。把间隔压到分钟级 ≠ 成本上涨，前提是执行器只在 `--list-queued` 非空时才起会话。

## 三、即时触发途径核查

| 途径 | 状态 | 说明 |
|---|---|---|
| 常驻 poller（本地脚本） | **已验证** | 唯一实测可用的「近即时」通道，延迟=间隔 |
| 无头 CLI `codebuddy -p` | **未验证** | 二进制存在（`D:/workburry/.../cli/bin/codebuddy`）；需一次 `/login`，桌面版凭据不共享；此前报 Authentication required |
| MCP 直连 GitHub | 不支持 | 当前无 GitHub 写入型 MCP 连接器 |
| 直接调用 Muse / WorkBuddy 智能体本体 | 不支持 | 无对外 API；只能经 Git 异步交换 |
| 事件通知（webhook → 本机） | 不支持 | 需公网入口，属网络安全变更，需审批 |

## 四、建议的最小风险快速通道

1. **默认档（不动任何配置）**：保持每小时 WorkBuddy 定时任务 + `queue` 模式。人工在场时手动 `--once`，即得即时响应。
2. **快速档（推荐，需用户确认）**：登录后双击一次 `start.bat` 让常驻 poller 跑起来（10 分钟间隔、零模型成本），定时任务退化为「兜底 + 执行器」。两者共存安全：`--once` 与常驻互不冲突，去重台账共用。
3. **执行档（暂缓）**：CLI 完成 `/login` 后切 `mode=execute`，由 poller 自身执行，摆脱每小时模型会话。

失败与去重：
- 重试退避 1/5/15 分钟、最多 3 次，仅在 `execute` 模式生效；`queue` 模式不重试，靠下一轮重新领取。
- 去重依据 `state/seen_tasks.json` 文件名键。
- **发现缺陷（需批准后修）**：`--mark-done` 把状态写成 `SUCCESS`，而主循环的跳过集合只含 `EXECUTED / QUEUED / BLOCKED_REFUSED`，因此 SUCCESS 任务下一轮会被重新入队并覆盖已完成报告。本次执行后已把状态归一到 `EXECUTED` 规避；根治建议把跳过集合改为 `EXECUTED / SUCCESS / PARTIAL / BLOCKED_REFUSED`，FAILED 且 attempts≥max 仍跳过。

## 五、下一步最小测试

1. 用户双击 `start.bat` → `status.bat` 应显示 `mode=queue interval=10min token=yes`，且 `state/poller.pid` 指向存活进程。
2. 由 ChatGPT 写一张 `read_only` 测试卡，记录写入时间 T0；观察 `logs/poller-<date>.log` 首次领取时间 T1，验证真实延迟是否 ≤10 分钟。
3. 验证 `tasks/done/` 报告状态与 `seen_tasks.json` 状态一致，确认不会重复入队。

## 六、安全声明

本报告不含 Token、密钥、内网地址、客户数据；所有结论均来自本机日志、状态文件与配置文件，未做任何写操作（仅新增本报告文件）。
