# 论文外错误 · 实验对比汇总（每次运行自动更新）

- **生成时间**：2026-05-20 10:49:17 UTC
- **本文件路径**：`member1_outputs/ISSUES_RUN_SUMMARY.md`
- **说明**：下方「使用过 Comfrey 的结果」随 `run_comfrey_experiment` / `run_real_apps_experiment` 跑批更新；数据准备案例在运行 `prepare_datasets.py` 后刷新本文件也会更新数据集段。

---

## 三个案例总览

| 案例 | 文档章节 | 运行命令 | 有无 Comfrey 修前/修后 |
|------|----------|----------|------------------------|
| **案例一** 替代/子集数据 | §二 案例1 | `prepare_datasets.py` | 无（仅输入数据） |
| **案例二** Mock 检测缺口 | §三 案例1 | `run_comfrey_experiment.py --mode mock` | 有 |
| **案例三** 真实应用 + BOM | §二 案例2 + §三 案例2 | `run_real_apps_experiment.py` | 有 |

---

## 案例一：替代数据集 / 子集（§二 案例1）

### 运行命令

```powershell
cd C:/Users/ZHAIJIA/Desktop/comfrey-paper-repro-work
python scripts/prepare_datasets.py --limit 300 --out datasets/processed --fail-log outputs/dataset_prepare_failures.csv --job-timeout 600
```

### 对比文件（本案例不使用 Comfrey）

| 用途 | 路径 |
|------|------|
| 八类数据登记 | `outputs/dataset_summary.csv` |
| 实际输入 | `datasets/processed/*.jsonl` |
| 替代集说明 | `docs/dataset_sources.md` |

### 本次运行结果（数据准备）

| 数据集 | 文件 | 条数 | 状态 |
|--------|------|------|------|
| MultiWOZ | `datasets/processed/multiwoz.jsonl` | 300 | success |
| Taskmaster-1 | `datasets/processed/taskmaster1.jsonl` | 300 | success |
| MBPP | `datasets/processed/mbpp.jsonl` | 300 | success |
| DS-1000 | `datasets/processed/ds1000.jsonl` | 300 | success |
| APPS | `datasets/processed/apps.jsonl` | 300 | success |
| NarrativeQA | `datasets/processed/narrativeqa.jsonl` | 300 | success |
| BBC News | `datasets/processed/bbc_news.jsonl` | 300 | success |
| arXiv | `datasets/processed/arxiv.jsonl` | 300 | success |

### 定位输入 — 复制搜索

```powershell
Import-Csv outputs/dataset_summary.csv
Get-Content datasets/processed/multiwoz.jsonl -TotalCount 3
```

---

## 案例二：Mock 注入 — Comfrey 修前 / 修后（§三 案例1）

### 运行命令

```powershell
python scripts/run_comfrey_experiment.py --data-summary outputs/dataset_summary.csv --mode mock --limit 15 --out member1_outputs/comfrey_mock_rerun --open-report
```

### 对比文件位置

| | 路径 | 关键字段 |
|--|------|----------|
| **修前** | `member1_outputs/comfrey_mock_rerun/raw_outputs.jsonl` | `scan.issues`、`raw_output`、`injection_kind` |
| **修后** | `member1_outputs/comfrey_mock_rerun/repaired_outputs.jsonl` | `scan_after.issues`、`repaired_output`、`repair_actions` |
| **数字汇总** | `member1_outputs/comfrey_mock_rerun/metrics.json` | `pass_rate_before` / `pass_rate_after` |
| **逐条总表** | `member1_outputs/comfrey_mock_rerun/comfrey_comparison_all.csv` | 列 `status`、`issues_before`、`issues_after` |
| **可读报告** | `member1_outputs/comfrey_mock_rerun/COMFREY_BEFORE_AFTER.md` | 仍失败 / 漏检列表 |

### 使用过 Comfrey 的结果（随 Mock 运行更新）

- **修前（≈未用 Comfrey 修复）**：`pass_rate_before` = **0.325**
- **修后（用过 Comfrey）**：`pass_rate_after` = **0.991667**
- `total_cases` = **120**
- `injected_failures` = **120**
- `detected_failures` = **81**
- `repaired_failures` = **80**
- `detection_recall` = **0.675**
- `repair_rate` = **0.987654**
- **漏检**（注入但修前未检出）：**39** 条 （其中修后仍 fail：**0**，修后碰巧通过：**39**）
- **检出未修好**：**1** 条
- **已修好**：**80** 条

#### 仍无法通过（修后仍 fail）— 输入定位

  - `taskmaster1.jsonl:6:110` | 注入 `repetition` → 源数据 datasets/processed/taskmaster1.jsonl 第 7 行（0-based 索引 6）

```powershell
Select-String -Path "member1_outputs/comfrey_mock_rerun/raw_outputs.jsonl" -Pattern "<test_id>"
Select-String -Path "member1_outputs/comfrey_mock_rerun/repaired_outputs.jsonl" -Pattern "<test_id>"
```

#### 漏检示例（修前 issues 为空）— 输入定位

  - `apps.jsonl:3:2` | 注入 `repetition` → 源数据 datasets/processed/apps.jsonl 第 4 行（0-based 索引 3）
  - `apps.jsonl:6:5` | 注入 `repetition` → 源数据 datasets/processed/apps.jsonl 第 7 行（0-based 索引 6）
  - `apps.jsonl:9:8` | 注入 `repetition` → 源数据 datasets/processed/apps.jsonl 第 10 行（0-based 索引 9）
  - `apps.jsonl:12:11` | 注入 `repetition` → 源数据 datasets/processed/apps.jsonl 第 13 行（0-based 索引 12）
  - `apps.jsonl:15:14` | 注入 `repetition` → 源数据 datasets/processed/apps.jsonl 第 16 行（0-based 索引 15）
  - `arxiv.jsonl:3:17` | 注入 `repetition` → 源数据 datasets/processed/arxiv.jsonl 第 4 行（0-based 索引 3）
  - `arxiv.jsonl:6:20` | 注入 `repetition` → 源数据 datasets/processed/arxiv.jsonl 第 7 行（0-based 索引 6）
  - `arxiv.jsonl:9:23` | 注入 `repetition` → 源数据 datasets/processed/arxiv.jsonl 第 10 行（0-based 索引 9）

CSV 筛漏检：`Import-Csv member1_outputs/comfrey_mock_rerun/comfrey_comparison_all.csv | Where-Object status -like 'missed*'`

---

## 案例三：真实应用 + BOM（§二 案例2 + §三 案例2）

### 运行命令

```powershell
python scripts/run_real_apps_experiment.py --data-summary outputs/dataset_summary.csv --limit 2 --out member1_outputs/real_apps_rerun --llm-provider ollama --model qwen2.5:3b
```

### 对比文件位置

| | 路径 | 关键字段 |
|--|------|----------|
| **修前** | `member1_outputs/real_apps_rerun/app_raw_outputs.jsonl` | `detected_before`、`context`、`raw_app_output` |
| **修后** | `member1_outputs/real_apps_rerun/app_repaired_outputs.jsonl` | `detected_after`、`repaired_output` |
| **数字汇总** | `member1_outputs/real_apps_rerun/app_metrics.json` | `pass_rate_before` / `pass_rate_after`、`failure_breakdown` |
| **可读报告** | `member1_outputs/real_apps_rerun/COMFREY_BEFORE_AFTER.md` | 未执行 / 未修好 |

### 使用过 Comfrey 的结果（随真实应用运行更新）

- **修前（≈未用 Comfrey 修复）**：`pass_rate_before` = **0.0**
- **修后（用过 Comfrey）**：`pass_rate_after` = **0.0**
- `total_cases` = **8**
- `detected_failures` = **2**
- `repaired_failures` = **0**
- `app_success` = **2**
- `app_failed` = **6**
- **failure_breakdown**：
  - missing_GROQ_API_KEY (BabyAGI fork requires Groq): **2**
  - Traceback (most recent call last):: **2**
  - URLError:<urlopen error [WinError 10061] 由于目标计算机积极拒绝，无法连接。>: **2**
- **Comfrey 未执行**（app_error）：**2** 条
- **检出未修好**：**0** 条
- **已修好**：**0** 条

#### §二 案例2 — BOM（工程噪声）— 输入定位

- （未在 `app_raw_outputs.jsonl` 中检测到 BOM，或尚未跑真实应用）

#### §三 案例2 — Comfrey 未执行 — 输入定位

  - `babyagi:taskmaster1:0` | missing_GROQ_API_KEY (BabyAGI fork requires Groq) → 源数据 datasets/processed/taskmaster1.jsonl 第 1 行（0-based 索引 0）
  - `babyagi:taskmaster1:1` | missing_GROQ_API_KEY (BabyAGI fork requires Groq) → 源数据 datasets/processed/taskmaster1.jsonl 第 2 行（0-based 索引 1）

#### §三 案例2 — 检出未修好 — 输入定位

  - `h2ogpt:multiwoz:4` | 修前 `['format']` → 源数据 datasets/processed/multiwoz.jsonl 第 5 行（0-based 索引 4）
  - `h2ogpt:multiwoz:5` | 修前 `['format']` → 源数据 datasets/processed/multiwoz.jsonl 第 6 行（0-based 索引 5）

```powershell
Select-String -Path "member1_outputs/real_apps_rerun/app_raw_outputs.jsonl" -Pattern "<test_id>"
Select-String -Path "member1_outputs/real_apps_rerun/app_repaired_outputs.jsonl" -Pattern "<test_id>"
```

---

## 手动刷新本汇总

```powershell
python scripts/generate_issues_run_summary.py
python scripts/generate_issues_run_summary.py --open
```

*由 `scripts/generate_issues_run_summary.py` 自动生成。*