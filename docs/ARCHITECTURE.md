# Architecture — v1.image

## The problem this repo solves

LLMs are *very* good at writing visually polished HTML. Anyone who has read
a Claude-generated HTML report has seen this — the typography, layout,
colour systems, inline SVG illustrations, and hierarchy are easily on par
with what a senior designer produces in a Figma session.

LLMs are *not* good at producing PowerPoint. Every "AI → PPT" tool we
evaluated forces the LLM through some intermediate representation:

| Tool | Bottleneck |
|---|---|
| Marp | Markdown + a single `<style>` block per deck. CSS surface is real but the markdown layer strips embedded HTML, so any `<div class="card">` becomes literal text. |
| Slidev | Vue components. The LLM has to know your component API. |
| gamma / Tome / Beautiful.AI | Internal slide JSON schema. LLM emits content; their renderer applies the design. The design ceiling is fixed by their template library. |
| Presenton | Same shape — schema-driven generator. |
| python-pptx direct | Element-by-element placement in EMU coordinates. Theoretically unlimited, practically the LLM cannot reason about overlap, text overflow, font metrics. Output looks like 1995 PowerPoint. |

In all cases the LLM never gets to use its strongest design medium. We
proved this by hand: a Marp deck produced by Claude looked plain; a
free-form HTML page produced by the same model in the same session looked
like a magazine.

## The bet

**Drop the schema. Treat HTML as the source of truth and PowerPoint as a
delivery format.**

```
                  ┌─────────────────────┐
   user prompt ──▶│ LLM (Sonnet/Opus)   │── outline.md ──┐
                  └─────────────────────┘                │
                                                         ▼
                  ┌─────────────────────────────────────────────────────┐
                  │ For each slide section, in parallel:                │
                  │   prompt = outline_section + DESIGN.md +            │
                  │            slide_type_template                      │
                  │   response = full HTML/CSS+SVG for one .slide       │
                  └─────────────────────────────────────────────────────┘
                                                         │
                                              N HTML files (one per slide)
                                              concatenated into one page
                                                         │
                                                         ▼
                  ┌─────────────────────┐
                  │ Headless Chromium   │── one PNG per .slide @ 3840×2160
                  │ (Playwright)        │
                  └─────────────────────┘
                                                         │
                                                         ▼
                  ┌─────────────────────┐
                  │ python-pptx         │── full-bleed picture per slide
                  │ blank layout        │   (16:9, 13.333" × 7.5")
                  └─────────────────────┘
                                                         │
                                                         ▼
                                                    output.pptx
```

## What v1.image gives up

**In-PowerPoint editability.** Every slide is a single picture. Users
cannot click into a text box and change "¥3000 万" to "¥3500 万". This is
the explicit price of design freedom.

The product answer: editing happens upstream by re-prompting. A user who
notices a typo says "把第 5 页的 ¥3000 万改成 ¥3500 万" and the system
re-runs the LLM call for slide 5 only, re-renders, and swaps the picture.
That's the AI-native workflow this repo is designed for. If you need a
classical "double-click to edit" PPT, this isn't your tool.

## Engineering decisions

### Why per-slide LLM calls (not one big call)

A single call asking for 10 slides at once produces a noticeable design
quality drop after slide ~7 — context dilution, attention budget. Calling
N times in parallel restores per-slide attention and roughly equalises
quality across the deck. Cost goes up linearly; wall-clock stays roughly
constant when parallelised. Failure isolation is a free side benefit:
slide 7 retry doesn't invalidate slides 1-6.

### Why 1920×1080 not 3840×2160 in CSS

CSS pixels stay at 1920×1080 (the LLM's mental model is desktop-resolution).
We bump `device_scale_factor=2` at the Playwright layer so the rendered PNG
is 3840×2160. This way:

- the LLM writes natural CSS (`font-size: 64px` not `128px`)
- the screenshot is retina-sharp when displayed at slide size
- the PPTX file size is acceptable (~600 KB / slide)

### Why screenshot the `.slide` element, not the full page

Two reasons. First, the example HTML uses dark gutter backgrounds + slide
shadows for "floating card" visual presentation in browser, but those
should not appear in PPT. Second, element-bound screenshots crop exactly
to the slide's box, so any user-side scroll padding or between-slide
markers is automatically excluded.

### Why blank PPT layout, not master-derived

We use `slide_layouts[6]` (fully blank). Master placeholders would compete
with our picture and add invisible structural noise. The blank slide is
literally a transparent canvas onto which we paste one full-bleed picture.

### Font loading

Playwright is told to wait until `document.fonts.ready` resolves before
screenshotting. This handles Google Fonts, `@font-face` imports, and any
other web font source. There is a 500ms safety margin after `fonts.ready`
to allow the first paint with the new font to settle — without it you
occasionally get a half-rendered glyph on the first character.

## Open questions for v2

- **v2.dom** — walk the rendered DOM and reconstruct as native PPT shapes,
  giving up some visual fidelity (gradients, complex SVG) for full
  editability. Maps roughly to `dom-to-pptx` library territory.
- **v1.5.hybrid** — render background as image but extract heading text and
  re-create as editable text frames overlaid in the same position. Keeps
  the design but lets users tweak headlines.
- **Brand consistency at scale** — when `DESIGN.md` is loaded as a skill
  into N parallel LLM calls, does each slide actually obey the same colour
  palette / typography ladder? Or do the calls drift? Needs measurement.
- **Render budget** — Playwright cold start is ~2-3s per call, screenshot
  is ~200ms per slide. For 10 slides total wall-clock is dominated by LLM
  generation (~30-60s), but if we move to 50-page reports the render
  pipeline becomes the bottleneck. Consider a long-lived browser process.
