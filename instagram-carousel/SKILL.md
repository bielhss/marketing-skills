---
name: instagram-carousel
description: >
  Creates high-quality Instagram carousels as individual HTML strings
  (1080×1350px), one per slide, packed into a JSON ready for n8n + html2png.dev.
  Handles the full workflow: brand setup, slide copy, visual design system
  (colors, fonts, components), image fetching via image-search skill, and HTML
  generation. Use this skill whenever the user asks to create, design, or
  generate an Instagram carousel, carrossel, slides para Instagram, or any
  Instagram multi-image post — even if they don't explicitly say "carousel" or
  "skill". Also trigger for requests to "create a post with multiple slides",
  "fazer carrossel", or "exportar slides para o Instagram".
---

# Instagram Carousel Generator

Generates fully self-contained HTML strings — one per slide — at exactly
1080×1350px, fetching contextual images via the **image-search skill**, then
outputs a single JSON with all slides ready for html2png.dev via n8n.

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

## Step 2: Fetch Images via image-search Skill

### ⚠️ CRITICAL: Never call image provider tools directly

This skill does NOT call `Search Unsplash`, `Search Pexels`, or any image
provider tool directly. **All image fetching is delegated to the image-search
skill.**

**How to request an image:**

For each slide that needs an image, call the **image-search skill** with:

```json
{
  "topic": "<slide headline or short topic description>",
  "orientation": "portrait" | "landscape"
}
```

The image-search skill returns:

```json
{
  "url":         "https://...",
  "source":      "unsplash" | "pexels" | null,
  "alt_text":    "descriptive text",
  "orientation": "landscape"
}
```

Use `url` directly in the `<img src="">` tag.
If `url` is null, render the slide without an image using the text-only
fallback (see Fallback Rules below).

---

### Which slides get images

| Slide type | Image usage | Layout variant |
|---|---|---|
| **Hero (slide 1)** | Full background with dark overlay | Layout A — overlay |
| **Solution / Gradient** | Decorative block | Random: C1, C2, or C3 |
| **Features / Steps** | Decorative block | Random: C1, C2, or C3 (≠ previous) |
| Problem / Challenge | ❌ No image | Dark bg is enough |
| CTA (last slide) | ❌ No image | Brand gradient is enough |

---

### Orientation by slide type

| Slide type | Orientation to request |
|---|---|
| Hero (slide 1) | `portrait` |
| All other image slides | `landscape` |

---

### Topic description for image-search

Pass a short English description of what the slide is about — not the
headline text verbatim. The image-search skill will apply its own query
strategy on top of this.

Examples:

| Slide headline | Topic to pass to image-search |
|---|---|
| "Automatize seus processos" | `"workflow automation"` |
| "Reduza erros operacionais" | `"quality checklist process"` |
| "IA + RPA trabalhando juntos" | `"automation AI integration"` |
| "Como começar em 3 passos" | `"workflow process steps"` |

---

### Fetch sequence (run ALL fetches before writing any HTML)

1. Pick a random layout for each image slide (C1 / C2 / C3 — see Layout
   Selection below)
2. For each image slide, call image-search with topic + orientation
3. Store all returned URLs (or null values)
4. Only then build the HTML for all slides

---

## Step 3: Layout Selection

For each image slide, pick one of three layouts at random.
Never assign the same layout to two consecutive image slides.

```
Available layouts: C1, C2, C3

Slide 3 → pick from [C1, C2, C3]         → e.g. C2
Slide 4 → pick from [C1, C3] (not C2)    → e.g. C3
Slide 6 → pick from [C1, C2] (not C3)    → e.g. C1
```

---

## Step 4: Embed Images in HTML

### A — Full background with overlay (Hero slide)

Used when image-search returns a non-null URL for the Hero slide.

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

**Hero logo — MANDATORY:** always render the icon logo using the real image
URL from brand details. Never recreate it as text, initials, or a colored shape:

```html
<img src="{ICON_LOGO_URL}"
     style="position:relative;z-index:2;height:48px;width:auto;object-fit:contain;display:block;margin-bottom:20px;"
     alt="logo">
```

If image-search returns null for the Hero, fall back to `LIGHT_BG`
background — no overlay needed.

---

### B — Decorative image layouts (Solution + Content slides)

#### C1 — Image top, text bottom

**Pixel budget:**
- Image: `top:0` → `height:580px` (rounded bottom corners)
- Text: `top:600px` → `height:560px`
- Progress bar: bottom 140px

```html
<div class="slide" style="justify-content:flex-start;padding:0;">
  <img src="{IMAGE_URL}"
       onerror="this.style.display='none';var t=document.getElementById('txt');if(t){t.style.top='100px';t.style.height='1110px';}"
       style="position:absolute;top:0;left:0;right:0;
              width:100%;height:580px;object-fit:cover;
              border-radius:0 0 32px 32px;z-index:1;opacity:0.97;">
  <div id="txt"
       style="position:absolute;left:80px;top:600px;right:80px;
              height:560px;overflow:hidden;
              display:flex;flex-direction:column;justify-content:flex-start;z-index:3;">
    <!-- tag, headline (max 2 lines at 84px), body (max 3 lines at 42px) -->
  </div>
  <!-- progress bar: position:absolute;bottom:0;left:0;right:0;z-index:10 -->
  <!-- swipe arrow: z-index:9 -->
</div>
```

#### C2 — Image top, floating card overlay

**Pixel budget:**
- Image: `top:0` → `height:680px`
- Card: `top:600px` → `bottom:100px`
- Progress bar: bottom 100px

```html
<div class="slide" style="justify-content:flex-start;padding:0;background:{SLIDE_BG};">
  <img src="{IMAGE_URL}"
       onerror="this.style.display='none';var c=document.getElementById('card');if(c){c.style.top='100px';c.style.bottom='100px';c.style.boxShadow='none';}"
       style="position:absolute;top:0;left:0;right:0;
              width:100%;height:680px;object-fit:cover;z-index:1;">
  <div id="card"
       style="position:absolute;left:56px;right:56px;
              top:600px;bottom:100px;
              background:{CARD_BG};border-radius:28px;
              box-shadow:0 -8px 40px rgba(0,0,0,0.18);
              padding:44px 56px;overflow:hidden;
              display:flex;flex-direction:column;justify-content:flex-start;
              z-index:4;">
    <!-- tag, headline, body -->
  </div>
  <!-- progress bar: position:absolute;bottom:0;left:0;right:0;z-index:10 — OUTSIDE card -->
  <!-- swipe arrow: z-index:9 -->
</div>
```

**CARD_BG:** light slides → `#FFFFFF` · dark slides → `{DARK_BG}`

#### C3 — Text top, image bottom

**Pixel budget:**
- Text: `top:100px` → `height:560px`
- Image: `top:700px` → `bottom:0` (height:650px, rounded top corners)
- Progress bar overlays image bottom — use white track/fill

```html
<div class="slide" style="justify-content:flex-start;padding:0;background:{SLIDE_BG};">
  <div id="txt"
       style="position:absolute;left:80px;top:100px;right:80px;
              height:560px;overflow:hidden;
              display:flex;flex-direction:column;justify-content:flex-start;z-index:3;">
    <!-- tag, headline, body -->
  </div>
  <img src="{IMAGE_URL}"
       onerror="this.style.display='none';var t=document.getElementById('txt');if(t){t.style.top='100px';t.style.height='1110px';}"
       style="position:absolute;top:700px;left:0;right:0;bottom:0;
              width:100%;height:650px;object-fit:cover;
              border-radius:32px 32px 0 0;z-index:1;opacity:0.97;">
  <!-- progress bar: white track rgba(255,255,255,0.2) + white fill — sits over image -->
  <!-- swipe arrow: z-index:9 -->
</div>
```

#### ⚠️ Critical rules for ALL C layouts

1. **Zones never overlap** — verify pixel coordinates before writing HTML
2. **`overflow:hidden` mandatory** on every text wrapper and card
3. **Font sizes are fixed** — if content doesn't fit, remove words, never shrink
4. **Body text: MAX 3 lines at 42px** across all C layouts
5. **Progress bar lives on `.slide`** — never inside a card or text wrapper
6. **C2 card: `bottom:100px` is non-negotiable**
7. **C3 progress bar**: white track `rgba(255,255,255,0.2)` and white fill `#fff`
8. **Always add `onerror`** on every `<img>` in C layouts — hide image and
   expand text/card zone on failure
9. **If image-search returned null** for this slide → omit the `<img>` tag
   entirely and set `.slide` to `justify-content:center; padding:100px 90px 140px`

---

## Step 5: Derive the Full Color System

```
BRAND_PRIMARY   = {main accent}             // Progress bar, icons, tags
BRAND_LIGHT     = {primary lightened ~20%}  // Secondary accent
BRAND_DARK      = {primary darkened ~30%}   // CTA text, gradient anchor
LIGHT_BG        = {warm or cool off-white}  // Light slide background
LIGHT_BORDER    = {slightly darker than LIGHT_BG}
DARK_BG         = {near-black with brand tint}
```

- DARK_BG: near-black with subtle brand tint (e.g. `#1A1918` or `#0F172A`)
- Brand gradient: `linear-gradient(165deg, BRAND_DARK 0%, BRAND_PRIMARY 50%, BRAND_LIGHT 100%)`

---

## Step 6: Set Up Typography

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

| Element | Size | Weight | Line-height | Notes |
|---|---|---|---|---|
| Tag / category label | 22px | 700 | — | Letter-spacing: 3px, uppercase |
| Main headline | **88–100px** | 700 | 1.08 | Absolute floor: 72px — NEVER below |
| Subheadline / supporting | **52–60px** | 400–500 | 1.25 | |
| Feature label / step title | **40px** | 600 | 1.3 | |
| Feature description | **30px** | 400 | 1.45 | |
| Counter / small UI | 22px | 500 | — | Progress bar counter |

**If content doesn't fit at these sizes → REMOVE words, not pixels.**

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

Content must **never overlap the progress bar** — `140px` bottom padding
guarantees clearance.

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
- When decorative image is present → max-width 580px for content
- If content exceeds limits → split into more slides or simplify

---

## Slide Sequences

### Standard (7 slides — default)

| # | Type | Background | Image |
|---|------|------------|-------|
| 1 | Hero | image + dark overlay | ✅ portrait |
| 2 | Problem | DARK_BG | ❌ |
| 3 | Solution | LIGHT_BG | ✅ landscape (random C layout) |
| 4 | Features | DARK_BG or LIGHT_BG | ✅ landscape (random C layout, ≠ slide 3) |
| 5 | Details | DARK_BG | ❌ |
| 6 | How-to | LIGHT_BG | ✅ landscape (random C layout, ≠ slide 4) |
| 7 | CTA | Brand gradient | ❌ |

---

## Slide Architecture

### Progress Bar (required on every slide)

```html
<div style="position:absolute;bottom:0;left:0;right:0;padding:32px 60px 40px;z-index:10;display:flex;align-items:center;gap:20px;">
  <div style="flex:1;height:6px;background:{TRACK_COLOR};border-radius:4px;overflow:hidden;">
    <div style="height:100%;width:{PCT}%;background:{FILL_COLOR};border-radius:4px;"></div>
  </div>
  <span style="font-size:22px;color:{LABEL_COLOR};font-weight:500;">{N}/{TOTAL}</span>
</div>
```

- Light slides: track `rgba(0,0,0,0.08)`, fill `BRAND_PRIMARY`, label `rgba(0,0,0,0.3)`
- Dark/gradient/overlay: track `rgba(255,255,255,0.12)`, fill `#fff`, label `rgba(255,255,255,0.4)`

### Swipe Arrow (every slide except last)

```html
<div style="position:absolute;right:0;top:0;bottom:0;width:100px;z-index:9;display:flex;align-items:center;justify-content:center;background:linear-gradient(to right,transparent,{BG});">
  <svg width="48" height="48" viewBox="0 0 24 24" fill="none">
    <path d="M9 6l6 6-6 6" stroke="{STROKE}" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round"/>
  </svg>
</div>
```

- Light slides: bg `rgba(0,0,0,0.06)`, stroke `rgba(0,0,0,0.25)`
- Dark/gradient/overlay: bg `rgba(255,255,255,0.08)`, stroke `rgba(255,255,255,0.35)`

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

### CTA button (final slide only)
```html
<div style="display:inline-flex;align-items:center;gap:12px;padding:20px 48px;background:{LIGHT_BG};color:{BRAND_DARK};font-family:'{BODY_FONT}',sans-serif;font-weight:600;font-size:36px;border-radius:56px;">
  {CTA text}
</div>
```

### Logo Lockup

```html
<!-- Complete logo (CTA/last slide) -->
<img src="{COMPLETE_LOGO_URL}"
     style="height:52px;width:auto;object-fit:contain;display:block;margin-bottom:16px;"
     alt="logo">

<!-- Icon logo (hero slide top-left) -->
<img src="{ICON_LOGO_URL}"
     style="height:48px;width:48px;object-fit:contain;display:block;margin-bottom:20px;"
     alt="logo">
```

Rules:
- **Never** apply `border-radius` to logo images
- **Never** recreate the logo with text, initials, or colored circles
- If a logo URL fails to load, omit the logo — do not substitute with text

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
    body { width: 1080px; height: 1350px; overflow: hidden; background: {SLIDE_BG}; }
    .slide {
      position: relative;
      width: 1080px; height: 1350px;
      display: flex; flex-direction: column;
      justify-content: center;
      padding: 100px 90px 140px;
      overflow: hidden;
    }
    .serif { font-family: '{HEADING_FONT}', serif; }
    .sans  { font-family: '{BODY_FONT}', sans-serif; }
    h1, h2, p, span, div { word-wrap: break-word; overflow-wrap: break-word; }
  </style>
</head>
<body>
  <div class="slide">
    <!-- content -->
    <!-- progress bar (absolute, z-index:10) -->
    <!-- swipe arrow (absolute, z-index:9) -->
  </div>
</body>
</html>
```

---

## Generation Flow

### Step 1 — Plan layouts and fetch all images upfront

**A — Pick layouts for image slides**

Pick one of C1 / C2 / C3 at random for each image slide.
Never repeat the same layout on consecutive image slides.

**B — Call image-search skill for every image slide**

Make ALL image-search calls before writing any HTML.

| Slide | Topic example | Orientation |
|---|---|---|
| Hero (slide 1) | `"office team laptop working"` | `portrait` |
| Solution (slide 3) | `"workflow automation"` | `landscape` |
| Features (slide 4) | `"productivity efficiency"` | `landscape` |
| How-to (slide 6) | `"workflow process steps"` | `landscape` |

Replace example topics with ones matching the actual carousel subject.

**C — After all image-search calls complete → build HTML**

### Step 2 — Build all slide HTML strings

```python
# Use images["hero"], images["solution"], etc. when constructing each slide
# If value is None/null, omit the <img> tag and render at full width
slide_1_html = f"""<!DOCTYPE html>..."""
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
```

### Step 4 — Copy and present

```bash
cp /home/claude/carousel_output.json /mnt/user-data/outputs/carousel_output.json
```

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

### n8n Code node — split slides

```javascript
const { slides } = $input.first().json;
return slides.map(slide => ({
  json: { slide: slide.slide, html: slide.html }
}));
```

---

## Design Principles

<<<<<<< HEAD
1. **Images are optional enhancements** — slides always render correctly without them
2. **Hero image creates impact** — dark overlay ensures text legibility at any image
3. **Decorative images add context** — right-side positioning never competes with copy
4. **Content always wins** — if image + text don't fit, remove words, never shrink fonts
5. **Light/dark alternation** — visual rhythm across swipes
6. **Typography is fixed** — same scale on every slide, no exceptions
7. **Vertical centering by default** — `flex-end` only for 4+ item lists
8. **One JSON output** — all slides as HTML strings, ready for html2png.dev via n8n
=======
1. **image-search handles all image fetching** — never call provider tools directly
2. **Images are optional enhancements** — slides always render correctly without them
3. **Hero image creates impact** — dark overlay ensures text legibility
4. **Decorative images add context** — never compete with copy
5. **Content always wins** — if image + text don't fit, remove words, never shrink fonts
6. **Light/dark alternation** — visual rhythm across swipes
7. **Typography is fixed** — same scale on every slide, no exceptions
8. **One JSON output** — all slides as HTML strings, ready for html2png.dev via n8n
>>>>>>> 58fdcef (skill)
