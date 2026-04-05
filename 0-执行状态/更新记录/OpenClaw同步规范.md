---
title: OpenClaw同步规范
created: 2026-04-05
updated: 2026-04-05
type: skill
tags: [OpenClaw, 部署, 同步]
---

# OpenClaw同步规范

> 仓库：`~/openclaw-custom/` → `github.com:youayou-Lee/openclaw-custom`（私有）
>
> 模式：**v2.0 全量同步**（除飞书凭证外，所有配置完全一致）

## 仓库结构

```
openclaw-custom/
├── deploy.py              ← 跨平台部署脚本（推荐）
├── deploy.sh              ← Linux 部署脚本
├── sync-push.sh           ← 开发机：本地→仓库同步
├── openclaw.json.template ← 全量配置模板（只有飞书用占位符）
├── config.env.example     ← 飞书凭证模板（gitignored）
├── config.env             ← 飞书凭证（gitignored）
├── CHANGELOG.md
├── agents/                → 复制到 ~/.openclaw/agents/
│   ├── main/              ← 调度员（GLM-5）
│   ├── lawyer/            ← 律师（GLM-5）
│   ├── reviewer-deepseek/ ← 对方律师（DeepSeek Chat）
│   └── reviewer-qwen/     ← 法官（Qwen3.6 Plus）
├── workspace/             → 复制到 ~/.openclaw/workspace/
│   ├── SOUL.md
│   └── AGENTS.md
├── workspace-lawyer/      → 复制到 ~/.openclaw/workspace-lawyer/
│   ├── SOUL.md
│   └── AGENTS.md
└── skills/                → 复制到 ~/.agents/skills/（27个Skill）
```

## Agent 架构

```
用户消息 → [main 调度员]
              ├── 简单问题 → 直接回答
              └── 法律任务 → sessions_spawn → [lawyer 律师]
                                                    │
                                                    ├── 完成后自动审查
                                                    ├── → [reviewer-deepseek 对方律师]
                                                    └── → [reviewer-qwen 法官]
```

| Agent | 模型 | 角色 |
|-------|------|------|
| main | GLM-5 | 调度员：接收任务、委托律师、汇报结果 |
| lawyer | GLM-5 | 律师：法律分析、证据审查、文书起草 |
| reviewer-deepseek | DeepSeek Chat | 对方律师：全力攻击方案弱点 |
| reviewer-qwen | Qwen3.6 Plus | 法官：客观评判，识别真实风险 |

## 两个方向

| 方向 | 命令 | 谁用 | 场景 |
|------|------|------|------|
| Push | `./sync-push.sh` | 你 | 改了本地OpenClaw配置 → 同步到仓库 |
| Deploy | `deploy.py` / `deploy.sh` | 律师 | 从仓库拉取最新配置部署到本地 |

## 规矩

### 同步的4类文件

| 类别 | 源 → 目标 | 数量 | 说明 |
|------|----------|------|------|
| Agent 定义 | `agents/` → `~/.openclaw/agents/` | 4个Agent | 只操作 `agent/` 子目录，不动 `sessions/` |
| Workspace 指令 | `workspace*/` → `~/.openclaw/workspace*/` | 4个md | 只覆盖 SOUL.md + AGENTS.md |
| Skills | `skills/` → `~/.agents/skills/` | 27个Skill | 全量覆盖（先删后复制） |
| 主配置 | `openclaw.json.template` → `~/.openclaw/openclaw.json` | 1个 | 模板渲染，替换占位符 |

### 脱敏规则

| 内容 | 处理 |
|------|------|
| 飞书 App ID / Secret | 脱敏为 `%%FEISHU_APP_ID%%` / `%%FEISHU_APP_SECRET%%` |
| HOME 路径 | 脱敏为 `{HOME}` |
| API Key | **不脱敏**（团队共享） |

sync-push.sh 自动处理脱敏。deploy.py/deploy.sh 反向替换。

### 绝对不提交

- config.env（飞书凭证）
- agents `auth-profiles.json`（本地认证缓存）
- workspace 运行时文件（memory/、MEMORY.md、TOOLS.md、USER.md、BOOTSTRAP.md）

### 提交规范

```
feat: 新增XXX Agent
fix: 修复审查员prompt
chore: 同步配置更新
sync: 配置更新 YYYY-MM-DD
```

每次更新 `CHANGELOG.md`。

## 操作速查

```bash
# 你改了OpenClaw后，同步到仓库
cd ~/openclaw-custom
./sync-push.sh --dry-run     # 先看变化
./sync-push.sh               # 脱敏+提交+推送

# 律师端部署（推荐，跨平台）
cd ~/openclaw-custom && git pull
python deploy.py --dry-run    # 预览
python deploy.py              # 正式部署
# 或传入飞书凭证：
python deploy.py --feishu-id cli_xxxx --feishu-secret xxxx

# 律师端部署（Linux备选）
cd ~/openclaw-custom && git pull && ./deploy.sh

# 验证部署
python deploy.py --verify-only

# 重启
sudo systemctl restart openclaw-gateway
```

## 故障排查

| 问题 | 解决 |
|------|------|
| 审查员没调度 | 检查 main agent 的 AGENTS.md 中 sessions_spawn 配置 |
| 飞书凭证没替换 | `cp config.env.example config.env` 填入凭证 |
| openclaw.json 被破坏 | 从备份恢复：`ls ~/.openclaw/openclaw.json.bak.* \| tail -1` |
| deploy.py 找不到 python | 用 `uv run python deploy.py` 或 `python3 deploy.py` |
| Skills 没生效 | 确认路径是 `~/.agents/skills/` 不是 `~/.openclaw/skills/` |

---
## 变更历史

| 日期 | 版本 | 变更 |
|------|------|------|
| 2026-04-04 | v1.0 | 初始增量补丁模式：reviewer Agent + jq脚本 + aglets-md |
| 2026-04-05 | v2.0 | 重构为全量同步：4 Agent + Workspace + Skills + 模板渲染，移除增量模式 |
