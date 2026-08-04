---
name: us-visa
description: 美国B1/B2签证 DS-160 表单审核与家庭面签模拟。当用户提到"DS-160"、"美签"、"签证审核"、"面签模拟"、"签证准备"、"美国签证"时使用。读取 DS-160 完整表单 PDF，从签证官视角做四维审核（前后矛盾/高风险因素/常见错误/真实情况一致性）+ 跨家庭成员一致性比对，输出中文审核报告；或模拟家庭一起面签的英文问答并逐答中文点评。
dependencies:
  - tencentcloud-ocr
---

# 美签：DS-160 审核与面签模拟

面向美国 **B1/B2**（旅游/商务）签证申请准备的 Claude Code 技能，以**家庭为单位**（含未成年子女）服务。两项核心能力：**① DS-160 完整表单审核**、**② 家庭一起面签模拟**。

> 完整需求与设计依据见 [PRD/PRJ-001](../../PRD/PRJ-001/)（需求清单、成功标准、业务流程、技术规格）。

---

## 一、触发与角色

**触发方式：** 用户输入 `/us-visa`，或用自然语言提到"帮我审核 DS-160 / 面签模拟 / 美签准备"等。

**技能承担的角色：**
- **审核助手**：从签证官视角审核 DS-160，提前暴露问题与风险
- **面签陪练**：扮演签证官英文提问 + 点评官中文反馈

**涉及角色：**
- 用户/主申请人：主要使用者，面签主应答者
- 家庭成员：一起面签，含未成年子女（被问简单问题）

---

## 二、开始流程

1. 确认用户需求：**审核 / 面签模拟 / 全流程（先审核后面签模拟）**
2. 审核/全流程模式 → 让用户**指定输入位置**（文件或目录），技能不得擅自扫描其他位置
3. 确认输入文件数量，读取后推断家庭成员数，并**请用户确认主申请人**

---

## 三、模式一：DS-160 审核

### 3.1 读取与解析
1. 逐个读取用户指定位置的 PDF。
2. **文字版 PDF**：直接用 Read 工具提取文本。
3. **图片/扫描型 PDF**：无法直接提取时，提示用户提供 **腾讯云 OCR 凭据**（见第六节），经 OCR 识别后提取。
4. 按 [references/ds160-fields.md](references/ds160-fields.md) 提取并核对结构化字段。

### 3.2 家庭成员识别
- 依各 PDF 的**出生日期**判定成员类型：成年成员 / 未成年子女。
- 信息不足时默认按成人处理并提示用户确认。
- 请用户确认**主申请人**（面签主应答者）。

### 3.3 四维审核
对照 [references/audit-rules.md](references/audit-rules.md) 逐份执行：

| 维度 | 内容 | 说明 |
|------|------|------|
| ① 前后矛盾 | 表单内部自相矛盾 | 逐字段核对，定位矛盾字段 |
| ② 高风险因素 | 可能触发 214(b) 拒签或行政审查的组合 | 给出风险等级（高/中/低）与规避建议 |
| ③ 常见填写错误 | 格式、必填遗漏、逻辑不合理 | 对照官方字段规范 |
| ④ 真实情况一致性 | 表单与实际差异 | **依赖用户补充真实情况**；未补充时标注"未执行"，不得默认通过 |

### 3.4 跨成员一致性比对
- 仅当 PDF 数量 **≥ 2** 时执行。
- 比对应一致字段：家庭住址、在美联系人、行程计划、直系亲属信息等，以成员维度展示差异。

### 3.5 生成审核报告
输出**中文**审核报告，结构：
```
总体结论
├─ 逐成员审核（每人：四维审核结果）
├─ 跨成员一致性（如有）
└─ 面签提示 / 易被质疑点（结合审核发现）
```
每条发现包含：**风险等级（高/中/低）+ 字段定位 + 说明 + 修改建议**。
报告**输出到用户指定位置**，命名含日期（如 `2026-08-03-DS160审核报告.md`）。

---

## 四、模式二：面签模拟

### 4.1 场景准备
- 若有已解析的成员数据：**个性化 + 通用题库**结合提问（个性化为主）。
- 若无解析数据：退化为**纯通用题库**，不阻塞使用。

### 4.2 模拟流程
```
问候 → 主申请人提问（可穿插切换其他成员）→ 追问 → 收尾
```
1. **中文出题**：说明本轮要模拟的主题与场景。
2. **签证官英文提问**：成人用标准题库，未成年子女用**简化题库**（姓名、年龄、学校、随行人员等，不涉及工作/收入/移民倾向）。
3. **用户英文回答**。
4. **中文逐答点评**：质量评估 + 风险提示 + 优化建议（附可复用的英文表述示例）。
5. 循环至用户结束，输出**收尾总结**（整体表现、薄弱点、改进建议）。

### 4.3 题库
- 个性化问题：引用具体 DS-160 资料（具体工作、行程日期、家庭成员等），占整体提问 ≥ 60%。
- 通用题库：[references/question-bank.md](references/question-bank.md)，覆盖 ≥ 6 类主题。

---

## 五、核心规则

1. **输入/输出位置由用户指定**，技能不擅自决定、不越界扫描。
2. **不脱敏**：敏感数据保持原始，仅在本地仓库内流转。
3. **外部服务唯一例外**：腾讯云 OCR API（图片型 PDF 用），经依赖技能 `tencentcloud-ocr` 调用，见第六节。
4. **四类审核维度全部执行**；一致性维度在无补充信息时标注"未执行"。
5. **跨成员比对**仅 ≥2 份时执行。
6. **报告/点评用中文**，模拟对话用英文，无混用。

---

## 六、图片型 PDF 识别（腾讯云 OCR）

仅当 PDF 为图片/扫描型且需直接识别时启用。本技能不自带 OCR 脚本，经依赖技能 **tencentcloud-ocr**（腾讯云通用文字识别·高精度版）识别文字。

### 6.1 依赖安装（首次使用前）

`tencentcloud-ocr` 为外部依赖，**需单独安装**到当前 Agent 的 skills 目录（来源 GitHub：[TencentCloud/tencentcloud-ocr-skills](https://github.com/TencentCloud/tencentcloud-ocr-skills)）：

```bash
# 手动安装（推荐，目录名 tencentcloud-ocr 与依赖名一致）
git clone --depth 1 https://github.com/TencentCloud/tencentcloud-ocr-skills.git
cp -R tencentcloud-ocr-skills/tencentcloud-ocr <你的 skills 目录>/tencentcloud-ocr
```

> 也可用 skills CLI：`npx skills add TencentCloud/tencentcloud-ocr-skills -s tencentcloud-ocr-generalaccurate --copy -y`
> ⚠️ CLI 按技能 frontmatter `name` 命名，会装成 `tencentcloud-ocr-generalaccurate`——若走此路，须把本技能 frontmatter 的依赖名同步改为 `tencentcloud-ocr-generalaccurate`，二者保持一致。

Python 依赖：`pip install tencentcloud-sdk-python`（运行识别脚本需用装有该 SDK 的解释器）。

> ⚠️ 上游 `scripts/main.py` 有两处 bug，安装后须修复，否则解析不了结果：
> 1. `call_json()` 已返回 dict，删除对其多余的 `json.loads()`；
> 2. 其返回统一信封 `{"Response": {...}}`，需先 `resp_json = resp_json.get("Response", resp_json)` 解包再取字段。

### 6.2 识别流程

1. 提示用户："检测到图片型页面，需要腾讯云 OCR 识别，请输入你的 **SecretId** 与 **SecretKey**。"（若已配置 `TENCENTCLOUD_SECRET_ID` / `TENCENTCLOUD_SECRET_KEY` 环境变量，可直接使用）
2. **凭据由用户每次手动输入**，仅本次会话使用；**绝不写入任何文件**、不缓存、不写入 git。
3. 在 `tencentcloud-ocr` 技能目录下，用装有 SDK 的解释器执行（env 前缀传入凭据，密钥不落盘）：

   ```bash
   cd <tencentcloud-ocr 技能目录>
   TENCENTCLOUD_SECRET_ID=<SecretId> TENCENTCLOUD_SECRET_KEY=<SecretKey> \
     python3 scripts/main.py \
     --image-base64 "<图片/PDF路径>" --is-pdf true --pdf-page-number <N>
   ```

4. **多页 PDF 需逐页识别**：该接口单次仅识别 1 页，对每一页分别调用一次（`--pdf-page-number 1 / 2 / 3 …`），再按页合并为完整文本。
5. 解析返回 JSON：取 `raw_text` 为识别文本；`raw_text` 为空且带 `message` 表示该页无文字，照实反馈。（提示：用 `printf` 或写临时文件读 JSON，勿用 `echo` 传参——zsh 会转义 `\n` 破坏 JSON 结构）
6. 识别失败时明确告知用户该份跳过，不中断其他文件；结束即丢弃凭据。

---

## 七、明确不做（排除范围）

- 签证缴费
- 面签预约
- 材料清单辅助
- 签证状态查询
- 其他国家签证相关功能

---

## 八、文件索引

| 文件 | 用途 |
|------|------|
| [SKILL.md](SKILL.md) | 本技能主文件 |
| [references/ds160-fields.md](references/ds160-fields.md) | DS-160 字段核对清单 |
| [references/audit-rules.md](references/audit-rules.md) | 四维审核规则与高风险模式 |
| [references/question-bank.md](references/question-bank.md) | B1/B2 通用题库（成人+儿童） |
| tencentcloud-ocr（外部依赖） | 依赖技能：腾讯云通用文字识别（高精度版）。来源 [GitHub](https://github.com/TencentCloud/tencentcloud-ocr-skills)，安装见第六节 |
