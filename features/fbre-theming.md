---
title: FBRE Theming Guide
applies-to:
  - "@sonata-innovations/fiber-fbre@^4.0"
read-when: "Theming FBRE-rendered forms: colorScheme presets, palette knobs, loading a brand font, raw --fbre-* token overrides, precedence model, full token catalog."
---

# FBRE Theming Guide

How to theme a form rendered by **FBRE** (`@sonata-innovations/fiber-fbre`) — from a one-line accent change to a full custom brand palette, light or dark.

> This is the canonical theming reference. It ships inside the `@sonata-innovations/fiber-fbre` package (`docs/features/fbre-theming.md`), so the copy you are reading matches the version you installed. `fbre/README.md` carries a condensed summary; this document is authoritative.

---

## The mental model

FBRE's appearance is driven entirely by **CSS custom properties** (the `--fbre-*` tokens). Everything you see — every fill, border, label, and state color — resolves from one of these tokens. Theming is the act of supplying values for them.

There are **two independent axes**:

| Axis | Controls | Set via |
| --- | --- | --- |
| **`style`** (FlowStyleType) | the *shape*: borders vs. underlines, fills, spacing, radius rhythm | `theme.style` (e.g. `"clean"`, `"defined-outlined"`) |
| **palette** | the *colors*: ground, surfaces, text, borders, accent, state colors | `colorScheme` + the palette knobs, or raw `--fbre-*` overrides |

`style` and palette are orthogonal: any style works with any palette. This guide is about the **palette** axis. (For the list of styles, see the [Flow Schema](../schema/flow-schema.md#style-types).)

---

## The precedence chain

Palette values resolve through four layers. Each layer overrides the ones above it, so you only specify what you want to change:

```
1.  Light defaults                    base.css  .fbre-container            (lowest)
2.  Preset:  colorScheme: "dark"      base.css  .fbre-container[data-mode="dark"]
3.  Knobs:   theme.{surface,text,…}   inline custom properties on the container
4.  Raw CSS: your own --fbre-* rules   your stylesheet
```

- **Layer 1–2 (presets)** seed the *entire* token set with a coherent light or dark palette. Picking `colorScheme` swaps which one is active.
- **Layer 3 (knobs)** are written as **inline** custom properties on the container, so they beat any stylesheet rule targeting the same element — no selector can out-specify an inline declaration.
- **Layer 4 (raw CSS)** beats the presets and is the escape hatch for every token no knob touches. For a token a knob *has* set, a stylesheet rule on `.fbre-container` loses to the inline knob — to override a knob-set token you need `!important`, or set the token on a **descendant** element so it takes effect lower in the tree.

This is why you can set `colorScheme: "dark"` and then override just `surface` and `text` — the rest of the form stays coherent with the dark preset, and your two knobs take effect on top.

### Where the knob values come from

Layer 3 above is "the effective `theme`", which is itself resolved from three sources before any token is written — the flow's own theme sits in the middle:

```
themeDefaults prop  <  flow.config.theme  <  theme prop
(host house style)     (what the author chose)  (host override)
```

- **`themeDefaults`** applies only where the flow left a key unset, so a flow that deliberately picked a palette keeps it.
- **`theme`** wins outright. Live theme editing depends on this: [`ThemeEditor`](../integration/theme-editor.md) renders a real FBRE and feeds it the in-progress theme, which has to beat the flow's stored one for the preview to show anything.

This resolution is key-level, so the three sources compose rather than replace: `themeDefaults={{ radius: "4px" }}` still applies to a flow whose theme sets only `color`.

---

## 1. Color scheme (presets)

`theme.colorScheme` selects the built-in starter palette. Default is `"light"`.

```tsx
// Via the theme prop (takes precedence over flow config)
<FBRE flow={flow} theme={{ colorScheme: "dark" }} onFlowComplete={done} />

// Via the flow config
const flow = { ...base, config: { ...base.config, theme: { colorScheme: "dark" } } };

// Via the themeDefaults prop (a flow that sets colorScheme itself still wins)
<FBRE flow={flow} themeDefaults={{ colorScheme: "dark" }} onFlowComplete={done} />
```

It applies `data-mode="light" | "dark"` on the `.fbre-container` and seeds all `--fbre-*` tokens with light- or dark-appropriate values.

> **Migration:** `colorScheme` replaces the former `darkMode: boolean`. `darkMode: true` → `colorScheme: "dark"`; `darkMode: false`/absent → `colorScheme: "light"`. There is no runtime fallback — pre-migrate stored flows.

There is intentionally **no `"auto"`** scheme yet (following the viewer's `prefers-color-scheme`). It is an additive future option; today a flow renders one chosen scheme for every viewer.

---

## 2. Palette knobs

Knobs live on `theme` and override individual tokens **on top of** the active preset. Set any subset; unset knobs fall through.

```tsx
<FBRE
  flow={flow}
  theme={{
    colorScheme: "dark",
    color: "#bcd3cd",       // accent
    background: "#1a2a3a",  // form ground
    surface: "#243244",     // input fills / cards
    text: "#e8ede9",
    border: "#2e4158",
    radius: "3px",
    fontFamily: '"Inter", system-ui, sans-serif',
  }}
  onFlowComplete={done}
/>
```

Each knob sets a **primary token** and **derives** the related tokens from it, so you get a coherent result from a single value:

| Knob | Primary token | Also derives |
| --- | --- | --- |
| `color` | `--fbre-theme-color` | `--fbre-theme-light`, `--fbre-theme-dark` |
| `background` | `--fbre-bg` | — |
| `surface` | `--fbre-surface` | `--fbre-surface-hover/-alt/-subtle`, `--fbre-control-surface`, `--fbre-input-bg`, `--fbre-input-bg-focus` |
| `text` | `--fbre-text` | `--fbre-text-secondary/-placeholder/-disabled`, `--fbre-label` |
| `border` | `--fbre-border` | `--fbre-border-hover/-light/-subtle` |
| `radius` | `--fbre-radius` | — |
| `fontFamily` | `--fbre-font` | — (see [Loading a brand font](#loading-a-brand-font)) |
| `error` | `--fbre-error` | `--fbre-error-light` |
| `success` | `--fbre-success` | `--fbre-success-bg`, `--fbre-success-border` |
| `warning` | `--fbre-warning` | `--fbre-warning-bg`, `--fbre-warning-border` |

### Loading a brand font

`fontFamily` is a CSS family *string*. Setting it changes which family the form asks for; it does not make that family available. If no matching `@font-face` is registered in the document, the browser falls back to `system-ui` — quietly, and to something that looks approximately fine, which is why this failure tends to go unreported rather than get fixed.

The host cannot always fix it from outside, either. FBRE is designed to mount inside a **shadow root**, and `@font-face` rules declared inside a shadow root's stylesheet are **not registered at all** — faces resolve at document level only. So the one place a customer-facing form actually runs is the place where injecting a stylesheet the way you inject the FBRE stylesheet (`adoptedStyleSheets`, see the [shadow-root section](../integration/fbre.md#mounting-inside-a-shadow-root)) does not work for fonts.

To have FBRE load the font itself, give `fontFamily` an object instead of a string:

```tsx
<FBRE
  flow={flow}
  theme={{
    fontFamily: {
      family: "Brand Sans",
      src: [{ url: "https://cdn.example.com/brand.woff2", format: "woff2" }],
      faces: [
        { src: [{ url: "https://cdn.example.com/brand-bold.woff2", format: "woff2" }], weight: "700" },
      ],
    },
  }}
  onFlowComplete={done}
/>
```

FBRE registers those faces against the **owning document** of the element it mounted into — which is the host document even from inside a shadow root — and writes `"Brand Sans", system-ui, …` to `--fbre-font`. Because Fiber owns both halves, the family string and the loaded face cannot drift apart. Set `stack` to control the fallback chain explicitly.

Registration is idempotent per document, so several FBRE instances sharing a page register each face once. A face that fails to fetch is reported but never thrown: the form renders in the fallback stack, exactly as it would have before, so a bad font URL cannot take a form down. A later render retries it.

**Calling it yourself.** FBRE does this on mount, which means the first paint can happen before the face is ready. To register earlier — or against a document FBRE has not mounted into yet — call the helper directly:

```ts
import { ensureFontLoaded } from "@sonata-innovations/fiber-fbre";

// Before rendering, or at app boot with the tenant's theme
await ensureFontLoaded(theme, document);
```

It no-ops for a theme with no font, for a plain family string (that font is the host page's job), and where the FontFace API is unavailable. `fontFamilyStack(theme.fontFamily)` returns the stack FBRE writes, if you need to match it elsewhere in your UI.

**A plain string is still correct** when the host page already loads the font — a self-hosted `@font-face` in the app's own CSS, or a Google Fonts `<link>`. Use the object form when the theme is data (a tenant picking their brand font) rather than something the app was built with.

### How derivation works

Knobs are applied as inline custom properties on the container; derived tokens are emitted as `color-mix()` expressions that reference other tokens rather than literal colors — e.g. `--fbre-text-secondary` is roughly `color-mix(in srgb, var(--fbre-text) 64%, var(--fbre-bg))`. Because the inputs are *tokens*, derivations stay **mode-aware**: if you set only `surface`, its derived hover/alt surfaces still mix against the active scheme's `--fbre-bg`/`--fbre-text`. A knob's derived tokens are emitted **only when that knob is set**, so an unset knob leaves the preset's values untouched. The mapping lives in one place: `buildThemeVars()` in `fbre/src/lib/theme-vars.ts`.

> Knobs are *colors and a few scalars*, not shape. To change borders-vs-underlines, fills, or spacing rhythm, choose a different `theme.style` — see the orthogonal-axes note above.

---

## 3. Raw CSS escape hatch

For anything not exposed as a knob — or for surgical control — override the `--fbre-*` tokens directly in your own stylesheet. Scope to your wrapper so you out-specify the engine's defaults regardless of load order:

```css
.my-form-wrapper .fbre-container {
  --fbre-surface: #243244;
  --fbre-star-filled: #bcd3cd;   /* not a knob — set it raw */
  --fbre-shadow-color: rgba(0, 0, 0, 0.4);
}
```

Knobs are applied inline on the container, so a knob and a raw stylesheet rule targeting the *same* token on the same element will see the knob win regardless of selector specificity — override a knob-set token with `!important` or by setting it on a descendant element. In practice: use knobs for the common palette, raw CSS for the long tail of tokens no knob touches.

---

## Token catalog

Every themeable value, with light and dark preset defaults. Tokens marked **(derived)** are produced from a knob when that knob is set; you can also set any of them raw.

### Accent

| Token | Light | Dark | Knob |
| --- | --- | --- | --- |
| `--fbre-theme-color` | `#1976d2` | `#42a5f5` | `color` |
| `--fbre-theme-light` | `#e3f2fd` | `#1a2332` | (derived) |
| `--fbre-theme-dark` | `#1565c0` | `#1e88e5` | (derived) |
| `--fbre-on-primary` | `#fff` | `#fff` | — |

### Ground & surfaces

| Token | Light | Dark | Knob |
| --- | --- | --- | --- |
| `--fbre-bg` | `#fff` | `#121212` | `background` |
| `--fbre-surface` | `#fff` | `#1e1e1e` | `surface` |
| `--fbre-surface-hover` | `#f5f5f5` | `#2a2a2a` | (derived) |
| `--fbre-surface-alt` | `#e0e0e0` | `#333` | (derived) |
| `--fbre-surface-subtle` | `#e8e8e8` | `#2a2a2a` | (derived) |
| `--fbre-control-surface` | `#fff` | `#e0e0e0` | (derived) |
| `--fbre-input-bg` | `var(--fbre-surface)` | `var(--fbre-surface)` | (derived) |
| `--fbre-input-bg-focus` | `var(--fbre-bg)` | `var(--fbre-bg)` | (derived) |

> Note: some style variants override tokens inside their `[data-style]` block: `defined-outlined` overrides `--fbre-input-bg` and `--fbre-label`, `soft-outlined` overrides `--fbre-input-bg`, and `airy-clean` overrides `--fbre-label`; the outlined variants also adjust `--fbre-layout-gap`. A knob still wins over all of these, because knobs are inline. (Their dark-mode blocks reset input-bg/label back to surface-derived values.)

### Text

| Token | Light | Dark | Knob |
| --- | --- | --- | --- |
| `--fbre-text` | `#212121` | `#e0e0e0` | `text` |
| `--fbre-text-secondary` | `#666` | `#9e9e9e` | (derived) |
| `--fbre-text-placeholder` | `#aaa` | `#666` | (derived) |
| `--fbre-text-disabled` | `#bbb` | `#555` | (derived) |
| `--fbre-label` | `var(--fbre-text-secondary)` | `var(--fbre-text-secondary)` | (derived) |

### Borders

| Token | Light | Dark | Knob |
| --- | --- | --- | --- |
| `--fbre-border` | `#ccc` | `#444` | `border` |
| `--fbre-border-hover` | `#999` | `#666` | (derived) |
| `--fbre-border-light` | `#ddd` | `#444` | (derived) |
| `--fbre-border-subtle` | `#eee` | `#333` | (derived) |

### State colors

| Token | Light | Dark | Knob |
| --- | --- | --- | --- |
| `--fbre-error` | `#d32f2f` | `#ef5350` | `error` |
| `--fbre-error-light` | `#fdecea` | `#2c1b1b` | (derived) |
| `--fbre-success` | `#2d6a28` | `#a8e6a0` | `success` |
| `--fbre-success-bg` | `#ddffda` | `#1a3318` | (derived) |
| `--fbre-success-border` | `#b8e6b4` | `#2d5a28` | (derived) |
| `--fbre-warning` | `#7a5900` | `#ffe082` | `warning` |
| `--fbre-warning-bg` | `#fff8e1` | `#332b00` | (derived) |
| `--fbre-warning-border` | `#ffe082` | `#665500` | (derived) |

### Controls

| Token | Light | Dark | Knob |
| --- | --- | --- | --- |
| `--fbre-toggle-track` | `#ccc` | `#555` | — |
| `--fbre-star-empty` | `#ddd` | `#444` | — |
| `--fbre-star-filled` | `#ffc107` | `#ffca28` | — |
| `--fbre-slider-track` | `#ddd` | `#444` | — |
| `--fbre-shadow-color` | `rgba(0,0,0,.12)` | `rgba(0,0,0,.4)` | — |

### Shape & motion

| Token | Default | Knob |
| --- | --- | --- |
| `--fbre-radius` | `4px` | `radius` |
| `--fbre-font` | `"Segoe UI", system-ui, -apple-system, sans-serif` | `fontFamily` |
| `--fbre-layout-gap` | `4px` (varies by style) | — |
| `--fbre-transition-duration` | `250ms` | — |
| `--fbre-transition-easing` | `cubic-bezier(0.4, 0, 0.2, 1)` | — |

The control and motion tokens above have **no dedicated knob** — set them raw if you need to. (Tokens are defined in `fbre/src/styles/base.css`; transition tokens in `transitions.css`.)

---

## Worked example: a dark brand form

Goal: a sage-on-navy intake form using the `defined-outlined` style. Two equivalent ways to express it.

**With knobs (recommended)** — the engine derives the supporting tokens and keeps the dark scheme coherent:

```tsx
<FBRE
  flow={flow}
  theme={{
    style: "defined-outlined",
    colorScheme: "dark",
    color: "#bcd3cd",
    background: "#1a2a3a",
    surface: "#243244",
    text: "#e8ede9",
    border: "#2e4158",
    radius: "3px",
    fontFamily: '"Inter", system-ui, sans-serif',
  }}
  onFlowComplete={done}
/>
```

**With raw CSS** — equivalent for these tokens, and the only option for non-knob tokens (e.g. recoloring rating stars):

```css
.brand .fbre-container {
  --fbre-bg: #1a2a3a;
  --fbre-surface: #243244;
  --fbre-input-bg: #243244;
  --fbre-text: #e8ede9;
  --fbre-text-secondary: #8fa899;
  --fbre-label: #8fa899;
  --fbre-border: #2e4158;
  --fbre-theme-color: #bcd3cd;
  --fbre-radius: 3px;
  --fbre-font: "Inter", system-ui, sans-serif;
}
```

The knob form needs no `data-mode` gymnastics and no per-element selector overrides — input fills, labels, and focus states all follow the tokens.

---

## See also

- [Flow Schema — ThemeConfig](../schema/flow-schema.md) — the `theme` object's fields and types
- [FBRE Integration Guide](../integration/fbre.md) — embedding FBRE in a parent app
- `fbre/README.md` — FBRE props and a condensed theming summary
