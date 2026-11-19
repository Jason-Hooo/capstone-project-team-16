# ATLAS：Atomic Fact To LAy Summary

ATLAS 是一個以「原子事實（atomic fact）」為核心的生醫論文**通俗摘要（lay summary）**生成框架，本專案為其論文實驗程式碼，資料集使用 BioLaySumm 2025（PLOS、eLife）。

ATLAS 用三個模組分別降低 LLM 直接生成摘要時常見的三種錯誤：

| 模組 | 處理的問題 | 做法 |
|---|---|---|
| Module 1 | 遺漏重要內容 | 從全文抽出大量候選原子事實（重視召回率） |
| Module 2 | 幻覺（寫出原文不支持的內容） | 只保留能被 Abstract 支持的事實，再依此生成摘要 |
| Module 3 | 表達錯誤（寫錯、寫模糊、方向相反） | 每個事實出一題 1T3F+NOTA 選擇題，摘要答錯就做最小幅度改寫 |

**1T3F+NOTA**：1 個正確選項（原子事實本身）＋ 3 個相近但錯誤的干擾選項 ＋ E「None of the above」。作答模型只能依據摘要作答，答錯即表示摘要沒有正確保留該事實。

```text
[輸入] 全文 + Abstract
   ▼
[Module 1]  extract_candidate_atomic_facts   全文 → 候選原子事實
   ▼
[Module 2]  filter_against_abstract          逐條判斷是否被 Abstract 支持
   ▼
[摘要生成]  generate_fact_driven_summary     Abstract + 最終事實 → 初版摘要
   ▼
[Module 3a] generate_question                每個最終事實各出一題
   ▼
[Module 3b] verify_and_rewrite_if_needed     依序逐題作答；答錯則由 Module 3c 改寫，
   ▼                                         改寫後的摘要作為下一題的輸入
[輸出] 最終通俗摘要
```

---

## 安裝與設定

```bash
pip install -r requirements.txt                    # API 模型 + 資料集 + 評估指標
pip install -r requirements-local-backbones.txt    # 選用：本地開源模型（需 CUDA GPU）
```

程式**不會讀取 `.env`**，API 金鑰需設為環境變數：

```bash
export OPENAI_API_KEY=...   # GPT 系列
export GEMINI_API_KEY=...   # Gemini 系列
export HF_TOKEN=...         # 選用：下載資料集、Llama 等受限模型
```

---

## 輸入資料格式

輸入為 JSONL，每行一篇文章，以下欄位缺一不可（`atlas.io.load_articles` 會驗證：欄位齊全、`id` 唯一、內容非空）：

```json
{"id": "plos_0", "source_dataset": "PLOS", "article": "全文", "abstract": "摘要", "expert_summary": "專家通俗摘要"}
```

可用腳本直接從 HuggingFace 下載並轉換 BioLaySumm 資料：

```bash
python -m experiments.prepare_biolaysumm_input --split validation --output inputs/validation.jsonl
```

---

## 如何使用 ATLAS 模組

> 請在**專案根目錄**執行（Python 腳本或 `python -m ...`），否則無法 `import atlas`。

### 1. 批次執行完整流程

```python
from pathlib import Path
from atlas import AtlasPipeline
from atlas.io import load_articles

articles = load_articles(Path("inputs/validation.jsonl"))
pipeline = AtlasPipeline("gpt41_mini")                  # 指定骨幹模型簡稱
results = pipeline.run(articles, Path("outputs/atlas"))  # 所有中間結果寫入 outputs/atlas/gpt41_mini/
print(results[0]["final_summary"])
```

`run()` 回傳每篇文章的結果 dict，並在 `<output_dir>/<model_key>/` 寫出：

| 檔案 | 內容 |
|---|---|
| `module_1_candidate_atomic_facts.jsonl` | 候選事實（`fact_id`、`atomic_fact`、`source_span`、`selection_reason`） |
| `module_2_final_atomic_facts.jsonl` | 通過過濾的事實（另含 `abstract_support`、`filter_reason`） |
| `fact_driven_lay_summaries.jsonl` | Module 3 之前的初版摘要 |
| `module_3_questions.jsonl` | 每個事實的題目、選項、正確答案、干擾策略 |
| `module_3_verification_and_targeted_rewrites.jsonl` | 每題作答結果與改寫前後摘要 |
| `final_lay_summaries.jsonl` | **最終摘要**（`article_id`、`source_dataset`、`summary`），可直接送評估腳本 |

### 2. 單篇執行

```python
result = pipeline.run_article(articles[0])   # 不寫檔
```

回傳鍵值：`article_id`、`source_dataset`、`candidate_atomic_facts`、`final_atomic_facts`、`initial_summary`、`questions`、`verification_and_rewrites`、`final_summary`。

### 3. 逐步呼叫各模組（除錯或替換某一步時使用）

```python
article = articles[0]
aid = str(article["id"])

candidates  = pipeline.extract_candidate_atomic_facts(aid, article["article"])   # Module 1
final_facts = pipeline.filter_against_abstract(article["abstract"], candidates)  # Module 2
summary     = pipeline.generate_fact_driven_summary(article["abstract"], final_facts)

for fact in final_facts:                                                          # Module 3
    question = pipeline.generate_question(fact)
    check = pipeline.verify_and_rewrite_if_needed(summary, question)
    summary = check["summary_after"]        # 務必接續使用改寫後的摘要
```

### 4. 直接生成基線

```python
from atlas.pipeline import run_direct_generation
run_direct_generation(articles, "gpt41_mini", Path("outputs/direct"))  # → outputs/direct/gpt41_mini/direct_lay_summaries.jsonl
```

### 5. 可用模型（`atlas/clients.py` 的 `MODEL_SPECS`）

| 簡稱 | 實際模型 | 呼叫方式 |
|---|---|---|
| `gpt41` / `gpt41_mini` / `gpt4o_mini` | gpt-4.1 / gpt-4.1-mini / gpt-4o-mini | OpenAI Chat（strict JSON schema） |
| `gpt54_mini` | gpt-5.4-mini-2026-03-17（reasoning: medium） | OpenAI Responses API |
| `gemini25_flash` / `gemini3_flash_preview` | gemini-2.5-flash / gemini-3-flash-preview | Google GenAI |
| `qwen25_7b_instruct` / `llama3_8b_instruct` | Qwen2.5-7B-Instruct / Llama-3-8B-Instruct | 本地 transformers（需 CUDA） |

新增模型：在 `MODEL_SPECS` 加一筆 `"簡稱": ModelSpec(provider, model)`，`provider` 為 `openai`、`openai_responses`、`gemini`、`local_hf` 之一。

也可傳入自訂用戶端（例如測試用的假模型），只要它有 `generate(instruction, inputs, schema) -> dict` 方法：

```python
pipeline = AtlasPipeline("gpt41_mini", client=my_client)
```

### 使用注意事項

- **錯誤會直接拋出例外**：模型回傳空欄位、選項重複、Module 2 丟棄全部事實等情況都會中斷，沒有重試或續跑機制，大量資料建議自行分批。
- **成本**：Module 2 每個事實呼叫 1 次，Module 3 每個事實呼叫 2–3 次，總呼叫量約為「文章數 × 事實數 × 3～4」。
- **可重現**：API 模型 `temperature=0`（推理模型除外），本地模型 greedy decoding；選項順序以 `fact_id` 雜湊為種子，固定不變。
- **提示詞**：所有方法都從 `prompts/` 讀取提示詞，修改提示詞即可調整行為；若刪掉程式需要的 `{placeholder}` 會報錯。

---

## 專案架構

```text
.
├── requirements.txt                   # 基本依賴
├── requirements-local-backbones.txt   # 本地開源模型依賴（torch 等）
├── atlas/                             # ATLAS 核心套件
├── experiments/                       # 命令列腳本：資料準備、論文實驗、評估、排行榜
├── evaluation/                        # BioLaySumm 官方評估程式
└── prompts/                           # 所有 LLM 提示詞
```

### `atlas/` — 核心套件

| 檔案 | 功能 |
|---|---|
| `__init__.py` | 匯出 `AtlasPipeline` |
| `pipeline.py` | **主體**。`AtlasPipeline` 實作 Module 1/2/3、事實驅動摘要生成、`run_article`/`run`，以及實驗與評估用的 LLM 操作（拆解專家摘要、涵蓋率判斷、LLM 評審等）；`run_direct_generation` 為直接生成基線；另含提示詞載入與 placeholder 替換工具 |
| `clients.py` | `StructuredLLM`：統一呼叫 OpenAI、Gemini、本地 HF 模型，強制回傳符合 JSON Schema 的物件；`MODEL_SPECS` 為模型對照表 |
| `schemas.py` | 每種 LLM 呼叫的輸出 JSON Schema（候選事實、過濾決策、摘要、題目、作答、改寫、評分等） |
| `io.py` | JSONL 讀寫、輸入文章驗證、`mean` |
| `experiment_support.py` | 前導實驗工具：在摘要中定位並替換支持某事實的句子、取錯誤選項、計算比例 |

### `experiments/` — 命令列腳本（皆以 `python -m experiments.<名稱> --help` 查看參數）

| 類別 | 腳本 | 功能 |
|---|---|---|
| 資料準備 | `prepare_biolaysumm_input.py` | 下載 BioLaySumm 2025 PLOS + eLife，轉成輸入 JSONL |
| 前導實驗 | `pre_experiment_1_reliability_of_1t3f_nota.py` | 驗證 1T3F+NOTA 題型：專家摘要可否答對、摘要被竄改後模型是否依上下文作答 |
| | `pre_experiment_2_multiple_choice_vs_direct_judgment.py` | 製造保留/刪除/竄改事實的受控摘要，比較選擇題與直接是/否判斷的錯誤偵測率 |
| | `pre_experiment_3_fact_coverage_of_module_1_extraction.py` | 衡量不同模型的 Module 1 候選事實能涵蓋多少專家摘要內容 |
| 主實驗 | `main_experiment.py` | 三個骨幹模型各跑直接生成與 ATLAS，並輸出逐模組消融摘要（`ablation/<model>/stage_summaries.jsonl`） |
| 補充實驗 | `recent_reasoning_model_baseline.py` | GPT-5.4-mini 直接生成基線 |
| | `supplementary_backbone_validation.py` | 以 Qwen2.5-7B、Llama-3-8B 執行 ATLAS（需 GPU） |
| 評估 | `evaluate_biolaysumm_metrics.py` | 分 PLOS/eLife 計算官方 10 項指標並取平均 |
| | `evaluate_expert_summary_recall.py` | 專家摘要召回率：專家摘要的事實有多少被生成摘要涵蓋 |
| | `llm_as_a_judge_evaluation.py` | LLM 對相關性、可讀性、事實性各評 1–5 分 |
| 後處理 | `leaderboard_comparison.py` | 將多個系統的指標 min-max 正規化，換算排行榜分數與名次 |

評估腳本的 `--summaries` 輸入格式為每行 `{"article_id": ..., "summary": ...}`，即 `final_lay_summaries.jsonl` 的格式。

### `evaluation/` — 官方評估程式

| 檔案 | 功能 |
|---|---|
| `evaluation_final.py` | BioLaySumm 主辦單位釋出的評估程式，`evaluate_all()` 計算 ROUGE、BLEU、METEOR、BERTScore（相關性）、FKGL、DCRS、CLI、LENS（可讀性）、AlignScore、SummaC（事實性） |

需自行在專案根目錄放置 `AlignScore/` 與 `summac/`（含 `summac_conv_vitc_sent_perc_e.bin`）原始碼，並需 CUDA GPU。

### `prompts/` — 提示詞

| 檔案 | 使用處 |
|---|---|
| `module_1_candidate_atomic_fact_extraction.txt` | Module 1 候選事實抽取 |
| `module_2_abstract_based_atomic_fact_filtering.txt` | Module 2 以 Abstract 過濾 |
| `fact_driven_summary_generation.txt` | 依最終事實生成初版摘要 |
| `module_3_question_generation.txt` / `module_3_answering.txt` / `module_3_targeted_rewrite.txt` | Module 3 出題 / 作答 / 最小幅度改寫 |
| `direct_generation.txt` | 直接生成基線 |
| `llm_as_a_judge_evaluation.txt` | LLM 評審 |
| `pre_experiment_1_*.txt` / `pre_experiment_2_*.txt` / `pre_experiment_3_*.txt` | 各前導實驗的作答、判斷、拆解專家摘要、涵蓋率判斷 |
| `experiment_operations/` | 實驗操作用：Module 1-only 消融摘要、受控摘要改寫、專家摘要召回率判斷 |
