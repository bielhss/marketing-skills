---
name: instagram-carousel
description: >
  Creates high-quality Instagram carousels as individual HTML strings
  (1080×1350px), one per slide, packed into a JSON ready for n8n + html2png.dev.
  Handles the full workflow: brand setup, slide copy, visual design system
  (colors, fonts, components), Unsplash image fetching, and HTML generation.
  Use this skill whenever the user asks to create, design, or generate an
  Instagram carousel, carrossel, slides para Instagram, or any Instagram
  multi-image post — even if they don't explicitly say "carousel" or "skill".
  Also trigger for requests to "create a post with multiple slides",
  "fazer carrossel", or "exportar slides para o Instagram".
---

# Instagram Carousel Generator

Generates fully self-contained HTML strings — one per slide — at exactly
1080×1350px, fetching contextual images from Unsplash where appropriate,
then outputs a single JSON with all slides ready for html2png.dev via n8n.

---

## Step 1: Collect Brand Details

<!-- Default brand values — always use, without asking the user -->

1. **Brand name** — rpa teste
2. **Instagram handle** — @rpatestesimp
3. **Primary brand color** — #221a4a, #51515a, #ff6e00, #00b1bc
4. **Logo** — https://raw.githubusercontent.com/bielhss/marketing-skills/refs/heads/main/instagram-carousel/assets/complete%20logo.png
5. **Font preference** — see typography table below, or specific Google Fonts
6. **Tone** — accessible consultative tone, demonstrate technical expertise, but like a close colleague who understands the challenges, not a distant specialist.
7. **Images** — screenshots, product images, etc.
   1. profile photo — https://raw.githubusercontent.com/bielhss/marketing-skills/refs/heads/main/instagram-carousel/assets/icon%20logo.png
8. **Idioma dos slides** — default: **Português (BR)**
9. **Carousel format** — standard (7 slides) or alternate sequence (see sequences section)

If the user provides a website URL or brand assets, derive colors and style from those.

---

## Step 2: Fetch Unsplash Images

### API Setup

The Unsplash API key is injected by n8n directly into the system prompt as a variable. It will appear in the context as a plain string — read it from there and assign it to `UNSPLASH_KEY` in Python before making any requests:

```python
import urllib.request
import urllib.parse
import json

# The key is injected by n8n into the system prompt as {{ $credentials.unsplash_key }}
# When writing the Python code, replace the placeholder below with the actual value
# that was injected — it will be visible in the conversation context as a string.
UNSPLASH_KEY = "INJECTED_BY_N8N_SYSTEM_PROMPT"
```

**How to configure in n8n:**
1. In the AI Agent node, open the **System Prompt**
2. Create a credential of type **Header Auth** in n8n with:
   - Name: `unsplash_key` (or any name you prefer)
   - Value: your Unsplash Access Key
3. Reference it in the system prompt with an expression:
   ```
   Unsplash API Key: {{ $credentials.unsplash_key.value }}
   ```
4. The agent will receive the key as plain text in its context and use it when following this skill

If `UNSPLASH_KEY` is empty or missing from the context, skip all image fetching silently and generate slides without images — never raise an error or halt generation.

### Which slides get images

| Slide type | Image usage | Layout variant |
|---|---|---|
| **Hero (slide 1)** | Full background with dark overlay | Text centered on top of image |
| **Solution / Gradient** | Decorative block | Alternates: right-side or top-banner |
| **Features / Steps (content slides)** | Decorative block | Alternates: right-side or bottom-banner |
| Problem / Challenge | ❌ No image | Dark bg is enough |
| CTA (last slide) | ❌ No image | Brand gradient is enough |

### ⚠️ Query Strategy — CRITICAL

Bad queries are the main reason images feel irrelevant. Follow these rules strictly:

**Rule 1 — Query must match the slide's specific subject, not the general topic**

The query is derived from the **main noun or action of that specific slide**, not the overall carousel theme.

| Slide headline | ❌ Wrong query | ✅ Correct query |
|---|---|---|
| "Por que unir IA e RPA?" | `"technology"` | `"robot arm factory automation"` |
| "Junte IA à automação" | `"business"` | `"computer screen data processing"` |
| "Decisão automática" | `"artificial intelligence"` | `"dashboard analytics decision"` |
| "Escalabilidade" | `"growth"` | `"server rack data center scale"` |
| "Atendimento automatizado" | `"customer"` | `"headset call center software"` |
| "Leitura de documentos" | `"document"` | `"document scanning ocr paper"` |
| "Integração de sistemas" | `"technology"` | `"api connection integration cables"` |
| "Processos financeiros" | `"finance"` | `"spreadsheet invoice accounting desk"` |
| "Monitoramento" | `"monitoring"` | `"screen graphs monitoring alert"` |

**Rule 2 — Self-check before using a query**

Before calling `fetch_unsplash()`, ask internally:
> "If someone searches this on a photo site, would the results show images directly related to what this slide is about?"

If the answer is "probably not" or "maybe" → rewrite the query or use `None` (no image).

**Rule 3 — Banned generic terms**

Never use these as the primary term in a query:
`technology`, `business`, `nature`, `landscape`, `background`, `abstract`, `people`, `team`, `success`, `growth`, `digital`, `future`, `innovation`

These return random stock photos with no semantic link to the slide content.

**Rule 4 — Always in English, 2–4 words, concrete nouns preferred**

Good queries describe a physical object, scene, or action visible in a photo:
- ✅ `"robot arm welding factory"`
- ✅ `"laptop screen code terminal"`
- ✅ `"server rack blue light"`
- ❌ `"smart automation future"`
- ❌ `"digital transformation business"`

### How to fetch an image

```python
def fetch_unsplash(query: str, orientation: str = "portrait") -> str | None:
    """
    Returns the best image URL for the query, or None if unavailable.
    Always validate the query against the rules above before calling this function.
    orientation: 'portrait' for hero full-bg, 'squarish' for decorative blocks
    """
    if not UNSPLASH_KEY or UNSPLASH_KEY == "INJECTED_BY_N8N_SYSTEM_PROMPT":
        return None
    try:
        params = urllib.parse.urlencode({
            "query": query,
            "orientation": orientation,
            "per_page": 3,
            "order_by": "relevant"
        })
        url = f"https://api.unsplash.com/search/photos?{params}"
        req = urllib.request.Request(url, headers={
            "Authorization": f"Client-ID {UNSPLASH_KEY}",
            "Accept-Version": "v1"
        })
        with urllib.request.urlopen(req, timeout=8) as resp:
            data = json.loads(resp.read())
            results = data.get("results", [])
            if results:
                # Pick the first result — per_page:3 gives the API room to rank better
                return results[0]["urls"]["regular"]
    except Exception:
        pass
    return None
```

---

## Step 3: Embed Images in HTML

### A — Full background with overlay (Hero slide)

Used when `image_url` is returned for the Hero slide.

```html
<!-- Inside .slide div, before all content — z-index layering is critical -->

<!-- Layer 0: background image -->
<img src="{IMAGE_URL}"
     style="position:absolute;inset:0;width:100%;height:100%;object-fit:cover;z-index:0;">

<!-- Layer 1: dark overlay so text stays readable -->
<div style="position:absolute;inset:0;background:rgba(0,0,0,0.55);z-index:1;"></div>

<!-- Layer 2+: all slide content (text, logo, progress bar, arrow) -->
```

When using a dark overlay on the Hero:
- Switch all text to white (`#fff`)
- Use `rgba(255,255,255,0.6)` for the tag label
- Use `rgba(255,255,255,0.12)` progress bar track and `#fff` fill
- Use `rgba(255,255,255,0.35)` swipe arrow stroke

If `fetch_unsplash()` returns `None` for the Hero, fall back to `LIGHT_BG` background — no overlay needed.

### B — Decorative block layouts (Solution + Content slides)

These slides alternate between two layout variants to create visual rhythm across the carousel.
Choose the variant based on the slide's position — odd-indexed content slides use Layout B1, even-indexed use Layout B2.

#### B1 — Image right, text left (odd content slides)

Text occupies the left 560px; image floats on the right, vertically centered.

```html
<!-- Decorative image — right side, vertically centered -->
<img src="{IMAGE_URL}"
     style="position:absolute;right:72px;top:50%;transform:translateY(-50%);
            width:390px;height:460px;object-fit:cover;border-radius:20px;
            z-index:2;opacity:0.95;box-shadow:0 20px 56px rgba(0,0,0,0.22);">

<!-- Content wrapper — left side only -->
<div style="position:relative;z-index:3;max-width:560px;">
  <!-- tag, headline, body content -->
</div>
```

#### B2 — Image top-right banner, text below (even content slides)

A wide banner image sits in the upper-right quadrant; text is anchored below it on the left.

```html
<!-- Decorative image — top right banner -->
<img src="{IMAGE_URL}"
     style="position:absolute;right:72px;top:100px;
            width:440px;height:300px;object-fit:cover;border-radius:20px;
            z-index:2;opacity:0.95;box-shadow:0 16px 48px rgba(0,0,0,0.18);">

<!-- Content wrapper — full width, positioned in lower half -->
<div style="position:relative;z-index:3;margin-top:440px;">
  <!-- tag, headline, body content -->
</div>
```

#### Rules for both B layouts:
- **Never reduce font sizes** — keep the mandatory typography scale
- If content is too long → remove words, not pixels
- If `fetch_unsplash()` returns `None` → render slide at full width, no layout shift
- On **gradient/dark slides** using B1: add `box-shadow:0 20px 56px rgba(0,0,0,0.4)` for depth
- On **light slides** using B1 or B2: shadow is `rgba(0,0,0,0.18)`

---

## Step 4: Derive the Full Color System

```
BRAND_PRIMARY   = {main accent}             // Progress bar, icons, tags
BRAND_LIGHT     = {primary lightened ~20%}  // Secondary accent — tags on dark, pills
BRAND_DARK      = {primary darkened ~30%}   // CTA text, gradient anchor
LIGHT_BG        = {warm or cool off-white}  // Light slide background (never pure #fff)
LIGHT_BORDER    = {slightly darker than LIGHT_BG}
DARK_BG         = {near-black with brand tint}
```

- DARK_BG: near-black with subtle brand tint (e.g. `#1A1918` or `#0F172A`)
- Brand gradient: `linear-gradient(165deg, BRAND_DARK 0%, BRAND_PRIMARY 50%, BRAND_LIGHT 100%)`

---

## Step 5: Set Up Typography

| Style | Heading Font | Body Font |
|-------|-------------|-----------|
| Editorial / premium | Playfair Display | DM Sans |
| Modern / clean | Plus Jakarta Sans (700) | Plus Jakarta Sans (400) |
| Warm / approachable | Lora | Nunito Sans |
| Technical / sharp | Space Grotesk | Space Grotesk |
| Bold / expressive | Fraunces | Outfit |
| Classic / trustworthy | Libre Baskerville | Work Sans |
| Rounded / friendly | Bricolage Grotesque | Bricolage Grotesque |

---

## ⚠️ Typography Scale — MANDATORY AND FIXED

These sizes are **non-negotiable**. Apply them identically on every slide, regardless of content volume.

| Element | Size | Weight | Line-height | Notes |
|---|---|---|---|---|
| Tag / category label | 22px | 700 | — | Letter-spacing: 3px, uppercase |
| Main headline | **88–100px** | 700 | 1.08 | Absolute floor: 72px — NEVER below |
| Subheadline / supporting | **52–60px** | 400–500 | 1.25 | Use when headline needs context |
| Feature label / step title | **40px** | 600 | 1.3 | List items, step names |
| Feature description | **30px** | 400 | 1.45 | Supporting text under labels |
| Counter / small UI | 22px | 500 | — | Progress bar counter |

**If content doesn't fit at these sizes → REMOVE words, not pixels.**

Apply via CSS classes `.serif` (heading font) and `.sans` (body font) throughout all slides.

---

## ⚠️ Vertical Alignment — MANDATORY RULES

| Slide type | `justify-content` | `padding` |
|---|---|---|
| Hero (slide 1) | `center` | `100px 90px 140px` |
| Problem / Challenge | `center` | `100px 90px 140px` |
| Solution / Gradient | `center` | `100px 90px 140px` |
| Features / list (≤3 items) | `center` | `100px 90px 140px` |
| Features / list (4+ items) | `flex-end` | `100px 90px 140px` |
| How-to / Steps | `center` | `100px 90px 140px` |
| CTA (last slide) | `center` | `100px 90px 140px` |

**Default is always `center`.** `flex-end` only when 4+ list items genuinely overflow.
Content must **never overlap the progress bar** — `140px` bottom padding guarantees clearance.

---

## Slide 1 — Hook Rules

| Hook format | Example |
|---|---|
| Afirmação polêmica | "Você está usando IA errado" |
| Número + benefício | "7 ferramentas que substituem seu designer" |
| Pergunta que dói | "Por que seus carrosséis têm 0 salvamentos?" |
| Resultado concreto | "Esse post gerou 4.200 seguidores em 3 dias" |
| Inversão de expectativa | "Mais esforço no design = menos alcance" |

- Never start with the brand name as headline
- Hook must promise value that the following slides deliver

---

## Content Density Rules

- Maximum 3 items per list on centered slides
- Maximum 4 items only on `flex-end` slides
- Maximum 2 lines per item description
- When decorative image is present → max-width 580px for content; remove words before reducing font size
- If content exceeds limits → split into more slides or simplify — **NEVER compress**

---

## Slide Sequences

### Standard (7 slides — default)

| # | Type | Background | Image | Layout |
|---|------|------------|-------|--------|
| 1 | Hero | image + dark overlay | ✅ Full background | A — overlay |
| 2 | Problem | DARK_BG | ❌ | — |
| 3 | Solution | Brand gradient | ✅ Decorative block | B1 — image right |
| 4 | Features | LIGHT_BG | ✅ Decorative block | B2 — image top-right banner |
| 5 | Details | DARK_BG | ❌ | — |
| 6 | How-to | LIGHT_BG | ✅ Decorative block | B1 — image right |
| 7 | CTA | Brand gradient | ❌ | — |

### Listicle / Tutorial / Comparação

- Apply image decoration to **content slides only** (light or gradient background)
- Dark background slides never receive images
- Alternate B1 and B2 layouts between consecutive image slides for visual rhythm
- Never apply the same layout variant to two consecutive image slides

---

## Slide Architecture

### Required Elements on Every Slide

#### Progress Bar

```html
<div style="position:absolute;bottom:0;left:0;right:0;padding:32px 60px 40px;z-index:10;display:flex;align-items:center;gap:20px;">
  <div style="flex:1;height:6px;background:{TRACK_COLOR};border-radius:4px;overflow:hidden;">
    <div style="height:100%;width:{PCT}%;background:{FILL_COLOR};border-radius:4px;"></div>
  </div>
  <span style="font-size:22px;color:{LABEL_COLOR};font-weight:500;">{N}/{TOTAL}</span>
</div>
```

- Light slides: track `rgba(0,0,0,0.08)`, fill `BRAND_PRIMARY`, label `rgba(0,0,0,0.3)`
- Dark/gradient/image-overlay slides: track `rgba(255,255,255,0.12)`, fill `#fff`, label `rgba(255,255,255,0.4)`

#### Swipe Arrow (every slide except last)

```html
<div style="position:absolute;right:0;top:0;bottom:0;width:100px;z-index:9;display:flex;align-items:center;justify-content:center;background:linear-gradient(to right,transparent,{BG});">
  <svg width="48" height="48" viewBox="0 0 24 24" fill="none">
    <path d="M9 6l6 6-6 6" stroke="{STROKE}" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round"/>
  </svg>
</div>
```

- Light slides: bg `rgba(0,0,0,0.06)`, stroke `rgba(0,0,0,0.25)`
- Dark/gradient/overlay slides: bg `rgba(255,255,255,0.08)`, stroke `rgba(255,255,255,0.35)`

---

## Reusable Components

### Tag / Category Label
```html
<span class="sans" style="display:inline-block;font-size:22px;font-weight:700;letter-spacing:3px;text-transform:uppercase;color:{COLOR};margin-bottom:24px;">{TAG TEXT}</span>
```

### Feature list item
```html
<div style="display:flex;align-items:flex-start;gap:20px;padding:16px 0;border-bottom:1px solid {LIGHT_BORDER};">
  <span style="color:{BRAND_PRIMARY};font-size:28px;width:36px;text-align:center;">{icon}</span>
  <div>
    <div class="sans" style="font-size:40px;font-weight:600;color:{TEXT_COLOR};line-height:1.3;">{Label}</div>
    <div class="sans" style="font-size:30px;font-weight:400;color:#8A8580;line-height:1.45;">{Description}</div>
  </div>
</div>
```

### Numbered steps
```html
<div style="display:flex;align-items:flex-start;gap:24px;padding:16px 0;border-bottom:1px solid {LIGHT_BORDER};">
  <span class="serif" style="font-size:52px;font-weight:300;color:{BRAND_PRIMARY};min-width:48px;line-height:1;">01</span>
  <div>
    <div class="sans" style="font-size:40px;font-weight:600;color:{TEXT_COLOR};line-height:1.3;">{Step title}</div>
    <div class="sans" style="font-size:30px;font-weight:400;color:#8A8580;line-height:1.45;">{Step description}</div>
  </div>
</div>
```

### Prompt / quote box
```html
<div style="padding:28px 32px;background:rgba(0,0,0,0.15);border-radius:16px;border:1px solid rgba(255,255,255,0.08);margin-top:32px;">
  <p class="sans" style="font-size:22px;color:rgba(255,255,255,0.5);margin-bottom:12px;">{Label}</p>
  <p class="serif" style="font-size:36px;color:#fff;font-style:italic;line-height:1.4;">"{Quote text}"</p>
</div>
```

### CTA button (final slide only)
```html
<div style="display:inline-flex;align-items:center;gap:12px;padding:20px 48px;background:{LIGHT_BG};color:{BRAND_DARK};font-family:'{BODY_FONT}',sans-serif;font-weight:600;font-size:36px;border-radius:56px;">
  {CTA text}
</div>
```

### Logo Lockup (first and last slides)
- Icon logo img 56px circle + brand name at 26px weight 600
- Or initials: 56px circle BRAND_PRIMARY bg, white letter

---

## Line Break Optimization

❌ `"Por que unir IA e RPA?"`
✅ `"Por que unir<br>IA e RPA?"`

---

## HTML Slide Structure

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <link rel="preconnect" href="https://fonts.googleapis.com">
  <link href="https://fonts.googleapis.com/css2?family={HEADING_FONT}:wght@300;600;700&family={BODY_FONT}:wght@400;500;600&display=swap" rel="stylesheet">
  <style>
    * { margin: 0; padding: 0; box-sizing: border-box; }
    body {
      width: 1080px;
      height: 1350px;
      overflow: hidden;
      background: {SLIDE_BG};
    }
    .slide {
      position: relative;
      width: 1080px;
      height: 1350px;
      display: flex;
      flex-direction: column;
      justify-content: center; /* see Vertical Alignment table */
      padding: 100px 90px 140px;
      overflow: hidden;
    }
    .serif { font-family: '{HEADING_FONT}', serif; }
    .sans  { font-family: '{BODY_FONT}', sans-serif; }
  </style>
</head>
<body>
  <div class="slide">
    <!-- [Hero only] background image (z-index:0) + dark overlay (z-index:1) -->
    <!-- [B1] decorative image right: absolute right:72px, vertically centered, z-index:2 -->
    <!-- [B2] decorative image top-right: absolute right:72px top:100px, z-index:2 -->
    <!-- content wrapper (z-index:3) -->
    <!--   B1: max-width:560px -->
    <!--   B2: margin-top:440px -->
    <!--   No image: full width -->
      <!-- tag label -->
      <!-- headline -->
      <!-- subheadline or body content -->
    <!-- /content wrapper -->
    <!-- progress bar (absolute, z-index:10) -->
    <!-- swipe arrow (absolute, z-index:9, all except last) -->
  </div>
</body>
</html>
```

---

## Generation Flow

### Step 1 — Fetch all Unsplash images upfront

Before building any HTML, fetch all needed images in sequence. Store results in a dict.

```python
import os, json, urllib.request, urllib.parse
from pathlib import Path

UNSPLASH_KEY = os.environ.get("UNSPLASH_ACCESS_KEY", "")

def fetch_unsplash(query: str, orientation: str = "portrait") -> str | None:
    if not UNSPLASH_KEY:
        return None
    try:
        params = urllib.parse.urlencode({
            "query": query,
            "orientation": orientation,
            "per_page": 1,
            "order_by": "relevant"
        })
        req = urllib.request.Request(
            f"https://api.unsplash.com/search/photos?{params}",
            headers={"Authorization": f"Client-ID {UNSPLASH_KEY}", "Accept-Version": "v1"}
        )
        with urllib.request.urlopen(req, timeout=8) as resp:
            data = json.loads(resp.read())
            results = data.get("results", [])
            if results:
                return results[0]["urls"]["regular"]
    except Exception:
        pass
    return None

# Fetch images for each slide that needs one
# Derive queries from the carousel topic
images = {
    "hero":     fetch_unsplash("{topic_query}", orientation="portrait"),
    "solution": fetch_unsplash("{topic_query}", orientation="landscape"),
    "features": fetch_unsplash("{topic_query}", orientation="landscape"),
    "howto":    fetch_unsplash("{topic_query}", orientation="landscape"),
}
# Any None value means: render that slide without an image — never raise errors
```

### Step 2 — Build all slide HTML strings

```python
# Use images["hero"], images["solution"], etc. when constructing each slide
# If value is None, omit the <img> tag and render at full width
slide_1_html = f"""<!DOCTYPE html>..."""
# ... etc
slides_html = [slide_1_html, slide_2_html, ...]
```

### Step 3 — Save JSON output

```python
output = {
    "slides": [
        {"slide": i + 1, "html": html}
        for i, html in enumerate(slides_html)
    ]
}
Path("/home/claude/carousel_output.json").write_text(
    json.dumps(output, ensure_ascii=False), encoding="utf-8"
)
print(f"Generated {len(slides_html)} slides.")
```

### Step 4 — Copy and present

```bash
cp /home/claude/carousel_output.json /mnt/user-data/outputs/carousel_output.json
```

Call `present_files` with `/mnt/user-data/outputs/carousel_output.json`.

---

## Output Format

```json
{
  "slides": [
    { "slide": 1, "html": "<!DOCTYPE html>..." },
    { "slide": 2, "html": "<!DOCTYPE html>..." }
  ]
}
```

### Structured Output Parser schema (n8n)

```json
{
  "type": "object",
  "properties": {
    "slides": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "slide": { "type": "integer" },
          "html":  { "type": "string" }
        },
        "required": ["slide", "html"]
      }
    }
  },
  "required": ["slides"]
}
```

### n8n Code node — split slides

```javascript
const { slides } = $input.first().json;
return slides.map(slide => ({
  json: { slide: slide.slide, html: slide.html }
}));
```

---

## Setting Up the Unsplash API Key

### 1. Criar a chave no Unsplash
1. Acesse [unsplash.com/developers](https://unsplash.com/developers) e crie uma conta
2. Clique em **New Application**, preencha nome e descrição
3. Copie o **Access Key** gerado

### 2. Criar a credential no n8n
1. No n8n, vá em **Settings → Credentials → New Credential**
2. Escolha o tipo **Header Auth**
3. Preencha:
   - **Name:** `Unsplash Access Key` (ou qualquer nome)
   - **Value:** cole o Access Key copiado do Unsplash
4. Salve

### 3. Injetar no system prompt do agente
No nó **AI Agent**, adicione ao final do System Prompt:

```
Unsplash API Key: {{ $credentials["Unsplash Access Key"].value }}
```

O agente receberá a chave como texto no contexto e a utilizará ao seguir esta skill.

### Limites do plano gratuito
- **50 requests/hora** — mais que suficiente para geração de carrosséis
- Máximo de 3 imagens por carrossel nesta skill (hero + solution + features/steps)

---

## Design Principles

1. **Images are optional enhancements** — slides always render correctly without them
2. **Hero image creates impact** — dark overlay ensures text legibility at any image
3. **Decorative images add context** — right-side positioning never competes with copy
4. **Content always wins** — if image + text don't fit, remove words, never shrink fonts
5. **Light/dark alternation** — visual rhythm across swipes
6. **Typography is fixed** — same scale on every slide, no exceptions
7. **Vertical centering by default** — `flex-end` only for 4+ item lists
8. **One JSON output** — all slides as HTML strings, ready for html2png.dev via n8n