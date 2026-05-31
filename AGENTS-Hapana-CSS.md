# AGENTS-Hapana-CSS.md — Hapana Custom CSS Context

> Context document for AI assistants working on this repo.
> For full infrastructure detail see `AGENTS-Master.md` in the `ditto-master-it-ops` repo.

---

## What This Repo Is

Hapana is the studio management and booking SaaS used by Ditto Club. It provides embed widgets (booking schedule, package/membership purchase, dashboard, etc.) that are iframed into the Ditto Club website.

The widgets ship with Hapana's default MUI (Material UI) styling — generic blue, wrong fonts, nothing on-brand.

Hapana supports injecting a single custom CSS file via a URL registered in their admin settings. This repo contains that CSS file (`hapana-style.css`) and everything needed to work on it.

---

## Files in This Repo

| File | Purpose |
|---|---|
| `hapana-style.css` | The actual stylesheet — this is what Hapana loads |
| `AGENTS-Hapana-CSS.md` | This file — AI context |
| `hapana-documentation.md` | Scraped Hapana docs: how CSS customisation works, full CSS selector reference |
| `html-structure.md` | Detailed DOM tree of each Hapana widget, inspected live. Use this when targeting specific elements. |

---

## CSS Delivery Architecture

### Current Setup

Hapana requires the CSS URL to be HTTPS and to be served with `Content-Type: text/css`. Direct GitHub Raw URLs don't work because GitHub serves them as `text/plain` and Hapana rejects them due to `X-Content-Type-Options: nosniff`.

**Solution:** A Cloudflare Worker sits in front. It fetches the raw CSS from GitHub and re-serves it with the correct headers.

- **URL registered with Hapana:** `https://ditto-hapana-css.peter-c-christie.workers.dev/hapana-style.css`
- **CSS source (prod):** `https://raw.githubusercontent.com/pcchristie/dittoclub-hapana-css/main/hapana-style.css`
- **Cache-Control:** `no-cache` — changes go live immediately without cache purge

### History (why not simpler approaches)

| Approach | Problem |
|---|---|
| GitHub Raw URL directly | `Content-Type: text/plain` → rejected by Hapana |
| jsDelivr CDN | Correct content-type but aggressive caching; purge requests throttled after a few in quick succession |
| Cloudflare Worker (current) | Correct content-type + no-cache; instant updates |

### Planned: Test/Prod Branching

The website has two environments: `test.ditto.club` (test branch) and `ditto.club` (prod). Only one CSS URL can be registered with Hapana.

**Plan:** Extend the Worker to read the `Referer` header. When Hapana loads the CSS, it passes the page URL as `Referer`. The Worker branches on this to serve from the `test` or `main` GitHub branch.

A `test` branch in this repo needs to be created. Until then, there is only `main` (prod).

**Fallback if Referer is unreliable:** Embed `?env=test` as a query param on the Hapana embed URL on the test Astro site, then branch in the Worker on `searchParams.get('env')`.

Full Worker code for both the current and planned setup lives in `AGENTS-Master.md`.

---

## The Widget — Key Technical Facts

- The widgets are **React + Material UI (MUI)** components loaded inside an iframe.
- CSS class naming convention: `Mui-<widgetName>-OFS-<elementName>` — e.g., `.Mui-packageCard-OFS-card`
- **Do not target `jss*` classes** (e.g., `jss123`). These are dynamically generated per Hapana release and will break on any widget update. Always target the stable `Mui-*-OFS-*` classes or standard MUI classes instead.
- When a stable selector isn't in `hapana-documentation.md`, use browser DevTools to inspect the live widget, find the nearest ancestor with a `Mui-*-OFS-*` class, and write a descendant selector from that.

### Astro: use `is:inline` on the script tag

When embedding Hapana widgets in an Astro page, the script tag **must** have `is:inline`:

```html
<script is:inline src="https://widget.hapana.com/hapana_widget.js"></script>
```

Without it, Astro converts the tag into a module script. Module scripts are subject to CORS rules — the browser will block the load because Hapana's server doesn't include your domain in its allowed-origins headers. A regular script tag (which `is:inline` preserves) bypasses CORS entirely. This was confirmed as the cause of a blank white widget box on `test.ditto.club`.

---

## Brand Tokens

Full brand reference: `AGENTS-Branding.md` in the `ditto-master-it-ops` repo.

Quick reference for CSS work:

```css
/* Colours */
--color-cobalt:    #171C8F;   /* primary brand blue — text, headings */
--color-off-white: #F9F8F2;   /* background */
--color-spearmint: #B9DCD2;   /* primary CTA buttons */
--color-asphalt:   #333F48;   /* secondary / footer */
--color-tan:       #E2DABA;   /* accent */
--color-black:     #0F1010;   /* body text */
```

```css
/* Fonts — must be declared in hapana-style.css; the widget iframe cannot read fonts from the parent page */
@font-face {
  font-family: 'Acumin Variable Concept';
  src: url('https://raw.githubusercontent.com/pcchristie/dittoclub-hapana-css/main/fonts/AcuminVariableConcept.otf');
  font-weight: 100 900;
  font-stretch: 75% 100%;
}
@font-face {
  font-family: 'Instrument Serif';
  src: url('https://raw.githubusercontent.com/pcchristie/dittoclub-hapana-css/main/fonts/InstrumentSerif-Regular.ttf');
  font-style: normal;
}
@font-face {
  font-family: 'Instrument Serif';
  src: url('https://raw.githubusercontent.com/pcchristie/dittoclub-hapana-css/main/fonts/InstrumentSerif-Italic.ttf');
  font-style: italic;
}
```

> **Note:** Fonts are currently loaded via Squarespace's CDN in the old site CSS. If fonts aren't self-hosted in this repo, the URLs above will need updating. Check whether font files have been added to this repo before using these URLs.

### Typography roles

| Role | Font | Notes |
|---|---|---|
| Headings | Acumin Variable Concept | Weight 750, stretch 80%, uppercase |
| Sub-headings / decorative | Instrument Serif | Use regular or italic variant |
| Body / labels | Acumin Variable Concept | Weight 400, stretch 100% |
| Buttons | Acumin Variable Concept | Weight 700 |

### Primary CTA button

Spearmint background (`#B9DCD2`) + Cobalt text (`#171C8F`), no border, no border-radius (square), uppercase.

```css
.Mui-button-OFS-containedPrimary,
.Mui-button-OFS-containedSecondary {
  background-color: #B9DCD2 !important;
  color: #171C8F !important;
  border: none !important;
  border-radius: 0 !important;
  text-transform: uppercase;
  font-weight: 700;
}
```

---

## Key Widgets Used by Ditto Club

| Widget | Embed location | Purpose |
|---|---|---|
| Classes / Schedule | `/classes/schedule` | Browse and book classes by date |
| Packages / Memberships | `/classes/membership` | Purchase class packs and memberships (including intro offer) |
| Dashboard | (member area) | Member self-service — bookings, account |

See `html-structure.md` for the full DOM tree of each widget.

---

## Open Items

- [x] Deploy updated Worker code (Referer-based test/prod branching) — live and confirmed working
- [x] Create `test` branch in this repo
- [x] Verify branching works on test.ditto.club — confirmed via spearmint border test
- [ ] Fix Hapana booking widget mobile CSS — widget currently doesn't scale well on mobile. **Do not hide it** (old Squarespace site hid it on mobile and showed an app-download message instead — that was a workaround, not a solution). Needs proper responsive CSS.
- [x] Fix widget fonts on test.ditto.club — removed all `@font-face` declarations and custom font-family names (`'INSTRUMENTSERIF'`, `'ACUMIN'`) from test branch. All rules now reference `'Instrument Serif'` and `'Acumin Variable Concept'` to match global.css. Widget is DOM-injected on the Astro site so page fonts are in the same cascade — no separate font loading needed. Go-live: merge test → main, same approach applies.
- [x] ~~Confirm whether fonts need to be self-hosted inside this repo or if the Squarespace CDN URLs remain usable during transition~~ — resolved by removing @font-face entirely from test branch; using page fonts directly.
