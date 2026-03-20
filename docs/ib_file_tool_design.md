# 投行承做文件处理工具搭建方案

## 1. 目标与范围

本工具面向投行承做团队，目标是将重复、易错、耗时的文件整理工作标准化和自动化，覆盖以下 3 个高频场景：

1. 合并多个同表头的 `dat/csv` 文件为一张汇总 `Excel`。
2. 签章页自动匹配与 PDF 拼接：按签章页文本中的协议名称匹配原件，并将原件最后一页替换为签章页。
3. 底稿资产池目录整理：根据 Excel 目录树（银行→客户）从资料池中抽取客户文件并重建目录结构。

---

## 2. 产品形态建议

推荐分两层：

- **执行引擎（Core Engine）**：Python 实现，负责文件扫描、解析、规则匹配、执行、日志和回滚。
- **交互层（UI）**：
  - MVP 期：命令行 + 配置文件（YAML）
  - 稳定期：桌面端（PySide6 / Electron）或 Web 内网服务（FastAPI + 前端）

建议先做 **CLI MVP**，2~4 周可落地并跑通全部场景，再封装为图形界面。

---

## 3. 总体架构

## 3.1 模块划分

1. `ingest`：文件读取层（csv/dat/excel/pdf）
2. `normalize`：命名规范化、编码处理、文本清洗
3. `match`：规则匹配引擎（协议名匹配、客户名匹配）
4. `transform`：具体任务处理器（merge、pdf_replace_last_page、asset_pool_export）
5. `audit`：日志、执行报告、差异记录、异常清单
6. `safety`：预览（dry-run）、备份、回滚、幂等控制

## 3.2 目录建议

```text
ib-file-tool/
  app/
    cli.py
    config.py
    tasks/
      merge_table.py
      pdf_signature_replace.py
      asset_pool_organize.py
    services/
      file_scanner.py
      excel_service.py
      pdf_service.py
      matcher.py
      name_normalizer.py
      report_service.py
  configs/
    default.yaml
  templates/
    mapping_template.xlsx
  output/
  logs/
```

---

## 4. 场景方案设计

## 4.1 场景一：同表头 dat/csv 合并为 Excel

### 输入

- 文件夹：包含多个 `*.csv`、`*.dat`
- 可选配置：
  - 编码优先级（`utf-8-sig`, `gbk`, `gb18030`）
  - 分隔符候选（`,`、`\t`、`|`）
  - 表头标准化映射（可选）

### 处理流程

1. 扫描文件并尝试多编码读取。
2. 自动识别分隔符。
3. 读取首行作为表头，去空格/全半角统一。
4. 校验表头一致性：
   - 严格模式：列名和顺序完全一致
   - 宽松模式：列名集合一致，按标准列顺序重排
5. 合并数据并追加来源字段：`source_file`, `source_row_no`, `import_time`。
6. 输出：`merged.xlsx`（主表）+ `rejects.xlsx`（异常文件记录）。

### 关键技术点

- 使用 `pandas` 读写，统一数据类型（尤其金额、日期、证券代码）。
- 大文件建议分块读取（`chunksize`）避免内存峰值。
- 输出前做基础质量检查：空值率、重复率、金额列格式。

---

## 4.2 场景二：PDF 签章页匹配并替换原文件最后一页

### 输入

- `signed_pages.pdf`：扫描签字页合集（每页一个签章页）
- `originals/`：原件文件夹，文件名以协议名称命名

### 假设与规则

- 每页签章页中可 OCR/文本提取到协议名称关键词。
- 原件文件名中包含协议名称，可通过规范化后匹配。

### 处理流程

1. **签章页拆分**：将 `signed_pages.pdf` 拆为单页流。
2. **文本提取**：
   - 优先 `PyMuPDF/pdfplumber` 提取可选文本；
   - 提取不到时走 OCR（`PaddleOCR` 或 `Tesseract`）。
3. **协议名识别**：基于关键词字典 + 正则抽取（如“XX协议”“XX合同”）。
4. **原件索引**：扫描 `originals/`，对文件名做标准化（去空格、括号、版本号等）。
5. **匹配策略**（按优先级）：
   - 精确匹配（标准化后全等）
   - 包含匹配（A 包含 B）
   - 模糊匹配（相似度阈值，如 0.88）
6. **替换逻辑**：
   - 打开原件 PDF，保留前 `n-1` 页
   - 追加对应签章页
   - 输出到 `output/replaced/`
7. **报告输出**：
   - `matched.xlsx`：匹配成功清单
   - `unmatched_signed.xlsx`：签章页未匹配
   - `unmatched_original.xlsx`：原件未命中
   - `ambiguous.xlsx`：多候选冲突需人工确认

### 风控与可追溯

- 默认不覆盖原文件（仅输出新文件）。
- 每个输出 PDF 写入元信息：匹配来源、算法版本、时间戳。
- 支持 `--dry-run`，只出匹配结果不改文件。

---

## 4.3 场景三：底稿资产池目录树驱动的资料抽取

### 输入

- 一张目录树 Excel（示例字段）
  - `银行`
  - `客户姓名`
  - 可选：`客户编号`、`证件号后四位`、`产品期次`
- 一个资料池文件夹（混合文档）

### 处理流程

1. 读取 Excel 目录树，生成目标结构（银行→客户）。
2. 扫描资料池并建立索引：
   - 文件名关键词
   - 文件内文本（可选，耗时）
3. 客户资料匹配（多键匹配）：
   - 客户姓名精确/规范化匹配
   - 客户编号、证件后四位辅助确认
4. 输出目录重建：
   - `output/资产池整理/{银行}/{客户}/...`
5. 生成核对清单：
   - 每客户命中文件数
   - 未命中客户
   - 重复命中（一个文件命中多个客户）

### 降低误匹配策略

- 姓名同名场景必须启用二级字段（编号/证件后四位）
- 设置“最低置信度阈值”，低于阈值进入人工复核队列
- 输出“匹配依据”字段（姓名命中、编号命中、文本命中）

---

## 5. 配置与规则中心

建议单独设计 `YAML` 配置，便于非开发人员维护。

示例：

```yaml
encoding_candidates: ["utf-8-sig", "gbk", "gb18030"]
delimiter_candidates: [",", "\t", "|"]
match:
  fuzzy_threshold: 0.88
  normalize_rules:
    remove_tokens: ["（", "）", "(", ")", "-扫描件", "-盖章版"]
pdf:
  ocr_enabled: true
  overwrite_original: false
safety:
  dry_run: true
  backup_enabled: true
```

---

## 6. 技术选型建议

- 语言：Python 3.11+
- 表格处理：`pandas`, `openpyxl`
- PDF 处理：`PyMuPDF`（fitz）, `pypdf`
- OCR：`PaddleOCR`（中文效果更好）
- 模糊匹配：`rapidfuzz`
- 接口层（可选）：`FastAPI`
- 桌面 GUI（可选）：`PySide6`
- 日志：`loguru` / `logging`

---

## 7. 质量与安全控制

1. **Dry-run**：必须支持，先看匹配与变更清单。
2. **不可变原件**：默认只输出到 `output/`，原始目录只读。
3. **可回滚**：记录每次任务输入哈希与输出路径。
4. **审计留痕**：运行日志 + 结果报表 + 错误栈。
5. **权限控制**（若做 Web）：按项目组隔离目录访问。
6. **敏感信息保护**：日志中对证件号、账号脱敏。

---

## 8. 交互设计（MVP）

CLI 示例：

```bash
python -m app.cli merge-table --input ./data/raw --output ./output/merged.xlsx
python -m app.cli replace-sign-page --signed ./data/signed_pages.pdf --originals ./data/originals --output ./output/replaced
python -m app.cli organize-asset-pool --tree ./data/tree.xlsx --source ./data/pool --output ./output/asset_pool
```

统一参数：

- `--config configs/default.yaml`
- `--dry-run`
- `--report ./output/report.xlsx`
- `--log-level INFO`

---

## 9. 实施路线图

### Phase 1（1~2 周）

- 完成场景 1 + 场景 3 的核心能力
- 建立统一日志、配置、报告框架

### Phase 2（2~3 周）

- 完成场景 2（含 OCR 兜底）
- 完善匹配策略与人工复核导出

### Phase 3（1~2 周）

- 上线 GUI/Web
- 增加批处理队列、任务历史、权限体系

---

## 10. 验收指标（建议）

1. 场景 1：10 万行级别合并成功率 100%，异常文件可追溯。
2. 场景 2：自动匹配准确率 ≥ 95%，其余可进入人工复核。
3. 场景 3：目录重建正确率 ≥ 98%，误匹配率 < 1%。
4. 单次任务全链路日志与报表齐全。

---

## 11. 后续可扩展能力

- 批量重命名与命名规范检查
- OCR 关键字段抽取（签署日期、主体名称）
- 多版本对比（合同版本 diff）
- 与 DMS/SharePoint/企业网盘对接

