# 图片型 PDF 识别（腾讯云 OCR）

> 用途：**兜底 OCR 流程**，仅当 DS-160 PDF 页面整页为扫描图片（无任何可提取文字）时使用。
> **DS-160 多为浏览器打印的文字版 PDF，通常可直接提取文字，无需 OCR。**
> 文本提取顺序：先用 pypdf 提取 PDF 文字层；仅当某页整页为扫描图片才走本节。
> 嵌入图片（照片、条形码、签名、签证页图）一律不处理、不 OCR、不读取。

---

## 依赖安装（首次使用前）

`tencentcloud-ocr` 为外部依赖，**需单独安装**到当前 Agent 的 skills 目录（来源 GitHub：[TencentCloud/tencentcloud-ocr-skills](https://github.com/TencentCloud/tencentcloud-ocr-skills)）：

```bash
# 手动安装（推荐，目录名 tencentcloud-ocr 与依赖名一致）
git clone --depth 1 https://github.com/TencentCloud/tencentcloud-ocr-skills.git
cp -R tencentcloud-ocr-skills/tencentcloud-ocr <你的 skills 目录>/tencentcloud-ocr
```

> 也可用 skills CLI：`npx skills add TencentCloud/tencentcloud-ocr-skills -s tencentcloud-ocr-generalaccurate --copy -y`
> ⚠️ CLI 按技能 frontmatter `name` 命名，会装成 `tencentcloud-ocr-generalaccurate`——若走此路，须把技能 frontmatter 的依赖名同步改为 `tencentcloud-ocr-generalaccurate`，二者保持一致。

Python 依赖：`pip install tencentcloud-sdk-python`（运行识别脚本需用装有该 SDK 的解释器）。

> ⚠️ 上游 `scripts/main.py` 有两处 bug，安装后须修复，否则解析不了结果：
> 1. `call_json()` 已返回 dict，删除对其多余的 `json.loads()`；
> 2. 其返回统一信封 `{"Response": {...}}`，需先 `resp_json = resp_json.get("Response", resp_json)` 解包再取字段。

---

## 识别流程

1. 提示用户："检测到图片型页面，需要腾讯云 OCR 识别，请输入你的 **SecretId** 与 **SecretKey**。"（若已配置 `TENCENTCLOUD_SECRET_ID` / `TENCENTCLOUD_SECRET_KEY` 环境变量，可直接使用）
2. **凭据由用户每次手动输入**，仅本次会话使用；**绝不写入任何文件**、不缓存、不写入 git。
3. 在 `tencentcloud-ocr` 技能目录下，用装有 SDK 的解释器执行（env 前缀传入凭据，密钥不落盘）：

   ```bash
   cd <tencentcloud-ocr 技能目录>
   TENCENTCLOUD_SECRET_ID=<SecretId> TENCENTCLOUD_SECRET_KEY=<SecretKey> \
     python3 scripts/main.py \
     --image-base64 "<图片/PDF路径>" --is-pdf true --pdf-page-number <N>
   ```

4. **多页 PDF 需逐页识别**：该接口单次仅识别 1 页，对每一页分别调用一次（`--pdf-page-number 1 / 2 / 3 …`），再按页合并为完整文本。逐页识别时记录页码（写入中间文件 `field_pages`，见 intermediate-schema.md）。
5. 解析返回 JSON：取 `raw_text` 为识别文本；`raw_text` 为空且带 `message` 表示该页无文字，照实反馈。（提示：用 `printf` 或写临时文件读 JSON，勿用 `echo` 传参——zsh 会转义 `\n` 破坏 JSON 结构）
6. 识别失败时明确告知用户该份跳过，不中断其他文件；结束即丢弃凭据。

---

## 调用方

- **ds160-processor**：仅当某页整页为扫描图片（pypdf 无可提取文字）时才调用本节；识别后合并文本并继续提取字段（含页码记录）。
- 若 OCR 识别失败，跳过该份并在中间文件 `meta` 标注，不中断其他文件。
