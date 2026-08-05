---
name: us-visa
description: 美签 B1/B2 DS-160 审核与家庭面签模拟（主 agent + 三子 agent 流水线）。触发词：DS-160、美签、签证审核、面签模拟、签证准备、美国签证。
dependencies:
  - tencentcloud-ocr
---

# 美签：DS-160 审核与面签模拟

面向美国 **B1/B2**（旅游/商务）签证申请准备的 Claude Code 技能，以**家庭为单位**（含未成年子女）服务。采用**主 agent + 三子 agent** 架构：主 agent 按「DS-160 处理 → 审核 → 面签模拟」流水线编排，子 agent 之间经**中间文件**衔接。两项核心能力：**① DS-160 完整表单审核**、**② 家庭一起面签模拟**。

> 完整需求与设计依据见 [PRD/PRJ-001](../../PRD/PRJ-001/)（需求清单、成功标准、业务流程、技术规格）。

---

## 一、触发与角色

**触发方式：** 用户输入 `/us-visa`，或用自然语言提到"帮我审核 DS-160 / 面签模拟 / 美签准备"等。

**主 agent（编排层）：**
- 确认操作模式与输入位置，按流水线调度三个子 agent
- 维护中间文件（生成衔接、会话结束清理）
- 汇总子 agent 结果反馈给用户

**三个子 agent**（skill 范围定义于 `agents/` 目录，**不注册到全局**；主 agent 用 Agent 工具调用）：

| 子 agent | 职责 |
|---------|------|
| **ds160-processor**（DS-160 处理） | 解析多份 DS-160 PDF → 生成结构化中间文件（JSON） |
| **ds160-auditor**（审核） | 读中间文件 → 四维审核 + 跨成员比对 → 中文审核报告 |
| **interview-simulator**（面签模拟） | 读成员数据 → 家庭一起面签的英文模拟 + 中文逐答点评 |

**涉及角色：**
- 用户/主申请人：主要使用者，面签主应答者
- 家庭成员：一起面签，含未成年子女（被问简单问题）

---

## 二、开始流程

1. 确认用户需求：**审核 / 面签模拟 / 全流程（先审核后面签模拟）**
2. 审核/全流程模式 → 让用户**指定输入位置**（文件或目录），技能不得擅自扫描其他位置
3. 确认输入文件数量，读取后推断家庭成员数，并**请用户确认主申请人**
4. 按模式调度子 agent（定义见 [agents/](agents/) 目录）：

| 模式 | 调度链 |
|------|--------|
| 全流程 | ds160-processor → ds160-auditor → interview-simulator |
| 仅审核 | ds160-processor（若无中间文件）→ ds160-auditor |
| 仅模拟 | interview-simulator（无成员数据时退化为纯通用题库） |

> **子 agent 调用方式：** 子 agent 定义于技能 `agents/` 目录（skill 范围内，不注册到全局）。主 agent 用 Agent 工具调用：以 agent 名称（`ds160-processor` / `ds160-auditor` / `interview-simulator`）作为子 agent 类型，或读取 `agents/<name>.md` 定义作为任务指令；并在任务中传入输入/输出路径与技能 `references/` 目录路径。

---

## 三、模式一：DS-160 审核（编排）

### 3.1 DS-160 处理（子 agent：ds160-processor）

1. 调用 `ds160-processor`，传入：PDF 位置、中间文件输出位置、技能 references 路径。
2. 子 agent 逐个读取 PDF：**先用 pypdf 提取 PDF 文字层**（文字版直接产出文本）；仅当页面无任何可提取文字（整页为扫描图片）才经腾讯云 OCR（见 [references/ocr-pdf-recognition.md](references/ocr-pdf-recognition.md)）。
3. 按 `references/ds160-fields.md` 核对字段，识别成员类型（成人/儿童），提取结构化字段。
4. 生成中间文件 JSON（结构见 [references/intermediate-schema.md](references/intermediate-schema.md)），写入用户指定位置。
5. 请用户确认**主申请人**（面签主应答者）。

### 3.2 DS-160 审核（子 agent：ds160-auditor）

1. 调用 `ds160-auditor`，传入：中间文件路径、报告输出位置、技能 references 路径。
2. 子 agent 读取中间文件，对照 [references/audit-rules.md](references/audit-rules.md) 逐成员执行四维审核：

| 维度 | 内容 | 说明 |
|------|------|------|
| ① 前后矛盾 | 表单内部自相矛盾 | 逐字段核对，定位矛盾字段 |
| ② 高风险因素 | 可能触发 214(b) 拒签或行政审查的组合 | 给出风险等级（高/中/低）与规避建议 |
| ③ 常见填写错误 | 格式、必填遗漏、逻辑不合理 | 对照官方字段规范 |
| ④ 真实情况一致性 | 表单与实际差异 | **依赖用户补充真实情况**；未补充时标注"未执行"，不得默认通过 |

3. 成员数 **≥ 2** 时执行跨成员一致性比对（家庭住址、在美联系人、行程计划、直系亲属互填等，以成员维度展示差异）。
4. 生成**中文**审核报告到用户指定位置，结构：
```
总体结论
├─ 逐成员审核（每人：四维审核结果）
├─ 跨成员一致性（如有）
└─ 面签提示 / 易被质疑点（结合审核发现）
```
每条发现包含：**风险等级（高/中/低）+ 字段定位（含表单页码）+ 说明 + 修改建议**。报告命名含日期（如 `2026-08-03-DS160审核报告.md`）。

---

## 四、模式二：面签模拟（编排）

1. 调用 `interview-simulator`，传入：中间文件路径（如有）、主申请人信息、技能 references 路径。
2. 子 agent 执行模拟流程：
```
问候 → 主申请人提问（可穿插切换其他成员）→ 追问 → 收尾
```
   1. **中文出题**：说明本轮要模拟的主题与场景。
   2. **中文问答（开场）**：签证官中文问候提问，用户中文回答，熟悉流程与口径。
   3. **切换英文（警示触发）**：遇**需要警惕/小心**处（高风险/易被追问点，如工作经历时间线、资金与消费水平、职业转型、行程矛盾、约束力弱、首次赴美等），签证官**切换英文提问**。
   4. **切换后全英文**：一旦英文提问，后续问答必须全英文，直至收尾。
   5. **中文逐答点评**：质量评估 + 风险提示 + 优化建议（附可复用的英文表述示例）。
   6. 循环至用户结束，输出**收尾总结**（整体表现、薄弱点、改进建议）。
3. 无中间文件时退化为**纯通用题库**，不阻塞使用。
4. 题库：成人标准 + 儿童简化见 [references/question-bank.md](references/question-bank.md)；个性化问题引用中间文件中的具体 DS-160 资料（个性化为主，≥ 60%）。
5. **语言阶段规则**：中文开场热身；遇需警惕/小心处切英文提问（警示信号）；**切换后后续问答必须全英文**；出题/点评始终中文。

---

## 五、中间文件

- **生成**：ds160-processor 将解析结果写为 JSON 到用户指定位置（默认 `ds160-intermediate.json`），结构见 [references/intermediate-schema.md](references/intermediate-schema.md)。
- **衔接**：ds160-auditor / interview-simulator **只读中间文件**，不重复解析源 PDF（一次解析、多次复用）。
- **清理**：中间文件为**临时产物**，会话结束由主 agent 清理，不纳入版本库；用户要求保留时跳过。
- **回退**：中间文件缺失/失效时，回到 ds160-processor 重新生成。

---

## 六、核心规则

1. **输入/输出位置由用户指定**，技能不擅自决定、不越界扫描。
2. **不脱敏**：敏感数据保持原始，仅在本地仓库内流转。
3. **外部服务唯一例外**：腾讯云 OCR API（图片型 PDF 用），经依赖技能 `tencentcloud-ocr` 调用，见 [references/ocr-pdf-recognition.md](references/ocr-pdf-recognition.md)。
4. **四类审核维度全部执行**；一致性维度在无补充信息时标注"未执行"。
5. **跨成员比对**仅 ≥2 份时执行。
6. **报告/点评用中文**，模拟对话用英文，无混用。
7. **中间文件为临时产物**：用户指定位置落盘，会话末清理，不纳入版本库。
8. **子 agent 只读中间文件**，不重复解析源 PDF；单个子 agent 失败不中断整体流程，可跳过/重试。
9. **嵌入图片（照片、条形码、签名）不处理、不审核**：ds160-processor 用 pypdf 只提取 PDF 文字层，跳过一切无文字对象（照片/条形码/签名等），不 OCR 表单内图片；ds160-auditor 不审核照片合规性等图像类问题。

---

## 七、图片型 PDF 识别（腾讯云 OCR）

**DS-160 多为浏览器打印的文字版 PDF，通常可直接提取文字，无需 OCR。** 仅当某页整页为扫描图片（无任何可提取文字）时才启用 OCR 兜底。完整流程（依赖安装、凭据处理、逐页识别、上游 bug 修复）见 [references/ocr-pdf-recognition.md](references/ocr-pdf-recognition.md)。

---

## 八、明确不做（排除范围）

- 签证缴费
- 面签预约
- 材料清单辅助
- 签证状态查询
- 其他国家签证相关功能

---

## 九、文件索引

| 文件 | 用途 |
|------|------|
| [SKILL.md](SKILL.md) | 本技能主文件（编排层） |
| [agents/ds160-processor.md](agents/ds160-processor.md) | 子 agent：DS-160 处理，生成中间文件 |
| [agents/ds160-auditor.md](agents/ds160-auditor.md) | 子 agent：四维审核 + 跨成员比对，生成报告 |
| [agents/interview-simulator.md](agents/interview-simulator.md) | 子 agent：家庭面签模拟 |
| [references/ds160-fields.md](references/ds160-fields.md) | DS-160 字段核对清单 |
| [references/audit-rules.md](references/audit-rules.md) | 四维审核规则与高风险模式 |
| [references/question-bank.md](references/question-bank.md) | B1/B2 通用题库（成人+儿童） |
| [references/intermediate-schema.md](references/intermediate-schema.md) | 中间文件 JSON Schema |
| [references/ocr-pdf-recognition.md](references/ocr-pdf-recognition.md) | 图片型 PDF 的腾讯云 OCR 兜底流程（罕见） |
| tencentcloud-ocr（外部依赖） | 依赖技能：腾讯云通用文字识别（高精度版）。来源 [GitHub](https://github.com/TencentCloud/tencentcloud-ocr-skills)，使用见 references/ocr-pdf-recognition.md |
