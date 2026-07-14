# Impact Theme Integration Brief — Client Design → Shopify-Functional

> **Purpose:** Convert the client's static HTML/CSS/JS designs (Header, Footer, Home, Product pages) into a working Shopify store built on the **Impact theme (Maestrooo)**. The client design is the **visual layer**; Impact remains the **functional engine** (cart drawer, search drawer, product form, AJAX cart). Do not rebuild what Impact already does — reuse it.

---

## 0. Golden Rules (read before writing any code)

1. **Impact is the engine. Never break it.** Do not delete or disable Impact's cart drawer, search drawer, product-form, pub/sub, or cart AJAX. Layer on top of them.
2. **Discover before you build.** The full Impact theme is in this project. Read the real source to find exact custom-element tags, event names, and payloads. **Never guess or hardcode event names from memory** — they vary between Impact versions.
3. **Reuse over reinvent — customize before you replace.** Prefer wrapping the client's markup *around* Impact's existing components over writing new logic (especially the product form). **Decision rule for every section:** if an Impact section can be adapted to match the client's design at reasonable cost, adapt it in place — you inherit its wiring for free. Build a separate/new section only when the customization cost clearly outweighs building fresh. Default to customizing; replace when it's genuinely cheaper or cleaner.
4. **Everything is merchant-editable.** Every converted section ships with a full `{% schema %}` — settings + blocks — so it's editable in the theme editor. No hardcoded copy/images that should be settings.
5. **Scope the client CSS.** The client's stylesheet must not clobber Impact's design tokens or component styles. Namespace/scope it and reuse Impact's CSS variables where they exist.
6. **One section per design block.** Keep sections small, composable, and OS 2.0-native.

---

## 1. Discovery Phase (do this FIRST, output a short notes file)

Before building, inspect the Impact source and record findings in `impact-discovery-notes.md`. Answer each of these **from the actual code**, with file paths and line references:

### Cart
- [ ] Cart drawer custom element tag + defining file (e.g. `assets/*.js`, `snippets/`/`sections/` for markup).
- [ ] How the cart drawer is **opened** (custom DOM event? element method? attribute/`is=` hook? a global like `theme.cart`?).
- [ ] The **cart-update mechanism**: is there a pub/sub utility (e.g. `publish`/`subscribe` + an events map)? What is the exact event name fired after an item is added? What payload does it expect?
- [ ] How the drawer **re-renders** after add (Shopify Section Rendering API — which `sections=` / `section_id`s does Impact request on `/cart/add.js` or `/cart.js`?).
- [ ] The cart line-item count / bubble element that needs updating.

### Product form
- [ ] Impact's `<product-form>` (or equivalent) custom element — its file, the input names it submits, and how it calls the AJAX cart.
- [ ] The variant picker component and how it updates variant id / price / availability.
- [ ] Whether the form already supports a `selling_plan` input (subscriptions) — search for `selling_plan`.

### Search
- [ ] Search drawer custom element + how it's opened/closed (event/attribute/method).
- [ ] Predictive search component, if any.

### Design system
- [ ] CSS custom properties / design tokens (colors, spacing, type scale) and where they're defined.
- [ ] Breakpoints and any container/grid utilities.
- [ ] Section schema conventions Impact uses (naming, `enabled_on`/`disabled_on`, common settings) — mirror these.

> **Rule:** Every integration instruction below defers to what you find here. If reality differs from this brief, trust the source and note the deviation.

---

## 2. Header

**Goal:** Match the client design by **modifying Impact's existing header in place** — the two are nearly identical, so customize rather than replace (Golden Rule 3). This keeps all of Impact's cart/search trigger wiring intact for free.

- **Do not build a new header section.** Edit Impact's existing header markup/CSS to match the client design.
- **Primary change: icons → text labels.** The client uses text triggers ("Cart", "Search") instead of Impact's icons. Swap the icon markup for text labels **but keep the same trigger element, attributes, and handlers** Impact already uses — change the label, not the wiring. Open behavior then keeps working with zero rewiring.
- Keep Impact's `cart-drawer` and `search` drawer intact and mounted; you're only restyling their triggers.
- Adjust layout, spacing, and typography to the client design using Impact's tokens where possible.
- Navigation stays **dynamic** via the existing menu (`linklists`) binding — restyle, don't rebuild.
- Verify the cart-count bubble still updates (it should, since you kept Impact's element).
- Only if a specific header element genuinely can't be adapted, isolate just that piece — don't replace the whole section.

**Acceptance:** Header matches the client design with text "Cart"/"Search" triggers; clicking them opens Impact's cart/search drawers with no rewiring; nav still editable; cart count still updates.

---

## 3. Home Page

**Goal:** One editable section per design block.

- Split the client's home HTML/CSS into discrete OS 2.0 sections (hero, featured collection, banners, testimonials, etc.).
- Each section: full `{% schema %}` with **settings + blocks** (fully editable). Repeatable content → blocks with `{% for block in section.blocks %}` and `{{ block.shopify_attributes }}`.
- Use Shopify objects where content is dynamic (collections, products, images via `image_url`/`image_tag`, metafields if relevant) instead of static assets.
- Reuse Impact's color-scheme / section-padding settings pattern so sections feel native in the editor.
- Add `{% schema %}` `presets` so sections are addable from the editor.

**Acceptance:** Every home block is a section, addable/removable/reorderable in the editor, all copy/images are settings.

---

## 4. Footer

**Goal:** Direct replacement with the client's footer.

- Replace Impact's footer with the client's design.
- Keep it **editable**: menus via `linklists`, newsletter form via Shopify's `{% form 'customer' %}`, social links / payment icons / copyright as settings or blocks.
- If the client footer drops a newsletter, wire it to Shopify's real customer form (don't fake it).

**Acceptance:** Footer matches design, menus + newsletter are functional and editable.

---

## 5. Product Page

Two categories of section on this page.

### 5a. Non-functional sections (description, cross-sells, tabs, trust badges, etc.)
- Straight HTML/CSS → Liquid section conversion, same editable-schema standard as home (settings + blocks).
- Pull dynamic data from the `product` object / metafields where the design implies real content.

### 5b. Product Info section (the functional one) — **highest care**

Layout per client design: **left column = media slider**, **right column = product info** (title, price, short description, variant selectors, quantity, **subscription UI: One-time / Recurring**, add-to-cart).

**Add-to-cart strategy — in priority order:**

1. **Preferred: wrap Impact's existing `<product-form>` component.** Keep Impact's product-form element and its inputs; restyle and re-lay-out the surrounding markup to match the client design. This gives you AJAX add-to-cart **and** cart-drawer sync for free, because you're using Impact's own path.
2. **Only if the design truly can't accommodate it:** build a custom form, but it must still:
   - POST to Shopify's AJAX cart the same way Impact does (same endpoint + the `sections` param Impact uses to re-render the drawer, from Phase 1).
   - After a successful add, **publish Impact's own cart-update event** (exact name from Phase 1) so the drawer refreshes and opens. Do not roll your own drawer.
   - Update the cart-count bubble via Impact's mechanism.
   - Handle errors (sold out, quantity limits) using Impact's patterns.

**Media slider (left column):**
- Prefer reusing Impact's media-gallery/slider component styled to the client design.
- If building custom, keep it **fully independent of cart logic** (presentation only) so it can't interfere with add-to-cart sync. Sync it to variant changes (variant image) using the variant-change event from Phase 1.

**Variant selection / price:**
- Reuse Impact's variant picker + variant-change events so price, availability, media, and the selected `id` stay correct. Don't duplicate variant JS.

**Acceptance:** Selecting variants updates price/media/availability; add-to-cart opens and updates Impact's cart drawer; no second cart UI exists; works on mobile.

---

## 6. Subscriptions — Recharge (One-time / Recurring UI)

Store uses **Recharge**. First confirm it runs **Recharge's Shopify Checkout Integration (SCI)** — the modern setup where Recharge syncs subscription plans into Shopify as native **selling plans**. On SCI the theme-side work is clean and app-agnostic: read `product.selling_plan_groups` and submit a `selling_plan` id.

**Recommended path (SCI + selling plans):**
- Render One-time / Recurring options from `product.selling_plan_groups` (frequencies come from the plans). Do not hardcode frequencies.
- The toggle sets a hidden **`selling_plan`** input: empty for one-time, the chosen plan id for recurring.
- Submit it through Impact's `<product-form>` (§5b option 1) — it rides through Impact's add-to-cart and Recharge picks it up at checkout. No custom cart logic needed.
- Update the displayed price from the selected plan's `selling_plan_allocation` price/discount.
- Confirm Impact's **cart drawer shows the plan name/frequency** on subscription line items (native line-item data: `line_item.selling_plan_allocation.selling_plan.name`). If the client design requires it and the drawer omits it, add it to the drawer's line-item template.

**Verify / fallback:**
- If the store is NOT on SCI (legacy Recharge with its own cart/checkout), stop and confirm the intended integration — that path uses Recharge's widget/SDK and line-item properties instead of selling plans and changes the cart flow entirely.
- Do not mix the two models.

**Acceptance:** Recurring adds a subscription line item with the correct Recharge selling plan, shows correctly in Impact's cart drawer, and carries through to Recharge at checkout; one-time behaves as a normal add.

---

## 7. CSS & Assets

- Reuse Impact's design tokens/CSS variables where the client design maps onto them; introduce client tokens only where needed.
- **Scope client CSS** so selectors can't override Impact component internals (cart drawer, search, product-form). Prefer section-scoped styles / a namespace class on the client sections.
- Follow the project's existing asset conventions (from the current CLAUDE.md / file structure). Keep JS additive — attach behavior without re-binding or overriding Impact's component lifecycles.
- No inline `<script>` that duplicates Impact utilities; import/extend Impact's modules where possible.

---

## 8. Do NOT

- ❌ Remove or disable Impact's cart drawer, search drawer, product-form, pub/sub, or cart AJAX.
- ❌ Build a second cart UI or a parallel add-to-cart that bypasses Impact's drawer sync.
- ❌ Hardcode event names, element tags, or `section_id`s from memory — always confirm in source.
- ❌ Hardcode text/images that a merchant should edit — use schema.
- ❌ Let client CSS/JS leak into and override Impact component internals.

---

## 9. Suggested Build Order

1. Discovery notes (§1).
2. Header — wire triggers to Impact drawers, dynamic nav (§2).
3. Footer (§4).
4. Home sections (§3).
5. Product non-functional sections (§5a).
6. Product Info section (§5b) — start by wrapping Impact's product-form.
7. Subscriptions layer (§6).
8. CSS scoping pass + QA (§7).

---

## 10. Final Acceptance Checklist

- [ ] Cart icon (new header) opens Impact's cart drawer.
- [ ] Search icon (new header) opens Impact's search drawer.
- [ ] Custom product form adds to cart **and** the Impact drawer refreshes + opens.
- [ ] Variant change updates price, availability, and media.
- [ ] Subscription toggle produces correct one-time vs recurring line items through to checkout.
- [ ] All sections (header, footer, home, product) are editable in the theme editor.
- [ ] Nav + footer menus driven by Shopify linklists.
- [ ] No console errors; no duplicated Impact markup; client CSS doesn't break Impact components.
- [ ] Mobile + desktop verified.