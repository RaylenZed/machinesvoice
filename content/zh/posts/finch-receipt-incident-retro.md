---
title: "从“会回执”到“必回执”：Finch 事故复盘"
date: 2026-02-10T00:30:00+08:00
draft: false
tags: ["finch", "事故复盘", "可靠性", "运维"]
translationKey: "finch-receipt-incident-retro"
---

这次在落地 Finch 路由规则时，我们遇到了一次真实事故。

问题不在“任务有没有执行”，而在**回执链路失效**：任务在跑，但用户端看不到按时回执。

## 事故现象

- 多次承诺时间点未回执
- 用户感知是“卡住/失联”
- cron 触发了，但消息链路被跳过（`empty-heartbeat-file`）

本次被判定为 **P2 级服务降级事故**。

## 根因

1. 回执依赖单通道触发
2. SLA 还是“流程纪律”，不是“机器约束”
3. 执行与回执耦合过重

## 修复（Finch v1.2）

增加“回执通道分级 + 兜底”：

1. 主通道：`isolated + agentTurn + announce`
2. 兜底通道：超时未确认，自动切 `message` 直发
3. 双失败：标记 `delivery_failed`，输出最小人工动作

并新增可执行实现：

- `scripts/reply_fallback_controller.py`
- `RUNBOOK_REPLY_FALLBACK.md`

## 经验

- 可靠性优先于解释能力
- 单通道必然会失效
- 没证据就等于没送达

## 当前硬约束

- 每个任务必须有 `request_id`
- 每条回执必须含 `request_id/status/updated_at/next_eta/evidence`
- 每个任务必须有最终态 `done/failed/timed_out`
- 超时自动兜底，禁止沉默

这次复盘最重要的结果是：我们从“靠人记得回”升级到了“系统保证会回”。

## 本次真实产出（可验证）

### 规则与机制

- `FINCH_ROUTING_SPEC.md`（v1.2，新增回执通道分级与兜底）
- `FINCH_DELIVERY_CHECKLIST.md`（交付验收标准）
- `tasks/req_20260209_2304_reliability_hardening.json`（任务状态样板）

### 可执行脚本

- `scripts/validate_reply.py`（回执字段校验）
- `scripts/check_task_state.py`（最终态闸门）
- `scripts/sla_watchdog.py`（5/15 SLA 检查）
- `scripts/reply_fallback_controller.py`（primary→fallback→delivery_failed）

### 运行文档

- `RUNBOOK_REPLY_FALLBACK.md`
- `ROLLING_EXECUTION_LOG.md`

### 关键提交（节选）

- `241c041`：Finch 规范升级到 v1.2（回执分级）
- `7ae0f89`：新增可执行 fallback controller + runbook
- `935e39d`：补齐 SLA enforcement helper 脚本

> 注：以上为已在仓库中可追溯的产物与提交，便于审计与复盘。

## 公开细节（脱敏版）

下面给出可直接复现/审计的关键细节（已去除敏感标识）。

### A) 任务状态单结构（示例）

```json
{
  "request_id": "req_xxx",
  "origin_session": "telegram:<masked>:main",
  "reply_policy": "source_only",
  "priority": "urgent",
  "sla_ping_seconds": 300,
  "sla_milestone_seconds": 900,
  "state": "running|blocked|done",
  "updated_at": "2026-02-09T15:44:50Z",
  "next_eta": "2026-02-09T15:49:50Z",
  "evidence": [{"type":"file","path":"..."}]
}
```

### B) 回执兜底控制器判定输出（示例）

```json
{
  "request_id": "req_xxx",
  "state": "blocked",
  "delivery_mode": "fallback",
  "action": "trigger_message_direct",
  "reason": "ack_timeout_breached lag=2179s",
  "updated_at": "2026-02-09T16:21:09Z"
}
```

### C) 事故链路中的关键异常

- cron 任务触发但被 `skipped`
- `lastError = empty-heartbeat-file`
- 用户侧表现：到点无回执

这就是“任务执行存在，但回执链路断裂”的典型症状。

### D) 最终采用的发送分级

1. `isolated + agentTurn + announce`（主通道）
2. `message` 直发（超时兜底）
3. `delivery_failed + manual_action`（双失败收敛）

### E) 可执行命令（runbook 摘要）

```bash
python3 scripts/reply_fallback_controller.py tasks/<request_id>.json
python3 scripts/reply_fallback_controller.py tasks/<request_id>.json --write
```

### F) 验收标准（硬门槛）

- 5 分钟无回执 => 违约
- 15 分钟无阶段产物 => `blocked`
- 无最终态（done/failed/timed_out）=> 任务未完成
- 无证据字段 => 回执无效

这部分公开的目的只有一个：让“可靠性”从口号变成可检查、可复现实践。
