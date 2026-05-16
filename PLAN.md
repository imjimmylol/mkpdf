# PDF-to-HTML Agent — Architecture Plan

## 1. Overview

一個針對**學術論文**的 PDF → HTML 自動轉換 agent，透過 **ML-based 初始轉換 + Vision LLM 迭代修復迴圈**，產出結構語義正確且關鍵視覺元素忠實的 HTML。

### Core Loop

```
PDF → Docling 初始轉換 → HTML (含 anchor markers + data-source-page 頁碼標籤)
                              ↓
                    ┌─→ 按 Page Origin Map 分組 → Playwright 截圖 + DOM Mapping Table
                    │         ↓
                    │   Qwen2.5-VL Judge (2 張原 PDF 頁面 vs 1 張對應 HTML 截圖)
                    │         ↓
                    │     Pass? ──→ Yes ──→ 輸出最終 HTML
                    │      No (+ 結構化錯誤報告 + bounding box 標註)
                    │         ↓
                    │   Code Model Fixer (search-replace / 整段重寫)
                    │         ↓
                    └─── 更新 HTML，回到截圖步驟
                    
              (達到迭代上限仍 fail → 降級策略 + 錯誤報告)
```

---

## 2. Step 1 — 初始轉換

| 項目 | 決策 |
|------|------|
| **工具** | Docling (MIT License, 免費開源) |
| **模型** | Granite-Docling-258M (Apache 2.0) |
| **輸出格式** | HTML (Docling 原生支援直接 export) |
| **公式處理** | LaTeX 原始碼保留於 HTML，前端用 MathJax 渲染 |
| **LLM 參與** | 否 — 此階段純 ML-based，不呼叫 LLM |

### 為什麼選 Docling

- 直接輸出 HTML，不需要額外 Markdown → HTML 轉換
- 內建 layout analysis (DocLayNet) + table structure recognition (TableFormer)
- Granite-Docling-258M 表現：表格 TEDS 0.97、公式 F1 0.968、code F1 0.988
- DocTags 格式天然分離內容與 layout，方便後續注入 anchor marker
- GitHub 30k+ stars，社群活躍，持續更新

### 初始轉換後處理

1. Docling 輸出 HTML
2. 注入 **`data-source-page` 頁碼標籤**：標記每個 HTML 區塊對應原始 PDF 的第幾頁（例如 `data-source-page="3"`）。Docling 的 provenance 資訊可提供此映射，若不足則用 PDF 頁面的文字內容比對定位
3. 注入 `data-region-id` anchor markers（每個段落、表格、公式、圖片）
4. 建立 **Page Origin Map**：`{ source_page: [anchor_id_1, anchor_id_2, ...] }`，記錄每個原始頁碼包含哪些 HTML 元素
5. 用於後續 debug + bounding box → DOM 元素映射

---

## 3. Step 2 — 迭代修復迴圈

### 3.1 比對粒度

| 項目 | 決策 |
|------|------|
| **粒度** | 以原始 PDF 兩頁為一組，比對該兩頁對應的所有 HTML 內容 |
| **理由** | 學術論文內容完整性極少跨越兩頁；單頁可能切斷跨頁元素，三頁以上資訊過多 |

#### 關鍵設計：Page Origin Label

原始 PDF 的 2 頁內容轉成 HTML 後，渲染長度不固定（表格展開、圖片縮放等），因此：

- **比對單位是「原始 PDF 頁碼」，不是「渲染後 HTML 的視覺頁面」**
- 每個 HTML 元素帶有 `data-source-page` 屬性，標記它來自原始 PDF 第幾頁
- 比對時，透過 Page Origin Map 收集 source page 3-4 的所有 HTML 元素，整段渲染截圖
- Judge 收到的是：2 張原始 PDF 頁面截圖 vs 1 張（可能更長的）HTML 渲染截圖

```
原始 PDF page 3-4 (固定 2 張圖)
        ↕ 比對
HTML 中 data-source-page="3" 和 "4" 的所有元素 (渲染後可能超過 2 頁長)
```

#### Playwright 截圖策略

由於 HTML 對應內容可能比原始 2 頁更長，截圖時：
- 不用固定視窗高度分頁，而是**按 `data-source-page` 分組，full-height 截圖**
- 若截圖過長（例如超過 3000px），可切分為多張送入 judge，但標註它們屬於同一組

### 3.2 渲染與截圖

**工具：Playwright (headless browser)**

每輪迭代，針對每個尚未 pass 的 page pair 執行：
1. 從 Page Origin Map 取出該 page pair 對應的所有 HTML 元素
2. 渲染這些元素並 full-height 截圖（長度不固定，由內容決定）
3. 同步執行 JS 抽取該範圍內所有可見元素的 `getBoundingClientRect()`
4. 建立 **座標 → DOM 映射表**：`{ element_id, anchor_id, source_page, bbox: [x, y, w, h] }`

### 3.3 Vision Judge

| 項目 | 決策 |
|------|------|
| **模型** | Qwen2.5-VL (建議 72B，可依成本降至 7B) |
| **輸入** | 原始 PDF 頁面圖片 (2 張) + 對應 HTML 渲染截圖 (1 張，長度不固定) |
| **輸出** | 分數 + 結構化錯誤描述 + bounding box 圖片標註 |
| **判斷標準** | 中間路線 — 結構語義正確 + 關鍵視覺元素正確 |

#### Judge 輸出格式 (建議 JSON)

```json
{
  "page_pair": [3, 4],
  "score": 6,
  "pass": false,
  "errors": [
    {
      "type": "table_structure",
      "bbox": [120, 340, 450, 520],
      "anchor_id": "table-2",
      "description": "表格第三欄的邊框缺失，且 header row 與 data row 沒有分隔線",
      "severity": "high"
    },
    {
      "type": "equation_rendering",
      "bbox": [80, 600, 400, 650],
      "anchor_id": "eq-7",
      "description": "公式 (7) 的上標 \\alpha^{(t)} 渲染為 \\alpha(t)，缺少上標格式",
      "severity": "medium"
    }
  ]
}
```

#### Judge 評分維度

- **內容完整性**：文字有無遺漏或多出
- **表格結構**：行列數、合併儲存格、邊框
- **公式正確性**：LaTeX 渲染結果與原 PDF 視覺一致
- **圖片位置與大小**：圖片是否在正確位置、比例是否合理
- **閱讀順序**：多欄 layout 的閱讀順序是否正確

### 3.4 Bounding Box → DOM 映射

Vision judge 回報的 bounding box 座標，透過以下流程映射回 HTML 元素：

1. 從 DOM 映射表中，計算每個元素與 judge bbox 的 **IoU (Intersection over Union)**
2. 取 IoU 最高的元素作為目標
3. 用該元素的 `data-region-id` (anchor marker) 定位 HTML 片段
4. 抽取該片段 + 周邊 context，送給 Fixer

### 3.5 Code Model Fixer

| 項目 | 決策 |
|------|------|
| **模型** | 純文字 code model（Claude Sonnet / GPT-4o / DeepSeek-Coder） |
| **與 Judge 分開** | 是 — Judge 專責看圖挑錯，Fixer 專責改 code |
| **輸入** | 問題 HTML 片段 + 周邊 context + Judge 的結構化錯誤描述 |
| **輸出** | Diff/Patch |

#### Patch 策略（兩層 fallback）

```
嘗試 Search-Replace
    ├─ 成功 → Apply patch，進入下一輪
    └─ 失敗 (find 字串不存在或不唯一)
           ↓
       整段重寫 (用 anchor_id 定位區塊，整段替換)
```

#### Fixer Prompt 結構 (建議)

```
你是一個 HTML/CSS 修復專家。以下是一段有問題的 HTML 片段和錯誤描述。
請用 search-replace 格式輸出修復。

## 錯誤描述
{judge 的結構化錯誤 JSON}

## 問題 HTML 片段 (anchor: {anchor_id})
{抽取的 HTML 片段}

## 周邊 Context
{前後各 N 行的 HTML}

## 輸出格式
{
  "strategy": "search_replace" | "full_rewrite",
  "patches": [
    {
      "anchor_id": "table-2",
      "find": "要替換的舊 HTML",
      "replace": "修復後的新 HTML"
    }
  ]
}
```

### 3.6 迭代控制

| 項目 | 決策 |
|------|------|
| **最大迭代次數** | 3-5 輪（MVP 先設 3） |
| **通過門檻** | Judge score ≥ 8/10 (可調) |
| **提前停止** | 所有 page pair 都 pass 時提前結束 |
| **per-page tracking** | 已 pass 的 page pair 不再重新處理 |

---

## 4. 降級策略 (Fallback)

當某個 page pair 在迭代上限後仍未通過：

1. **識別 fail 的區塊**：根據最後一輪 judge 的錯誤報告，定位具體的 fail 元素
2. **區塊級降級**：將 fail 的區塊（例如壞掉的表格、渲染錯誤的公式）替換為原 PDF 對應區域的截圖 (`<img>`)
3. **保留 HTML**：其他正常的部分維持 HTML 格式
4. **附上錯誤報告**：最終輸出包含一份結構化的降級報告

### 降級報告格式

```json
{
  "status": "partial_degradation",
  "total_pages": 12,
  "passed_pages": [1,2,3,4,5,6,9,10,11,12],
  "degraded_pages": [7, 8],
  "degraded_elements": [
    {
      "page": 7,
      "anchor_id": "table-5",
      "type": "table_structure",
      "reason": "複雜的多層合併儲存格表格，3 輪修復後仍無法正確呈現",
      "fallback": "image_replacement",
      "last_score": 5
    }
  ],
  "iterations_used": 3,
  "final_overall_score": 7.8
}
```

---

## 5. 技術棧總覽

| 元件 | 技術選擇 | 角色 |
|------|----------|------|
| **Orchestration** | LangGraph | Pipeline 狀態管理、迴圈控制、conditional branching |
| **初始轉換** | Docling + Granite-Docling-258M | PDF → HTML |
| **公式渲染** | MathJax + LaTeX 原始碼保留 | HTML 中公式的前端渲染 |
| **截圖 & DOM Mapping** | Playwright | 渲染 HTML、截圖、抽取元素座標 |
| **Vision Judge** | Qwen2.5-VL | 比對原 PDF vs 渲染 HTML，輸出分數 + 錯誤 + bbox |
| **Code Fixer** | Claude Sonnet / GPT-4o | 根據錯誤報告修復 HTML |
| **Patch 機制** | Search-Replace → 整段重寫 fallback | Apply 修復到 HTML |

---

## 6. LangGraph State 設計

```python
from typing import TypedDict, List, Optional
from langgraph.graph import StateGraph

class PagePairState(TypedDict):
    page_numbers: tuple[int, int]
    score: float
    passed: bool
    errors: list[dict]
    iteration_count: int

class PipelineState(TypedDict):
    # 輸入
    pdf_path: str
    
    # 初始轉換
    raw_html: str
    html_with_anchors: str
    
    # 頁碼映射
    page_origin_map: dict    # {source_page: [anchor_id_1, anchor_id_2, ...]}
    total_source_pages: int  # 原始 PDF 總頁數
    
    # 迭代狀態
    current_html: str
    page_pairs: list[PagePairState]
    current_iteration: int
    max_iterations: int  # default: 3
    
    # DOM Mapping (per page pair, 每輪重建)
    dom_mapping: dict  # {anchor_id: {source_page, bbox, element_html}}
    
    # 截圖
    html_screenshots: dict  # {(source_page_start, source_page_end): image_bytes}
    pdf_screenshots: dict   # {(source_page_start, source_page_end): [page_img_1, page_img_2]}
    
    # 最終輸出
    final_html: str
    degradation_report: Optional[dict]
```

### LangGraph Node 定義

```
[convert_pdf]          # Docling 轉換 + 注入 anchors + 注入 data-source-page 標籤 + 建立 Page Origin Map
      ↓
[render_and_capture]   # 按 page pair 從 Page Origin Map 取元素 → Playwright 截圖 + DOM mapping
      ↓
[judge_pages]          # Qwen2.5-VL 逐 page pair 評分 (2 張 PDF 圖 vs 1 張 HTML 截圖)
      ↓
[check_completion]     # 全部 pass? 或達上限?
    ↙        ↘
[fix_html]    [finalize]
    ↓              ↓
[apply_patches]  [degrade_and_report]  
    ↓
 → 回到 [render_and_capture]
```

---

## 7. MVP 實作計畫

### Phase 1 — 基礎 Pipeline (Week 1-2)

- [ ] Docling 安裝與測試，確認學術論文 PDF → HTML 品質
- [ ] `data-source-page` 頁碼標籤注入邏輯（利用 Docling provenance 資訊）
- [ ] Page Origin Map 建立
- [ ] Anchor marker 注入邏輯
- [ ] Playwright 按 page pair 分組截圖 + DOM mapping 抽取
- [ ] PDF 頁面截圖（用 PyMuPDF 或 pdf2image）

### Phase 2 — Judge + Fixer 迴圈 (Week 2-3)

- [ ] Qwen2.5-VL judge prompt 設計與測試
- [ ] Judge 輸出 JSON parsing + bounding box 標註
- [ ] Bounding box → DOM 元素 IoU 映射
- [ ] Code fixer prompt 設計
- [ ] Search-replace + 整段重寫 fallback 實作

### Phase 3 — LangGraph 整合 (Week 3-4)

- [ ] LangGraph state 定義
- [ ] Node functions 實作
- [ ] Conditional edge 邏輯（pass/fail/max_iteration）
- [ ] 降級策略實作
- [ ] 錯誤報告生成

### Phase 4 — 測試與調參 (Week 4-5)

- [ ] 收集 10-20 篇不同複雜度的學術論文做測試集
- [ ] 調整 judge 通過門檻
- [ ] 調整最大迭代次數
- [ ] 觀察常見 failure pattern，優化 fixer prompt
- [ ] 成本分析（每篇論文的 API call 成本）

---

## 8. 已知風險與待解決問題

| 風險 | 影響 | 緩解方案 |
|------|------|---------|
| Qwen2.5-VL judge 評分不穩定（同輸入不同分數） | 迴圈可能不收斂 | 多次評分取平均；固定 temperature=0 |
| 複雜表格（多層合併儲存格）修復困難 | 反覆修不好浪費迭代 | 降級策略兜底；考慮對表格用專門的修復邏輯 |
| HTML 很長超出 fixer context window | Fixer 無法看到足夠 context | 只送問題片段 + 周邊 context，不送整份 HTML |
| MathJax 渲染與原 PDF 字體差異導致 judge 扣分 | 不必要的迭代 | Judge prompt 明確說明字體差異可接受 |
| Docling 對某些 PDF 格式解析失敗 | 初始轉換品質差 | 預處理檢查 PDF 品質；嚴重時 fallback 到 Marker |

---

## 9. 成本估算 (per paper, ~15 pages)

| 元件 | 單次成本 | 最壞情況 (3 輪迭代) |
|------|---------|-------------------|
| Docling 初始轉換 | 本地推理，~免費 | 免費 |
| Playwright 截圖 | 本地執行，免費 | 免費 |
| Qwen2.5-VL Judge (per 輪) | ~8 page pairs × vision tokens | ×3 輪 |
| Code Fixer (per 輪) | ~修復 2-4 個問題 × text tokens | ×3 輪 |
| **預估總計** | 視模型定價而定 | 如用 self-hosted Qwen2.5-VL 可大幅降低 |

> 💡 MVP 階段建議先用 API（快速迭代），確認 pipeline 可行後再考慮 self-host Qwen2.5-VL 降低成本。