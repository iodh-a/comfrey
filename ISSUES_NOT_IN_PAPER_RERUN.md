# 复现过程中发现的论文外错误记录

> 生成说明：本文档在仓库根目录 `comfrey-paper-repro-work` 下 **实际执行脚本后** 根据终端 JSON 摘要、`member1_outputs` 与 `outputs` 下产物整理。日期：2026-05-16


## 一、实验输入、应用结果与本文记录范围

**1. 本次实验涉及的数据集**

下列八类数据由 `scripts/prepare_datasets.py` 从公开源导出到 `datasets/processed/`，并在 `outputs/dataset_summary.csv` 中登记路径（本次复跑后 **每个文件 300 条**，见下文「命令」）：

| 逻辑名 | 输出文件 | 说明 |
|--------|----------|------|
| MultiWOZ | `multiwoz.jsonl` | 对话类 |
| Taskmaster-1 | `taskmaster1.jsonl` | 任务管理类 |
| MBPP | `mbpp.jsonl` | 程序合成 |
| DS-1000 | `ds1000.jsonl` | 程序合成 |
| APPS | `apps.jsonl` | 程序合成 |
| NarrativeQA | `narrativeqa.jsonl` | 长文 QA |
| BBC News | `bbc_news.jsonl` | 新闻类（README 注明为 **Substitute** 数据集） |
| arXiv | `arxiv.jsonl` | 摘要元数据（README 注明为 **Substitute**） |

更细的上游 HF 数据集名见仓库内 `docs/dataset_sources.md`。

**2. 本次实际跑过的应用 / 管线**

| 类型 | 内容 | 结果概览 |
|------|------|----------|
| **Harness：mock Comfrey** | `scripts/run_comfrey_experiment.py`，对八份 JSONL 各取 15 行注入再检测修复 | **成功结束**；指标见 §四案例 |
| **Harness：真实应用子实验** | `scripts/run_real_apps_experiment.py`，四应用各调度 2 条（`--limit 2`） | **6 条应用级成功，2 条失败**（均为 BabyAGI） |
| **数据准备** | `scripts/prepare_datasets.py` 两次（先 `--limit 5` 用于探测，再 `--limit 300` **恢复**仓库常用规模） | 八数据集均 **success**；小样本阶段的失败日志见 `member1_outputs/dataset_prepare_failures_rerun.csv`（**仅表头，无失败行**） |

涉及的开源/上游对齐名称：**BabyAGI（Groq fork）**、**PaperQA**、**h2oGPT**、**LightGPT 风格代码生成脚本**（见 §三、§四案例中的链接）。

**3. 本文记录重点**

- 论文中没有出现的错误。

---

## 二、错误类型一：本地导出数据与「论文级严格输入」或上游镜像不完全一致

**总体说明**：论文可能引用某基准名（如 MultiWOZ、BBC 新闻正文等）；本仓库为可复现与体量控制，采用 **HF 上的替代打包或子集**，属于 **数据工程上的差异**。论文 **通常不会** 把「用了 HF 替代镜像」单独列为一种应用输出错误。

### 案例 1：八类基准 JSONL 全量成功，但存在「替换/子集」与论文全量实验的差距

- **开源项目链接**：数据通过 Hugging Face [`datasets`](https://github.com/huggingface/datasets) 库拉取；各具体源见 `docs/dataset_sources.md`（例如 MultiWOZ 使用 `Brendan/multiwoz_turns_v22` 等 **Substitute packaging** 说明）。  
- **使用数据集**：上述八类；本次写回 **`--limit 300` 每文件**。  
- **运行命令**：

```powershell
python scripts/prepare_datasets.py --limit 300 --out datasets/processed --fail-log outputs/dataset_prepare_failures.csv --job-timeout 600
```

（此前为探测曾执行 `--limit 5` 并写入 `member1_outputs/dataset_prepare_failures_rerun.csv`，**无失败行**；随后用本命令恢复为 300 条/文件。）

- **输出目录**：`datasets/processed/*.jsonl`；汇总 `outputs/dataset_summary.csv`。  
- **最后输出**：`dataset_summary.csv` 中八行均为 `status=success`，`num_samples=300`。  
- **观察到的问题**：与论文可能使用的 **原始发布形态或全量规模** 不一定一致；README 已写明 BBC、arXiv 等为 **Substitute**（见根目录 `README.md` 表格）。本次复现 **未暴露** 下载失败，但 **暴露的是方法论风险**：若读者忽略 `meta.substitute_note`，会把结果误当成「与论文完全同分布的输入」。  
- **与论文口径的区别**：属于 **数据来源与论文叙述范围之间的 gap**，不是模型吐出来的 format/syntax/repetition。

### 案例 2：NarrativeQA 上下文带 UTF-8 BOM 前缀（工程噪声）

- **开源项目链接**：[PaperQA / paper-qa](https://github.com/Future-House/paper-qa)（本批真实子实验中 PaperQA 使用该库路径：`Comfrey_2026-main/real_apps/_paper_candidates/paper-qa`）。  
- **使用数据集**：`narrativeqa.jsonl` 中前两行（`paperqa:narrativeqa.jsonl:0` / `:1`）。  
- **运行命令**：python scripts/run_real_apps_experiment.py --data-summary outputs/dataset_summary.csv --limit 2 --out member1_outputs/real_apps_rerun --llm-provider ollama --model qwen2.5:3b  
- **输出目录**：`member1_outputs/real_apps_rerun/`。  
- **最后输出**：PaperQA 两条在 `app_metrics.json` 中计入成功；`app_raw_outputs.jsonl` 内 `context` 字段开头可见 **`ï»¿`**（UTF-8 BOM），属 **导出/拼接文本时的编码残留**。  
- **观察到的问题**：对 RAG/解析可能引入 **额外噪声**；论文一般不会把「BOM 字符」列为与三类输出错误并列的研究对象。  
- **与论文口径的区别**：**数据清洗与管线工程问题**，不是论文核心归纳的语义/格式错误类型。

---

## 三、错误类型二：Harness 检测 / 修复链路与「注入失败标签」不完全对齐（指标层暴露）

**总体说明**：`run_comfrey_experiment.py --mode mock` 会 **保证注入失败**，但 `comfrey_lite` **未必每条都能检出或修回**；`detection_recall` 等数字反映的是 **本仓库轻量实现的能力边界**，不是论文里对真实应用错误的完备枚举。

### 案例 1：Mock 注入 120 条失败，仅检出 81 条（检测召回约 0.675）

- **开源项目链接**：检测/修复逻辑在本仓库 `scripts/comfrey_lite.py`（非独立 GitHub 产品）；输入数据来自八份 JSONL（上游 HF 数据经 `prepare_datasets.py` 处理，见 §三）。  
- **使用数据集**：`outputs/dataset_summary.csv` 指向的八份 `datasets/processed/*.jsonl`，**每文件最多 15 行**（`8 × 15 = 120` 条总用例）。  
- **运行命令**：

```powershell
python scripts/run_comfrey_experiment.py --data-summary outputs/dataset_summary.csv --mode mock --limit 15 --out member1_outputs/comfrey_mock_rerun
```

- **输出目录**：`member1_outputs/comfrey_mock_rerun/`  
- **最后输出**（`metrics.json` 原文摘要）：

```json
{
  "mode": "mock",
  "total_cases": 120,
  "injected_failures": 120,
  "detected_failures": 81,
  "repaired_failures": 80,
  "detection_recall": 0.675,
  "repair_rate": 0.987654,
  "pass_rate_before": 0.325,
  "pass_rate_after": 0.991667,
  "detected_targeted_hits": 81
}
```

- **观察到的问题**：在 **明知已注入失败** 的前提下，仍有约 **39 条** 未被记为 `detected_failures`（`120 - 81`）；修复率按已检出子集约 **98.8%**，但整体 **「注入 → 检出」链路存在缺口**。  
- **与论文口径的区别**：论文研究 **应用输出错误**；此处暴露的是 **本复现 harness 的检测召回不足**，属于 **方法论与实现限制**，不应直接说成「某应用漏检了论文中的某类错误」。

### 案例 2：真实子实验中「模型输出被标为 format」与「BabyAGI 完全未跑」并存

- **开源项目链接**：[LightGPT 对齐仓库](https://github.com/darshit001/LightGPT)；本批调用的是仓库内脚本 `Comfrey_2026-main/runs/paper_apps/lightgpt_groq_sql_repro.py`（见 `app_metrics.json` 的 `paths.lightgpt_reference`）。  
- **使用数据集**：`mbpp.jsonl`（`lightgpt:mbpp.jsonl:7` 等在 `app_metrics.json` 的 `example_detected_nonpass` 中有摘要）。  
- **运行命令**：同 §二案例 1。  
- **输出目录**：`member1_outputs/real_apps_rerun/`。  
- **最后输出**：`app_metrics.json` 中 `detected_failures` **1**（LightGPT 一条被 `comfrey_lite` 标为 `format`）；`repaired_failures` **0**；`avg_latency_s` **约 7.13**。  
- **观察到的问题**：在 **同一批 `--limit 2`** 下，**部分应用** 走通 Ollama，**BabyAGI** 却因密钥 **完全未调用模型**——整批实验的 **可比性** 受工程条件影响，而不是单一「应用错误类型」能概括。  
- **与论文口径的区别**：论文外错误体现在 **实验条件不齐、指标混合了「未运行」与「已运行」**，不是论文对三类输出错误的定义本身。

---

**本次实际执行的命令汇总（按顺序）**

1. `New-Item -ItemType Directory -Force -Path member1_outputs`  
2. `python scripts/run_comfrey_experiment.py --data-summary outputs/dataset_summary.csv --mode mock --limit 15 --out member1_outputs/comfrey_mock_rerun`  
3. `python scripts/run_real_apps_experiment.py --data-summary outputs/dataset_summary.csv --limit 2 --out member1_outputs/real_apps_rerun --llm-provider ollama --model qwen2.5:3b`  
4. `python scripts/prepare_datasets.py --limit 5 --out datasets/processed --fail-log member1_outputs/dataset_prepare_failures_rerun.csv --job-timeout 180`（探测用）  
5. `python scripts/prepare_datasets.py --limit 300 --out datasets/processed --fail-log outputs/dataset_prepare_failures.csv --job-timeout 600`（**恢复**常用 300 条/文件）

---

**归类对照（便于检索）**

| 论文外错误类型 | 归入章节 |
|----------------|----------|
| 替换数据集 / 子集规模 / BOM 等数据工程差异 | **§二** |
| Mock 检测召回不足、真实批混合成功与未跑通应用 | **§三** |
