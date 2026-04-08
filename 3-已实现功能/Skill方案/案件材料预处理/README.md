---
title: 案件材料预处理
type: skill-overview
created: 2026-04-09
updated: 2026-04-09
tags: [skill, pdf, ocr, 知识图谱, 案件预处理]
---

# 案件材料预处理

## 技术方案
OpenClaw Skill + Python脚本（pdf_extract.py）

## 一句话说明
将PDF案件材料转化为结构化知识基座：文本提取 → 材料目录 → 关系图谱。

## 部署位置
- Claude Code: `~/.claude/skills/case-preprocess/`
- OpenClaw: `~/.agents/skills/case-preprocess/`
- 源码仓库: `~/openclaw-custom/skills/case-preprocess/`

## 三阶段管线

### Stage 1: 文本提取（Python脚本）
- 脚本：`scripts/pdf_extract.py`
- 检测逻辑：先用 `pdftotext` 提取，每页平均字符 < 50 判定为扫描型
- 扫描型调用 PaddleOCR API（`PaddleOCR-VL-1.5` 模型）
- 输出：每个PDF对应一个 `.md` 文件 + `提取报告.json`

### Stage 2: 材料目录（Agent执行）
- 逐文件读取 `材料/` 下的MD，识别文书类型（起诉状/答辩状/判决书/合同/流水/证据等）
- 生成三个模块：材料位置索引 + 信息定位指引 + 案件概要
- 参考专家指引知识（`references/expert-guide.md`）

### Stage 3: 关系图谱（Agent执行）
- 从材料中提取六类实体：人物(蓝)、组织(青)、金额(红)、日期(绿)、法律关系(橙)、证据(紫)
- 生成 Obsidian Canvas JSON 格式的 `.canvas` 文件
- 布局：人物左侧 → 法律关系中间 → 金额中上 → 证据右侧 → 日期下 → 文件最下

## 关键设计决策

1. **PDF类型自动检测**：先试pdftotext（零成本），不行的才调OCR（有API成本）
2. **脚本而非Agent执行Stage 1**：文本提取是确定性操作，脚本比Agent更可靠
3. **材料目录作为检索中间层**：Agent搜索时先查目录定位，避免全量扫描所有文件
4. **Canvas图谱解决关键词检索局限**：通过关系链发现grep搜不到的关联证据

## ⚠️ 验证状态
- **测试环境**：OpenClaw已部署（lawyer agent）
- **OpenClaw端到端运行**：✅已验证（2026-04-09）
- **场景覆盖**：杨刚案（民事借贷，8 PDF），陈明飞诈骗案（刑事，4 PDF + 1 WPS）
- **律师验证**：❌未验证

## 已知限制
- PaddleOCR API Token 有过期风险，需定期更新
- 单文件超过50页时OCR耗时较长，需用户确认
- WPS文件（.wps）由pdftotext处理，部分格式可能丢失
- Canvas图谱布局为固定规则，复杂案件可能出现节点重叠

## 目录索引
- [[测试]] — 一键测试命令 + 验证方法
- [[思考]] — 设计决策记录
