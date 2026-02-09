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
