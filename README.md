# AppBoard

A single-file HTML whiteboard for rapid mobile app prototyping. No build step, no dependencies, no setup.

---

## TL;DR

1. Duplicate `index.html` and rename it for your project
2. Open it in a browser
3. Share the file with an AI along with context about what you're building — *"this is a [type of app], here's the problem I'm exploring"*
4. Ask for prototype pages and variants — different layouts, flows, IA models, interaction patterns
5. Iterate in HTML until the design feels right
6. Use the final prototype as the visual spec when building in Swift (or any platform)

The HTML isn't throwaway work. It's the design. Iterate here first, then build.

---

## What's in this repo

| File | Purpose |
|---|---|
| `index.html` | Blank template — duplicate this for every new project |
| `example.html` | Working Airbnb-like prototype — reference for how components compose |
| `SPEC.md` | Full component reference for LLMs and developers |
| `LLMCONTEXT.md` | Build guide for LLMs — read this before generating a prototype |

---

## The idea

AppBoard is a desktop viewer — a sidebar and a centered phone frame — that you fill with mobile app prototypes. Each file is one project. The sidebar has a **page dropdown** at the top and a list of **variants** below it. Pages are distinct scenarios or user types. Variants are design alternatives within the same scenario.

The goal isn't production code. It's speed and feel. Draw the experience, click through it, decide if it works.

---

## Mental model

**One file = one project.** All exploration for a product lives in the same file.

**Pages = distinct scenarios.** Page 1, Page 2… Each page is a different angle on the problem — a different user type, a different flow, a different surface model. Switch pages with the dropdown at the top of the sidebar.

**Variants = design alternatives within a page.** Variant A, Variant B… Variant Z, Variant AA… Each variant is a different approach to the same scenario. Listed in the sidebar below the dropdown.

**Inside a variant = whatever the idea needs.** A single layout. Two screens in a flow. A tab bar with drill-in detail. The variant contains as many screens as it takes to express the idea.

---

## Naming system

| Thing | Name |
|---|---|
| Pages | Page 1, Page 2, Page 3… |
| Variants | Variant A, Variant B… Variant Z, Variant AA, Variant AB… |
| Page IDs | `p1`, `p2`, `p3`… |
| Variant IDs | `p1vA`, `p1vB`… `p2vA`, `p2vB`… |

---

## Getting started

1. Duplicate `index.html` → rename it for your project (e.g. `myapp.html`)
2. Open in Chrome or Safari
3. Update the `<title>` tag and sidebar logo text
4. Build your first prototype inside the existing `variant-p1vA` slot — or replace it with your own
5. Open `example.html` to see a working reference if you need a pattern

---

## Adding a page

**1. Page dropdown option** — in `#page-select`:
```html
<option value="p2">Page 2</option>
```

**2. Variant group** — in the sidebar, after the last existing `.variant-group`:
```html
<div class="variant-group" id="page-p2">
  <div class="sidebar-section">
    <div class="sidebar-section-label">Variants</div>
    <div class="sidebar-item active" onclick="switchVariant('p2vA', this)">
      <div class="sidebar-item-dot" style="background: var(--accent)"></div>
      Variant A
    </div>
  </div>
</div>
```

**3. Variant block** — inside `.phone-frame`:
```html
<div class="variant" id="variant-p2vA">
  <!-- screens, subpages, tab bar go here -->
</div>
```

**4. About description** — in `.sidebar-about`:
```html
<div class="sidebar-about-p2vA" style="display:none;">
  <div class="sidebar-section-label">About</div>
  <div style="padding:0 8px; font-size:11px; color:var(--text-secondary); line-height:1.5;">
    One sentence on what this variant is exploring.
  </div>
</div>
```

---

## Adding a variant to an existing page

**1. Sidebar item** — inside the page's `.variant-group`:
```html
<div class="sidebar-item" onclick="switchVariant('p1vB', this)">
  <div class="sidebar-item-dot" style="background: #3b82f6"></div>
  Variant B
</div>
```

**2. Variant block** — inside `.phone-frame`:
```html
<div class="variant" id="variant-p1vB">
  <!-- screens, subpages, tab bar go here -->
</div>
```

**3. Variant CSS** — in the marked section of `<style>`:
```css
/* ── p1vB: Your variant ── */
.p1vB-card { ... }
```
Prefix all classes with the variant ID to avoid conflicts.

---

## Component reference

See **SPEC.md** for the full reference with complete HTML patterns for every component.

### Quick list

**Layout:** `.m-page` · `.m-subpage` · `.m-tab-bar` · `.m-tab` · `.m-fab`

**Headers:** `.m-header` · `.m-back-header` · `.m-section-header` · `.m-label`

**Lists:** `.m-item-row` · `.m-cat-row` · `.m-col-card`

**Scroll:** `.m-h-scroll` + `.m-h-scroll-end` · `.m-chips` · `.m-chip`

**Misc:** `.m-divider` · `.m-source` · `.m-avatar` · `.m-intent[data-i="..."]`

**Animation:** add `class="stagger"` to any container for fade-up children

### Navigation JS

```js
switchPage('p2')              // switch to a different page (via dropdown)
switchVariant('p1vB', el)     // activate a variant from the sidebar
switchMTab('p1vA', 'home', el) // switch tab within a variant
openSubpage('p1vA-detail')    // slide in a subpage
closeSubpage('p1vA-detail')   // slide back out
openAdd('p1vA')               // open the add/save bottom sheet
closeAdd('p1vA')              // close it
```

---

## Design tokens

Edit `:root` in the CSS to match your project:

```css
:root {
  --accent:     #059669;   /* brand color */
  --phone-w:    390px;     /* iPhone 14 width */
  --phone-h:    844px;     /* iPhone 14 height */
  --tab-bar-h:  80px;
}
```

---

## Tips

- **picsum.photos for images** — `https://picsum.photos/seed/any-word/400/300`. Same seed = same image every reload.
- **Always end `.m-h-scroll` with `<div class="m-h-scroll-end"></div>`** — browsers clip trailing padding on overflow containers. The spacer div fixes it.
- **Stagger animation** — add `class="stagger"` to any flex container for auto-sequenced fade-up on children.
- **Read `example.html`** — the Airbnb prototype shows how everything composes. Copy patterns from it freely.

---

## For LLMs

Read **LLMCONTEXT.md** before generating a prototype — it covers orientation, what to build, what not to build, patterns, and gotchas. Full component spec in **SPEC.md**.
