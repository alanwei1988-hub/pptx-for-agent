# pptx-for-agent

Turn LLM-authored HTML into editable-quality PowerPoint slides without
sacrificing design fidelity.

## Version: v1.image

This is the **image-embedding** strategy. The LLM writes free-form
HTML/CSS+SVG for each slide; headless Chromium renders each `.slide` element
to a 3840×2160 PNG; `python-pptx` embeds the PNG full-bleed onto a 16:9
slide. The PPTX opens in PowerPoint, exports cleanly to PDF, and looks
exactly like the source HTML.

| | v1.image (this repo) |
|---|---|
| Design ceiling | LLM's full HTML/CSS+SVG ability — no framework cap |
| In-PowerPoint editability | None (rasterised). Edits = re-prompt the LLM. |
| Render fidelity | 1:1 with browser (verified) |
| Stack | Playwright + python-pptx |
| Output size | ~600 KB / slide @ 2x retina |

Future variants may live on other branches (e.g. `v2.dom` — DOM walked into
native PPT shapes — would restore editability at the cost of CSS expressiveness).

## Why this matters

Every existing "AI → PPT" tool (Marp, Slidev, gamma, Tome, Presenton,
Beautiful.AI) constrains the LLM's design ability through some intermediate
schema — Marp markdown, slide JSON, layout templates. That schema is the
bottleneck: the LLM never gets to use the design vocabulary it's strongest
at (raw HTML/CSS+SVG, the same medium that makes Claude's HTML reports look
hand-crafted).

`v1.image` removes the schema entirely. The LLM emits a complete HTML page;
we treat it as the source of truth and rasterise. PowerPoint becomes a
*delivery format*, not a *design environment*.

## Quick start

```bash
# Install
python -m venv .venv
.venv\Scripts\activate            # Windows
# source .venv/bin/activate       # macOS/Linux
pip install -r requirements.txt
playwright install chromium

# Run on the bundled 3-slide demo
python build_pptx.py examples/demo_3slides.html
# → build/demo_3slides.pptx
```

## Input contract

Your HTML must contain one or more elements with `class="slide"`, each
sized `1920×1080` (CSS pixels). Anything else on the page (background,
between-slide markers, scroll padding) is ignored — the screenshot crops to
the `.slide` element's bounding box.

Recommended page skeleton:

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8">
  <link href="https://fonts.googleapis.com/css2?family=Noto+Sans+SC:wght@300;400;700;900&display=swap" rel="stylesheet">
</head>
<body>
  <section class="slide">…1920×1080 of free design…</section>
  <section class="slide">…</section>
</body>
</html>
```

Web fonts are awaited via `document.fonts.ready` before screenshot — Google
Fonts and any `@font-face` will load correctly.

## Roadmap

- [x] Pipeline MVP (this repo)
- [ ] Per-slide LLM driver: `outline.md` → N parallel LLM calls → N HTMLs
- [ ] DESIGN.md skill loader (style guardrails passed into each LLM call)
- [ ] Slide-type prompt library (cover / problem / market / traction / …)
- [ ] Failure recovery (per-slide retry, render-time validation)
- [ ] Watermark / brand stamp injector
- [ ] Optional: image background + editable text overlay (v1.5.hybrid?)

See [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) for the full design rationale.

## License

Private. Authored by @alanwei1988-hub for the 苏秦 (suqin) AI colleague
product. Not open source yet.
