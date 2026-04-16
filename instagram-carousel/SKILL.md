---
name: instagram-carousel
description: >
  Creates high-quality Instagram carousels as base64-encoded PNG images
  (1080×1350px), one image per slide, ready for n8n integration.
  Handles the full workflow: brand setup, slide copy, visual design system
  (colors, fonts, components), HTML generation, and headless PNG rendering.
  Use this skill whenever the user asks to create, design, or generate an
  Instagram carousel, carrossel, slides para Instagram, or any Instagram
  multi-image post — even if they don't explicitly say "carousel" or "skill".
  Also trigger for requests to "create a post with multiple slides",
  "fazer carrossel", or "exportar slides para o Instagram".
---

# Instagram Carousel Generator

Generates fully self-contained HTML slides at exactly 1080×1350px,
renders each one headlessly via Playwright, and outputs a JSON file
containing one base64-encoded PNG string per slide — ready for n8n.

---

## Step 1: Collect Brand Details

<!-- Default brand values — always use, without asking the user -->

1. **Brand name** — rpa teste
2. **Instagram handle** — @rpatestesimp
3. **Primary brand color** — #221a4a, #51515a, #ff6e00, #00b1bc, 
4. **Logo** — https://raw.githubusercontent.com/bielhss/marketing-skills/refs/heads/main/instagram-carousel/assets/complete%20logo.png
5. **Font preference** — see typography table below, or specific Google Fonts
6. **Tone** — accessible consultative tone, demonstrate technical expertise, but like a close colleague who understands the challenges, not a distant specialist.
7. **Images** — screenshots, product images, etc.
  1. profile photo - https://raw.githubusercontent.com/bielhss/marketing-skills/refs/heads/main/instagram-carousel/assets/icon%20logo.png
8. **Idioma dos slides** — default: **Português (BR)** 
9. **Carousel format** — standard (7 slides) or alternate sequence (see sequences section)

If the user provides a website URL or brand assets, derive colors and style from those.

---

## Handling User-Provided Images

**This section applies from the very first HTML generation — not only during export.**

When the user provides an image file path (e.g., `/home/user/gestante.png`, `/mnt/user-data/uploads/foto.jpg`):

### ⚠️ Critical Rules

1. **NEVER use relative paths** (`gestante.png`) — they break in every browser context except the exact folder the HTML lives in.
2. **NEVER use `background: url(filepath)`** — leads to 1.5MB+ base64 inline strings that crash the browser parser.
3. **ALWAYS embed as base64 `data:` URI** — works in preview, export, and any environment.
4. **ALWAYS generate the HTML via Python** (`Path.write_text()`) — shell heredocs interpolate `$` and backticks, corrupting base64 strings.

### Step-by-step: embed an image

```bash
# 1. Check the actual file format (extension may lie)
file /path/to/image.png
```

```python
import base64
from pathlib import Path

# 2. Read and encode
img_path = Path("/path/to/image.png")
# Use "image/jpeg" if `file` command says JPEG, else "image/png"
mime = "image/jpeg"  # or "image/png"
b64 = base64.b64encode(img_path.read_bytes()).decode()
data_uri = f"data:{mime};base64,{b64}"

# 3. Inject into HTML template as a Python variable — never via shell
html = f"""
<div style="position:relative;width:100%;height:100%;">
  <img src="{data_uri}"
       style="position:absolute;inset:0;width:100%;height:100%;object-fit:cover;z-index:0;">
  <div style="position:absolute;inset:0;background:rgba(255,255,255,0.35);z-index:1;"></div>
  <!-- slide content goes here, z-index:2 -->
</div>
"""

Path("/home/claude/carousel.html").write_text(html, encoding="utf-8")
```

### Image as slide background (most common use)

```html
<!-- Inside the slide div, before any content -->
<img src="{data_uri}"
     style="position:absolute;inset:0;width:100%;height:100%;object-fit:cover;z-index:0;">
<!-- Semi-transparent overlay so text stays readable -->
<div style="position:absolute;inset:0;background:rgba(255,255,255,0.35);z-index:1;"></div>
<!-- All slide content must have z-index:2 or higher -->
```

For dark slides, use `rgba(0,0,0,0.45)` as the overlay instead.

### Common image mistakes to avoid

| Mistake | What goes wrong | Fix |
|---------|----------------|-----|
| `<img src="gestante.png">` | Broken image — relative path only works if HTML and image share the same folder | Always use base64 `data:` URI |
| `background: url('data:...')` inline with 1.5MB base64 | Browser parser crash, 1.3M token context | Use `<img>` tag with `object-fit:cover` |
| Generating HTML via shell `echo` or heredoc | `$` and backtick characters in base64 get interpolated and corrupt the string | Always use Python `Path.write_text()` |
| Assuming `.png` extension = PNG format | File may actually be JPEG; wrong MIME type breaks rendering | Run `file` command to detect actual format |

---

## Step 2: Derive the Full Color System

From the user's **single primary brand color**, generate the full 6-token palette:

```
BRAND_PRIMARY   = {user's color}                    // Main accent — progress bar, icons, tags
BRAND_LIGHT     = {primary lightened ~20%}           // Secondary accent — tags on dark, pills
BRAND_DARK      = {primary darkened ~30%}            // CTA text, gradient anchor
LIGHT_BG        = {warm or cool off-white}           // Light slide background (never pure #fff)
LIGHT_BORDER    = {slightly darker than LIGHT_BG}    // Dividers on light slides
DARK_BG         = {near-black with brand tint}       // Dark slide background
```

**Rules for deriving colors:**
- LIGHT_BG: tinted off-white complementing the primary (warm → warm cream, cool → cool gray-white)
- DARK_BG: near-black with subtle brand tint (warm → #1A1918, cool → #0F172A)
- LIGHT_BORDER: always ~1 shade darker than LIGHT_BG
- Brand gradient: `linear-gradient(165deg, BRAND_DARK 0%, BRAND_PRIMARY 50%, BRAND_LIGHT 100%)`

---

## Step 3: Set Up Typography

Based on the user's font preference, pick a **heading font** and **body font** from Google Fonts.

| Style | Heading Font | Body Font |
|-------|-------------|-----------|
| Editorial / premium | Playfair Display | DM Sans |
| Modern / clean | Plus Jakarta Sans (700) | Plus Jakarta Sans (400) |
| Warm / approachable | Lora | Nunito Sans |
| Technical / sharp | Space Grotesk | Space Grotesk |
| Bold / expressive | Fraunces | Outfit |
| Classic / trustworthy | Libre Baskerville | Work Sans |
| Rounded / friendly | Bricolage Grotesque | Bricolage Grotesque |

**Font size scale (fixed across all brands):**
- Headings: 80–120px, weight 700, line-height 1.08–1.12
- Body: 32–44px, weight 400–500, line-height 1.35–1.45
- Tags/labels: 18–22px, weight 700, letter-spacing 2–3px
- Step numbers: heading font, 26px, weight 300
- Small text: 22–26px

Apply via CSS classes `.serif` (heading font) and `.sans` (body font) throughout all slides.

---

## Slide 1 — Hook Rules

The first slide must stop the scroll in under 1 second. Prioritize these formats:

| Hook format | Example |
|---|---|
| Afirmação polêmica | "Você está usando IA errado" |
| Número + benefício | "7 ferramentas que substituem seu designer" |
| Pergunta que dói | "Por que seus carrosséis têm 0 salvamentos?" |
| Resultado concreto | "Esse post gerou 4.200 seguidores em 3 dias" |
| Inversão de expectativa | "Mais esforço no design = menos alcance" |

**Rules:**
- Never start with the brand name as headline
- Visual proof on Slide 1 whenever possible (screenshot, result, real number)
- Hook must promise value that the following slides deliver

---

## Typography Scaling Rule (CRITICAL)

This system is NOT web-based. All typography must be optimized for Instagram viewing on mobile devices.

- Text must be readable at arm's length on a phone
- Prioritize large, bold headlines
- Reduce text density instead of shrinking font size
- Prefer fewer words over smaller typography

If content does not fit:
→ REMOVE content
→ NEVER reduce font size below defined scale
---

## Content Density Rules

- Maximum 3 bullet points per slide
- Maximum 2 lines per bullet point
- Maximum 6–8 total lines of text per slide

If content exceeds limits:
→ Split into more slides
→ OR simplify text
→ NEVER compress layout

## Slide Sequences

### Standard (7 slides — default)

| # | Type | Background | Purpose |
|---|------|------------|---------|
| 1 | Hero | LIGHT_BG | Hook — bold statement, logo lockup, optional watermark |
| 2 | Problem | DARK_BG | Pain point — what's broken, frustrating, or outdated |
| 3 | Solution | Brand gradient | The answer — what solves it, optional quote/prompt box |
| 4 | Features | LIGHT_BG | What you get — feature list with icons |
| 5 | Details | DARK_BG | Depth — customization, specs, differentiators |
| 6 | How-to | LIGHT_BG | Steps — numbered workflow or process |
| 7 | CTA | Brand gradient | Call to action — logo, tagline, CTA button. **No arrow. Full progress bar.** |

### Listicle (5–10 slides)

| # | Type | Background |
|---|------|------------|
| 1 | Hero | LIGHT_BG |
| 2–N | Item N | Alternating LIGHT/DARK |
| Last | CTA | Brand gradient |

Use for: "X ferramentas", "X erros", "X dicas"

### Tutorial (7 slides)

| # | Type | Background |
|---|------|------------|
| 1 | Hero | LIGHT_BG |
| 2 | Contexto / Por quê | DARK_BG |
| 3–5 | Passo 1, 2, 3 | Alternating |
| 6 | Resultado esperado | DARK_BG |
| 7 | CTA | Brand gradient |

### Comparação (5 slides)

| # | Type | Background |
|---|------|------------|
| 1 | Hero (o que será comparado) | LIGHT_BG |
| 2 | Opção A | LIGHT_BG |
| 3 | Opção B | DARK_BG |
| 4 | Veredicto | Brand gradient |
| 5 | CTA | DARK_BG |

**General rules for all sequences:**
- Start with a hook — first slide must stop the scroll
- End CTA on brand gradient — no swipe arrow, progress bar at 100%
- Alternate light and dark backgrounds for visual rhythm
- Adapt sequence to topic — not every carousel needs all slides

---

## Slide Architecture

### Format
- Dimensions: **1080×1350px** (Instagram carousel standard, 4:5)
- Each slide is rendered to a **base64 PNG string** via Playwright
- All slides use `width:1080px; height:1350px` fixed in the `<body>` and root container
- Alternate LIGHT_BG and DARK_BG backgrounds for visual rhythm

### Required Elements on Every Slide

#### 1. Progress Bar (bottom of every slide)

Shows position in the carousel. Fixed fill per slide.

- Position: absolute bottom, full width, 60px horizontal padding, 40px bottom padding
- Track: 6px height, rounded corners
- Fill width: `((slideIndex + 1) / totalSlides) * 100%`
- Light slides: `rgba(0,0,0,0.08)` track, `BRAND_PRIMARY` fill, `rgba(0,0,0,0.3)` counter
- Dark slides: `rgba(255,255,255,0.12)` track, `#fff` fill, `rgba(255,255,255,0.4)` counter
- Counter label beside the bar: "1/7" format, 22px, weight 500

```javascript
function progressBar(index, total, isLightSlide) {
  const pct = ((index + 1) / total) * 100;
  const trackColor = isLightSlide ? 'rgba(0,0,0,0.08)' : 'rgba(255,255,255,0.12)';
  const fillColor = isLightSlide ? BRAND_PRIMARY : '#fff'; // use actual BRAND_PRIMARY value
  const labelColor = isLightSlide ? 'rgba(0,0,0,0.3)' : 'rgba(255,255,255,0.4)';
  return `<div style="position:absolute;bottom:0;left:0;right:0;padding:32px 60px 40px;z-index:10;display:flex;align-items:center;gap:20px;">
    <div style="flex:1;height:6px;background:${trackColor};border-radius:4px;overflow:hidden;">
      <div style="height:100%;width:${pct}%;background:${fillColor};border-radius:4px;"></div>
    </div>
    <span style="font-size:22px;color:${labelColor};font-weight:500;">${index + 1}/${total}</span>
  </div>`;
}
```

⚠️ **Important:** Always replace `BRAND_PRIMARY` with the actual hex value before rendering. Never leave it as a variable name in the HTML output.

#### 2. Swipe Arrow (right edge — every slide EXCEPT the last)

Subtle chevron guiding the user to keep swiping. Removed on the last slide.

- Position: absolute right, full height, 100px wide
- Background: gradient fade transparent → subtle tint
- Chevron: 48×48 SVG, rounded strokes
- Light slides: `rgba(0,0,0,0.06)` bg, `rgba(0,0,0,0.25)` stroke
- Dark slides: `rgba(255,255,255,0.08)` bg, `rgba(255,255,255,0.35)` stroke

```javascript
function swipeArrow(isLightSlide) {
  const bg = isLightSlide ? 'rgba(0,0,0,0.06)' : 'rgba(255,255,255,0.08)';
  const stroke = isLightSlide ? 'rgba(0,0,0,0.25)' : 'rgba(255,255,255,0.35)';
  return `<div style="position:absolute;right:0;top:0;bottom:0;width:100px;z-index:9;display:flex;align-items:center;justify-content:center;background:linear-gradient(to right,transparent,${bg});">
    <svg width="48" height="48" viewBox="0 0 24 24" fill="none">
      <path d="M9 6l6 6-6 6" stroke="${stroke}" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round"/>
    </svg>
  </div>`;
}
```

---

## Reusable Components

### Strikethrough pills
```html
<span style="font-size:11px;padding:5px 12px;border:1px solid rgba(255,255,255,0.1);border-radius:20px;color:#6B6560;text-decoration:line-through;">{Old tool}</span>
```

### Tag pills
```html
<span style="font-size:11px;padding:5px 12px;background:rgba(255,255,255,0.06);border-radius:20px;color:{BRAND_LIGHT};">{Label}</span>
```

### Prompt / quote box
```html
<div style="padding:16px;background:rgba(0,0,0,0.15);border-radius:12px;border:1px solid rgba(255,255,255,0.08);">
  <p class="sans" style="font-size:13px;color:rgba(255,255,255,0.5);margin-bottom:6px;">{Label}</p>
  <p class="serif" style="font-size:15px;color:#fff;font-style:italic;line-height:1.4;">"{Quote text}"</p>
</div>
```

### Feature list
```html
<div style="display:flex;align-items:flex-start;gap:14px;padding:10px 0;border-bottom:1px solid {LIGHT_BORDER};">
  <span style="color:{BRAND_PRIMARY};font-size:15px;width:18px;text-align:center;">{icon}</span>
  <div>
    <span class="sans" style="font-size:14px;font-weight:600;color:{DARK_BG};">{Label}</span>
    <span class="sans" style="font-size:12px;color:#8A8580;">{Description}</span>
  </div>
</div>
```

### Numbered steps
```html
<div style="display:flex;align-items:flex-start;gap:16px;padding:14px 0;border-bottom:1px solid {LIGHT_BORDER};">
  <span class="serif" style="font-size:26px;font-weight:300;color:{BRAND_PRIMARY};min-width:34px;line-height:1;">01</span>
  <div>
    <span class="sans" style="font-size:14px;font-weight:600;color:{DARK_BG};">{Step title}</span>
    <span class="sans" style="font-size:12px;color:#8A8580;">{Step description}</span>
  </div>
</div>
```

### Color swatches
```html
<div style="width:32px;height:32px;border-radius:8px;background:{color};border:1px solid rgba(255,255,255,0.08);"></div>
```

### CTA button (final slide only)
```html
<div style="display:inline-flex;align-items:center;gap:8px;padding:12px 28px;background:{LIGHT_BG};color:{BRAND_DARK};font-family:'{BODY_FONT}',sans-serif;font-weight:600;font-size:14px;border-radius:28px;">
  {CTA text}
</div>
```

### Tag / Category Label
```html
<span class="sans" style="display:inline-block;font-size:10px;font-weight:600;letter-spacing:2px;color:{color};margin-bottom:16px;">{TAG TEXT}</span>
```
- Light slides: `BRAND_PRIMARY`
- Dark slides: `BRAND_LIGHT`
- Brand gradient slides: `rgba(255,255,255,0.6)`

### Logo Lockup (first and last slides)
- If logo icon: 40px circle (BRAND_PRIMARY bg) + icon centered + brand name beside
- If initials: 40px circle with first letter in white
- Brand name: 13px, weight 600, letter-spacing 0.5px

---

## Layout Rules

- Content padding: `0 72px` standard (scales with 1080px width)
- Bottom-aligned slides with progress bar: `0 72px 100px` to clear the bar
- **Hero/CTA slides:** `justify-content: center`
- **Content-heavy slides:** `justify-content: flex-end`
- **Content must never overlap the progress bar** — use `padding-bottom: 100px`

---

## Layout Rules Update

- Default vertical alignment: center
- Use flex-end ONLY for dense content slides
- Prefer visual balance over strict structure

---

## Line Break Optimization

Headlines should be manually broken into 2–3 lines for better visual rhythm.

Avoid long single-line headings.

Example:
❌ "Por que unir IA e RPA?"
✅ "Por que unir\nIA e RPA?"

## HTML Slide Structure (per slide)

Each slide is a fully self-contained HTML string — no JavaScript, no interactivity.
It is written to a temp file and rendered to PNG by Playwright.

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <link rel="preconnect" href="https://fonts.googleapis.com">
  <link href="https://fonts.googleapis.com/css2?family={HEADING_FONT}:wght@300;600;700&family={BODY_FONT}:wght@400;600&display=swap" rel="stylesheet">
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
      justify-content: flex-end; /* or center for hero/CTA */
      padding: 120px 90px 140px;
      overflow: hidden;
    }
    .serif { font-family: '{HEADING_FONT}', serif; }
    .sans  { font-family: '{BODY_FONT}', sans-serif; }
  </style>
</head>
<body>
  <div class="slide">
    <!-- slide content here -->
    <!-- progress bar -->
    <!-- swipe arrow (all slides except last) -->
  </div>
</body>
</html>
```

---

## Generation Flow

### Step 1 — Install Playwright (once per session)

```bash
pip install playwright --break-system-packages
python -m playwright install chromium
```

### Step 2 — Write all slide HTML files to disk

Use Python `Path.write_text()` — never shell heredocs (breaks base64 strings).

```python
from pathlib import Path

slides_html = [slide_1_html, slide_2_html, ...]  # list of HTML strings

for i, html in enumerate(slides_html, start=1):
    Path(f"/home/claude/slide_{i}.html").write_text(html, encoding="utf-8")
```

### Step 3 — Render each slide to base64 PNG via Playwright

```python
import base64
import json
from pathlib import Path
from playwright.sync_api import sync_playwright

results = []

with sync_playwright() as p:
    browser = p.chromium.launch()
    for i in range(1, len(slides_html) + 1):
        page = browser.new_page(viewport={"width": 1080, "height": 1350})
        html_path = Path(f"/home/claude/slide_{i}.html").resolve()
        page.goto(f"file://{html_path}")
        # Wait for Google Fonts to load
        page.wait_for_timeout(2000)
        png_bytes = page.screenshot(full_page=False)
        b64 = base64.b64encode(png_bytes).decode("utf-8")
        results.append({
            "slide": i,
            "base64": b64,
            "media_type": "image/png"
        })
    browser.close()

# Save JSON output for n8n
output = {"slides": results}
Path("/home/claude/carousel_output.json").write_text(
    json.dumps(output), encoding="utf-8"
)
print(f"Generated {len(results)} slides.")
```

### Step 4 — Copy output to user-accessible directory

```bash
cp /home/claude/carousel_output.json /mnt/user-data/outputs/carousel_output.json
```

### Step 5 — Present the output file to the user

After copying, call `present_files` with `/mnt/user-data/outputs/carousel_output.json`.

---

## Output Format

The final output is a single JSON file: `carousel_output.json`

```json
{
  "slides": [
    {
      "slide": 1,
      "base64": "iVBORw0KGgoAAAANSUhEUgAABD...",
      "media_type": "image/png"
    },
    {
      "slide": 2,
      "base64": "iVBORw0KGgoAAAANSUhEUgAABD...",
      "media_type": "image/png"
    }
  ]
}
```

### Using in n8n

In n8n, after receiving this JSON:
- Use a **Code node** or **Set node** to iterate `slides`
- Each `base64` value can be decoded directly as a binary PNG
- Pass to HTTP Request, Google Drive, Instagram API, or any image-consuming node

---

## Design Principles

1. **Every slide is export-ready** — arrow and progress bar are part of the slide image
2. **Light/dark alternation** — creates visual rhythm across swipes
3. **Heading + body font pairing** — display font for impact, body for readability
4. **Brand-derived palette** — all colors stem from one primary, keeping everything cohesive
5. **Progressive disclosure** — progress bar fills and arrow guides forward
6. **Last slide is special** — no arrow, full progress bar, clear CTA
7. **Consistent components** — same tag style, list style, spacing across all slides
8. **Content padding clears UI** — body text never overlaps progress bar or arrow
9. **Hook-first copy** — Slide 1 exists to stop the scroll, not to introduce the brand
10. **One JSON output** — all slides as base64 PNGs in a single file, ready for n8n