# PRJ-001：美签DS-160审核与面签模拟Skill

| 属性 | 值 |
|------|-----|
| 项目编码 | PRJ-001 |
| 项目名称 | 美签DS-160审核与面签模拟Skill |
| 项目描述 | 创建Claude Code技能：读取审核DS-160表单、模拟美国签证面签问答 |
| 状态 | 进行中 |
| 当前阶段 | 需求分析完成（P1~P5） |
| 创建日期 | 2026-08-03 |
| 最后更新 | 2026-08-04 |

## 项目简介

用户希望创建一个 Claude Code 技能（Skill），用于辅助美国签证申请流程：

1. **DS-160 读取审核**：读取用户的 DS-160 表单内容，按美国签证官视角审核潜在问题、前后矛盾、风险点
2. **面签模拟**：根据用户个人资料模拟美国签证面签的问答环节，帮助用户准备面签

本项目将产出完整的需求分析文档，作为该 Skill 后续设计与实现的输入基础。

> **2026-08-04 更新：** 项目已从 Obsidian Vault（`05-Life/VISA/`）迁移至独立仓库 `/Users/neo/src/us-visa-skill/`；OCR 兜底方案确定为依赖技能 `tencentcloud-ocr`（见 [技术约束](05-technical-spec/technical-constraints.md) TC-INT-003）。
>
> **2026-08-04 更新：** 新增三子 agent 架构需求——DS-160 处理 / 审核 / 面签模拟，由主 agent 按流水线编排，经中间文件衔接（见 [需求清单](02-requirements/requirements-list.md) 模块四）。

## 文档索引

- [需求澄清记录](01-clarification/clarification.md)
- [需求清单](02-requirements/requirements-list.md)
- [成功标准](03-success-criteria/success-criteria.md)
- [业务流程](04-business-processes/processes.md)
- [功能需求](05-technical-spec/functional-requirements.md)
- [非功能需求](05-technical-spec/non-functional-requirements.md)
- [技术约束](05-technical-spec/technical-constraints.md)
