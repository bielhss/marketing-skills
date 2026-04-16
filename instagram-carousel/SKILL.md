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

The Unsplash API key is injected by n8n into the system prompt as a plain text line in this format:
```
Unsplash API Key: 926582kIHOqYSR-Dl43mr8JxrPouwnfabkJXjibDJB41mraXQ
```

**⚠️ When writing the Python code, you MUST read the actual key value from the system prompt context and hardcode it into the `UNSPLASH_KEY` variable.** Do not copy the placeholder — copy the real key string that is visible in your context window.

If the system prompt contains `Unsplash API Key: abc123xyz`, then your Python must have:
```python
UNSPLASH_KEY = "abc123xyz"
```

Never leave `UNSPLASH_KEY` as a placeholder string. If you cannot find the key in the context, set `UNSPLASH_KEY = ""` and skip image fetching entirely.

```python
import urllib.request
import urllib.parse
import json

# The key is injected by n8n into the system prompt as {{ $credentials.unsplash_key }}
# When writing the Python code, replace the placeholder below with the actual value
# that was injected — it will be visible in the conversation context as a string.
# ⚠️ CRITICAL: Read the Unsplash API Key from the system prompt context.
# The key was injected as: "Unsplash API Key: <value>"
# Extract the actual key value from that line and assign it here — never leave it as a placeholder.
# Example: if system prompt contains "Unsplash API Key: 926582kIH..." then:
UNSPLASH_KEY = "<extract_the_actual_key_value_from_system_prompt_context>"
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

### ⚠️ Image Type Decision — Run This First

Before fetching any image, classify the slide topic as **abstract** or **physical**:

| Type | Definition | Image source |
|---|---|---|
| **Abstract** | Concept with no single physical representation: decision-making, AI, integration, workflow logic, automation intelligence | → **Illustration** (Unsplash illustration query) |
| **Physical** | Something you can photograph: a desk, a server, a document, a screen, a headset | → **Photo** (Unsplash photo query) |

**Quick test:** "Could a photographer walk into a room and take a picture of this concept?"
- No → Abstract → Illustration
- Yes → Physical → Photo

---

### ⚠️ Query Strategy — CRITICAL

Bad queries are the #1 reason images feel irrelevant. Follow every rule below without exception.

---

#### Rule 1 — Use the pre-validated query library — MANDATORY FIRST STEP

**Before writing ANY query, you MUST check the table below.**
- If the topic matches an entry → **copy that exact string verbatim. Do NOT rephrase, simplify, or translate it.**
- Only proceed to Rule 2 if the topic genuinely has NO match in either table.
- Using a self-invented query when a library match exists is a violation of this rule.

**Abstract topics → Illustration queries** (append `"flat illustration"` or `"minimal illustration"` to signal style)

| Topic | Illustration query |
|---|---|
| Automação inteligente / RPA + IA | `"automation workflow flat illustration"` |
| Inteligência Artificial / Machine Learning | `"artificial intelligence flat illustration"` |
| Decisão automática | `"decision tree flat illustration"` |
| Integração de sistemas / API | `"system integration flat illustration"` |
| Fluxo de trabalho / Workflow | `"workflow process flat illustration"` |
| Dados / Analytics | `"data analytics flat illustration"` |
| Conectividade / Redes | `"network connection flat illustration"` |
| Transformação digital | `"digital transformation flat illustration"` |
| Escalabilidade | `"cloud computing flat illustration"` |
| Monitoramento / Alertas | `"monitoring dashboard flat illustration"` |
| Eficiência / Produtividade | `"productivity efficiency flat illustration"` |
| Segurança / Compliance | `"cybersecurity flat illustration"` |

**Physical topics → Photo queries** (concrete objects, no acronyms like RPA/IA/API)

| Topic | Photo query |
|---|---|
| Automação industrial / Robôs físicos | `"robot arm factory conveyor"` |
| Servidores / Infraestrutura | `"server rack data center"` |
| Documentos / OCR / Papel | `"document scanner paper desk"` |
| E-mail / Inbox | `"laptop email inbox screen"` |
| Atendimento ao cliente | `"headset desk computer support"` |
| Processos financeiros | `"spreadsheet calculator accounting desk"` |
| Código / Desenvolvimento | `"code editor screen dark terminal"` |
| Escritório / Trabalho | `"office desk laptop working"` |
| Reunião / Equipe | `"meeting room whiteboard team"` |
| Relatórios | `"report paper printed desk"` |
| Monitoramento físico | `"monitor screen graphs dark room"` |

If the topic is NOT in either table → apply Rule 2.

---

#### Rule 2 — Derive a query when topic is not in the library

First classify (abstract or physical), then write a query following the same pattern:
- Abstract → `"{concrete_concept} flat illustration"`
- Physical → `"{physical_object} {context}"`

| Slide headline | Classification | ✅ Correct query |
|---|---|---|
| "Reduza erros no processo" | Physical | `"quality checklist clipboard desk"` |
| "Relatórios automáticos" | Physical | `"report paper printed desk"` |
| "Lógica de decisão" | Abstract | `"decision logic flat illustration"` |
| "Orquestração de robôs" | Abstract | `"workflow orchestration flat illustration"` |

---

#### Rule 3 — Self-check before every fetch_unsplash() call

Ask internally:
> "If I search this exact phrase on Unsplash, will the top results clearly illustrate what THIS SLIDE is specifically about — and match the abstract/physical classification I chose?"

- "Yes, clearly" → proceed
- "Maybe / probably not" → rewrite or return `None`

---

#### Rule 4 — Banned terms (never use as primary term)

`RPA`, `IA`, `API`, `ERP`, `technology`, `business`, `nature`, `landscape`, `mountain`, `forest`, `river`, `sky`, `abstract`, `people`, `team`, `success`, `growth`, `digital`, `future`, `innovation`, `background`, `wallpaper`

`RPA`, `IA`, `API` are acronyms unknown to Unsplash — they return random unrelated results.
`nature`, `mountain`, `forest`, `sky` return scenic photos with zero relevance to tech/business topics.

❌ Real violation examples seen in production:
- Slide about "automações inteligentes" → query `"sofa outdoors"` — WRONG (unrelated)
- Slide about "RPA + IA" → query `"mountain stars night"` — WRONG (banned nature term)
- Correct for both: check library first → `"automation workflow flat illustration"`

---

#### Rule 5 — Always English, 3–5 concrete words

- ✅ `"automation workflow flat illustration"`
- ✅ `"server rack data center"`
- ✅ `"decision tree flat illustration"`
- ❌ `"RPA automation"` (acronym)
- ❌ `"artificial intelligence business"` (too generic)
- ❌ `"digital transformation technology"` (banned terms)

### How to fetch an image

```python
def fetch_unsplash(query: str, orientation: str = "landscape") -> str | None:
    """
    Fetches an image URL from Unsplash.

    orientation:
      - 'portrait'   → Hero full-background slides only
      - 'landscape'  → C1 and C2 image-top layouts (default for all content slides)

    Always run the Image Type Decision and Query Strategy rules before calling.
    Illustration queries (ending in 'flat illustration') work with any orientation.
    Returns None on any failure — caller must handle gracefully.
    """
    if not UNSPLASH_KEY or UNSPLASH_KEY.startswith("<extract"):  # guard: key was not properly set
        return None
    try:
        params = urllib.parse.urlencode({
            "query": query,
            "orientation": orientation,
            "per_page": 5,       # fetch 5, pick the most relevant-looking result
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
                return results[0]["urls"]["regular"]
    except Exception:
        pass
    return None


# ── Image fetch plan — run BEFORE building any HTML ──────────────────────
# For each slide that gets an image:
#   1. Classify topic: abstract → illustration query / physical → photo query
#   2. Pick query from library or derive following the rules
#   3. Call fetch_unsplash() with appropriate orientation
#   4. Store result; None = render that slide without image

# Example for a RPA+IA carousel:
images = {
    # Hero (slide 1) — physical: robot/automation hardware, portrait
    "hero":     fetch_unsplash("robot arm factory conveyor", orientation="portrait"),

    # Solution (slide 3) — abstract: concept, landscape for C1 top-banner
    "solution": fetch_unsplash("automation workflow flat illustration", orientation="landscape"),

    # Features (slide 4) — depends on specific features listed:
    #   if features are abstract concepts → illustration
    #   if features involve physical things (docs, screens) → photo
    "features": fetch_unsplash("productivity efficiency flat illustration", orientation="landscape"),

    # How-to (slide 6) — landscape for C1 top-banner
    "howto":    fetch_unsplash("workflow process flat illustration", orientation="landscape"),
}
# C1 and C2 layouts always use landscape orientation — wide images fill the top banner better
# Adjust queries per carousel topic — these are examples for RPA+IA theme
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

### B — Image-top layouts (Solution + Content slides)

These slides place the image in the **top portion** and the text in the **bottom portion** — a clean, native-Instagram feel. Two layout variants alternate for visual rhythm.

#### C1 — Full-width image top, text bottom (odd-indexed content slides: 3, 6 ...)

The slide is divided vertically:
- **Top zone** (0–620px): full-width image with rounded bottom corners
- **Text zone** (640px–1210px): exactly 570px tall, `overflow:hidden` enforced
- **Progress bar zone** (1210px–1350px): 140px reserved — text zone NEVER enters this area

```html
<div class="slide" style="justify-content:flex-start;padding:0;">

  <!-- Full-width image — TOP ZONE -->
  <img src="{IMAGE_URL}"
       style="position:absolute;top:0;left:0;right:0;
              width:100%;height:620px;object-fit:cover;
              border-radius:0 0 32px 32px;
              z-index:1;opacity:0.97;">

  <!-- Text content — BOTTOM ZONE, fixed height, overflow hidden -->
  <!-- Available: 1350 - 640 (top) - 140 (progress bar) = 570px -->
  <div style="position:absolute;left:80px;top:640px;right:80px;
              height:570px;overflow:hidden;
              display:flex;flex-direction:column;
              justify-content:flex-start;z-index:3;">
    <!-- tag label (22px + margin ~40px) -->
    <!-- headline: MAX 2 lines at 88px (~200px) -->
    <!-- body: MAX 2 lines at 48px (~130px) -->
  </div>

  <!-- Progress bar — z-index:10, always on top -->
  <!-- Swipe arrow — z-index:9 -->
</div>
```

⚠️ **C1 copy limits** — `overflow:hidden` clips silently, so stay within bounds:
- Headline: **max 2 lines** at 88px
- Body: **max 2 lines** at 48px — cut words before exceeding

#### C2 — Image top with floating card overlay (even-indexed content slides: 4 ...)

The image fills the top ~55% and a card floats at the bottom, overlapping the image edge slightly.

- **Image zone** (0–700px): bleeds full width, no border-radius
- **Card zone** (620px–1250px): fixed height 630px, `overflow:hidden` enforced
- **Progress bar zone** (1250px–1350px): 100px reserved — card `bottom` must respect this

```html
<div class="slide" style="justify-content:flex-start;padding:0;background:{SLIDE_BG};">

  <!-- Full-width image — TOP ZONE -->
  <img src="{IMAGE_URL}"
       style="position:absolute;top:0;left:0;right:0;
              width:100%;height:700px;object-fit:cover;
              z-index:1;">

  <!-- Floating card — overlaps image bottom edge by ~80px -->
  <!-- bottom:100px ensures progress bar always visible below card -->
  <div style="position:absolute;left:60px;right:60px;
              top:620px;bottom:100px;
              background:{CARD_BG};border-radius:28px;
              box-shadow:0 -8px 40px rgba(0,0,0,0.18);
              padding:44px 56px 44px;
              overflow:hidden;
              display:flex;flex-direction:column;justify-content:flex-start;
              z-index:4;">
    <!-- tag label (22px + margin ~40px) -->
    <!-- headline: MAX 2 lines at 84px (~190px) -->
    <!-- body: MAX 2 lines at 44px (~120px) -->
  </div>

  <!-- Progress bar — position:absolute;bottom:0;z-index:10 — ALWAYS outside the card -->
  <!-- Swipe arrow — z-index:9 -->
</div>
```

**CARD_BG values:**
- Light slides: `#FFFFFF` — text `#1a1a1a`
- Dark slides: `{DARK_BG}` — text `#ffffff`

⚠️ **C2 copy limits** — card height is ~530px usable after padding:
- Headline: **max 2 lines** at 84px
- Body: **max 2 lines** at 44px — cut words before exceeding
- `overflow:hidden` on the card prevents content from leaking outside

#### ⚠️ Critical rules for ALL C layouts:

1. **Pixel zones must never overlap** — C1 text starts at 640px, C2 card starts at 620px; verify before writing HTML
2. **Always use `overflow:hidden`** on the text wrapper (C1) and the card (C2) — this is the hard stop against overflow
3. **Never reduce font sizes** — if content doesn't fit, remove words. Font sizes are fixed by the typography scale
4. **Progress bar is always `position:absolute;bottom:0;left:0;right:0;z-index:10`** — it lives on `.slide`, NOT inside any card or text wrapper
5. **C2 card must have `bottom:100px`** — this guarantees the progress bar zone is always visible below the card
6. **If `fetch_unsplash()` returns `None`** → remove the image tag entirely, set `.slide` to `justify-content:center; padding:100px 90px 140px`, render full-width text layout
7. **Image orientation**: always `orientation="landscape"` for C1 and C2 — never portrait or squarish

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
| 3 | Solution | LIGHT_BG | ✅ Image top | C1 — full-width image top, text bottom |
| 4 | Features | DARK_BG or LIGHT_BG | ✅ Image top + floating card | C2 — image top, floating card |
| 5 | Details | DARK_BG | ❌ | — |
| 6 | How-to | LIGHT_BG | ✅ Image top | C1 — full-width image top, text bottom |
| 7 | CTA | Brand gradient | ❌ | — |

### Listicle / Tutorial / Comparação

- Apply image to **content slides only** (light or dark background)
- Dark background slides may use C2 with a dark card overlay
- Alternate C1 and C2 layouts between consecutive image slides for visual rhythm
- Never apply the same layout variant to two consecutive image slides
- Always use `orientation="landscape"` when fetching images for C1 and C2 layouts

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
  <!-- ═══ NO IMAGE (Problem, CTA, Details) ═══ -->
  <div class="slide" style="justify-content:center;padding:100px 90px 140px;">
    <!-- tag, headline, body -->
    <!-- progress bar (absolute, z-index:10) -->
    <!-- swipe arrow (absolute, z-index:9) -->
  </div>

  <!-- ═══ HERO — layout A (full background) ═══ -->
  <div class="slide" style="justify-content:center;padding:100px 90px 140px;">
    <!-- bg image: position:absolute;inset:0;z-index:0 -->
    <!-- dark overlay: position:absolute;inset:0;background:rgba(0,0,0,0.55);z-index:1 -->
    <!-- content: position:relative;z-index:2 — tag, headline, subheadline -->
    <!-- progress bar (absolute, z-index:10) -->
    <!-- swipe arrow (absolute, z-index:9) -->
  </div>

  <!-- ═══ C1 — full-width image top, text bottom ═══ -->
  <div class="slide" style="justify-content:flex-start;padding:0;">
    <!-- image: position:absolute;top:0;left:0;right:0;width:100%;height:700px;object-fit:cover;border-radius:0 0 32px 32px;z-index:1 -->
    <!-- text: position:absolute;left:80px;top:740px;right:80px;bottom:140px;z-index:3 -->
    <!-- progress bar (absolute, z-index:10) -->
    <!-- swipe arrow (absolute, z-index:9) -->
  </div>

  <!-- ═══ C2 — image top, floating card overlay ═══ -->
  <div class="slide" style="justify-content:flex-start;padding:0;">
    <!-- image: position:absolute;top:0;left:0;right:0;width:100%;height:780px;object-fit:cover;z-index:1 -->
    <!-- card: position:absolute;left:60px;right:60px;top:680px;bottom:100px;border-radius:28px;box-shadow:0 -8px 40px rgba(0,0,0,0.18);padding:52px 60px;z-index:4 -->
    <!-- progress bar (absolute, z-index:10) -->
    <!-- swipe arrow (absolute, z-index:9) -->
  </div>
</body>
</html>
```

---

## Generation Flow

### Step 1 — Classify topics and fetch all Unsplash images upfront

Before writing any HTML, run the classification and fetch for every slide that needs an image.

```python
import json, urllib.request, urllib.parse
from pathlib import Path

# Key injected by n8n system prompt — replace placeholder with actual value
# ⚠️ CRITICAL: Read the Unsplash API Key from the system prompt context.
# The key was injected as: "Unsplash API Key: <value>"
# Extract the actual key value from that line and assign it here — never leave it as a placeholder.
# Example: if system prompt contains "Unsplash API Key: 926582kIH..." then:
UNSPLASH_KEY = "<extract_the_actual_key_value_from_system_prompt_context>"

def fetch_unsplash(query: str, orientation: str = "landscape") -> str | None:
    if not UNSPLASH_KEY or UNSPLASH_KEY.startswith("<extract"):  # guard: key was not properly set
        return None
    try:
        params = urllib.parse.urlencode({
            "query": query,
            "orientation": orientation,
            "per_page": 5,
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

# ── STEP A: For each image slide, classify topic then pick query ──────────
#
# Abstract concept (no physical photo possible) → "... flat illustration"
# Physical thing (can be photographed)          → concrete noun query
#
# Hero always uses orientation="portrait"
# C1 and C2 slides always use orientation="landscape" (full-width top banner)
#
# Replace the example queries below with ones matching the actual carousel topic,
# following the pre-validated library in the Query Strategy section above.

images = {
    "hero":     fetch_unsplash("robot arm factory conveyor",               orientation="portrait"),
    "solution": fetch_unsplash("automation workflow flat illustration",    orientation="landscape"),
    "features": fetch_unsplash("productivity efficiency flat illustration", orientation="landscape"),
    "howto":    fetch_unsplash("workflow process flat illustration",        orientation="landscape"),
}
# None = render that slide without image — never raise errors, never halt generation
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