# Comfrey 论文错误类型、案例与运行指南

本文档对齐 **Comfrey 论文（ICSE 2026）Table 2 / §4.3–§4.5** 与仓库 **README / PAPER_REPRO.md** 的正式复现路径。  
目标：先按论文分类整理错误类型，再给出每类的代表案例，并说明在本仓库中如何运行与查看结果。

---

## 1. 文档范围与边界

### 1.1 论文内（Comfrey 要处理的问题）

Comfrey 针对 **LLM/RAG 输出** 与下游软件期望之间的三类集成失败：

| 维度 | 论文缺陷（Table 2） | 章节 |
|------|---------------------|------|
| **Format** | 模板不一致（Template discrepancy） | §4.3.1 |
| **Format** | 分段不当（Improper data segmentation） | §4.3.2 |
| **Format** | 上下文构建错误（Incorrect context construction / RAG） | §4.3.3 |
| **Syntax** | 语法–解析器不对齐（Syntax-parser misalignment） | §4.4.1 |
| **Syntax** | 词法特征不一致（Inconsistent lexical features） | §4.4.2 |
| **Repetition** | 冗余软件行为（Redundant software behavior） | §4.5.1 |
| **Repetition** | 语义重复（Redundant semantics） | §4.5.2 |

判定口径（论文评估）：对每条测试，下游 **gold oracle** 在 **Comfrey 修复后** 仍失败 → 记为 Comfrey **未解决**（Table 3 的 `failures_after_repair`）。

### 1.2 论文外（本文档不当作 Comfrey 缺陷）

以下在 `paper-apps` 日志里常见，但 **不在** Table 2 中，也 **不能** 用 Comfrey 修复：

- Groq **429 / 限流**
- **aiohttp / TLS / ConnectionReset**（paper-qa 批量跑时）
- BabyAGI **子进程早退**（无 `TASK CREATION` 等输出）
- 依赖缺失、clone 失败、无 `GROQ_API_KEY`

### 1.3 本仓库三类「可运行」层级

| 层级 | 含义 | 能否列出「Comfrey 未解决的逐条任务」 |
|------|------|--------------------------------------|
| **A. 论文聚合** | `reproduce_paper_results.py` → `reproduction_outputs/` | 仅有 **5550** 等总数，无 task ID |
| **B. 正式复现链** | `run_paper_reproduction.py --phase all` | 当前为 **paper-apps**（四应用真实 LLM）；**非** 30k Comfrey 评估 |
| **C. 方法演示** | `Comfrey_2026-main/experiments/run_10_apps.py`（`comprehensive`） | **有** 逐条 `oracle_pass`；**不是** 论文 50/100 应用 |

论文 **30k 完整 Comfrey 跑批** 需要作者冻结日志；见 `REPRO_GAP.md`。

---

## 2. 论文规模：Comfrey 仍未解决的任务（聚合）

来源：`Comfrey_2026-main/reproduction_outputs/summary.json`（论文发表数字校验，**非本机重跑**）。

| 指标 | 数值 |
|------|------|
| 评估失败总数（无工具） | 15,124 |
| **Comfrey 修后仍失败（未解决）** | **5,550** |
| Comfrey 阻止的失败 | 9,574 |
| 修后通过率 | 81.5% |
| 检出 Recall / Precision | 75.1% / 96.6% |

**Table 4 消融（说明哪类修复最影响「未解决」）** — `paper_table4_ablation.csv`：

| 配置 | 阻止失败占比 | 相对完整版少阻止 |
|------|--------------|------------------|
| 完整 Comfrey | 63.3% | — |
| 去掉 Format 修复 | 24.9% | 5,806（60.6%） |
| 去掉 Syntax 修复 | 49.1% | 2,155（22.5%） |
| 去掉 Repetition 修复 | 38.0% | 3,827（40.0%） |

解读：残留 **5,550** 条在类型上仍属 Format/Syntax/Repetition；**Format 维（含 RAG 上下文）** 对「能否修到 oracle 通过」影响最大。

---

## 3. 错误类型总览（与仓库脚本映射）

```
论文 Table 2
├── Format
│   ├── 模板不一致      → run_10_apps: app01, app09；论文应用: BabyAGI, h2oGPT(ReAct)
│   ├── 分段不当        → run_10_apps: app06；RAG 切块
│   └── 上下文构建错误  → run_10_apps: app07；论文应用: paper-qa, Chroma QA
├── Syntax
│   ├── 解析器不对齐    → run_10_apps: app05；论文应用: LightGPT(SQL/代码)
│   └── 词法不一致      → run_10_apps: app08
└── Repetition
    ├── 冗余工具调用    → run_10_apps: app10
    └── 语义重复        → run_10_apps: app02；论文应用: BabyAGI 任务列表
```

---

## 4. Format 类

### 4.1 模板不一致（Template discrepancy）

**论文含义**：LLM 输出字段顺序、标签名、编号样式与下游模板/FSA 不一致。

**论文关联应用**：任务管理类（如 BabyAGI 任务列表）、Agent ReAct 轨迹（h2oGPT）。

#### 案例 A — 合成演示（`app01_task_format`）

- **坏输出**（`case=bad`）：

```text
Task 1: Prepare agenda
Task 2: Send invitation
```

- **期望**：编号用 `Task 1.` 而非 `Task 1:`（见 `oracle_01`）。
- **好输出**：

```text
Task 1. Prepare agenda
Task 2. Send invitation
```

#### 案例 B — ReAct 标签顺序（`app09_action_template`）

- **坏输出**：

```text
Action: search
Thought: Need product info
Action input: Sony WH-1000XM5
```

- **期望**：`Thought` → `Action` → `Action input` 顺序。

#### 案例 C — 论文点名应用 BabyAGI

- **论文问题**：任务创建/排序段落的列表格式、与 objective 回声（见 `paper_issue_oracles.babyagi_repetition_oracle` 中 `format` 分支）。
- **本仓 paper-apps**：`paper_issue_target: format`；需有效 `raw_llm_output` 才能评。

#### 如何运行

| 目的 | 命令（仓库根目录） |
|------|-------------------|
| **完整 Comfrey + 看修后仍 fail** | `cd Comfrey_2026-main` → `python experiments/run_10_apps.py --modes comfrey --inputs-per-app 20` |
| **无 Comfrey 基线** | `python experiments/run_10_apps.py --modes without --inputs-per-app 20` |
| **论文四应用 LLM** | `python scripts/run_paper_reproduction.py --phase paper-apps`（需 `GROQ_API_KEY`） |
| **BabyAGI + 运行时 Comfrey** | 见 `Comfrey_2026-main/REAL_REPRO.md`：`COMFREY_WRAP=1` + `run_matrix.py --apps babyagi` |

#### 如何查看

```powershell
# 10-app：Comfrey 仍失败条数
python -c "import json;from pathlib import Path;p=Path('Comfrey_2026-main/runs/raw/10apps_comfrey.jsonl');f=[json.loads(l) for l in p.read_text().splitlines() if l.strip() and not json.loads(l)['oracle_pass']];print(len(f),'still failing')"

# 报告
type Comfrey_2026-main\runs\metrics\10apps_report.md
```

本地 `10apps_comfrey` 快照：约 **10/200** 条 `app01_task_format` 在 Comfrey 后仍 `oracle_pass=false`。

---

### 4.2 分段不当（Improper data segmentation）

**论文含义**：RAG 或流式输出把词/句在错误边界切开（如 `comput- || er`）。

#### 案例 — `app06_rag_segmentation`

- **坏输出**：

```text
comput- || er vision is useful || for image retrieval
```

- **期望**：`||` 分隔块完整，无 `er`/`ing` 等孤立碎片，且含 `computer vision`。

#### 如何运行

```powershell
cd Comfrey_2026-main
python experiments/run_10_apps.py --modes comfrey without --inputs-per-app 20
```

输出：`runs/raw/10apps_comfrey.jsonl`，筛选 `app_id == "app06_rag_segmentation"`。

---

### 4.3 上下文构建错误（Incorrect context construction / RAG）

**论文含义**：检索进上下文的片段与问题无关，或回答承认「上下文无信息」。

**论文关联应用**：paper-qa、context-based QA（NarrativeQA / BBC / arXiv）。

#### 案例 A — 合成（`app07_rag_context`）

- **坏输出**：

```text
Context 1: The warranty covers battery replacement.
Context 2: Football teams often change tactics.
Answer: The warranty covers battery replacement.
```

- **问题**：Context 2 与问答无关（含 `football`），oracle 判 fail。

#### 案例 B — 本仓 paper-qa（有效 LLM 时）

- **检出**：`rag_context` oracle fail（`paper_issue_20260521T104212Z.jsonl`）。
- **典型证据**：回答写 “Based on the provided context…” 但上下文与问题关键词不匹配。

#### 如何运行

```powershell
# 论文 paper-apps（含 paper-qa）
cd c:\Users\ZHAIJIA\Desktop\comfrey-paper-repro-work
$env:GROQ_REQUEST_SLEEP_S = "3"
python scripts/run_paper_reproduction.py --phase paper-apps

# 合成 10-app
cd Comfrey_2026-main
python experiments/run_10_apps.py --modes comfrey --inputs-per-app 20

# 八类数据 + mock 注入 + full Comfrey（无外部 LLM）
python scripts/run_full_comfrey_all.py --mock-only
# 或限量：
python scripts/run_comfrey_experiment.py --mode mock --engine full --comfrey-config comprehensive --data-summary outputs/dataset_summary.csv --limit 50 --out outputs/comfrey_experiment_smoke50
```

#### 如何查看

```powershell
Get-ChildItem Comfrey_2026-main\runs\full_reproduction\paper_issue_logs\paper_issue_*.jsonl |
  Sort-Object LastWriteTime -Descending | Select-Object -First 1

python -c "
import json
from pathlib import Path
p = max(Path('Comfrey_2026-main/runs/full_reproduction/paper_issue_logs').glob('paper_issue_*.jsonl'), key=lambda x: x.stat().st_mtime)
for line in p.read_text(encoding='utf-8').splitlines():
    r = json.loads(line)
    if r.get('app_id')=='paper-qa' and r.get('detected_failure_type')=='rag_context':
        print(r.get('input_text','')[:80])
        print(r.get('answer_excerpt', r.get('raw_llm_output',''))[:200])
        print('---')
        break
"
```

---

### 4.4 JSON / 结构化格式（归入 Format 维）

#### 案例 — `app03_json_schema`

- **坏输出**：

```json
{"action": "search", "action_input": }
```

- **期望**：合法 JSON，且含 `thought`, `action`, `action_input`。

#### 案例 — `app04_code_fence`

- **坏输出**（无 fence）：

```python
def add(a, b):
    return a + b
```

- **期望**：包裹在 ` ```python ... ``` ` 中。

运行方式同 §4.1 的 `run_10_apps.py`。

---

## 5. Syntax 类

### 5.1 语法–解析器不对齐（Syntax-parser misalignment）

**论文含义**：生成 SQL/代码无法被 `sqlparse`、AST、编译器接受。

**论文关联应用**：LightGPT（程序合成/SQL）。

#### 案例 A — 合成（`app05_sql_syntax`）

- **坏输出**（缺右括号、缺逗号、无分号）：

```sql
CREATE TABLE chat_sessions (id SERIAL PRIMARY KEY timestamp TIMESTAMP DEFAULT NOW()
```

- **好输出**：

```sql
CREATE TABLE chat_sessions (id SERIAL PRIMARY KEY, timestamp TIMESTAMP DEFAULT NOW());
```

#### 案例 B — paper-apps / lightgpt

- **论文目标**：`paper_issue_target: syntax`
- **oracle**：`lightgpt_syntax_oracle`（`sqlparse` / AST 粗检）
- **本仓**：有效 LLM 约 7/300 条里曾出现 `syntax` fail

#### 如何运行

```powershell
cd c:\Users\ZHAIJIA\Desktop\comfrey-paper-repro-work
python scripts/run_paper_reproduction.py --phase paper-apps

# 仅 lightgpt（在 Comfrey_2026-main 下）
cd Comfrey_2026-main
python experiments/full_reproduction/run_paper_app_issue_replication.py --app-ids lightgpt --tests-per-app 10

# 合成
python experiments/run_10_apps.py --modes comfrey --inputs-per-app 20
```

---

### 5.2 词法特征不一致（Inconsistent lexical features）

**论文含义**：同一集成链路中英式/美式拼写、脚本混用等导致解析或规则不一致。

#### 案例 — `app08_lexical_consistency`

- **坏输出**：

```text
Please authorize the user and set the colour preference.
```

- **期望**：与下游一致用 `color`（`authorize` + `color` 同时出现则 pass）。

```powershell
cd Comfrey_2026-main
python experiments/run_10_apps.py --modes comfrey without --inputs-per-app 20
```

---

## 6. Repetition 类

### 6.1 语义重复（Redundant semantics）

**论文含义**：任务列表、段落间语义近重复。

#### 案例 A — 合成（`app02_task_repetition`）

- **坏输出**：

```text
Task 1. Prepare thank-you messages
Task 2. Write thank-you notes
Task 3. Send appreciation messages
```

- **问题**：`thank` / `appreciation` 重复度过高（`oracle_02`）。

#### 案例 B — BabyAGI

- **检测**：`babyagi_repetition_oracle`（重复任务行、与 objective 回声 → `repetition` 或 `format`）。

```powershell
python scripts/run_paper_reproduction.py --phase paper-apps
# 或 REAL_REPRO：COMFREY_WRAP=1 对比 baseline
```

---

### 6.2 冗余软件行为（Redundant software behavior）

**论文含义**：同一工具/API 被重复调用且输入相同。

#### 案例 — `app10_tool_reinvocation`

- **坏输出**：

```text
SEARCH: area of France
SEARCH: area of France
RESULT: 551,695 square kilometers
```

- **期望**：仅一行 `SEARCH: area of France`。

```powershell
cd Comfrey_2026-main
python experiments/run_10_apps.py --modes comfrey --inputs-per-app 20
```

---

## 7. 用「完整 Comfrey」查看「修后仍失败」的标准流程

### 7.1 前置（README）

```powershell
cd c:\Users\ZHAIJIA\Desktop\comfrey-paper-repro-work

pip install datasets pandas tqdm

# 若无 8×300 数据
python scripts/prepare_datasets.py --limit 300 --out datasets/processed --fail-log outputs/dataset_prepare_failures.csv --job-timeout 600

# comprehensive 嵌入模型
python scripts/download_comfrey_embedding_model.py
python scripts/set_hf_offline_env.py

# Groq（paper-apps / 真实应用）
python scripts/persist_groq_env.py
```

### 7.2 路径一：论文正式复现（真实四应用 LLM）

```powershell
$env:GROQ_REQUEST_SLEEP_S = "3"
python scripts/run_paper_reproduction.py --phase all
```

| 产物 | 路径 |
|------|------|
| 规模校验 | `Comfrey_2026-main/runs/full_reproduction/test_inputs/build_report.json` |
| 应用日志 | `Comfrey_2026-main/runs/full_reproduction/paper_issue_logs/paper_issue_*.jsonl` |
| 汇总 | `Comfrey_2026-main/runs/full_reproduction/paper_app_issue_replication_report.json` |

**说明**：该阶段 **默认不在运行时包 Comfrey**；`comfrey_repair_audit` 仅作审计字段。要看 **运行时 comprehensive Comfrey**，用路径二或三。

### 7.3 路径二：8 数据集 × mock 注入 + full Comfrey

```powershell
python scripts/run_full_comfrey_all.py --mock-only
# 输出目录（若跑通）：
# member1_outputs/comfrey_mock_full_comprehensive/
```

或：

```powershell
python scripts/run_comfrey_experiment.py --mode mock --engine full --comfrey-config comprehensive --data-summary outputs/dataset_summary.csv --out outputs/comfrey_experiment
```

查看修后仍有问题：

```powershell
# metrics.json 中 detected vs repaired_failures
type outputs\comfrey_experiment\metrics.json
type outputs\comfrey_experiment\repaired_outputs.jsonl
```

### 7.4 路径三：10 合成应用 + `create_comprehensive_config()`（逐条 oracle）

```powershell
cd Comfrey_2026-main
python experiments/run_10_apps.py --modes without comfrey retry json_schema --inputs-per-app 20
```

| 文件 | 内容 |
|------|------|
| `runs/raw/10apps_without.jsonl` | 无工具 |
| `runs/raw/10apps_comfrey.jsonl` | **完整 Comfrey** |
| `runs/metrics/10apps_summary.csv` | 通过率对比 |
| `runs/metrics/10apps_report.md` | 摘要表 |

筛选 Comfrey 未解决（`oracle_pass == false`）：

```powershell
python -c "
import json
from pathlib import Path
p = Path('runs/raw/10apps_comfrey.jsonl')
for line in p.read_text(encoding='utf-8').splitlines():
    r = json.loads(line)
    if not r.get('oracle_pass'):
        print(r['app_id'], r['failure_kind'], (r.get('output') or '')[:100])
"
```

**本地快照（200 runs / mode）**：

| app_id | failure_kind | Comfrey 后仍 fail（约） |
|--------|--------------|-------------------------|
| app03_json_schema | format | 20 |
| app04_code_fence | format | 20 |
| app05_sql_syntax | syntax | 20 |
| app06_rag_segmentation | format | 20 |
| app07_rag_context | format | 20 |
| app09_action_template | format | 20 |
| app01_task_format | format | 10 |
| app02_task_repetition | repetition | 10 |

### 7.5 路径四：论文 Table 数字（无逐条任务）

```powershell
cd Comfrey_2026-main
python reproduce_paper_results.py
type reproduction_outputs\REPORT.md
type reproduction_outputs\paper_table3_detection_prevention.csv
```

---

## 8. 消融：观察「哪类错误 Comfrey 修不动」

在 `Comfrey_2026-main` 下：

```powershell
python experiments/run_10_apps.py --modes comfrey no_format no_syntax no_repetition --inputs-per-app 20
```

| mode | 含义 |
|------|------|
| `comfrey` | 完整三维修复 |
| `no_format` | 关闭 Format 修复 |
| `no_syntax` | 关闭 Syntax 修复 |
| `no_repetition` | 关闭 Repetition 修复 |

对比 `runs/metrics/10apps_summary.csv` 中 `failures` / `pass_rate`，与论文 Table 4 趋势一致：去掉 **Format** 后失败数上升最多。

---

## 9. 快速对照表

| 你想看… | 推荐命令 | 结果文件 |
|---------|----------|----------|
| 论文 5550 未解决（仅总数） | `python Comfrey_2026-main/reproduce_paper_results.py` | `reproduction_outputs/summary.json` |
| 完整 Comfrey + 逐条是否通过 | `cd Comfrey_2026-main` → `python experiments/run_10_apps.py --modes comfrey` | `runs/raw/10apps_comfrey.jsonl` |
| 真实 BabyAGI / paper-qa / lightgpt | `python scripts/run_paper_reproduction.py --phase paper-apps` | `paper_issue_logs/*.jsonl` |
| 八类数据 mock + full engine | `python scripts/run_full_comfrey_all.py --mock-only` | `member1_outputs/...`（需跑通后生成） |

---

## 10. 相关文档

- [PAPER_REPRO.md](../PAPER_REPRO.md) — 严格全量复现入口  
- [README.md](../README.md) — 数据集与 `run_full_comfrey_all.py`  
- [REPRO_GAP.md](../REPRO_GAP.md) — 30k 日志与 50/100 应用清单缺口  
- [Comfrey_2026-main/REAL_REPRO.md](../Comfrey_2026-main/REAL_REPRO.md) — BabyAGI `COMFREY_WRAP` 对比  
- [Comfrey_2026-main/reproduction_outputs/paper_table2_strategies.csv](../Comfrey_2026-main/reproduction_outputs/paper_table2_strategies.csv) — 论文策略表  

---

*生成说明：分类与章节号对齐 Comfrey 论文 §4.3–§4.5；案例来自 `experiments/run_10_apps.py` 与 `paper_issue_oracles.py`；运行命令对齐 README / PAPER_REPRO.md。*
