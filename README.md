`source .venv/bin/activate`

```bash
.venv/bin/python scripts/export_provenance_html.py \
  paper/JME_DL.pdf \
  outputs-test/JME_DL_provenance.html \
  --pipeline vlm \
  --image-mode referenced \
  --split-page-view
```

Notes:

- `--pipeline vlm` follows the notebook setup based on `granite_docling` + `MlxVlmEngineOptions()`.
- `--split-page-view` now uses Docling's built-in split-page HTML layout.
- With `--pipeline default --image-mode referenced`, Docling may spend 1-3 minutes in PDF conversion on CPU before HTML is written, especially when formula enrichment is enabled.
- Warnings from `transformers` during model loading do not usually mean the process is stuck.

Fast path when you only need provenance + referenced images and want to skip formula-model download:

```bash
.venv/bin/python scripts/export_provenance_html.py \
  paper/JME_DL.pdf \
  outputs-test/JME_DL_provenance.html \
  --pipeline default \
  --image-mode referenced \
  --skip-formula-enrichment
```

Outputs:

- `outputs-test/JME_DL_provenance.html`
- `outputs-test/JME_DL_provenance_page_origin_map.json`
- `outputs-test/JME_DL_provenance_dom_bbox_map.json`
