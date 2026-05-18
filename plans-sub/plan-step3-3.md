# Step 2-2 Plan: Vision Judge

## 目標

把 [notebooks/step2.ipynb](../notebooks/step2.ipynb) 既有的 per-page render pipeline (Step 2-1) 接上 PLAN.md §3.3 的 **Vision Judge**：對 pair 內的每一頁各跑 1 次 self-host 視覺模型 call（每 call 只送 1 張 PDF + 1 張 HTML render = 2 圖），兩頁平行；client 端 aggregate 成 pair-level 結構化錯誤 JSON。

這一階段先不做 bbox → DOM 的 IoU 映射 (§3.4)、Fixer (§3.5)、LangGraph 整合。先讓 judge 本體在 single pair 上跑通、輸出 schema 穩定。

---

## 設計核心：Judge 不見 anchor_id

### 為什麼 judge prompt 不揭露 anchor 資訊

`dom_bbox_map.json` 內每個元素都有 `anchor_id` (e.g. `region-0012`)，這是 HTML 端的 ground truth。但這份 ground truth 對 vision model 來說是「看不到的東西」——HTML render 出來的截圖裡沒有 marker 資訊，model 無從對齊。

如果在 prompt 裡把 anchor list 提供給 model 並要求它把 error 標上 `anchor_id`，會發生兩件壞事：

1. **幻覺**：model 會猜 anchor，產生看似合理但實際指錯區塊的對應
2. **責任邊界混淆**：「找錯」與「對齊到 DOM」是兩個獨立任務，混在同一個 prompt 會讓兩邊都做不好

新分工：

- Judge (§3.3, **本階段**)：只看圖、只回 `bbox` (像素座標) + `target_source_page`
- IoU 映射 (§3.4, **下一階段**)：拿 judge 的 bbox 跟 `dom_bbox_map.json` 算 IoU，反推 `anchor_id`

這也讓 judge 可以獨立 evaluate (不依賴 anchor 對齊正確性)，未來換模型也比較好做 A/B。

### 為什麼改成 per-page 1 call（2 圖），不是 pair 1 call（4 圖）

實測（dry run on `pair_01_02` against `qwen3.5:9b` via ollama）：

| 配置 | 結果 |
|------|------|
| 2 圖 + system message | ✅ `prompt_eval_count ≈ 3751`，正常描述兩張圖 |
| 1 圖 + no system | ✅ `prompt_eval_count ≈ 1589` |
| **4 圖 + system message** | ❌ `prompt_eval_count = 167` — server **silently drops images**，model 看不到圖直接幻覺 `score=10, errors=[]` |
| 4 圖 + no system | ❌ HTTP 500：`Failed to create new sequence: SameBatch may not be specified within numKeep (index: 3 numKeep: 4 SameBatch: 1565)` |

根因是 ollama runner 對每張圖開一個 `SameBatch`（~1500 tokens），第 4 張的 batch 邊界跨過 `numKeep` 保留區；加 system message 後走到另一條 code path、變成 silent drop。

**新方案**：per-page 1 call（2 圖：1 PDF + 1 HTML render），pair 內 2 頁用 `ThreadPoolExecutor` 平行送（ollama server 內部會 serialize 但 client 不阻塞）。

**代價**：跨頁元素 judge 看不到對側頁。緩解：
- **靠 overlap pair**：跨頁表格出現在 (n,n+1) 與 (n+1,n+2) 兩個 pair 的不同頁端
- **fixer 階段可跨頁**：拿到 per-page judge 報告後，fixer 端讀 HTML 不限於單頁

### 為什麼 aggregate 用 score min、不是 average

`min` 比 `avg` 保守：pair 內任一頁壞 = pair 壞，迴圈不會因為「另一頁好分數平均掉」就提前 pass。Errors 端直接 concat 兩頁的 errors，下游 IoU 階段自然按 `target_source_page` 對到正確頁的 DOM map。

### 為什麼 score 由 model 給、`passed` 由 client 算

讓 model 同時給 `score` 和 `passed` 會出現 self-inconsistency (e.g. score=8.5 但 passed=false)。Client 端用一個明確的 `PASS_THRESHOLD` 從 `score` 推 `passed`，single source of truth。

---

## 範圍

### In Scope

- Ollama 接入（`ollama` Python client，`ollama.chat(..., format="json", think=False)`）
- Pydantic schema 定義（`JudgeError`, `JudgeLLMOutput`, `JudgeReport`）
- Prompt 構造：**per-page 2 圖** + page_pixel_size + JSON output 規格
- `judge_single_page(page_info, model_id, num_ctx)` → `(JudgeLLMOutput | None, raw_str)`
- `judge_pair(input_path, output_path, model_id)`：load summary → 兩頁平行呼叫 `judge_single_page` → aggregate（score 取 min、errors concat、raw_output 兩段拼接）→ Pydantic validate → save `judge_report.json`
- Single pair dry run on `pair_01_02`
- 視覺 sanity check：把 aggregated errors 的 bbox 按 `target_source_page` 分組，分別畫在對應頁的 HTML screenshot 上

### Out of Scope

- bbox → anchor_id 的 IoU 映射 (§3.4)
- Fixer / patch apply (§3.5)
- LangGraph 整合、迭代控制、提前停止 (§3.6 / §6)
- 全篇 batch (先單 pair 驗證)
- vLLM server 自身的啟動 / 部署 (假設已 ready)
- Judge 結果的 aggregation 視覺化 dashboard

---

## 輸入 / 輸出

### Input

讀 `outputs-step2/pairs/pair_01_02/summary.json`，按其 `pages[*]` 解到：

對 pair 內每一頁（每頁一個獨立 ollama call）：
- `source_pdf.png` — 原始 PDF render
- `rendered_screenshot.png` — HTML render
- `dom_bbox_map.json` → 只取 `dom_header.page_pixel_size` (告訴 judge bbox 座標系統)，**不送 `dom[*]`**

兩頁的 call 在 client 端用 `ThreadPoolExecutor(max_workers=2)` 平行送。

### Output

寫到 `outputs-step2/pairs/pair_NN_MM/judge_report.json`。

Pydantic schema：

```python
from typing import Literal
from pydantic import BaseModel, Field

class JudgeError(BaseModel):
    type: Literal[
        "text_missing", "text_extra", "text_wrong",
        "table_structure", "equation_rendering",
        "figure_placement", "reading_order", "layout_other",
    ]
    target_source_page: int = Field(description="此錯誤所在的原始 PDF 頁碼，必須 ∈ pair")
    bbox: list[float] = Field(
        min_length=4, max_length=4,
        description="[x, y, w, h]，以 target_source_page 對應 HTML 截圖的像素為座標系",
    )
    description: str
    severity: Literal["high", "medium", "low"]

class JudgeLLMOutput(BaseModel):
    """LLM 每頁實際吐的 schema（最小集合，model 不知道 pair / pair_slug / threshold）。"""
    score: float = Field(ge=0.0, le=10.0)
    errors: list[JudgeError]

class JudgeReport(BaseModel):
    """Pair-level aggregated report（client 端組裝後寫檔）。"""
    pair: list[int]
    pair_slug: str
    score: float = Field(ge=0.0, le=10.0)  # 兩頁 LLM score 的 min
    passed: bool                            # score >= pass_threshold
    pass_threshold: float
    errors: list[JudgeError]                # 兩頁 errors concat
    model_id: str
    raw_output: str                         # 兩頁 raw 拼接：=== source_page=N === / {...}
```

---

## Notebook 變更

實作放在獨立的 [notebooks/step3.ipynb](../notebooks/step3.ipynb)（不再 append 到 step2.ipynb）。沿用 step2 的 `OUT_DIR` 結構，輸出寫到 `outputs-step3/pairs/{pair_slug}/judge_report.json`。

| # | Cell ID | Type | 內容 |
|---|---------|------|------|
| 1 | `3d52c988` | code | config：`INPUT_DIR`, `OUTPUT_DIR`, `SYSTEM_TEMPLATE`, `USER_TEMPLATE`（per-page，2 圖版本）|
| 2 | `bb18af28` | code | Pydantic schemas：`JudgeError`, `JudgeReport`, `JudgeLLMOutput` |
| 3 | `99938e6e` | code | helpers：`collect_pair_pages(pages)` → list of `{source_page, pdf_path, html_path, page_pixel_size}` |
| 4 | `949074d2` | code | 主流程：`get_model_context_length`, `make_pair_slug`, **`judge_single_page(page_info, model_id, num_ctx)`** → `(JudgeLLMOutput \| None, raw)`，**`judge_pair(...)`** 用 `ThreadPoolExecutor` 平行跑兩頁 → aggregate → 寫檔 |
| 5 | `8fb3dcf8` | code | dry run：`judge_pair(input_path=INPUT_DIR, output_path=OUTPUT_DIR)`，print pair/score/passed/errors + raw 前 800 字 |
| 6 | (新) | code | **Visual sanity check**：載 aggregated report → 按 `target_source_page` 分組 errors → 對每頁載 HTML screenshot、PIL `ImageDraw` 紅框 → inline display |

### Prompt 草稿（per-page）

```
[system]
你是學術論文 HTML 渲染品質的審稿員。給你一頁原始 PDF 圖片與對應的一頁 HTML 渲染截圖，
請比對結構與視覺正確性。只回報「明顯影響閱讀」的問題；字體、antialiasing、
細微 padding 差異不算錯。

評分標準 (0-10)：
- 10: 結構與內容完全對齊
- 8-9: 1-2 個 medium 以下小問題
- 5-7: 多個 medium 或 1 個 high (e.g. 表格欄位錯亂)
- 0-4: 多個 high (e.g. 整段遺漏、公式錯誤)

[user] (per page，pair 內呼叫 2 次)
以下是 source page = {p} 的素材，依序：
1. PDF page {p} (原始)             <image>
2. HTML render page {p} ({w}×{h})  <image>

請比對「PDF vs HTML」，輸出 JSON：
{
  "score": float,
  "errors": [
    {
      "type": "text_missing" | "text_extra" | "text_wrong"
            | "table_structure" | "equation_rendering" | "equation_wrong"
            | "figure_placement" | "reading_order" | "layout_other",
      "target_source_page": {p},
      "bbox": [x, y, w, h],   // HTML 截圖像素，原點左上
      "description": "...",
      "severity": "high" | "medium" | "low"
    }
  ]
}

bbox: 0 <= x <= {w}, 0 <= y <= {h}。target_source_page 必為 {p}。
只回 JSON，不要前後文。
```

### Aggregation 邏輯（client 端）

```python
# 兩頁平行
with ThreadPoolExecutor(max_workers=2) as ex:
    results = list(ex.map(lambda pi: judge_single_page(pi, model_id, num_ctx), pages_info))

per_page_scores, all_errors, raw_chunks = [], [], []
for pi, (parsed, raw) in zip(pages_info, results):
    raw_chunks.append(f"=== source_page={pi['source_page']} ===\n{raw}")
    if parsed is None:           # JSON 解析失敗
        per_page_scores.append(0.0)
        continue
    per_page_scores.append(parsed.score)
    all_errors.extend(parsed.errors)

score = min(per_page_scores)      # 保守：一頁壞 = pair 壞
passed = score >= PASS_THRESHOLD
```

---

## 既有可重用元件

| 元件 | 位置 | 用途 |
|------|------|------|
| `make_pair_slug` | step2.ipynb（亦在 step3.ipynb cell `949074d2` 重新定義） | 路徑命名一致 |
| `DOMBBoxHeader.page_pixel_size` | step2.ipynb cell `b1958ad0` 寫入 `dom_bbox_map.json` | 提供 judge prompt 所需座標系尺寸 |
| `summary.json` 的 `pages[*]` paths | step2.ipynb cell `e0063c6c` (`build_and_save_pair_summary`) | 一站取齊 pair 內每頁的 PDF/HTML render/bbox map |

---

## Risks / Open questions

### 已解決（per-page 1 call 後）

- ~~**4-image 路徑會 silently drop images / 500 SameBatch**~~：改成 per-page 2 圖後完全規避；ollama runner 在 2 圖路徑穩定。
- ~~**Multi-image token 預算爆 context**~~：2 圖約 ~3.7k prompt tokens，遠低於 `num_ctx`。

### 仍需注意

1. **bbox 座標漂移**
   VL 模型常輸出 normalized [0,1] 或 1000 scale 的 bbox，而非絕對像素。
   **緩解**：prompt 明確指定像素 + 帶 `page_pixel_size`。dry run 後用視覺化 cell 肉眼驗 bbox 是否落在合理區域；若系統性偏差再 iterate prompt（加 1-shot 範例 / 加 post-processing 自動偵測 scale 並還原）。

2. **JSON 輸出穩定性**
   `format="json"` 不保證 schema 完全符合 `JudgeLLMOutput`（多/少欄、型別錯）。
   **緩解**：`judge_single_page` 用 `try / except` 包住 `JudgeLLMOutput.model_validate_json`，parse 失敗時 `parsed=None`、aggregate 階段該頁計 0 分，raw_output 一律保留方便 debug。

3. **跨頁元素覆蓋**
   單 call 只看一頁，跨頁表格/段落在 judge 視角天然斷裂。
   **緩解**：靠 overlap pair `(n,n+1) / (n+1,n+2)`，跨頁元素至少會在某個 pair 的某一頁端被完整觀察到一半；fixer 階段拿到報告後可跨頁讀 HTML 整體修。完全不漏的覆蓋留給後續優化。

4. **ollama server 預設未啟動 / model 未 pull**
   notebook 不負責 spin up。
   **緩解**：dry run 前手動 `ollama serve` + `ollama pull qwen3.5:9b`（或對應 VL tag）。`get_model_context_length` 會 surfaces model 不存在的錯誤。

5. **Score aggregation 策略 (min vs avg)**
   目前用 `min`：保守，一頁壞 = pair 壞、不會提前 pass。但若一頁是 score=0 parse fail 而另一頁完美，整個 pair 會被誤判 fail。
   **緩解**：先觀察 dry run 的 parse 失敗率。若高（> 5%），改成 `min(scores where parsed is not None)` 並把 parse fail 另記到 report metadata。

---

## Verification

實作完成後依此驗證 end-to-end：

1. **Server check**：`ollama list` 看 `qwen3.5:9b`（或當前 `model_id`）在列表
2. **跑 dry run cell** (pair_01_02)：兩頁平行各跑一次 ollama call，wall time ≈ 單頁 latency；stdout 印 `pair=[1,2] score=… passed=… errors=…`
3. **檢查 prompt_eval_count**（必要時加 print）：每頁應 ≥ 3000（2 圖正常吃進去）；若 < 500 表示 image drop 又出現了，回頭看 ollama log
4. **檢查 `outputs-step3/pairs/pair_01_02/judge_report.json`**：
   - `pair == [1, 2]`, `pair_slug == "pair_01_02"`
   - `0.0 <= score <= 10.0`, `passed == (score >= pass_threshold)`, `score == min(per_page_scores)`
   - 每個 `errors[i].target_source_page ∈ {1, 2}`
   - 每個 `errors[i].bbox` 各分量為非負 float，且 `x+w <= page_pixel_size.width`、`y+h <= page_pixel_size.height` (依 `target_source_page` 對應的 page 尺寸；以 `dom_bbox_map.json` 為準)
   - `raw_output` 含 `=== source_page=1 ===` 與 `=== source_page=2 ===` 兩段
5. **跑視覺 sanity cell**：errors 按 `target_source_page` 分組後紅框應落在「肉眼看得出 PDF vs HTML 有差」的區域；兩頁各畫一張
6. **Pydantic strict validate** 全部過：`JudgeLLMOutput.model_validate_json` 在 `judge_single_page` 內 catch；`JudgeReport(...)` 構造在 `judge_pair` 內若 enum / 範圍違反會直接 raise，不會悄悄寫進 json
