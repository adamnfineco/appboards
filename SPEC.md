# AppBoard — Component Specification

Complete reference for building prototypes in AppBoard. Every component, every pattern, every convention. Written for LLMs building new prototypes.

---

## File structure

`index.html` is self-contained. Everything lives inside it:

```
<head>
  <style>
    /* 1. Reset */
    /* 2. Design tokens (:root) */
    /* 3. Desktop shell (sidebar + phone frame) */
    /* 4. SHARED MOBILE COMPONENTS — .m-* classes */
    /* 5. Detail page styles */
    /* 6. Add modal styles */
    /* 7. VARIANT-SPECIFIC STYLES — add yours here, prefix with variant ID */
  </style>
</head>
<body>
  <div class="desktop-shell">
    <div class="sidebar"> ... </div>
    <div class="phone-stage">
      <div class="phone-frame">
        <div class="variant active" id="variant-v1"> ... </div>
        <div class="variant" id="variant-v2"> ... </div>
      </div>
    </div>
  </div>
  <script>
    /* DATA */
    /* CTA HELPERS */
    /* NAVIGATION */
    /* DETAIL RENDERERS */
  </script>
</body>
```

---

## Design tokens

All tokens are CSS custom properties on `:root`. Reference them in any inline style or variant CSS.

```css
--bg:             #fafafa    /* page background */
--surface:        #ffffff    /* card/sheet background */
--text-primary:   #18181b    /* main text */
--text-secondary: #71717a    /* secondary text */
--text-tertiary:  #a1a1aa    /* placeholder, labels */
--border:         #e4e4e7    /* visible borders */
--border-subtle:  #f4f4f5    /* very light dividers */
--accent:         #059669    /* brand green — override for your project */
--accent-soft:    #ecfdf5    /* light accent background */
--accent-text:    #047857    /* accent text color */
--sidebar-w:      240px
--phone-w:        390px      /* iPhone 14 logical width */
--phone-h:        844px      /* iPhone 14 logical height */
--tab-bar-h:      80px
--radius-sm:      8px
--radius-md:      14px
--radius-lg:      20px
--radius-xl:      28px
--transition-fast: 180ms cubic-bezier(0.16, 1, 0.3, 1)
--transition-med:  320ms cubic-bezier(0.16, 1, 0.3, 1)
```

---

## Prototype structure (variant)

Every prototype is a `<div class="variant">` inside `.phone-frame`. Only one is visible at a time.

```html
<div class="variant active" id="variant-v1">

  <!-- PAGES — one per tab, identified by {variant}-{name} -->
  <div class="m-page active" id="v1-home"> ... </div>
  <div class="m-page" id="v1-library"> ... </div>

  <!-- SUBPAGES — drill-in detail views -->
  <div class="m-subpage" id="v1-detail"> ... </div>

  <!-- ADD MODAL (optional) -->
  <div class="add-overlay" id="v1-add-overlay" onclick="closeAdd('v1')">
    <div class="add-sheet" onclick="event.stopPropagation()"> ... </div>
  </div>

  <!-- TAB BAR (if needed) -->
  <div class="m-tab-bar"> ... </div>

  <!-- FAB (if needed) -->
  <button class="m-fab" onclick="openAdd('v1')"> ... </button>

</div>
```

**Rules:**
- `id` always follows pattern `variant-{id}` for the outer div
- Pages follow `{variant}-{pagename}` — e.g. `v1-home`, `v1-library`
- Subpages follow `{variant}-{name}` — e.g. `v1-listing-detail`
- First page gets `class="m-page active"`, rest just `class="m-page"`
- First tab button gets `class="m-tab active"`, rest just `class="m-tab"`

---

## Layout components

### `.m-page`

Top-level scrollable content area for a tab. Fades in from slightly below when activated.

```html
<div class="m-page active" id="v1-home">
  <!-- content -->
  <div style="height:20px"></div>  <!-- bottom breathing room -->
</div>
```

- `position: absolute; inset: 0; bottom: var(--tab-bar-h)` — sits above the tab bar
- If no tab bar, override: `style="bottom:0"`
- Add `<div style="height:100px"></div>` at the bottom when there's a FAB or sticky bar

---

### `.m-subpage`

Drill-in detail page that slides in from the right. Lives at `z-index: 100`, above pages.

```html
<div class="m-subpage" id="v1-detail">
  <!-- full-screen content -->
</div>
```

Trigger:
```js
openSubpage('v1-detail')   // activate
closeSubpage('v1-detail')  // deactivate
```

Always include a back button at the top of the subpage:
```html
<button onclick="closeSubpage('v1-detail')" style="position:absolute;top:48px;left:14px; ...">
  <svg viewBox="0 0 24 24"><path d="M15 18l-6-6 6-6"/></svg>
</button>
```
Or use `.detail-hero-back` for an image-overlay back button (see Detail page styles).

---

### `.m-tab-bar` + `.m-tab`

iOS-style frosted-glass tab bar, fixed to the bottom of the variant.

```html
<div class="m-tab-bar">
  <button class="m-tab active" onclick="switchMTab('v1','home',this)">
    <svg viewBox="0 0 24 24"><!-- icon --></svg>
    <span>Home</span>
  </button>
  <button class="m-tab" onclick="switchMTab('v1','library',this)">
    <svg viewBox="0 0 24 24"><!-- icon --></svg>
    <span>Library</span>
  </button>
</div>
```

- SVG icons: `width/height` not needed — CSS sets `22×22`, `stroke: var(--text-tertiary)`, `fill: none`
- Active tab: `stroke: var(--text-primary)` + label color updated automatically via `.m-tab.active`
- Use 3–5 tabs. Don't exceed 5.

---

### `.m-fab`

Floating action button, bottom-right, above the tab bar.

```html
<button class="m-fab" onclick="openAdd('v1')">
  <svg viewBox="0 0 24 24">
    <line x1="12" y1="5" x2="12" y2="19"/>
    <line x1="5" y1="12" x2="19" y2="12"/>
  </svg>
</button>
```

Position: `bottom: calc(var(--tab-bar-h) + 12px); right: 16px`

---

## Header components

### `.m-header`

Top of a page. Large title + optional right element.

```html
<div class="m-header">
  <h1>Home</h1>
  <div class="m-avatar"><span>M</span></div>
</div>
```

Padding: `52px 18px 14px` (accounts for status bar area).

---

### `.m-back-header`

Header for subpages or pushed screens. Back button + title.

```html
<div class="m-back-header">
  <button class="m-back-btn" onclick="closeSubpage('v1-detail')">
    <svg viewBox="0 0 24 24"><path d="M15 18l-6-6 6-6"/></svg>
  </button>
  <h2>Detail Title</h2>
</div>
```

---

### `.m-section-header`

Section title row, optionally with a right-side link.

```html
<div class="m-section-header">
  <h3>Section Title</h3>
  <button class="m-section-link">See all</button>
</div>
```

---

### `.m-label`

Uppercase micro-label above a content block.

```html
<div class="m-label">Coming up</div>
```

---

### `.m-avatar`

30px circle with initials. Use in headers.

```html
<div class="m-avatar"><span>M</span></div>
```

---

## List components

### `.m-item-row`

Standard list item: 64×64 thumbnail + title + meta row.

```html
<div class="m-item-row" onclick="...">
  <div class="m-item-img">
    <img src="https://picsum.photos/seed/example/128/128" alt="">
  </div>
  <div class="m-item-body">
    <h4>Item Title</h4>
    <div class="m-item-meta">
      <span class="m-intent" data-i="buy">buy</span>
      <span class="m-source">source.com</span>
    </div>
  </div>
</div>
<div class="m-divider"></div>
```

Always follow with `.m-divider` (except the last row).

---

### `.m-cat-row`

Category list row: 50×50 thumbnail + title + subtitle + chevron.

```html
<div class="m-cat-row" onclick="...">
  <div class="m-cat-thumb">
    <img src="https://picsum.photos/seed/cat/100/100" alt="">
  </div>
  <div class="m-cat-info">
    <h4>Category Name</h4>
    <p>42 items</p>
  </div>
  <svg class="m-cat-arrow" viewBox="0 0 24 24"><path d="M9 6l6 6-6 6"/></svg>
</div>
<div class="m-divider"></div>
```

---

### `.m-col-card`

Collection card for horizontal scroll rows. Colored dot + name + count.

```html
<div class="m-col-card" onclick="...">
  <div class="m-col-dot" style="background: #3b82f6"></div>
  <h4>Collection Name</h4>
  <p>47 items</p>
</div>
```

Width: `136px` fixed. Always inside `.m-h-scroll`.

---

## Scroll & filter components

### `.m-h-scroll`

Horizontal scroll container. Hides the scrollbar. Use for cards, chips, anything that scrolls sideways.

```html
<div class="m-h-scroll stagger">
  <div class="m-col-card"> ... </div>
  <div class="m-col-card"> ... </div>
  <div class="m-h-scroll-end"></div>  <!-- REQUIRED: prevents trailing bleed -->
</div>
```

**Always end with `<div class="m-h-scroll-end"></div>`** — browsers clip `padding-right` on overflow scroll containers. The spacer div prevents the last card from bleeding to the frame edge.

---

### `.m-chips` + `.m-chip`

Horizontal filter pill row.

```html
<div class="m-chips">
  <span class="m-chip active" onclick="toggleChip(this)">All</span>
  <span class="m-chip" onclick="toggleChip(this)">Filter A</span>
  <span class="m-chip" onclick="toggleChip(this)">Filter B</span>
</div>
```

`toggleChip(el)` deactivates all siblings, activates the clicked one.

---

## Utility components

### `.m-divider`

1px horizontal rule. Use between list rows.

```html
<div class="m-divider"></div>
```

---

### `.m-source`

Small grey text for domain/attribution labels.

```html
<span class="m-source">example.com</span>
```

---

### `.m-intent[data-i="..."]`

Colored intent badge. 14 types with distinct background/text color pairs.

```html
<span class="m-intent" data-i="buy">buy</span>
<span class="m-intent" data-i="cook">cook</span>
```

Available intents: `buy` `cook` `go` `create` `eat` `read` `watch` `listen` `do` `learn` `hire` `book` `gift` `inspire`

---

## Animation

### Stagger fade-up

Add `class="stagger"` to any flex container. Children animate in sequentially on load.

```html
<div class="m-h-scroll stagger">
  <!-- children fade up with 50ms delay between each -->
</div>
```

Supports up to 8 children with stagger delays (0ms → 350ms). Works on any flex container, not just scroll rows.

---

## Add modal (bottom sheet)

A form sheet that slides up from the bottom. Triggered by FAB or any button.

```html
<div class="add-overlay" id="v1-add-overlay" onclick="closeAdd('v1')">
  <div class="add-sheet" onclick="event.stopPropagation()">
    <div class="add-handle"></div>
    <h3>Sheet Title</h3>

    <div class="add-field">
      <label>Field Label</label>
      <input type="text" placeholder="...">
    </div>

    <div class="add-field">
      <label>Notes</label>
      <textarea rows="2" placeholder="..."></textarea>
    </div>

    <div class="add-field">
      <label>Tags (optional)</label>
      <div class="m-chips" style="padding:0;margin-top:4px;">
        <span class="m-chip" onclick="this.classList.toggle('active')">Tag A</span>
      </div>
    </div>

    <button class="add-submit">Save</button>
  </div>
</div>
```

JS: `openAdd('v1')` / `closeAdd('v1')`

The overlay click-outside-to-dismiss is already wired via `onclick="closeAdd('v1')"` on `.add-overlay`. The sheet's `event.stopPropagation()` prevents tap-through.

---

## Detail page styles

For item/content detail subpages with a hero image at top.

```html
<div class="m-subpage" id="v1-item-detail">
  <div class="detail-hero">
    <img src="https://picsum.photos/seed/item/780/400" alt="">
    <button class="detail-hero-back" onclick="closeSubpage('v1-item-detail')">
      <svg viewBox="0 0 24 24"><path d="M15 18l-6-6 6-6"/></svg>
    </button>
  </div>
  <div class="detail-body">
    <div class="detail-title">Item Title</div>
    <div class="detail-meta-row">
      <span class="m-intent" data-i="buy">buy</span>
      <span class="m-source">source.com</span>
    </div>
    <button class="detail-cta">
      <svg viewBox="0 0 24 24"><!-- icon --></svg>
      Open Link
    </button>
    <div class="detail-comment">"A note about this item."</div>
    <div class="detail-section-label">Details</div>
    <div class="detail-meta-grid">
      <div class="detail-meta-cell"><dt>Label</dt><dd>Value</dd></div>
      <div class="detail-meta-cell"><dt>Label</dt><dd>Value</dd></div>
    </div>
  </div>
</div>
```

Classes: `.detail-hero` · `.detail-hero-back` · `.detail-body` · `.detail-title` · `.detail-meta-row` · `.detail-cta` · `.detail-comment` · `.detail-section-label` · `.detail-meta-grid` · `.detail-meta-cell`

For list-style detail pages (category/collection): `.list-detail-header` · `.list-detail-title` · `.list-detail-dot`

---

## Navigation JS reference

```js
// Switch the active prototype in the sidebar
switchVariant('v2', el)

// Switch tab within a prototype (closes any open subpages first)
switchMTab('v1', 'home', el)

// Open/close a subpage
openSubpage('v1-detail')
closeSubpage('v1-detail')

// Open/close the add modal bottom sheet
openAdd('v1')
closeAdd('v1')

// Deactivate all .m-chip siblings, activate clicked one
toggleChip(el)
```

---

## Data objects (optional)

If building content-heavy prototypes with dynamic detail pages, populate these at the top of `<script>`:

```js
const ITEMS = {
  'key': {
    title: 'Item Title',
    source: 'domain.com',
    intent: 'buy',          // one of 14 intent types
    type: 'product',        // product | recipe | article | event | place | video
    format: 'link',         // link | video | audio | image | pdf
    img: 'seed-word',       // picsum seed — used as picsum.photos/seed/{img}/...
    comment: 'A note.',
    mentions: ['Name'],
    collections: [{ name: 'Collection Name', color: '#hex' }],
    category: 'Category Name',
    savedBy: 'Person Name',
    date: 'Feb 18, 2026',
  }
};

const CATEGORIES = {
  'key': {
    name: 'Category Name',
    count: 42,
    img: 'seed-word',
    subs: ['Sub A', 'Sub B'],
    items: ['item-key-1', 'item-key-2'],
  }
};

const COLLECTIONS = {
  'key': {
    name: 'Collection Name',
    color: '#3b82f6',
    count: 12,
    items: ['item-key-1'],
  }
};
```

Then use the detail renderers:
```js
openItemDetail('v1', 'item-key')         // → populates + opens #v1-item-detail
openCategoryDetail('v1', 'category-key') // → populates + opens #v1-category-detail
openCollectionDetail('v1', 'col-key')    // → populates + opens #v1-collection-detail
```

These subpage containers must exist inside your variant:
```html
<div class="m-subpage" id="v1-item-detail"></div>
<div class="m-subpage" id="v1-category-detail"></div>
<div class="m-subpage" id="v1-collection-detail"></div>
```

---

## Images

Use `picsum.photos` throughout. Consistent seeds give consistent images across reloads.

```
https://picsum.photos/seed/{word}/{width}/{height}
```

Common sizes used in AppBoard:
- Item thumbnails: `128/128`
- Category thumbnails: `100/100`
- Hero images: `780/400`
- Listing cards: `440/320`
- Host avatars: `88/88`
- Full-width hero: `780/520`

Pick descriptive seeds that reflect the content — `apt-brooklyn`, `cat-recipe`, `host-sarah`. Same word always produces the same image.

---

## Conventions summary

| Thing | Convention |
|---|---|
| Prototype ID | `v1`, `v2`, `v3` … |
| Variant outer div | `id="variant-{id}"` |
| Page div | `id="{id}-{pagename}"` e.g. `v1-home` |
| Subpage div | `id="{id}-{name}"` e.g. `v1-listing-detail` |
| Variant CSS prefix | `.{id}-classname` e.g. `.v1-card` |
| Sidebar item | `onclick="switchVariant('{id}', this)"` |
| About panel | `class="sidebar-about-{id}"` |
| Tab switching | `onclick="switchMTab('{id}','{pagename}',this)"` |
| Horizontal scroll | Always end with `<div class="m-h-scroll-end"></div>` |
| Bottom breathing room | `<div style="height:20px"></div>` at end of `.m-page` content; `100px` if FAB or sticky bar is present |
| Status bar offset | `.m-header` has `padding-top: 52px` built in — use it as the first element on every page |

---

## Extending components

Three levels of extension — in order of preference:

### 1. Inline style overrides

For one-off adjustments to shared components:

```html
<div class="m-item-img" style="width:80px;height:80px;">...</div>
<div class="m-section-header" style="padding-top:8px;">...</div>
```

### 2. Variant-specific classes

For patterns that repeat within a prototype. Add in the variant CSS section, prefix with variant id:

```css
/* ── v2: Listing prototype ── */
.v2-price-tag { font-size:13px; font-weight:700; }
.v2-rating { display:flex; align-items:center; gap:3px; font-size:12px; }
```

### 3. Promote to shared `.m-*`

When a pattern appears across 2+ prototypes and is genuinely generic, promote it to the `SHARED MOBILE COMPONENTS` CSS section and add it to this spec.

**Promote when:** truly generic, used in 2+ prototypes, matches the existing visual language.
**Don't promote:** brand-specific widgets, domain-specific components, anything with a concept name baked in.

---

## Typography

Font: **Outfit** (all weights). Fallback: `system-ui, sans-serif`.

| Use | Size | Weight | Letter-spacing |
|---|---|---|---|
| Page title `h1` | 26px | 700 | -0.03em |
| Detail title | 20px | 700 | -0.02em |
| Back header `h2` | 18px | 600 | -0.02em |
| Section header `h3` | 16px | 600 | -0.01em |
| Category row `h4` | 14px | 600 | -0.01em |
| Item row `h4` | 13px | 600 | -0.01em |
| Body text | 13px | 400 | 0 |
| Secondary | 12px | 400–500 | 0 |
| Tab labels | 10px | 500 | 0 |
| Uppercase micro-labels | 10–11px | 600 | 0.06–0.08em |
| Source/domain | 10px | 400 | 0 |

---

## Spacing

Horizontal padding from phone edge: always **18px**.

| Context | Value |
|---|---|
| Page horizontal padding | 18px |
| `.m-header` padding | 52px top · 18px sides · 14px bottom |
| `.m-section-header` | 22px top · 18px sides · 10px bottom |
| `.m-label` | 24px top · 18px sides · 8px bottom |
| Item / category row | 12px top/bottom · 18px sides |
| Chip row | 0 18px 10px |
| H-scroll | 0 18px 4px |
| Bottom breathing room | 20px (no FAB) · 100px (FAB or sticky bar) |

---

## SVG icons

All icons: `viewBox="0 0 24 24"`, `fill="none"`, stroke color via CSS or inline. Stroke width: 1.6–2.2. Size set by CSS, not SVG attributes.

Style: Feather Icons / Heroicons outline. Avoid filled icons.

Common paths:

```
Home:      <path d="M3 9.5L12 3l9 6.5V20a1 1 0 01-1 1H4a1 1 0 01-1-1V9.5z"/>
Search:    <circle cx="11" cy="11" r="7"/><path d="M21 21l-4.35-4.35"/>
Back:      <path d="M15 18l-6-6 6-6"/>
Chevron:   <path d="M9 6l6 6-6 6"/>
Plus:      <line x1="12" y1="5" x2="12" y2="19"/><line x1="5" y1="12" x2="19" y2="12"/>
Heart:     <path d="M20.84 4.61a5.5 5.5 0 00-7.78 0L12 5.67l-1.06-1.06a5.5 5.5 0 00-7.78 7.78l1.06 1.06L12 21.23l7.78-7.78 1.06-1.06a5.5 5.5 0 000-7.78z"/>
User:      <path d="M20 21v-2a4 4 0 00-4-4H8a4 4 0 00-4 4v2"/><circle cx="12" cy="7" r="4"/>
Map pin:   <path d="M21 10c0 7-9 13-9 13s-9-6-9-13a9 9 0 0118 0z"/><circle cx="12" cy="10" r="3"/>
Star:      <polygon points="12 2 15.09 8.26 22 9.27 17 14.14 18.18 21.02 12 17.77 5.82 21.02 7 14.14 2 9.27 8.91 8.26 12 2"/>
```

---

## Common gotchas

| Problem | Cause | Fix |
|---|---|---|
| Last card bleeds to phone edge | Browser clips `padding-right` on overflow scroll | End `.m-h-scroll` with `<div class="m-h-scroll-end"></div>` |
| Content hidden behind tab bar | No bottom breathing room | End `.m-page` content with `<div style="height:20px">` (or `100px`) |
| Navigation silently broken | Wrong page ID format | `id="{variant}-{pagename}"` must match `switchMTab('{variant}','{pagename}',this)` |
| Styles bleeding between prototypes | Unprefixed CSS class | Always prefix variant CSS: `.v1-card`, `.v2-hero` |
| Z-index conflicts | Reusing reserved z-index values | Pages:0 · Subpages:100 · Tab bar:200 · Add overlay:300 — don't use these values elsewhere |
| Content too close to top | Missing status bar offset | Use `.m-header` as first element (has 52px top padding built in) |
| Chips behave as multi-select | Using `toggleChip` for multi-select | For multi-select: `this.classList.toggle('active')` instead |
| Sticky bar floats mid-screen in subpage | `position:sticky` doesn't work without overflow | Use `position:absolute; bottom:0; left:0; right:0` on bottom bars inside `.m-subpage`, and add `padding-bottom:100px` to the scrollable body content above it |
