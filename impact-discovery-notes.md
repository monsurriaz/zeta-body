# Impact v7.2.0 — Discovery Notes (Zeta Body integration)

> Grounded in the actual theme source. Every integration decision in the build defers to what's recorded here. File/line refs are clickable.

## Theme shape

- **JS:** single bundle [`assets/theme.js`](assets/theme.js) (5,624 lines, not minified). All custom elements are defined here via `customElements.define(...)`. No per-component JS files.
- **CSS:** [`assets/theme.css`](assets/theme.css) (utility + component styles), [`assets/custom.css`](assets/custom.css) (our scoped layer — was empty), plus leftover `assets/gp-global.css` from a removed GemPages builder.
- **Leftover page-builder artifacts to IGNORE (do not touch):** `templates/*.gem-*`, `templates/*.gp-*`, `layout/theme.gempages.*`, `assets/gp-global.css`. Native OS 2.0 `index.json` / `product.json` are the live templates.

---

## Cart

- **Custom element:** `cart-drawer` (defined in `theme.js`; markup in [`sections/cart-drawer.liquid`](sections/cart-drawer.liquid)).
- **How it's opened:** a plain link whose `aria-controls` points at the drawer id. In the header the cart trigger is:
  - [`sections/header.liquid:333-347`](sections/header.liquid#L333) — `<a href="{{ routes.cart_url }}" ... aria-controls="cart-drawer">` (the `aria-controls` is only emitted when `settings.cart_type != 'page'` and not on the cart page). Opening is handled generically by `theme.js` off `aria-controls` → matching drawer/dialog element. **No JS rewiring needed; keep this attribute.**
- **Cart count bubble:** `<cart-count class="count-bubble">` at [`sections/header.liquid:344`](sections/header.liquid#L344). It self-updates on cart change — keep the element as-is.
- **Cart update mechanism (pub/sub):** events live in `theme.js`. Confirmed event names:
  - `cart:change`, `cart:refresh`, `cart:error`, `variant:add`, `variant:change`, `product:rerender`.
  - The cart drawer re-render + open after add is handled inside `theme.js` around lines 1804–1885 (`cart-drawer` logic reacting to `cart:change`). Since we submit through Impact's `<product-form>`, we inherit this for free.

## Product form

- **Custom element:** `product-form` — the form is created in [`snippets/buy-buttons.liquid:32`](snippets/buy-buttons.liquid#L32): `{%- form 'product', product, is: 'product-form', id: form_id -%}`.
  - Submits a hidden `name="id"` input ([`:33-40`](snippets/buy-buttons.liquid#L33)); this input is `disabled` when a `variant_picker` block exists, because the variant picker owns the live `id` input.
  - AJAX add-to-cart + drawer sync is done by the `product-form` element in `theme.js`. **Preferred path (README §5b option 1): render our buy box inside/around this form.**
- **Variant picker:** [`snippets/variant-picker.liquid`](snippets/variant-picker.liquid); emits/consumes `variant:change`. Price/availability/media stay in sync via this event — do NOT duplicate variant JS.
- **`selling_plan` support:**
  - NOT currently rendered in `buy-buttons.liquid` or `main-product.liquid` (no `selling_plan` input on the product form yet).
  - Theme DOES understand selling plans downstream: referenced in [`snippets/line-item.liquid`](snippets/line-item.liquid), [`snippets/product-card.liquid`](snippets/product-card.liquid), `product-card2.liquid`, `horizontal-product.liquid` (line-item shows plan name; cards show subscription price).
  - **Integration:** add a hidden `selling_plan` input to the product form (empty = one-time, plan id = recurring). App-agnostic — any app that syncs plans as native Shopify selling plans works. Client wants the One-time / Subscribe & Save UI regardless of which app; we build the UI and feed the input.

## Search

- **Custom element:** `search-drawer` (+ `predictive-search`), defined in `theme.js`.
- **How it's opened:** same `aria-controls` pattern — [`sections/header.liquid:314`](sections/header.liquid#L314) and `:102`: `<a href="{{ routes.search_url }}" ... aria-controls="search-drawer">`. Keep the attribute; swap only the icon → "Search" text.

## Product media / gallery (client wants: main image + L/R arrows, thumbnails 5–6 + L/R arrows)

- **Component:** [`snippets/product-gallery.liquid`](snippets/product-gallery.liquid) → `<product-gallery>` wrapping `<media-carousel>`.
- **Impact already supports this layout natively** via `section.settings.desktop_media_layout`:
  - `carousel_thumbnails_bottom` → main carousel with a **thumbnail strip below** ([`:102`, `:205-237`](snippets/product-gallery.liquid#L205)). ← matches the client design.
  - `carousel_thumbnails_left` → thumbnails on the left.
- **Arrows:** carousel navigation via `custom-cursor` ([`:71-81`](snippets/product-gallery.liquid#L71)) and `prev-button`/`next-button` custom elements (in `theme.js`). Thumbnail strip is a `<page-dots align-selected>` inside a `scroll-shadow` scroll-area.
- **Plan:** set `desktop_media_layout: carousel_thumbnails_bottom`, add explicit prev/next arrow buttons (`prev-button`/`next-button`) on both the main carousel and the thumbnail strip, and restyle via scoped CSS to the client look. Keep gallery **presentation-only** and independent of cart logic (README §5b); it syncs to variant via `variant:change`.

---

## Design system mapping (client V3 → Impact)

- **Impact color model:** RGB triples consumed as `rgb(var(--text-color) / .12)`. Color schemes defined in [`config/settings_data.json`](config/settings_data.json); fonts via `--heading-font-family` / `--text-font-family` (theme settings `heading_font` / `body_font`).
- **Client tokens** (canonical, from `Zeta-design-token.html`):

  | token | hex | role |
  |---|---|---|
  | `--bg` | `#F4F4F5` | page bg, buy box surface |
  | `--surface` | `#FFFFFF` | cards, tiles, hero panel |
  | `--ink` | `#0A1320` | text, dark bands, primary CTA |
  | `--teal` | `#006E73` | brand accent, italic emphasis, active |
  | `--bronze` | `#B89968` | warm accent, badges, stars |
  | `--sand` | `#EFE9DC` | soft section bg, badge bg |
  | `--mid` | `#6B7280` | secondary text |
  | `--border` | `#E2E0DC` | borders, dividers |
  | (home only) `--dark-surface` | `#111B2A` | dark product cards / pillars |

- **Type:** Cormorant Garamond (display headings, italic teal emphasis) + Inter (UI/body, weights 300–700). No third font. Teal italic = one emphasis moment per headline.
- **Approach:** namespace/scope client CSS under a `.zeta` wrapper on our sections + a `--zeta-*` token layer in `custom.css`; reuse Impact color-scheme vars where a section maps cleanly. Never override Impact component internals (cart drawer, search, product-form, variant-picker).

## Canonical header/footer decision

- Homepage HTML says *"Skip the header, footer of Home page. Use the header, footer of Zeta Product Page."* → **canonical chrome = the product-page nav + footer**:
  - Header: left wordmark, centered nav links, right `Account` + `Cart (0)` — already **text triggers** (matches README §2).
  - Footer: `Z E T A` wordmark, italic tagline, Health/Wealth/Youth trinity, Shop / Learn / Inner-Circle columns, legal (FDA disclaimer).

## Section schema conventions (mirror these)

- `main-product` is **block-based** ([`sections/main-product.liquid`](sections/main-product.liquid), 2,142 lines): blocks incl. `vendor`, `title`, `badges`, `price`, `variant_picker`, `description`, `tabs`, buy buttons. We adapt via blocks, not a rebuild.
- Home sections use color-scheme + section-padding settings and `presets` — mirror for editor-native feel.
