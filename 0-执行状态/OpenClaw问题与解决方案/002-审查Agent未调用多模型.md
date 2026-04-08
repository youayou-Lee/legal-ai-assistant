---
title: 审查Agent未能调用多模型
date: 2026-04-05
severity: medium
tags: [OpenClaw, Agent, 多模型, 排障]
status: resolved
---

# 审查 Agent 未能调用多模型

## 基本信息

| 项目 | 内容 |
|------|------|
| **日期** | 2026-04-05 |
| **严重程度** | 中（多模型交叉验证功能失效） |
| **影响范围** | 审查Agent只能用单一模型，无法实现多模型交叉验证 |
| **状态** | ✅ 已解决 |
| **涉及文件** | `~/.openclaw/openclaw.json`, Agent AGENTS.md |

## 问题描述

配置了 `reviewer-deepseek` 和 `reviewer-qwen` 两个审查 Agent，但测试时发现 main Agent 没有成功调用审查员。AI 只用自己的模型生成内容，没有走"生成→审查→修正"的闭环流程。

## 根因分析

main Agent 的 AGENTS.md 中没有写明审查调度的触发规则和调用方式。AI 不知道：
- 什么时候应该调用审查员
- 用什么命令/格式调用 subagent
- 审查结果如何合并（共识机制）

OpenClaw 的 Agent 不会"自己发现"可以调用 subagent，必须在配置中显式告知。

## 解决方案

在 Agent 配置（AGENTS.md）中明确写入：

1. **审查触发条件**：什么时候需要审查（文书生成后、法律分析后等）
2. **调度顺序**：先 main 生成 → 再并行调用两个审查员
3. **共识机制**：两个审查员都指出的问题必须修改，取高不取低
4. **具体的 subagent 调用命令格式**：让 AI 知道怎么调用

## 经验教训

- Agent 的 AGENTS.md 就是它的"操作手册"，必须写清楚每一个决策分支
- AI 不会"自己发现"可以调用 subagent，必须显式告知
- 配置优于代码——用 OpenClaw 原生配置而非写 Python 脚本调用不同 API