# Garment Construction Decoder

Describe a garment and get a structured technical breakdown a sample maker can use. Built with Astro, Netlify Functions, and the Claude API.

## What it does

Enter a garment description — or upload a photo — and get:

| Field | What it answers |
|---|---|
| Seam types | Which finish for each area, and why |
| Fabric behavior | How the fabric behaves before cutting; what the sample maker needs to know |
| Lining | Recommended or not, with type and reason |
| Interfacing | Location, weight, type, and reason — per piece |
| Closure options | Two options with pros, cons, and best-for context |
| Construction order | Numbered sequence; no steps that require undoing prior work |
| Construction notes | Fabric- and style-specific tips |

Optionally: **Traveler Mode** reprioritizes for packability, lightweight construction, and wrinkle resistance.

## Stack

- **Astro** — static site with component-scoped CSS Modules
- **Netlify Functions** — TypeScript serverless handler, keeps API key off the browser
- **Claude API** (`claude-haiku-4-5-20251001`) — structured JSON output, `max_tokens: 1536`, `temperature: 0.3`
- **SVG silhouettes** — seven garment outlines with annotation anchor points; overlay renders left-side leader lines after generation

## Structure

```
src/
  pages/
    index.astro                    # page shell, loads tokens, renders tool
  components/
    GarmentDecoderTool.astro       # full tool UI with silhouette panel
    GarmentDecoderTool.module.css  # silhouette panel, upload zone, annotation overlay
    garmentDecoder.ts              # form logic, API call, DOM rendering
    garmentAnnotations.ts          # SVG silhouette selection + annotation overlay
    garmentImageUpload.ts          # image upload state and drag-and-drop
    silhouettes/
      shirt-dress.svg
      blazer.svg
      slip-dress.svg
      trousers.svg
      a-line-skirt.svg
      oversized-coat.svg
      cropped-jacket.svg
  styles/
    tokens.css                     # CSS custom properties (colors, fonts)
    page.module.css                # page layout, panels, form, output sections

netlify/
  functions/
    garment-decoder.ts             # POST handler → Anthropic API, supports multimodal
```

## SVG silhouettes

Each SVG uses CSS custom properties (`var(--black)`, `var(--muted)`, `var(--border)`, `var(--off-white)`) — no hardcoded hex colors.

Annotation anchors: `<circle id="ap-{name}" cx="…" cy="…" r="2" fill="none"/>` inside a `<g id="ap-">` group. The `garmentAnnotations.ts` module resolves these by ID from the API response.

The annotation overlay renders as an SVG with `viewBox="-130 0 430 500"` positioned to the left of the silhouette, with leader lines from anchor points.

## Setup

### Prerequisites

- Node.js 18+
- Netlify CLI (`npm install -g netlify-cli`)
- An [Anthropic API key](https://console.anthropic.com/)

### Install

```bash
npm install
```

### API key

```bash
cp .env.example .env
# add your key to .env
```

### Local development

```bash
netlify dev
```

Runs Astro and the Netlify Functions together. The tool's API calls go to `/.netlify/functions/garment-decoder`.

> `astro dev` alone will 404 on API calls — use `netlify dev`.

### Build

```bash
npm run build
```

## Key design decisions

**The `reason` field in every recommendation.** Every seam type and closure option includes a reason field. Trust in the output increased significantly when the reasoning was visible.

**Closure disambiguation.** When placement is unclear, the prompt returns two options with tradeoffs rather than guessing. Early versions would confidently recommend the wrong closure for the fabric type.

**Construction order constraint.** The hardest prompt rule: "no steps that require prior steps to be undone." Early versions would sequence zipper insertion after side seams were sewn — physically impossible on certain styles.

**Domain scope limited to womenswear.** Menswear tailoring, knitwear, and technical outerwear are excluded. Different constraint sets would be needed, and the domain knowledge to validate those outputs isn't verified.

**Multimodal support.** Photo uploads are base64-encoded in the browser and sent as part of the API payload. When only a photo is provided (no text description), the response includes a note flagging that analysis is based on visual inference only.

**Annotation overlay math.** The SVG overlay uses `viewBox="-130 0 430 500"` with `width: 143.33%` (= 430/300) and `right: 0`. This ensures the 0–300 garment range maps pixel-perfectly to the silhouette width at all viewport sizes, with 130 units of label space on the left — matching the panel's `padding-left: 130px`.
