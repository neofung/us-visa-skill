# DS-160 中间文件 Schema

> 用途：**ds160-processor** 子 agent 解析 DS-160 PDF 后生成的结构化中间文件（JSON）规范，供 **ds160-auditor**（审核）、**interview-simulator**（面签模拟）读取。
> 中间文件为**临时产物**：存放于用户指定位置，会话结束由主 agent 清理，不纳入版本库（REQ-019/020、NFR-SEC-005、TC-INT-004/OTH-004）。

---

## 总体结构

```json
{
  "schema_version": "1.0",
  "meta": {
    "generated_by": "ds160-processor",
    "generated_at": "<YYYY-MM-DD>",
    "source_files": ["<每份源 PDF 文件名>"],
    "ocr_used": false
  },
  "members": [
    {
      "id": 1,
      "source_file": "<该成员对应的源 PDF 文件名>",
      "name": { "family": "<姓>", "given": "<名>" },
      "type": "adult | child",
      "is_main_applicant": true,
      "fields": { "<见下方字段分区>" },
      "field_pages": { "<字段名>": "<页码>" }
    }
  ]
}
```

- `members` 数组按成员排序；`id` 从 1 递增。
- `type`：依出生日期判定，未成年（一般 < 18 岁）为 `child`，否则 `adult`；信息不足默认 `adult` 并在 `meta` 或字段中标注待确认。
- `is_main_applicant`：默认仅第一个成员为 `true`，主申请人在会话中由用户确认后由主 agent 更新。
- `field_pages`：**字段 → 源 PDF 页码**映射（页数从 1 开始），由 ds160-processor 逐页提取时记录；供审核报告"字段定位"标注问题所在页。缺失页码的字段记 `null`。

---

## 字段分区（`fields`）

以 [ds160-fields.md](ds160-fields.md) 为字段基线，逐区提取。字段值为 `null` 表示表单未提供或打印版不可见。

### personal_information（个人信息）
| 字段 | 说明 |
|------|------|
| full_name | 姓名拼音，与护照完全一致 |
| birth_date | 出生日期 |
| birth_country / birth_city | 出生国家/城市 |
| nationality | 国籍 |
| gender | 性别 |
| marital_status | 婚姻状况 |
| national_id | 身份证号 |
| phone / email | 联系电话/邮箱 |
| address | 当前住址（完整到门牌号） |

### passport_information（护照信息）
| 字段 | 说明 |
|------|------|
| passport_number | 护照号 |
| issue_country / issue_place | 签发国/签发地 |
| issue_date / expiry_date | 签发/到期日 |
| has_old_passport | 是否有旧护照（null=未知） |

### travel_information（旅行信息）
| 字段 | 说明 |
|------|------|
| intended_visit_date | 拟到访美国日期 |
| intended_stay_days | 拟停留天数（null=未填） |
| us_address | 计划在美地址/城市 |
| expense_payer | 费用支付人（SELF / OTHER / 说明） |

### us_contact（在美联系人）
| 字段 | 说明 |
|------|------|
| contact_name_org | 在美联系人或组织（含 DO NOT KNOW / HOTEL 等） |
| relationship | 关系 |

### family_information（家庭信息）
| 字段 | 说明 |
|------|------|
| spouse | 配偶（姓名、出生日期等，无则 null） |
| children | 子女数组（姓名、出生日期） |
| parents | 父母信息 |
| us_relatives | 在美直系亲属（高风险字段） |
| siblings | 兄弟姐妹（含在美者标注） |

### education_employment（教育与工作经历）
| 字段 | 说明 |
|------|------|
| education | 教育经历数组（学校、起止时间） |
| current_employer | 当前雇主（名称、职位、起始日） |
| current_employer_address / phone | 雇主地址/电话 |
| employment_5y | 过去 5 年工作经历数组 |
| job_description | 工作描述 |

### security_background（安全和背景部分）
| 字段 | 说明 |
|------|------|
| all_no | 安全类问题是否全为"否" |
| refusals | 拒签/拒绝入境史（null=无） |
| overstay | 逾期停留/签证违规（null=无） |
| previous_visa_applications | 是否曾申请美签（null=无） |

### other（其他）
| 字段 | 说明 |
|------|------|
| prepared_by_third_party | 是否第三方协助填写 |
| application_barcode | 确认页条码/申请号（若可见） |

---

## 生成规则（ds160-processor 执行）

1. **一次解析、多次复用**：DS-160 处理 agent 只解析 PDF 一次，将结果全部写入中间文件；审核/模拟 agent 不得重新解析 PDF。
2. **逐页提取并记录页码**：用 pypdf 按页提取文字时，记录每个字段所在页（页数从 1 开始），写入该成员 `field_pages`（字段名 → 页码）。同一字段跨多页时取首次出现页；页码不可得记 `null`。
3. **图片/扫描型 PDF**：经依赖技能 tencentcloud-ocr 识别后再提取（见 SKILL.md 第七节），并在 `meta.ocr_used` 标 `true`；单份识别失败跳过并在 `meta` 标注。OCR 逐页识别时同样记录页码。
4. **字段缺失**：打印版不可见的字段记 `null`，不得臆造；在 `meta` 中列出缺失字段清单，供审核 agent 标注。
5. **成员互填字段**：家庭成员的配偶/子女互填信息保留原始填写，供审核 agent 跨成员比对。

---

## 读取规则（ds160-auditor / interview-simulator 执行）

1. 只读中间文件，**不得重新读取源 PDF**。
2. 中间文件缺失或 `schema_version` 不匹配时，返回"中间文件缺失/失效"，由主 agent 回到处理环节重新生成。
3. 审核报告、面签模拟均以中间文件字段为准，与 DS-160 表单保持逐字一致。
