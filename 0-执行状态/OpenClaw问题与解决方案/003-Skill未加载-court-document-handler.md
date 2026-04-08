---
title: Skill无法触发-court-document-handler未被加载
date: 2026-04-06
severity: high
tags: [OpenClaw, Skill, YAML, 排障]
status: resolved
---

# Skill 无法触发：court-document-handler 未被加载

## 基本信息

| 项目 | 内容 |
|------|------|
| **日期** | 2026-04-06 |
| **严重程度** | 高（核心功能完全不可用） |
| **影响范围** | 法院短信自动处理流程全部失效 |
| **状态** | ✅ 已解决 |
| **涉及文件** | `~/.agents/skills/court-document-handler/SKILL.md` |

## 问题描述

通过微信发送法院短信给 OpenClaw，期望触发 `court-document-handler` skill 执行自动化流程（访问链接→输入验证码→下载PDF→提取信息→微信汇报），但实际行为是 OpenClaw 使用 WebSearch 查询链接，回复：

> 这是法院的电子文书签收平台，需要手机验证码登录后才能查看具体文书内容。我无法直接获取文书PDF。

**关键线索**：
- 让 OpenClaw 自己查找有哪些 Skill 时，它不知道有 `court-document-handler`
- 但问它 skill 存储位置在哪，它能找到 `~/.agents/skills/` 目录
- `openclaw skills list` 输出中完全没有该 skill

## 根因分析

**直接原因**：SKILL.md 文件的 YAML frontmatter 缺少闭合的 `---` 分隔符。

### 错误的文件格式

```yaml
---
name: court-document-handler
description: Use when message contains "广西法院短信平台" or "法院向您发送" or "sfsdw" or "文书详见" or case number pattern (20XX)桂. 法院送达短信自动处理：**必须使用 openclaw browser 命令**（不能用 web_fetch），自动下载PDF、提取信息、微信汇报并发送PDF、设置提醒。全自动，无需人工干预。

# Court Document Handler

自动处理法院送达短信，下载文书、提取信息、微信汇报并发送PDF、设置提醒。**全自动处理，无需人工干预。**
```

### 问题链

1. YAML 解析器遇到 `---`（opening），开始读取 frontmatter
2. 在 `description:` 行之后没有找到闭合的 `---`
3. 解析器把后面的 Markdown 正文（`# Court Document Handler`、`自动处理法院送达短信...`、`## ⚠️ 重要...` 等）全部当作 YAML 内容
4. Markdown 中的 `#`、`**`、`##` 等符号在 YAML 中是非法语法
5. YAML 解析失败，抛出 `could not find expected ':'` 错误
6. **OpenClaw 静默处理该错误**，不报错、不警告，直接跳过该 skill
7. skill 从未被注册到可用 skill 列表中
8. 当 AI 收到法院短信时，扫描可用 skill 列表找不到匹配项，回退到默认行为（WebSearch）

### 验证方法

用 Python 快速验证 frontmatter 格式：

```bash
python3 -c "
import yaml
with open('/path/to/SKILL.md') as f:
    content = f.read()
parts = content.split('---', 2)
if len(parts) >= 3:
    try:
        meta = yaml.safe_load(parts[1])
        print('OK:', list(meta.keys()))
    except Exception as e:
        print('YAML ERROR:', e)
else:
    print('ERROR: no closing ---')
"
```

## 解决方案

### 1. 修复 frontmatter 格式

添加闭合 `---`，去掉 description 中的 Markdown 标记，加入 metadata：

```yaml
---
name: court-document-handler
description: Use when message contains "广西法院短信平台" or "法院向您发送" or "sfsdw" or "文书详见" or case number pattern (20XX)桂. 法院送达短信自动处理，必须使用 openclaw browser 命令（不能用 web_fetch），自动下载PDF、提取信息、微信汇报并发送PDF、设置提醒。全自动，无需人工干预。
metadata:
  {
    "openclaw": {
      "emoji": "⚖️",
      "os": ["linux", "darwin", "win32"]
    }
  }
---

# Court Document Handler

自动处理法院送达短信，下载文书、提取信息、微信汇报并发送PDF、设置提醒。**全自动处理，无需人工干预。**
```

### 2. 重启 gateway

```bash
pkill -9 -f openclaw-gateway || true
nohup openclaw gateway run --bind lan --port 18789 --force > /tmp/openclaw-gateway.log 2>&1 &
```

### 3. 验证

```bash
openclaw skills list | grep court
# 应输出: │ ✓ ready       │ ⚖️ court-document-handler
```

## 经验教训

1. **Frontmatter 必须闭合**：SKILL.md 的 YAML frontmatter 必须有 `---` 开头和 `---` 结尾，缺一不可
2. **错误是静默的**：OpenClaw 不会在 skill 加载失败时报错或警告，只会静默跳过
3. **验证习惯**：新建或修改 skill 后，务必运行 `openclaw skills list` 确认 skill 出现在列表中
4. **YAML 值中不要用 Markdown**：description 等字段中避免使用 `**`、`#` 等 Markdown 语法，会导致 YAML 解析混乱
5. **快速验证脚本**：可用 Python yaml 库快速验证 frontmatter 格式是否正确