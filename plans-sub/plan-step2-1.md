# Step 2-1 Plan: Per-Page Render Harness

## 目標

這份計畫聚焦在 Step 2 的第一個可驗證里程碑：先做出一個能穩定跑通的 `single source page -> single-page HTML -> browser render -> screenshot -> DOM bbox map -> PDF page image` harness，然後以**輕量 pair summary 層**把它們聚成 judge 評斷單位。

這一階段先不做 Vision Judge、Fixer、LangGraph loop，也不先處理 patch apply。目標是把後續所有節點會共用的輸入產物先定型。

---

## 設計核心：Per-Page 渲染，Pair 為純引用

### 為什麼不再對 pair 做合併截圖

原本「把 pair 兩頁聚成一張長截圖」的設計有三個問題：

1. **Judge 視覺切段沒有 ground truth**：要從 5000px 長圖識別 page 邊界、再對應兩張獨立 PDF 圖，切錯後續 bbox 全亂位。
2. **bbox 座標系統混淆**：合併圖 bbox 是 pair-local，IoU 比對前要做 page-relative 轉換，多一層靜默 bug 源。
3. **長截圖損失字級細節**：Qwen2.5-VL 等視覺模型對極長圖會 resize，公式上下標小細節因此糊掉。

新設計把渲染粒度切到 **page**：

- 每個 source page 各自 build 一份 single-page HTML、各自截圖、各自存 bbox map
- Judge 一次拿到 4 張獨立圖（PDF p3 + PDF p4 + HTML p3 + HTML p4），做 pairwise 比對
- bbox 永遠以「該頁截圖」為座標系統，下游 IoU 不需任何轉換
- Pair 只是「judge 一次評哪兩頁」的 logical grouping，靠 summary.json 帶顯式 paths 指向對應 page artifacts

### 為什麼 page render 去重

Pair 採 overlap `(1,2), (2,3), (3,4), ..., (n-1, n)`，每個中間 page 屬於 2 個 pair。若 render 跟著 pair 跑，每頁會被渲兩次：

- 浪費約 2x 渲染成本
- 同頁兩次 render 因 anti-aliasing 等浮動，bbox 可能微異，造成 debug 困擾

解法：**render 階段是 page-driven，跟 pair 完全解耦**。每個 unique page 只 render 一次，bbox map 也只有一份權威版本。Pair summary 純粹是 reference。

### 為什麼 pair 用 overlap

當表格 / 段落跨頁時，judge 必須在同一個 prompt context 看到兩頁才能評斷跨頁完整性。非 overlap `[(1,2), (3,4)]` 會把 page 2-3 邊界切到兩個 judge call，誰都看不到完整跨頁元素。

Overlap 的代價是 judge call 數量約 2x，但 render 仍 1x（因去重）。

---

## 這一階段要解決的問題

目前 Step 1 已完成初始轉換，且已有下列產物：

- `outputs-test-std/JME_DL_annotated.html`
- `outputs-test-std/JME_DL_page_origin_map.json`
- `outputs-test-std/JME_DL_region_index.json`

其中 `JME_DL_annotated.html` 已帶有：

- `data-source-page`
- `data-region-id`

這代表它已經具備 Step 2 最需要的定位能力，作為 Step 2-1 的唯一輸入 HTML。

---

## 範圍

### In Scope

- 載入 Step 1 產物
- `make_overlapping_page_pairs` 產出 pair list
- 從 `annotated.html` 萃取單一 source page 的 `.page` 區塊
- 組出可單獨渲染的 single-page HTML
- 用 Playwright 對 single-page HTML 截圖（每頁一張）
- 抽取該頁所有 `[data-region-id]` 的 bbox（座標相對該頁截圖）
- 用 PyMuPDF 輸出原始 PDF 對應頁面圖片
- 三階段 orchestration：清空 → render 所有 unique pages（dedup）→ build pair summaries → 寫 top-level index
- 在 notebook 中做單頁 dry run 與全篇 batch

### Out of Scope

- Vision Judge prompt 設計
- Judge JSON parsing
- bbox 與 judge error 的 IoU 對應
- HTML 修復與 patch apply
- LangGraph state 與 node wiring
- fallback image replacement

---

## 成功條件

完成後，針對任一 source page，例如 page 3，應能穩定產出：

- `outputs-step2/pages/page_03/rendered.html`
- `outputs-step2/pages/page_03/rendered_screenshot.png`
- `outputs-step2/pages/page_03/source_pdf.png`
- `outputs-step2/pages/page_03/dom_bbox_map.json`

對任一 pair，例如 `(3, 4)`，應能穩定產出：

- `outputs-step2/pairs/pair_03_04/summary.json`（內含 page 3 與 page 4 各 4 個 artifact paths）

全篇 batch 後應有：

- `outputs-step2/all_pairs_summary.json`（雙索引：pages + pairs + source paths）

人工確認項目：

- 單頁 HTML 可獨立打開，視覺上只顯示該頁
- HTML screenshot 樣式延續原 annotated.html
- `dom_bbox_map.json` 中每個元素的 `source_page` 都等於 header 的 `source_page`
- bbox 數量與該頁 region 數大致相符
- bbox 的 `y` 都從 0 起算（page-local，不會出現 pair-level 的偏移）

---

## Notebook 定位

`notebooks/step2.ipynb` 是未來 `render_and_capture` node 的 prototype。

核心責任：

1. 生成 judge 需要看的圖片（per-page，每頁 PDF + HTML 各一張）
2. 生成之後要拿來做 bbox 對映的 DOM metadata（per-page bbox map）
3. 生成 pair 層輕量 summary，作為 judge call 的 input manifest

---

## Notebook Cell Skeleton

按執行順序：

### Cell 1: Imports

```python
from pathlib import Path
import json
import shutil
import tempfile
from typing import Any

import fitz
from bs4 import BeautifulSoup, Tag
```

### Cell 2: Config

```python
ROOT = Path("..").resolve()
PDF_PATH = ROOT / "paper" / "JME_DL.pdf"
HTML_PATH = ROOT / "outputs-test-std" / "JME_DL_annotated.html"
PAGE_ORIGIN_MAP_PATH = ROOT / "outputs-test-std" / "JME_DL_page_origin_map.json"
REGION_INDEX_PATH = ROOT / "outputs-test-std" / "JME_DL_region_index.json"
OUT_DIR = ROOT / "outputs-step2"

VIEWPORT = {"width": 1440, "height": 1200}
PDF_ZOOM = 2.0
MAX_SCREENSHOT_HEIGHT = 3000

DRY_RUN_PAGE = 3
```

### Cell 3: Load Step 1 Artifacts

```python
page_origin_map = json.loads(PAGE_ORIGIN_MAP_PATH.read_text())
region_index = json.loads(REGION_INDEX_PATH.read_text())
soup = BeautifulSoup(HTML_PATH.read_text(), "html.parser")
```

### Cell 4: Helpers

宣告：`make_overlapping_page_pairs`、`make_pair_slug`、`make_page_slug`、`load_annotated_soup`、`extract_page_node`、`build_page_html`、`save_page_html`、`render_pdf_page`。

### Cell 5: Pydantic Schemas

宣告：`Viewport`、`PagePixelSize`、`DOMBBoxElement`、`DOMBBoxHeader`、`DOMBBoxMap`。

### Cell 6: Capture Function

`capture_page_screenshot_and_bboxes`，支援 optional `browser` 參數以利 batch reuse。

### Cell 7: Orchestration Functions

`run_page_pipeline`、`build_and_save_pair_summary`、`save_all_pairs_summary`。

### Cell 8: Single-Page Dry Run

```python
sp = load_annotated_soup(HTML_PATH)
dry_run_result = await run_page_pipeline(
    soup=sp, pdf_path=PDF_PATH, source_page=DRY_RUN_PAGE,
    out_dir=OUT_DIR, viewport=VIEWPORT, pdf_zoom=PDF_ZOOM,
    max_height=MAX_SCREENSHOT_HEIGHT,
)
```

### Cell 9: Batch Orchestration（三階段）

- Stage 0: `shutil.rmtree(OUT_DIR)` 清空、`mkdir`
- Stage 1: 列 unique pages、launch 一次 browser、loop 跑 `run_page_pipeline`
- Stage 2: 對每個 pair 跑 `build_and_save_pair_summary`
- Stage 3: `save_all_pairs_summary`

### Cell 10: Visual Check

並排顯示 `source_pdf.png` 與 `rendered_screenshot.png`，列印 `bbox_count` 與 `page_pixel_size`。

---

## 函式設計

### `make_overlapping_page_pairs`

```python
def make_overlapping_page_pairs(total_pages: int) -> list[tuple[int, int | None]]:
    if total_pages == 0:
        return []
    if total_pages == 1:
        return [(1, None)]
    return [(i, i + 1) for i in range(1, total_pages)]
```

#### 行為

- `total_pages == 0` → `[]`
- `total_pages == 1` → `[(1, None)]`（唯一會出現 `None` 的情況，作為 single-page document fallback）
- `total_pages >= 2` → `[(1,2), (2,3), ..., (n-1, n)]`，全部是真正的 2 頁 pair

#### 為什麼不再加 `(n, None)` tail

原本的 tail 是為了「非 overlap 設計下若 n 為奇數，page n 會被遺漏」而存在。但 overlap 設計步長為 1，page n 必然出現在最後一個 pair 的第二位，tail 永遠是冗餘的（且會多一次無增益 judge call）。

---

### `make_pair_slug`

```python
def make_pair_slug(pair: tuple[int, int | None]) -> str:
    start, end = pair
    if end is None:
        return f"pair_{start:02d}"
    return f"pair_{start:02d}_{end:02d}"
```

---

### `make_page_slug`

```python
def make_page_slug(source_page: int) -> str:
    return f"page_{source_page:02d}"
```

---

### `load_annotated_soup`

```python
def load_annotated_soup(html_path: Path) -> BeautifulSoup:
    return BeautifulSoup(html_path.read_text(), "html.parser")
```

---

### `extract_page_node`

```python
def extract_page_node(soup: BeautifulSoup, source_page: int) -> Tag:
    ...
```

#### 行為

- 從 soup 找所有 `div.page` 中 `data-source-page == str(source_page)` 的節點
- 找不到 → raise
- 找到多於一個 → raise（fail loud，反映 Step 1 注入有 bug）
- 找到唯一一個 → 回傳該 Tag

---

### `build_page_html`

```python
def build_page_html(
    original_soup: BeautifulSoup,
    page_node: Tag,
    title: str,
) -> str:
    ...
```

#### 行為

- 建一個新 soup（含 `<html><head><body>`）
- 深拷貝原 soup 的 `<head>`（保留 CSS / meta / fonts / mathjax 等）
- 設定 `<title>` 為傳入的 `title`（通常為 `page_slug`）
- 將 page_node 拷貝到新 body（注意是字串往返、避免移動原 Tag）

---

### `save_page_html`

```python
def save_page_html(html: str, out_dir: Path, source_page: int) -> Path:
    ...
```

#### 寫到

`out_dir / "pages" / make_page_slug(source_page) / "rendered.html"`

---

### `render_pdf_page`

```python
def render_pdf_page(
    pdf_path: Path,
    source_page: int,
    out_dir: Path,
    zoom: float = 2.0,
) -> Path:
    ...
```

#### 寫到

`out_dir / "pages" / make_page_slug(source_page) / "source_pdf.png"`

#### 注意

- PyMuPDF 是 0-based，schema 是 1-based，要 `doc.load_page(source_page - 1)`

---

### `capture_page_screenshot_and_bboxes`

```python
async def capture_page_screenshot_and_bboxes(
    html_file: str | Path,
    out_dir: Path,
    source_page: int,
    viewport: dict[str, int],
    max_height: int = 3000,
    browser: Browser | None = None,
) -> dict[str, Any]:
    ...
```

#### 行為

- 若傳入 `browser`，reuse；否則自己 `async_playwright` launch + close
- 開新 page、設定 viewport
- goto 本地 HTML，等待 networkidle + 200ms
- evaluate full page size
- evaluate 所有 `[data-region-id]` 的 bbox + metadata
- screenshot（高度截到 `min(full_height, max_height)`）
- 用 `DOMBBoxMap` 驗證後寫到 `pages/page_NN/dom_bbox_map.json`

#### Browser 端擷取欄位

- `data-region-id` → `anchor_id`
- `data-source-page` → `source_page`（element 層保留作 defensive check）
- `tagName.toLowerCase()` → `tag`
- `innerText` 前 100 字 → `text_preview`
- `getBoundingClientRect()` → `bbox`

---

### `run_page_pipeline`

```python
async def run_page_pipeline(
    soup: BeautifulSoup,
    pdf_path: Path,
    source_page: int,
    out_dir: Path,
    viewport: dict[str, int],
    pdf_zoom: float,
    max_height: int = MAX_SCREENSHOT_HEIGHT,
    browser: Browser | None = None,
) -> dict[str, Any]:
    ...
```

#### 串接五步

1. `extract_page_node`
2. `build_page_html`
3. `save_page_html`
4. `render_pdf_page`
5. `capture_page_screenshot_and_bboxes`

#### 回傳

```python
{
    "source_page": 3,
    "page_slug": "page_03",
    "rendered_html_path": "outputs-step2/pages/page_03/rendered.html",
    "rendered_screenshot_path": "outputs-step2/pages/page_03/rendered_screenshot.png",
    "source_pdf_path": "outputs-step2/pages/page_03/source_pdf.png",
    "dom_bbox_map_path": "outputs-step2/pages/page_03/dom_bbox_map.json",
    "region_count": 18,
}
```

---

### `build_and_save_pair_summary`

```python
def build_and_save_pair_summary(
    pair: tuple[int, int | None],
    page_artifacts: dict[int, dict[str, Any]],
    out_dir: Path,
) -> Path:
    ...
```

#### 行為

- 從 `page_artifacts` 取 pair 涉及的每一頁
- 組出 `pairs/pair_NN_MM/summary.json`（schema 見下）
- 純 reference 組裝，無 render

---

### `save_all_pairs_summary`

```python
def save_all_pairs_summary(
    unique_pages: list[int],
    pairs: list[tuple[int, int | None]],
    page_artifacts: dict[int, dict[str, Any]],
    source_pdf_path: Path,
    annotated_html_path: Path,
    out_dir: Path,
) -> Path:
    ...
```

#### 寫到

`out_dir / "all_pairs_summary.json"`

---

## 輸出資料夾結構

```text
outputs-step2/
  pages/
    page_01/
      rendered.html
      rendered_screenshot.png
      source_pdf.png
      dom_bbox_map.json
    page_02/
      ...
    page_NN/
      ...
  pairs/
    pair_01_02/
      summary.json
    pair_02_03/
      summary.json
    pair_NN_MM/
      summary.json
  all_pairs_summary.json
```

### 為什麼這樣拆

- **`pages/` 是 source of truth**：render artifact 只有一份。Pair overlap 不會造成重複 render。
- **`pairs/` 是 reference 層**：summary.json 帶顯式 paths，judge 讀一份檔拿齊 8 paths（per-page screenshot×2 + per-page PDF×2 + per-page bbox map×2），不需跨資料夾 join。
- **`all_pairs_summary.json` 是 document 層**：放 `source_pdf_path`、`annotated_html_path` 等 document-level metadata，並列出所有 unique pages 與 pairs。

---

## `dom_bbox_map.json` 格式

```json
{
  "dom_header": {
    "source_page": 3,
    "page_slug": "page_03",
    "html_path": "outputs-step2/pages/page_03/rendered.html",
    "screenshot_path": "outputs-step2/pages/page_03/rendered_screenshot.png",
    "viewport": { "width": 1440, "height": 1200 },
    "page_pixel_size": { "width": 1440, "height": 2688 },
    "bbox_count": 18
  },
  "dom": [
    {
      "anchor_id": "region-0039",
      "source_page": 3,
      "tag": "div",
      "text_preview": "The first paragraph on page 3 ...",
      "bbox": [80, 140, 620, 92]
    }
  ]
}
```

### 重要欄位

- `source_page`（header 與 element 層都有）：element 層作為 defensive check，與 header 不一致時可立即發現 Step 1 注入 bug
- `viewport`：紀錄 render 時 Playwright 用的 viewport，讓 bbox 可被重現
- `page_pixel_size`：實際截圖尺寸（可能被 `max_height` 截過）
- `bbox`：以該頁 screenshot 的 pixel 為座標系統，永遠以 (0, 0) 為左上角

---

## `pairs/pair_NN_MM/summary.json` 格式

```json
{
  "pair": [3, 4],
  "pair_slug": "pair_03_04",
  "pages": [
    {
      "source_page": 3,
      "page_slug": "page_03",
      "rendered_html_path": "outputs-step2/pages/page_03/rendered.html",
      "rendered_screenshot_path": "outputs-step2/pages/page_03/rendered_screenshot.png",
      "source_pdf_path": "outputs-step2/pages/page_03/source_pdf.png",
      "dom_bbox_map_path": "outputs-step2/pages/page_03/dom_bbox_map.json",
      "region_count": 18
    },
    {
      "source_page": 4,
      "page_slug": "page_04",
      "rendered_html_path": "outputs-step2/pages/page_04/rendered.html",
      "rendered_screenshot_path": "outputs-step2/pages/page_04/rendered_screenshot.png",
      "source_pdf_path": "outputs-step2/pages/page_04/source_pdf.png",
      "dom_bbox_map_path": "outputs-step2/pages/page_04/dom_bbox_map.json",
      "region_count": 20
    }
  ]
}
```

### Single-page edge case

`total_pages == 1` 時 pair 為 `(1, None)`，`pages` list 只有 1 個元素，pair_slug 為 `pair_01`。

---

## `all_pairs_summary.json` 格式

```json
{
  "total_source_pages": 26,
  "source_pdf_path": "paper/JME_DL.pdf",
  "annotated_html_path": "outputs-test-std/JME_DL_annotated.html",
  "pages": [
    { "source_page": 1, "page_slug": "page_01", "dom_bbox_map_path": "outputs-step2/pages/page_01/dom_bbox_map.json" }
  ],
  "pairs": [
    { "pair": [1, 2], "pair_slug": "pair_01_02", "summary_path": "outputs-step2/pairs/pair_01_02/summary.json" }
  ]
}
```

---

## Notebook 執行順序

### Phase A: 單頁 Dry Run

固定 `DRY_RUN_PAGE = 3`，跑 cell 8 (single-page dry run)。

### Phase B: 全篇 Batch

跑 cell 9 三階段：
1. `shutil.rmtree(OUT_DIR)` 清空
2. Render 所有 unique pages（共用一個 browser instance）
3. Build 所有 pair summaries
4. 寫 top-level index

---

## 驗證清單

### 結構驗證

- `outputs-step2/pages/` 下每頁都有 4 個 artifact（rendered.html / rendered_screenshot.png / source_pdf.png / dom_bbox_map.json）
- `outputs-step2/pairs/` 下每個 pair 都有 summary.json
- `outputs-step2/all_pairs_summary.json` 存在且可被重新載入
- 沒有 legacy 扁平結構（`pair_NN_MM/` 直接在 OUT_DIR 底下）

### 視覺驗證

- 直接打開 `pages/page_03/rendered.html`，瀏覽器只顯示 page 3
- `rendered_screenshot.png` 樣式延續原 annotated.html

### bbox 驗證

- `pages/page_03/dom_bbox_map.json` 內所有 element 的 `source_page == 3`
- bbox 的 `y` 都從 0 起算（page-local）
- bbox 數量與該頁實際 region 數大致相符

### Dedup 驗證

- 對 26 頁文件，`page_artifacts` 應有 26 entries、`pairs` 應有 25 entries（不是 50 entries）

### 路徑驗證

- pair summary 中的所有 paths 都實際存在
- `all_pairs_summary.json` 中 pages 與 pairs 列表都完整

### PDF 對應驗證

- `pages/page_03/source_pdf.png` 與原 PDF 第 3 頁視覺一致（驗證 0-based vs 1-based 轉換沒寫反）

---

## 已知風險

### 1. Playwright 截圖與 notebook 環境整合

- 本地未安裝 chromium：先跑 `playwright install chromium`
- async 在 notebook 內：用 `await` 直接呼叫，notebook event loop 支援

### 2. 單頁 HTML 抽出後樣式跑掉

- 某些 CSS 依賴整份 HTML 的外層容器
- 緩解：完整深拷貝原 `<head>`；單頁的 `.page` 結構本身已是獨立 visual unit

### 3. bbox 座標系統與 screenshot 尺寸不一致

- viewport 寬度 vs full page width 不一致時 bbox 的 `x` 會偏
- 緩解：明確記錄 `viewport` 與 `page_pixel_size`，judge 統一用 `page_pixel_size` 座標系統

### 4. screenshot 過長被 `max_height` 截斷

- 一些 page 渲染後可能超過 3000px，超過部分被截掉，下半 region 的 bbox 仍在 map 裡但實際不在圖上
- 第二版才處理：切多段、或對單頁設更高 max_height

### 5. Step 1 注入 `data-source-page` 有 bug

- `extract_page_node` 找到多於一個會 raise，立即發現
- `DOMBBoxElement.source_page` 與 header 不一致也能被驗證

---

## 第一版不追求的事

- 完整 class 抽象
- LangGraph node wiring
- 過長截圖切片
- judge-ready prompt
- patch apply

第一版只需要確保：後面不管誰要看圖、對 bbox、修 HTML，都有一致且可信的輸入。

---

## 實作優先順序

1. `make_overlapping_page_pairs`、`make_pair_slug`、`make_page_slug`
2. `load_annotated_soup`、`extract_page_node`
3. `build_page_html`、`save_page_html`
4. `render_pdf_page`
5. Pydantic schemas（`DOMBBox*`、`Viewport`、`PagePixelSize`）
6. `capture_page_screenshot_and_bboxes`（含 browser reuse）
7. `run_page_pipeline`
8. `build_and_save_pair_summary`
9. `save_all_pairs_summary`
10. Single-page dry run cell
11. Three-stage batch orchestration cell
12. Visual check cell

這個順序的好處是：純 Python 結構先穩定，最後才碰 Playwright；單頁 pipeline 跑通再做 batch；render 跑通再做 summary 組裝。

---

## 下一步銜接

Step 2-1 完成後，下一份子計畫應該處理：

- Judge input schema（4 張圖 + 2 份 bbox map）
- Judge output JSON schema
- bbox → DOM IoU mapping（per-page，無 pair-local 轉換）
- 單一 error 對應 HTML 片段抽取

Step 2-1 的完成定義不是「修好 HTML」，而是「把 judge/fixer 需要的中介層建好」。
