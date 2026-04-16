---
name: image-search
description: >
  Universal skill for searching and retrieving free, open-source images from
  multiple providers (Unsplash, Pexels). Returns a standardized { url, source,
  alt_text, orientation } object ready for use in any other skill (e.g.
  instagram-carousel). Activate this skill ANY TIME another skill or the user
  needs an image URL — before calling any image provider tool directly.
  Handles provider routing, query strategy, fallback logic, and result
  normalization. Does NOT generate or edit images.
compatibility: >
  Requires at least one of: Unsplash HTTP tool, Pexels HTTP tool (configured
  in n8n as HTTP Request nodes exposed as AI Agent tools).
metadata:
  version: "1.0"
---

# Image Search Skill

Universal routing skill for fetching free, license-safe images from open
sources. Returns one standardized image object per request — ready to drop
into any HTML, slide, or component.

---

## Scope

**Use this skill when:**
- Another skill (e.g. instagram-carousel) needs an image URL for a slide
- The user asks to find, fetch, or search for a photo or illustration
- You need to resolve a visual concept into a concrete, working URL

**Do NOT use this skill when:**
- Generating new images with AI (use an image generation skill instead)
- Editing or transforming existing images
- The caller already has a valid image URL

---

## Providers

| Provider | License | API | Tool name |
|----------|---------|-----|-----------|
| **Unsplash** | Unsplash License (free, commercial OK) | `https://api.unsplash.com/search/photos` | `Search Unsplash` |
| **Pexels** | Pexels License (free, commercial OK) | `https://api.pexels.com/v1/search` | `Search Pexels` |

Both licenses allow free use in any project — including commercial — without
attribution required (though attribution is appreciated).

---

## Routing Logic

Run this decision tree before every image fetch:

```
1. Is `Search Unsplash` tool available?
   └── YES → try Unsplash first (Step A)
         └── no results → try Pexels (Step B)
   └── NO  → try Pexels directly (Step B)
              └── no results → return null (Step C)
```

**Never skip the fallback.** If the primary provider returns no results or
errors, always attempt the next provider before returning null.

---

## Step A — Search Unsplash

### Tool call

```
Tool: Search Unsplash
Parameters:
  query       → string  (see Query Strategy below)
  orientation → "portrait" | "landscape" | "squarish"
  per_page    → 5
  order_by    → "relevant"
```

### Extract result

```javascript
const url = response.results[0]?.urls?.regular ?? null;
const alt = response.results[0]?.alt_description ?? "";
```

If `url` is null or `results` is empty → proceed to Step B.

---

## Step B — Search Pexels

### Tool call

```
Tool: Search Pexels
Parameters:
  query       → string  (same query used for Unsplash)
  orientation → "portrait" | "landscape" | "square"
  per_page    → 5
```

### Extract result

```javascript
// Pexels returns photos[N].src.large2x for high-res (2x) or src.large
const photo = response.photos[0];
const url   = photo?.src?.large2x ?? photo?.src?.large ?? null;
const alt   = photo?.alt ?? "";
```

If `url` is null or `photos` is empty → proceed to Step C.

---

## Step C — No result

Return a null image object:

```json
{ "url": null, "source": null, "alt_text": "", "orientation": "<requested>" }
```

The calling skill MUST handle null gracefully — render the slide or component
without an image. Never halt generation because an image was not found.

---

## Standardized Output Object

Every successful fetch returns this exact structure:

```json
{
  "url":         "https://images.unsplash.com/photo-xxxx?w=1080",
  "source":      "unsplash",
  "alt_text":    "A person working at a laptop in a modern office",
  "orientation": "landscape"
}
```

| Field | Type | Description |
|-------|------|-------------|
| `url` | string \| null | Direct image URL, ready to use in `<img src="">` |
| `source` | `"unsplash"` \| `"pexels"` \| null | Provider that returned the result |
| `alt_text` | string | Description for HTML `alt` attribute |
| `orientation` | `"portrait"` \| `"landscape"` \| `"squarish"` | As requested |

---

## Query Strategy

Bad queries are the #1 reason images feel irrelevant.
Follow every rule below without exception.

---

### Rule 0 — Classify first: abstract or physical?

Before writing any query, classify the image topic:

| Type | Definition | Query style |
|------|-----------|------------|
| **Abstract** | Concept with no single photographable form: AI, integration, workflow logic, decision-making, automation intelligence | `"{concrete_metaphor} flat illustration"` |
| **Physical** | Something a photographer could capture: a desk, a server, a document, a headset | `"{physical_object} {context}"` |

**Quick test:** "Could a photographer walk into a room and take a picture of
this concept?"
- No → Abstract → use illustration query
- Yes → Physical → use photo query

---

### Rule 1 — Use the pre-validated query library (MANDATORY FIRST STEP)

Check this table before writing any query.
If the topic matches → **copy the exact string verbatim. Do NOT rephrase.**

#### Abstract topics → Illustration queries

| Topic | Query |
|-------|-------|
| Intelligent automation / RPA + AI | `"automation workflow flat illustration"` |
| Artificial intelligence / ML | `"artificial intelligence flat illustration"` |
| Automatic decision-making | `"decision tree flat illustration"` |
| System integration / API | `"system integration flat illustration"` |
| Workflow / process | `"workflow process flat illustration"` |
| Data / analytics | `"data analytics flat illustration"` |
| Connectivity / networks | `"network connection flat illustration"` |
| Digital transformation | `"digital transformation flat illustration"` |
| Scalability / cloud | `"cloud computing flat illustration"` |
| Monitoring / alerts | `"monitoring dashboard flat illustration"` |
| Efficiency / productivity | `"productivity efficiency flat illustration"` |
| Security / compliance | `"cybersecurity flat illustration"` |

#### Physical topics → Photo queries

| Topic | Query |
|-------|-------|
| Industrial automation / robots | `"robot arm factory conveyor"` |
| Servers / infrastructure | `"server rack data center"` |
| Documents / OCR / paper | `"document scanner paper desk"` |
| Email / inbox | `"laptop email inbox screen"` |
| Customer service | `"headset desk computer support"` |
| Financial processes | `"spreadsheet calculator accounting desk"` |
| Code / development | `"code editor screen dark terminal"` |
| Office / work | `"office desk laptop working"` |
| Meeting / team | `"meeting room whiteboard team"` |
| Reports | `"report paper printed desk"` |
| Physical monitoring | `"monitor screen graphs dark room"` |

If the topic has NO match in either table → apply Rule 2.

---

### Rule 2 — Derive a query when the topic is not in the library

Follow the same pattern as the library:
- Abstract → `"{concrete_concept} flat illustration"`
- Physical → `"{physical_object} {context}"`

Examples:

| Slide headline | Classification | Correct query |
|----------------|---------------|---------------|
| "Reduce process errors" | Physical | `"quality checklist clipboard desk"` |
| "Automatic reports" | Physical | `"report paper printed desk"` |
| "Decision logic" | Abstract | `"decision logic flat illustration"` |
| "Robot orchestration" | Abstract | `"workflow orchestration flat illustration"` |

---

### Rule 3 — Self-check before every tool call

Ask internally:
> "If I search this exact phrase on Unsplash or Pexels, will the top results
> clearly illustrate what THIS specific slide is about — and match the
> abstract/physical classification I chose?"

- "Yes, clearly" → proceed
- "Maybe / probably not" → rewrite the query or skip image for this slide

---

### Rule 4 — Banned terms (never use as primary search term)

`RPA`, `IA`, `API`, `ERP`, `technology`, `business`, `nature`, `landscape`,
`mountain`, `forest`, `river`, `sky`, `abstract`, `people`, `team`,
`success`, `growth`, `digital`, `future`, `innovation`, `background`,
`wallpaper`

These either return unrelated results (acronyms unknown to providers) or
generic scenic photos with zero relevance to professional topics.

---

### Rule 5 — Always English, 3–5 concrete words

- ✅ `"automation workflow flat illustration"`
- ✅ `"server rack data center"`
- ✅ `"decision tree flat illustration"`
- ❌ `"RPA automation"` → acronym, banned
- ❌ `"artificial intelligence business"` → too generic + banned term
- ❌ `"digital transformation technology"` → two banned terms

---

## Orientation Reference

| Use case | Orientation |
|----------|------------|
| Hero / full-background slide | `portrait` |
| Decorative / side-panel image | `landscape` |
| Square thumbnail / icon | `squarish` |

---

## n8n Tool Configuration

### Unsplash HTTP Request node

```
Method:  GET
URL:     https://api.unsplash.com/search/photos
Params:
  query       → {{ $json.query }}
  orientation → {{ $json.orientation }}
  per_page    → 5
  order_by    → relevant
Headers:
  Authorization  → Client-ID {{ $credentials["Unsplash Access Key"].value }}
  Accept-Version → v1
```

### Pexels HTTP Request node

```
Method:  GET
URL:     https://api.pexels.com/v1/search
Params:
  query       → {{ $json.query }}
  orientation → {{ $json.orientation }}
  per_page    → 5
Headers:
  Authorization → {{ $credentials["Pexels API Key"].value }}
```

### Credentials setup

**Unsplash:**
1. Create account at unsplash.com/developers
2. Create application → copy Access Key
3. n8n: Settings → Credentials → New → Header Auth
   - Name: `Unsplash Access Key`
   - Value: paste Access Key

**Pexels:**
1. Create account at pexels.com/api
2. Your Email → Images & Video API → copy API Key
3. n8n: Settings → Credentials → New → Header Auth
   - Name: `Pexels API Key`
   - Value: paste API Key

---

## Integration with instagram-carousel

When the instagram-carousel skill needs an image for a slide:

1. **instagram-carousel** calls this skill with:
   ```json
   {
     "topic": "slide headline or topic description",
     "orientation": "portrait" | "landscape"
   }
   ```

2. **image-search** runs the full routing logic and returns:
   ```json
   {
     "url": "https://...",
     "source": "unsplash",
     "alt_text": "...",
     "orientation": "landscape"
   }
   ```

3. **instagram-carousel** uses `url` directly in the `<img src="">` tag of the
   appropriate layout (A / C1 / C2 / C3). If `url` is null, renders the slide
   without an image using the text-only fallback.

**The instagram-carousel skill should NEVER call Search Unsplash or Search
Pexels directly.** All image fetching goes through this skill.

---

## Error Handling

| Situation | Action |
|-----------|--------|
| Tool unavailable | Skip provider, try next in routing order |
| Tool returns HTTP error | Log error, try next provider |
| Empty results | Try next provider |
| All providers exhausted | Return null object |
| Invalid query | Apply Rule 3 self-check, rewrite query, retry once |

**Never halt generation due to a missing image.** Always return either a valid
URL or a null object, and let the calling skill decide how to handle it.

---

## Key Principles

1. **Caller never touches providers directly** — all routing happens here
2. **Standardized output** — callers always receive the same object shape
3. **Graceful null** — missing images never break the calling skill
4. **Query quality first** — a great query beats a fast query
5. **Fallback always runs** — Unsplash → Pexels → null, in order
6. **License safety** — both providers are free for commercial use
