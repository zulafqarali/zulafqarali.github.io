# CONTEXT.md — Zulafqar Ali Personal Site

Everything needed to understand, maintain and extend this repository.
Read this first before making changes.

---

## 1. What this is

A single-page personal portfolio/resume site for **Zulafqar Ali** (Principal Software Engineer), published at **https://zulafqarali.github.io** via GitHub Pages.

- **Architecture:** plain static HTML/CSS/JS. **No Jekyll, no build step, no frameworks, no dependencies.** The original `_config.yml` (Jekyll `minimal` theme) was deliberately deleted — do not re-add it unless you intentionally want Jekyll.
- **Design source:** modeled 1:1 on https://awaisashrafqa.github.io (structure, design system, interactions), with Zulafqar's content.
- **Content source of truth:** `~/Downloads/Zulafqar Ali - Resume (Updated).pdf` (extracted with `pypdf`). Resume source of truth for the PDF is now `resume.html` in this repo.

---

## 2. File inventory

| File | Purpose |
|---|---|
| `index.html` | The entire site: inline CSS (in `<style>`), markup, inline JS (one IIFE `<script>` at the end). ~700 lines. |
| `resume.html` | Printable resume source (A4, print CSS) — primary backend/PHP resume. Rendered to PDF — see §6. Also live at `/resume.html`. |
| `resume-python.html` | **Targeted resume** (Python · FastAPI · LLMs · GCP stack), same design. **Deliberately NOT linked from `index.html`** — it is public via the repo URL but not part of the site navigation/CTAs. |
| `Zulafqar_Ali_Resume.pdf` | Generated from `resume.html`. Linked from 2 places in `index.html` (hero CTA + contact CTA, each with `href` + `download`). |
| `Zulafqar_Ali_Resume_Python_FastAPI.pdf` | Generated from `resume-python.html`. Not linked from the site. |
| `learn/` | Interview study material (Markdown, not linked from the site): `README.md` (4-week roadmap), `python.md`, `fastapi.md` (deep Q&A), `stack.md` (GCP, LLM/LangChain, Vue, Kibana, Hadoop, Grafana). |
| `og-image.png` | 1200×630 social/share image, referenced by Open Graph + Twitter tags and JSON-LD. |
| `robots.txt` | Allows all crawlers, points to sitemap. |
| `sitemap.xml` | Single URL entry. **Update `<lastmod>` when content changes.** |
| `README.md` | Original one-line readme. |
| `CONTEXT.md` | This file. |

Nothing else should be needed. If you add files, link them from `sitemap.xml` if they are crawlable pages.

---

## 3. Design system (in `index.html` `<style>`)

- **Colors:** CSS custom properties on `:root` — light theme (`--bg:#F5F7FA`, `--surface:#FFF`, `--ink:#0F1B2D`, accent `--accent:#1D4ED8`) and dark theme overrides in two places: `@media (prefers-color-scheme: dark)` (system default) **and** `:root[data-theme="dark"]` (manual toggle). **If you change a color, change it in both dark blocks.**
- **Hero/contact band:** `--hero-bg:#0A1630` navy with radial blue gradients + 56px grid overlay (`.hero::before/::after`).
- **Fonts:** Google Fonts — Plus Jakarta Sans (`--f-display`, headings) + IBM Plex Sans (`--f-body`), loaded via `<link>` with preconnect; `display=swap`.
- **Layout:** `.wrap` max-width `--maxw:1340px` (1520px ≥1600px viewport), `clamp()` type scale, sections are `section.block` separated by 1px rules.
- **Motion:** hero load-in `rise` animation; `prefers-reduced-motion` disables all animation/transition.
- **Breakpoints:** 1060px (nav → hamburger), 920px (hero 1-col), 900px (about/contact 1-col), 860px (job bullets 1-col), 560px (CTA grid).

---

## 4. `index.html` structure

**Sections in DOM order** (nav scroll-spy hooks `section.block` by id):

1. `nav#nav` — brand mark (inline SVG logo) + `#navList` links + `#menuBtn` (mobile) + `#themeBtn` (light/dark toggle)
2. `header.hero#top` — availability badge, `h1`, role line, lead, `.facts` list, `.cta` buttons; right: `aside.glance` (ZA monogram, 4 `.stat` cards, `.stack` chips)
3. `#about` — label + `h2`, then `.about > .text > (3 <p> + .focus cards)`. **The `.focus` grid MUST stay inside `.text`** — that's the reference layout (inner 2-col grid: paragraphs col 1, cards col 2).
4. `#results` — 8 `.metric` cards (`.num` with `<em>` accent, `<p>`, `.co` company label)
5. `#experience` — `#filters` (rendered chips) + `#jobs` (rendered from JS)
6. `#skills` — `#skillSearch` input + `#searchOut` (aria-live) + `#skillGrid` (rendered from JS)
7. `#credentials` — 3 `.cred` cards (Education/Certification, Experience, Languages & Details)
8. `#contact` — `.contact-grid`: pitch + CTAs left, `.rows` contact cards right (email/phone with copy buttons, LinkedIn/GitHub as links)
9. `footer` + `div#toast` (copy feedback, `role="status"`)

**Heading hierarchy:** exactly one `h1` (hero), `h2` per section, `h3` inside cards — keep it that way for SEO.

---

## 5. The JS data arrays (edit content here)

All near the top of the single `<script>` in `index.html`.

### `TAGS` — filter chips for the career section
```js
var TAGS = { backend:'Backend & APIs', crm:'CRM & Integrations', ai:'AI Features',
             lead:'Leadership', cloud:'Cloud & CI/CD', frontend:'Frontend & Web' };
```

### `JOBS` — career entries (rendered into `#jobs`)
```js
{co:'Tkxel', ini:'T', role:'Principal Software Engineer',
 when:'Dec 2022 – Present · Lahore, Pakistan',
 items: [ ['backend cloud', 'Bullet HTML — <strong>bold</strong> allowed'], ... ]}
```
- `ini` = letters shown in the logo square (no logo images exist; initials only).
- First element of each item = space-separated tag keys matched against `TAGS` (drives filter chips + counts). Second = HTML bullet text.
- **Adding a job:** append an object. **Adding a filter:** add key to `TAGS` and use it in items.

### `SK` — skill cards (rendered into `#skillGrid`)
```js
{t:'Category Name', tags:['Tag','Tag']}                    // normal card
{t:'...', feature:true, lede:'intro text', tags:[...]}     // full-width navy card
{t:'Industry Domains', wide:true, tags:[...]}              // full-width light card
```
- JS/TS/Node.js live in the **"JavaScript & Frontend"** card (user-requested; JavaScript is used across the CRM/web projects — don't remove).
- AI tooling lives in the **"AI Tools & Practices"** card (GitHub Copilot, Claude Code, JetBrains AI Assistant, ChatGPT Codex, OpenCode, Cursor + context/memory, agents & skills, prompt engineering) — keep in sync with the resume's "AI-Assisted Dev" row.
- Card count badges and the "N skills across M areas" idle text are computed automatically.

### Static content that duplicates resume facts
Metrics (`#results`), about paragraphs, glance stats, credentials. Keep numbers consistent with the resume (§7).

---

## 6. Regenerating the resume PDF

The PDF is **generated**, not hand-edited. After any change to `resume.html` (or `resume-python.html` — same command with different filenames):

```bash
"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" \
  --headless=new --disable-gpu --no-pdf-header-footer \
  --print-to-pdf=Zulafqar_Ali_Resume.pdf "file://$PWD/resume.html"

"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" \
  --headless=new --disable-gpu --no-pdf-header-footer \
  --print-to-pdf=Zulafqar_Ali_Resume_Python_FastAPI.pdf "file://$PWD/resume-python.html"
```

- Verify visually: render page to PNG with PyMuPDF (`pymupdf`) and inspect; it must stay **2 pages, A4**.
- **Filename is referenced by `index.html` in 4 places** (2 CTAs × `href`/`download`). If you rename the PDF, update all of them.
- `resume.html` design mirrors Awais Ashraf's resume: header (name, blue role line, icon contact rows, monogram avatar), blue-bar section headings with rule, KPI strip (blue left borders), jobs with right-aligned dates, two-column skills table, note box. Print CSS: `@page{size:A4;margin:0}`, body padding 11mm/12mm, Arial.
- Python venv with `pypdf`/`pymupdf` used for inspection lives in the system temp dir (recreate: `python3 -m venv <tmp> && pip install pymupdf pypdf`).

### `og-image.png` regeneration
Generated by headless Chrome `--window-size=1200,630 --screenshot=og-image.png` from a standalone HTML card styled like the hero (source was a throwaway temp file — recreate from the hero styles if the brand/headline changes). Dimensions must stay **1200×630** (OG requirement).

---

## 7. Content facts (keep these consistent everywhere)

- Zulafqar Ali · Principal Software Engineer · Tkxel, Lahore, Pakistan (UTC+5) · open to remote
- 9+ years (Jul 2014 – Present), teams of 4–6 led, cross-functional team up to 40
- Projects: Cafe Zupas ordering (80+ branches, millions of users), eDocs, SuiteCRM healthcare recruitment (7,000+ CRM users), ttgat.us (200+ users, 50 GB, QuickBooks sync), Xension, CarSwitch
- Contact: zulafqarali143@gmail.com · +92-344-6800-107 · linkedin.com/in/zulafqar-ali-076357104 · github.com/zulafqarali · zulafqarali.github.io
- Education: BS CS, The University of Lahore, 2016 · Certification: SugarCRM Developer
- Languages: English (professional), Urdu (native)

---

## 8. SEO (already implemented — maintain, don't duplicate)

- `<head>`: keyword-rich `<title>`, meta description (≤160 chars), `keywords`, `author`, `robots` (`index, follow, max-image-preview:large`), canonical `https://zulafqarali.github.io/`, `rel="me"` identity links.
- Open Graph (`og:title/description/url/type/site_name/locale/image[+width/height/alt]`) + Twitter `summary_large_image`, all pointing at `/og-image.png`.
- **JSON-LD** (`<script type="application/ld+json">`): `@graph` of `Person` + `WebSite` — includes `sameAs`, `worksFor`, `alumniOf`, `knowsAbout`. Must remain valid JSON.
- `robots.txt` → `sitemap.xml` (**bump `<lastmod>` on content edits**).
- After deploy: submit sitemap in Google Search Console + Bing Webmaster (account step, not in repo).

---

## 9. Verification checklist (run before pushing)

```bash
# 1. JSON-LD parses, meta tags present, tags balanced
python3 - <<'EOF'
import re, json, xml.etree.ElementTree as ET
h = open('index.html').read()
json.loads(re.search(r'application/ld\+json">(.*?)</script>', h, re.S).group(1))
ET.parse('sitemap.xml')
for t in ['div','section','a','span','main','ul','li','button']:
    o,c = len(re.findall(r'<%s[\s>]'%t,h)), len(re.findall(r'</%s>'%t,h))
    assert o==c, (t,o,c)
print('OK')
EOF

# 2. inline JS syntax
python3 -c "import re;open('/tmp/s.js','w').write(re.search(r'<script>(.*?)</script>',open('index.html').read(),re.S).group(1))"
node --check /tmp/s.js && echo "JS OK"

# 3. eyeball it
open index.html
```

Known UI pitfalls already hit once: **`.focus` must stay inside `.about .text`** (§4), and **dark-theme colors live in two CSS blocks** (§3).

---

## 10. Git / deploy / security

- Remote: `https://github.com/zulafqarali/zulafqarali.github.io` (GitHub Pages, served from repo root).
- ⚠️ **The remote URL in `.git/config` has a personal access token (`ghp_...`) embedded. Revoke that token on GitHub** and re-add the remote with normal auth.
- Nothing has been committed yet as of this writing unless a later commit says otherwise — check `git status` first.
- Deploy is automatic on push to the default branch; no build step to run.
