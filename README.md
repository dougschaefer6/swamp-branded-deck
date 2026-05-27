# @dougschaefer/branded-deck

An on-brand slide deck pipeline for swamp. A workflow renders a deck from a
voiced brief and a design reference, and a bundled skill carries the creative
half: authoring the copy in a writing-voice profile, pinning brand tokens, and
converting the result to PPTX.

## What's inside

- **`compose-branded-deck` workflow** runs `@dougschaefer/openai-document`'s
  `webpageDesign` method: GPT vision matches a reference design (a page URL or
  image) and a headless browser renders the deck to HTML, PDF, and PNG.
- **`branded-deck` skill** covers the judgment the workflow cannot mechanize:
  load the voice profile, choose the reference, pull exact brand tokens, author
  a sparse voiced brief (slide-density and layout rules), run the workflow,
  review the render, and optionally convert the PDF to a 16:9 PPTX.

## Dependencies

Declared in the manifest and resolved on pull:

- `@dougschaefer/openai-document` — the render engine (`webpageDesign`).
- `@dougschaefer/writing-voice` — the copy voice profile.
- `@dougschaefer/openai-image` — optional original imagery to embed.

## Setup the consumer completes

The workflow and skill reference model instances and a secret, which are
per-environment and do not travel with the extension:

- A `doc-processor` instance of `@dougschaefer/openai-document` with an OpenAI
  API key in its vault (the `webpageDesign` step reads it).
- An `asei-voice` instance of `@dougschaefer/writing-voice` (or your own voice
  instance, adjusting the skill) for the copy voice.
- For the optional PPTX conversion: ImageMagick, Ghostscript, and python-pptx
  installed locally.
- A working headless browser. On WSL the render auto-uses Windows Chrome or
  Edge; otherwise `chromium-browser`, or set `SWAMP_CHROMIUM_BIN`.

## Usage

```bash
swamp workflow run compose-branded-deck --input '{
  "brief": "<slide-by-slide copy + layout directives, authored in voice>",
  "references": ["https://example.com"],
  "styleNotes": "<hex palette + fonts + motifs>",
  "outputName": "my-deck"
}'
```

Artifacts land in `.swamp/generated-documents/`. Follow the `branded-deck` skill
for the full process, including authoring the brief in voice and converting the
PDF to PPTX.

## Testing attestation

Verified in the ASEI lab: the workflow renders a multi-slide deck end to end via
`webpageDesign`, and the PDF-to-PPTX conversion produces a 16:9 deck (one
rendered page per slide).
