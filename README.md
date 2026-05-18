# AppBoard

A single-file HTML whiteboard for rapid mobile app prototyping. No build step, no dependencies, no setup.

---

## TL;DR

1. Duplicate `index.html` and rename it for your project
2. Open it in a browser
3. Share the file with an AI along with context about what you're building — *"this is a [type of app], here's the problem I'm exploring"*
4. Ask for prototype variations — different layouts, flows, IA models, interaction patterns
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

AppBoard is a desktop viewer — a sidebar and a centered phone frame — that you fill with mobile app prototypes. Each file is one project. Each sidebar entry is one idea. Inside a prototype you build whatever screens, flows, or layouts that idea needs to feel real.

The goal isn't production code. It's speed and feel. Draw the experience, click through it, decide if it works.

---

## Getting started

1. Duplicate `index.html` → rename it for your project (e.g. `myapp.html`)
2. Open in Chrome or Safari
3. Update the `<title>` tag and sidebar logo text
4. Build your first prototype inside the existing `variant-v1` slot — or replace it with your own
5. Open `example.html` to see a working reference if you need a pattern

---

## Mental model

**One file = one project.** All prototypes for a product live in the same file.

**Sidebar entries = prototypes.** Each entry is a different idea or approach — not a different screen. "Inbox model", "Card stack", "One surface". Name the concept, not the page.

**Inside a prototype = whatever the idea needs.** A single layout. Two screens in a flow. A tab bar with drill-in detail. The prototype contains as many screens as it takes to express the idea.

---

## Adding a prototype

**1. Sidebar entry** — in `.sidebar-section`:
```html
<div class="sidebar-item" onclick="switchVariant('v2', this)">
  <div class="sidebar-item-dot" style="background: #3b82f6"></div>
  Prototype B — Your Idea
</div>
```

**2. About description** — in `.sidebar-about`:
```html
<div class="sidebar-about-v2" style="display:none;">
  <div class="sidebar-section-label">About</div>
  <div style="padding:0 8px; font-size:11px; color:var(--text-secondary); line-height:1.5;">
    One sentence on what this prototype is exploring.
  </div>
</div>
```

**3. Variant block** — inside `.phone-frame`:
```html
<div class="variant" id="variant-v2">
  <!-- screens, subpages, tab bar go here -->
</div>
```

**4. Variant CSS** — in the marked section of `<style>`:
```css
/* ── v2: Your prototype ── */
.v2-card { ... }
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
switchVariant('v2', el)       // activate a prototype from the sidebar
switchMTab('v1', 'home', el)  // switch tab within a prototype
openSubpage('v1-detail')      // slide in a subpage
closeSubpage('v1-detail')     // slide back out
openAdd('v1')                 // open the add/save bottom sheet
closeAdd('v1')                // close it
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
