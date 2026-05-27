---
name: branded-deck
description: >-
  Produce an on-brand slide deck that matches a reference design and sounds like
  the writing-voice profile, via the compose-branded-deck workflow. Use when
  asked to build a branded deck, marketing deck, sales deck, pitch deck, slide
  deck, or on-brand presentation from a website or design reference. Triggers on
  "branded deck", "marketing deck", "sales deck", "pitch deck", "make a deck",
  "slide deck", "on-brand presentation", "deck from <site>".
---

# Branded Deck

Produce an on-brand slide deck by pairing the **writing-voice** profile (how the
copy sounds) with **openai-document.webpageDesign** (how it looks), rendered and
optionally converted to PPTX by the `compose-branded-deck` workflow. The
workflow handles the deterministic half (vision-match the reference, render
HTML/PDF/PNG, optional PPTX). This skill covers the judgment the workflow
cannot: authoring the brief in voice, choosing the reference, and pinning the
brand tokens.

## Prerequisites

- A `doc-processor` instance of `@dougschaefer/openai-document` with the OpenAI
  API key in its vault (the `webpageDesign` method reads it).
- An `asei-voice` instance of `@dougschaefer/writing-voice` for the copy voice.
- Optional PPTX step: ImageMagick (`convert`) + Ghostscript + python3 with
  `python-pptx` installed locally.
- A working headless browser. On WSL the extension auto-uses Windows
  Chrome/Edge; otherwise `chromium-browser`, or set `SWAMP_CHROMIUM_BIN`.

## Steps

1. **Load the voice.** Follow the `writing-voice` skill, or refresh and read the
   profile directly:
   ```bash
   swamp model method run asei-voice get --json >/dev/null
   swamp data get asei-voice voice-profile --json
   ```
   Internalize `voiceIdentity`, `proseRules`, `antiPatterns`, and `killList`.
   On a deck the voice governs **diction and tone, not word count** — keep
   slides sparse and let layout carry the weight.

2. **Choose the design reference.** A page URL (e.g. the product's site) or
   local image(s). `webpageDesign` rasterizes a page URL automatically, so pass
   the URL directly in `references`.

3. **Pin the brand tokens (recommended).** Vision infers color and type from the
   screenshot, but exact hex and fonts are more reliable stated explicitly. Pull
   them from the reference's CSS and pass as `styleNotes`:
   ```bash
   curl -sL <url> -o /tmp/ref.html
   grep -oE '#[0-9a-fA-F]{6}' /tmp/ref.html | sort | uniq -c | sort -rn | head
   grep -oiE 'font-family:[^;}"]+' /tmp/ref.html | sort -u | head
   ```

4. **Author the brief.** Write it slide by slide, verbatim copy plus layout
   directives, in the loaded voice. Hard rules: 16:9 paginated panels (one
   `<section class="slide">` at 1280x720, `page-break-after: always`, plus
   `@page { size: 1280px 720px; margin: 0 }`); sparse on-slide text; **no em
   dashes**; scan the copy against the `killList` before running. A row of four
   cards must be a 2x2 grid — a single row of four overflows 1280px.

5. **Run the workflow.**
   ```bash
   swamp workflow run compose-branded-deck --input '{
     "brief": "<the slide brief>",
     "references": ["https://swamp.club"],
     "styleNotes": "<hex palette + fonts + motifs>",
     "outputName": "my-deck"
   }' --json
   ```
   Artifacts land in `.swamp/generated-documents/` (HTML, PDF, PNG). Set
   `SWAMP_OUTPUT_COPY_DIR` to also drop copies somewhere handy (e.g. a Downloads
   folder).

6. **Review the render.** Convert a couple of PDF pages to PNG and inspect for
   overflow and legibility, then iterate the brief if needed:
   ```bash
   convert -density 96 "<deck>.pdf[2]" /tmp/slide3.png
   ```

7. **Optional: convert to PPTX.** Turn the rendered PDF into a 16:9 PPTX (one
   page per slide) for PowerPoint or Google Slides. This repo ships a one-shot
   wrapper; the equivalent inline needs only ImageMagick + python-pptx:
   ```bash
   bash scripts/deck-pdf-to-pptx.sh .swamp/generated-documents/<deck>-<ts>.pdf
   # or inline, anywhere ImageMagick + python-pptx are installed:
   convert -density 150 -background black <deck>.pdf -alpha remove /tmp/s.png
   python3 - <<'PY'
   import glob, os
   from pptx import Presentation
   from pptx.util import Inches
   prs = Presentation(); prs.slide_width = Inches(13.333); prs.slide_height = Inches(7.5)
   pages = sorted(glob.glob('/tmp/s*.png'),
                  key=lambda p: int(''.join(c for c in os.path.basename(p) if c.isdigit()) or 0))
   for f in pages:
       s = prs.slides.add_slide(prs.slide_layouts[6])
       s.shapes.add_picture(f, 0, 0, width=prs.slide_width, height=prs.slide_height)
   prs.save('<deck>.pptx')
   PY
   ```

## Notes

- The PPTX is a faithful image of each slide (one rendered page per slide), not
  editable text boxes. That is the expected PDF→PPTX tradeoff; it presents
  cleanly in PowerPoint and Google Slides.
- For original hero art or icons, generate them with the
  `@dougschaefer/openai-image` model and point the brief at the saved files;
  `webpageDesign` embeds local images you reference.
