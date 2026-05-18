# AppBoard — LLM Context & Build Guide

Read this before building a prototype. It covers orientation, how to think about the task, what to build, what not to build, and common failure modes.

---

## What you're working with

`index.html` is a self-contained mobile app prototyping tool. It renders a desktop shell — sidebar on the left, centered iPhone frame on the right — and displays mobile app prototypes inside the phone. You build prototypes by writing HTML inside it.

The file has:
- A full shared CSS component library (`.m-*` classes)
- A navigation system (tab bars, subpages, screen transitions)
- A working example prototype (Airbnb-like home screen)
- An empty data model ready to populate

Everything you need is already there. You add HTML and CSS. You don't install anything, import anything, or run a build.

---

## The mental model

**One file = one project.** All prototypes for a product live in one HTML file.

**Sidebar entry = prototype.** Each entry in the sidebar is a different idea or approach — "Inbox model", "Card stack", "One surface". Not a page name.

**Prototype = however many screens the idea needs.** You might draw:
- A single screen to nail a layout
- Two screens connected by a transition
- A full tab bar with drill-in detail pages
- A flow through 3 states of the same feature
- Whatever the concept needs to feel real

The prototype contains the experience. The sidebar switches between ideas.

---

## How to approach a build request

When asked to build a new prototype:

1. **Understand the concept first.** What is the idea? What should someone feel when they interact with it? A card-based inbox feels different from a flat list. A bottom sheet feels different from a pushed screen. Know what you're building before you write HTML.

2. **Pick the right screens.** Don't build every screen in the app. Build the 1–3 screens that express the core idea. If it's a booking flow, build the list and the detail. If it's a new nav pattern, build the home state and one transition. Prototype the idea, not the whole product.

3. **Use shared components first.** The `.m-*` library covers most patterns. Only write custom CSS when the shared components genuinely don't fit. When you do write custom CSS, prefix it with the variant ID (`.v1-`, `.v2-`, etc.).

4. **Make it feel interactive.** Tappable items should respond visually (`:active` states are built into most components). Subpages should slide in. Chips should toggle. The prototype should feel like an app, not a mockup.

5. **Don't over-build.** Empty states, loading states, error states — skip them unless the prototype is specifically about those things. Ship the happy path first.

---

## File editing rules

**Where to add things:**

| What | Where |
|---|---|
| New sidebar entry | Inside `.sidebar-section` div |
| New about description | Inside `.sidebar-about` div, keyed to variant id |
| New prototype HTML | Inside `.phone-frame`, after the last existing variant |
| New variant CSS | In the `VARIANT-SPECIFIC STYLES` section of `<style>`, prefixed with variant id |
| New JS helpers | After the `CHIP TOGGLE` section in `<script>` |
| Data (items, categories, collections) | At the top of `<script>` in the DATA section |

**What NOT to touch:**
- The reset, design tokens, desktop shell CSS
- The shared `.m-*` component CSS
- The detail page shared styles
- The add modal shared styles
- The core navigation functions (`switchVariant`, `switchMTab`, `openSubpage`, `closeSubpage`)
- The detail renderer functions (`openItemDetail`, `openCategoryDetail`, `openCollectionDetail`)

---

## ID naming convention

This is strict. Wrong IDs break navigation silently.

```
Variant outer:   id="variant-v1"          (must match sidebar onclick)
Page:            id="v1-home"             ({variant}-{pagename})
Subpage:         id="v1-listing-detail"   ({variant}-{name})
Add overlay:     id="v1-add-overlay"      ({variant}-add-overlay)
About panel:     class="sidebar-about-v1" (sidebar-about-{variant})
```

Tab `onclick`: `switchMTab('v1', 'home', this)` — first arg is variant id, second is page name (must match the page `id` suffix).

---

## CSS rules

1. **All variant CSS is prefixed.** `.v1-card`, `.v2-hero`, never `.card` or `.hero`. Unprefixed classes collide across prototypes.

2. **Use tokens, not hardcoded values.** `var(--text-secondary)` not `#71717a`. `var(--radius-md)` not `14px`. Tokens are in `:root`.

3. **Inline styles for one-offs.** If something is used exactly once and isn't conceptually a component, inline it. `style="padding:0 18px"` is fine. Don't create a class for it.

4. **Don't override `.m-*` classes.** If a shared component needs modification, wrap it or subclass it (`.v1-header .m-section-header { ... }`). Don't mutate the shared styles.

---

## Component gotchas

**Horizontal scroll trailing bleed.** Browsers clip `padding-right` on `overflow: auto` containers. Always end `.m-h-scroll` with `<div class="m-h-scroll-end"></div>`. Without it, the last card bleeds to the phone frame edge.

**Bottom breathing room.** `.m-page` needs space at the bottom or content disappears behind the tab bar. End every page with `<div style="height:20px"></div>`. Use `height:100px` when there's a FAB or sticky bar above the tab bar.

**Status bar offset.** The top of every page needs ~52px of top padding to clear the iOS status bar area. `.m-header` has this built in — use it as the first element on every page. If you're not using `.m-header`, add `padding-top: 52px` to your first element.

**Subpages sit above everything.** `.m-subpage` is `z-index: 100`. The tab bar is `z-index: 200`. The add overlay is `z-index: 300`. Don't use z-index values in this range for other elements or they'll conflict.

**Active page.** Only one `.m-page` per variant should have `class="m-page active"` on load. Same for `.m-tab`.

**Tab switching closes subpages.** `switchMTab` closes all subpages in the variant before switching. This is intentional — tapping a tab always returns to the tab root. If you don't want this, wire tab navigation differently.

**Chip toggle is radio-style.** `toggleChip(el)` deactivates all siblings and activates the one clicked. If you need multi-select, wire it differently: `this.classList.toggle('active')`.

---

## Patterns

### Pattern: Tab bar across multiple pages

```html
<div class="variant active" id="variant-v1">

  <div class="m-page active" id="v1-home"> ... </div>
  <div class="m-page" id="v1-explore"> ... </div>
  <div class="m-page" id="v1-profile"> ... </div>

  <div class="m-tab-bar">
    <button class="m-tab active" onclick="switchMTab('v1','home',this)">
      <svg viewBox="0 0 24 24"><!-- icon --></svg>
      <span>Home</span>
    </button>
    <button class="m-tab" onclick="switchMTab('v1','explore',this)">
      <svg viewBox="0 0 24 24"><!-- icon --></svg>
      <span>Explore</span>
    </button>
    <button class="m-tab" onclick="switchMTab('v1','profile',this)">
      <svg viewBox="0 0 24 24"><!-- icon --></svg>
      <span>Profile</span>
    </button>
  </div>

</div>
```

### Pattern: Drill-in from list to detail

```html
<!-- List item that opens detail -->
<div class="m-item-row" onclick="openSubpage('v1-item-detail')">
  ...
</div>

<!-- Detail subpage -->
<div class="m-subpage" id="v1-item-detail">
  <div class="detail-hero">
    <img src="..." alt="">
    <button class="detail-hero-back" onclick="closeSubpage('v1-item-detail')">
      <svg viewBox="0 0 24 24"><path d="M15 18l-6-6 6-6"/></svg>
    </button>
  </div>
  <div class="detail-body">
    <div class="detail-title">Title</div>
    ...
  </div>
</div>
```

### Pattern: No tab bar (full-screen single surface)

```html
<div class="variant active" id="variant-v1">
  <div class="m-page active" id="v1-main" style="bottom:0">
    <!-- bottom:0 overrides the default tab-bar-h offset -->
    ...
  </div>
  <!-- No .m-tab-bar -->
  <button class="m-fab" onclick="openAdd('v1')"> ... </button>
</div>
```

### Pattern: Sticky bottom bar (not a tab bar)

**Important:** `position:sticky` doesn't work inside `.m-subpage` because the content rarely overflows (no scroll = sticky never activates). Use `position:absolute; bottom:0` instead, and add `padding-bottom:100px` to the scrollable body to prevent content hiding behind the bar.

Inside a `.m-subpage`:
```html
<!-- Scrollable body — needs padding-bottom to clear the bar -->
<div class="v1-detail-body" style="padding:16px 18px 100px;">
  <!-- content -->
</div>

<!-- Bar pinned to bottom of subpage -->
<div style="position:absolute;bottom:0;left:0;right:0;
            padding:12px 18px 20px;
            background:rgba(255,255,255,0.95);backdrop-filter:blur(12px);
            -webkit-backdrop-filter:blur(12px);
            border-top:1px solid var(--border-subtle);
            display:flex;align-items:center;justify-content:space-between;
            z-index:10;">
  <div>
    <div style="font-size:16px;font-weight:700;">$185 <span style="font-size:12px;font-weight:400;color:var(--text-secondary);">/ night</span></div>
  </div>
  <button style="padding:12px 22px;border-radius:10px;background:#e61e4d;color:white;
                 border:none;font-family:inherit;font-size:14px;font-weight:700;">Reserve</button>
</div>
```

Inside a `.m-page` (above the tab bar), use `position:absolute; bottom:var(--tab-bar-h)` instead of `bottom:0`.

### Pattern: Hero image with overlay text

```html
<div style="margin:0 18px;border-radius:var(--radius-lg);overflow:hidden;position:relative;cursor:pointer;">
  <img src="https://picsum.photos/seed/hero/780/360" style="width:100%;height:180px;object-fit:cover;display:block;">
  <div style="position:absolute;inset:0;background:linear-gradient(to top,rgba(24,24,27,0.85) 0%,transparent 60%);"></div>
  <div style="position:absolute;bottom:0;left:0;right:0;padding:16px 18px;">
    <div style="font-size:11px;font-weight:600;color:#6ee7b7;margin-bottom:4px;">Label</div>
    <div style="font-size:17px;font-weight:700;color:white;letter-spacing:-0.02em;">Title</div>
  </div>
</div>
```

### Pattern: Profile / stat block

```html
<div style="padding:52px 18px 20px;display:flex;align-items:center;gap:14px;">
  <div style="width:56px;height:56px;border-radius:50%;
              background:linear-gradient(135deg,#6366f1,#8b5cf6);
              display:flex;align-items:center;justify-content:center;flex-shrink:0;">
    <span style="font-size:20px;font-weight:700;color:white;">M</span>
  </div>
  <div>
    <div style="font-size:18px;font-weight:700;letter-spacing:-0.02em;">Name</div>
    <div style="font-size:12px;color:var(--text-secondary);margin-top:2px;">Subtitle</div>
  </div>
</div>

<div style="display:flex;gap:10px;padding:0 18px 16px;">
  <div style="flex:1;background:var(--surface);border:1px solid var(--border-subtle);
              border-radius:var(--radius-md);padding:14px 12px;text-align:center;">
    <div style="font-size:22px;font-weight:700;letter-spacing:-0.03em;">142</div>
    <div style="font-size:11px;color:var(--text-secondary);font-weight:500;">Stat label</div>
  </div>
  <!-- repeat for each stat -->
</div>
```

### Pattern: Empty state

```html
<div style="padding:60px 18px;text-align:center;color:var(--text-secondary);">
  <div style="font-size:32px;margin-bottom:12px;">⬜</div>
  <div style="font-size:15px;font-weight:600;color:var(--text-primary);margin-bottom:6px;">Nothing here yet</div>
  <div style="font-size:13px;line-height:1.5;">One line explaining how to get started.</div>
</div>
```

### Pattern: Settings / menu row

```html
<div style="display:flex;align-items:center;gap:14px;padding:14px 18px;cursor:pointer;transition:background var(--transition-fast);"
     onmouseenter="this.style.background='var(--border-subtle)'" onmouseleave="this.style.background=''">
  <div style="width:44px;height:44px;border-radius:50%;background:var(--border-subtle);
              display:flex;align-items:center;justify-content:center;flex-shrink:0;">
    <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="var(--text-secondary)" stroke-width="1.8">
      <!-- icon -->
    </svg>
  </div>
  <div style="flex:1;">
    <div style="font-size:14px;font-weight:600;letter-spacing:-0.01em;">Row Title</div>
    <div style="font-size:12px;color:var(--text-secondary);">Subtitle</div>
  </div>
  <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="var(--text-tertiary)" stroke-width="2"><path d="M9 6l6 6-6 6"/></svg>
</div>
<div class="m-divider"></div>
```

### Pattern: Search bar

```html
<div style="padding:0 18px 10px;">
  <div style="display:flex;align-items:center;gap:10px;
              background:var(--surface);border:1px solid var(--border);
              border-radius:var(--radius-md);padding:11px 14px;
              box-shadow:0 1px 6px rgba(0,0,0,0.06);">
    <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="var(--text-tertiary)" stroke-width="2">
      <circle cx="11" cy="11" r="7"/><path d="M21 21l-4.35-4.35"/>
    </svg>
    <input type="text" placeholder="Search..." style="flex:1;border:none;outline:none;
           font-family:inherit;font-size:14px;color:var(--text-primary);background:none;">
  </div>
</div>
```

---

## Extending the component library

When a shared component doesn't quite fit, you have three options — in order of preference:

### Option 1: Inline style the variation

Most one-off adjustments don't need a new class. Override inline:

```html
<!-- Wider item image -->
<div class="m-item-img" style="width:80px;height:80px;">...</div>

<!-- Section header with no top padding -->
<div class="m-section-header" style="padding-top:8px;">...</div>
```

### Option 2: Add a variant-specific class

When the pattern repeats within a prototype, add a class in the variant CSS section:

```css
/* ── v2: Listing prototype ── */
.v2-price-tag {
  font-size: 13px; font-weight: 700;
  color: var(--text-primary);
}
.v2-rating {
  display: flex; align-items: center; gap: 3px;
  font-size: 12px; font-weight: 500;
}
```

Use it anywhere in the `v2` variant HTML.

### Option 3: Promote to shared component

When the same pattern appears across multiple prototypes and it belongs in the component library, add it to the `SHARED MOBILE COMPONENTS` CSS section with an `.m-` prefix and document it in SPEC.md.

**Only promote if:**
- It's truly generic (not tied to a specific app concept)
- It will be used in 2+ prototypes
- It follows the same visual language as existing components

Examples worth promoting: a new card shape, a badge style, a new list row variant. Not worth promoting: a specific color scheme, a domain-specific widget, anything with a brand name baked in.

---

## Growing the library over time

AppBoard is meant to accumulate. Every prototype you build adds to the vocabulary. Some practices that keep it healthy:

**Document new shared components as you add them.** Add a row to the component table in SPEC.md. Future prototypes (and future LLMs) will find it.

**Keep variant CSS lean.** The best prototypes use 80% shared components and 20% custom styles. If you're writing a lot of custom CSS, ask whether the shared library is missing something real, or whether you're over-designing.

**Retire old prototypes by commenting them out, not deleting.** Dead prototypes are references. A commented-out `variant-v3` can be revived or mined for patterns.

**Refactor when you see repetition.** If `v3` and `v4` both have nearly identical card styles, consolidate into a shared class. The file will stay readable.

---

## What makes a good prototype

**It feels interactive.** Tapping things responds. Pages slide in and out. Chips filter. The prototype behaves like an app, not a static mockup.

**It's focused.** It expresses one idea clearly. It doesn't try to show every screen. A great prototype of a booking flow might be: listing card → tap → detail → reserve button. That's it.

**It uses real-ish content.** Placeholder content makes prototypes feel fake and makes decisions harder. Use real place names, real prices, real category names. Picsum photos seeded to the content type. The content should feel like it could be real.

**It's fast to read.** If someone picks up the file cold, they should be able to understand the prototype's intent from the sidebar description and the first screen. Clear names. Clean structure.

**It doesn't have noise.** No unused CSS. No dead HTML. No commented-out sections from mid-build debugging. Clean output reflects clean thinking.

---

## Common mistakes

**Forgetting `.m-h-scroll-end`.** Last card bleeds to the phone frame edge. Always end `.m-h-scroll` with the spacer div.

**No bottom breathing room.** Content disappears behind the tab bar or FAB. End every `.m-page` with `<div style="height:20px">` (or `100px` if there's a FAB or sticky bar).

**Wrong ID pattern.** Navigation breaks silently if the page `id` doesn't match what `switchMTab` is looking for. `id="v1-home"` must match `switchMTab('v1', 'home', this)`.

**Unprefixed CSS.** `.card { ... }` defined in one variant bleeds into another. Always prefix: `.v1-card { ... }`.

**Hardcoded colors instead of tokens.** Breaks when the accent color changes. Use `var(--accent)`, `var(--text-secondary)`, etc.

**z-index conflicts.** Pages are 0, subpages are 100, tab bar is 200, add overlay is 300. Don't use these values for other elements.

**Building too much.** A prototype that tries to show every screen of an app is usually less useful than one that shows the core 2–3 screens with real content and real interactions. Constrain yourself.

---

## SVG icons

AppBoard uses inline SVGs throughout. The convention is `viewBox="0 0 24 24"`, `fill="none"`, `stroke="currentColor"` or a specific color, `stroke-width` between 1.6–2.2.

Size is always set by CSS, not the SVG element itself (the `width` and `height` attributes on SVGs in the HTML are for fallback only where needed).

Recommended icon style: Feather Icons or Heroicons (outline). Avoid filled icons unless specifically needed — the outline style matches the aesthetic.

Common icons used in AppBoard:

```
Home:       <path d="M3 9.5L12 3l9 6.5V20a1 1 0 01-1 1H4a1 1 0 01-1-1V9.5z"/>
Search:     <circle cx="11" cy="11" r="7"/><path d="M21 21l-4.35-4.35"/>
Grid:       <rect x="3" y="3" w="7" h="7" rx="1"/>...  (4 rects)
Heart:      <path d="M20.84 4.61a5.5 5.5 0 00-7.78 0L12 5.67l-1.06-1.06a5.5..."/>
Back:       <path d="M15 18l-6-6 6-6"/>
Chevron →:  <path d="M9 6l6 6-6 6"/>
Plus:       <line x1="12" y1="5" x2="12" y2="19"/><line x1="5" y1="12" x2="19" y2="12"/>
Map pin:    <path d="M21 10c0 7-9 13-9 13s-9-6-9-13a9 9 0 0118 0z"/><circle cx="12" cy="10" r="3"/>
User:       <path d="M20 21v-2a4 4 0 00-4-4H8a4 4 0 00-4 4v2"/><circle cx="12" cy="7" r="4"/>
Star:       <polygon points="12 2 15.09 8.26 22 9.27 17 14.14 18.18 21.02 12 17.77 5.82 21.02 7 14.14 2 9.27 8.91 8.26 12 2"/>
```

---

## Typography reference

AppBoard uses **Outfit** (body) and **JetBrains Mono** (code/labels, rarely used).

Sizes and weights used consistently across the shared components:

| Use | Size | Weight |
|---|---|---|
| Page title (h1) | 26px | 700 |
| Section title (h3) | 16px | 600 |
| Back header title (h2) | 18px | 600 |
| Item title (h4 in rows) | 13px | 600 |
| Category title (h4 in cat-row) | 14px | 600 |
| Detail page title | 20px | 700 |
| Body / description | 13px | 400 |
| Secondary text | 12px | 400–500 |
| Labels (uppercase) | 10–11px | 600 |
| Tab labels | 10px | 500 |
| Source / domain | 10px | 400 |

Letter-spacing: `-0.03em` on large headings (26px+), `-0.02em` on medium (18–20px), `-0.01em` on small headings (14–16px), `0` on body.

---

## Spacing reference

Consistent horizontal padding throughout: **18px** from the phone frame edge.

```
Page horizontal padding:    18px
Section header padding:     22px top, 18px sides, 10px bottom
Label padding:              24px top, 18px sides, 8px bottom
Item row padding:           12px top/bottom, 18px sides
Category row padding:       12px top/bottom, 18px sides
Chip row padding:           0 18px 10px
H-scroll padding:           0 18px 4px
Header padding:             52px top, 18px sides, 14px bottom
```

When in doubt: `18px` horizontal, `12–16px` vertical for interactive rows, `22–24px` for section openers.
