---
title: Theme Editor Integration Guide
applies-to:
  - "@sonata-innovations/fiber-theme-editor@^2.0"
read-when: "Embedding the plug-and-play theming widget: controlled value contract, knob sets, defaults, modal pattern, passing the theme to FBRE/FBT/FBTL."
---

# Theme Editor — Theming Widget

A self-contained reference for integrating `@sonata-innovations/fiber-theme-editor`
into a parent application. Code-first, complete API, copy-paste patterns.
No prior Fiber knowledge assumed.

## Quick Facts

- **Package:** `@sonata-innovations/fiber-theme-editor` (public npm, scoped).
- **What it is:** a controlled React widget for editing a Fiber form's theme —
  light/dark scheme, accent color, style, and every palette token — beside a
  **live FBRE preview** that renders a real form with the chosen theme.
- **Controlled value:** `{ theme }` (see _Contract_). You own the state.
- **Renders real FBRE** for its preview, so it shares your app's FBRE instance
  via peer dependencies (it does not bundle its own).
- **Two stylesheets required:** the editor's and FBRE's (see _Styles_).

### Installation

```bash
npm install @sonata-innovations/fiber-theme-editor
# peer dependencies (install if you don't already have them):
npm install react react-dom @sonata-innovations/fiber-fbre @sonata-innovations/fiber-types
```

Peer dependency ranges: `react`/`react-dom` `^18 || ^19`,
`@sonata-innovations/fiber-fbre` `^3.2.1`, `@sonata-innovations/fiber-types`
`^2.0.0`.

### Styles

Import **both** stylesheets once, anywhere in your app's entry path. The editor
chrome needs its own CSS; the preview renders FBRE and needs FBRE's CSS:

```ts
import "@sonata-innovations/fiber-theme-editor/styles";
import "@sonata-innovations/fiber-fbre/styles";
```

Forgetting `fbre/styles` is the most common mistake — the controls render fine
but the preview pane looks unstyled.

## Contract

The editor edits the **theme slice of a Fiber flow's configuration**. Its value
is `{ theme }`:

```ts
type ThemeEditorValue = {
  theme: ThemeConfig; // the palette / scheme / style knobs
};
```

`theme` lives on `FlowConfiguration.theme`. After editing, splice the result back
into your flow:

```ts
flow.config.theme = value.theme;
```

> **Upgrading from 1.x.** The value and the props no longer carry `mode` —
> `FlowConfiguration.mode` no longer exists. Drop the `mode` prop, drop
> `value.mode`, and stop writing `flow.config.mode`. The style a flow already
> uses carries its own presentation. See
> [Style Families](../features/style-families.md).

`ThemeConfig` is sparse — set only the knobs you want to override; unset knobs
fall through to the `colorScheme`/`style` preset that FBRE applies at render:

```ts
type ThemeConfig = {
  colorScheme?: "light" | "dark";
  style?: FlowStyleType; // see Style options below
  color?: string; // accent (CSS color)
  background?: string;
  surface?: string;
  text?: string;
  border?: string;
  radius?: string; // CSS length, e.g. "8px"
  fontFamily?: string | FontFamilyConfig; // CSS stack, or a font FBRE loads itself
  error?: string;
  success?: string;
  warning?: string;
};
```

## Minimal Setup

The widget is fully controlled — hold `{ theme }` in your own state and pass
`onChange` straight through:

```tsx
import { useState } from "react";
import {
  ThemeEditor,
  type ThemeEditorValue,
} from "@sonata-innovations/fiber-theme-editor";
import "@sonata-innovations/fiber-theme-editor/styles";
import "@sonata-innovations/fiber-fbre/styles";

function ThemePanel() {
  const [value, setValue] = useState<ThemeEditorValue>({
    theme: { colorScheme: "light" },
  });

  return <ThemeEditor theme={value.theme} onChange={setValue} />;
}
```

## Props Reference

| Prop          | Type                               | Default          | Notes                                                                               |
| ------------- | ---------------------------------- | ---------------- | ----------------------------------------------------------------------------------- |
| `theme`       | `ThemeConfig`                      | —                | Required. Sparse; unset knobs fall through to the preset.                           |
| `onChange`    | `(next: ThemeEditorValue) => void` | —                | Required. Fires on every change with the full `{ theme }`.                          |
| `defaultTab`  | `"settings" \| "palette"`          | `"settings"`     | Which pane is open on mount.                                                        |
| `knobs`       | `ThemeKnob[]`                      | full set         | Which palette-tab knobs to expose. Use `FBTL_KNOBS` / `FBT_KNOBS`.                   |
| `defaults`    | `ThemeConfig`                      | FBRE light/dark  | Baseline shown as placeholder/swatch for unset knobs, and merged under the preview. |
| `previewFlow` | `Flow`                             | built-in sink    | The flow rendered in the preview.                                                   |
| `className`   | `string`                           | —                | Extra class on the root element.                                                    |

### Knob sets

- `FBTL_KNOBS` — `background, surface, text, border, radius, fontFamily, error`.
- `FBT_KNOBS` — the above plus `success, warning` (callout-only colors).

The font knob is a plain text field: it edits a CSS stack, not a font file. If
the theme it is editing carries a loadable font (`fontFamily` as
`{ family, src }` — see [Loading a brand font](../features/fbre-theming.md#loading-a-brand-font)),
the field shows the stack that config resolves to, and editing it rewrites the
stack while **keeping the sources**. Clearing the knob drops the whole key,
sources included.

Pick the set that matches what your preview can actually show. Note the
**built-in preview adapts**: when `knobs` includes `success`/`warning`, the
default preview flow automatically includes a callout screen, so `FBT_KNOBS`
does not require a custom `previewFlow`. The guidance only bites with a custom
`previewFlow` that has no callouts — there, use `FBTL_KNOBS` so
`success`/`warning` aren't dead controls.

### The `defaults` prop

`theme` is sparse; the editor needs to know what each unset knob falls through to
in order to show it as a `"#rrggbb (preset)"` placeholder and swatch color. By
default it uses FBRE's built-in light/dark token values (chosen by
`theme.colorScheme`). Pass `defaults` to override that baseline — e.g. your brand
palette. The values you pass are also merged **under** `theme` in the preview, so
a knob you haven't overridden renders at your baseline. The emitted value stays
sparse — `defaults` is a display + preview baseline, never written into `theme`.

### Style options

All ten styles are always selectable — the vocabulary is flat. The dropdown
groups them by family purely for legibility:

- **Form:** `clean`, `outlined`, `refined-clean`, `airy-clean`, `soft-outlined`,
  `defined-outlined`
- **Focused:** `centered-minimal`, `stacked-cards`, `soft-float`,
  `bold-statement`

The focused four add a centered narrow column, animated entry and larger targets
on top of their own look. Nothing gates or reconciles the list any more: the
exported `STYLE_OPTIONS` array is exactly what the widget renders, and
`styleFamily()` from `fiber-types` is the source of truth for the grouping. See
[Style Families](../features/style-families.md).

## Integration Pattern — "Change theme" button + modal

This is the canonical pattern (used by the FBTL playground): a header button
opens the editor in a modal, and the result is passed into whatever renders/owns
the flow. Keep the theme in parent state and feed it both to the editor and to
the consumer of the theme.

```tsx
import { useState } from "react";
import {
  ThemeEditor,
  FBTL_KNOBS,
  type ThemeEditorValue,
} from "@sonata-innovations/fiber-theme-editor";
import { FBTL } from "@sonata-innovations/fiber-fbtl";
import "@sonata-innovations/fiber-theme-editor/styles";
import "@sonata-innovations/fiber-fbre/styles";

function Editor({ flow, onFlowChange }) {
  const [themeOpen, setThemeOpen] = useState(false);
  const [themeValue, setThemeValue] = useState<ThemeEditorValue>({
    theme: { colorScheme: "light", style: "centered-minimal" },
  });

  return (
    <>
      <header>
        <button onClick={() => setThemeOpen(true)}>Change theme</button>
      </header>

      {/* Pass the edited theme into the builder/renderer. */}
      <FBTL flow={flow} onChange={onFlowChange} theme={themeValue.theme} />

      {themeOpen && (
        <div className="overlay" onClick={() => setThemeOpen(false)}>
          <div onClick={(e) => e.stopPropagation()} role="dialog">
            <button onClick={() => setThemeOpen(false)}>Done</button>
            <ThemeEditor
              theme={themeValue.theme}
              onChange={setThemeValue}
              knobs={FBTL_KNOBS}
            />
          </div>
        </div>
      )}
    </>
  );
}
```

Notes from this reference integration:

- **Persist the theme in the flow.** The theme belongs in `flow.config`. When you
  save/emit, merge it in: `{ ...emittedFlow, config: { ...emittedFlow.config, theme: themeValue.theme } }`.
- **FBTL previews the flow's own style.** Its preview pane honours whatever
  `theme.style` the flow carries, so the editor and the preview agree by
  construction. FBTL's own defaults fill in `centered-minimal`, so defaulting the
  editor to a focused style keeps the two in step.
- **The editor and the consumer share FBRE.** Because FBRE is a peer dependency,
  the editor's preview and your `<FBTL>`/`<FBRE>` use the same instance — no
  duplication, no version drift, provided you load `fbre/styles` once.

## Passing the theme into FBRE / FBTL / FBT

```tsx
// Render the live form with the chosen theme (live override):
<FBRE flow={flow} theme={themeValue.theme} />

// FBTL also accepts a ThemeConfig for its preview pane:
<FBTL flow={flow} onChange={setFlow} theme={themeValue.theme} />
```

For FBRE the precedence is: a `theme` prop knob wins over `flow.config.theme`,
which wins over FBRE's built-in preset. So you can either pass `theme` as a prop
(live override) or write it into `flow.config.theme` (persisted) — or both.

**FBT is different**: it has **no `theme: ThemeConfig` prop** (its only theming
prop is `themeColor`, a single accent color for the builder chrome itself), and
its change callback is `onFlowChange`, not `onChange`. To theme an FBT-built
flow, persist the editor's output into the flow config:

```tsx
const themedFlow = {
  ...flow,
  config: { ...flow.config, theme: themeValue.theme },
};
```

## Public Exports

```ts
// Component
export { ThemeEditor };

// Knob sets + metadata
export { FBTL_KNOBS, FBT_KNOBS, KNOB_DEFS };

// Built-in default values (the "(preset)" hints)
export { LIGHT_THEME_DEFAULTS, DARK_THEME_DEFAULTS, resolveThemeDefaults };

// In-widget glossary copy
export { buildGlossary, SETTINGS_GLOSSARY, KNOB_GLOSSARY };

// The style dropdown's option list (all ten, in picker order)
export { STYLE_OPTIONS };

// Default preview flow
export { buildPreviewFlow, defaultPreviewFlow };

// Types
export type {
  ThemeEditorProps,
  ThemeEditorValue,
  ThemeEditorTab,
  ThemeKnob,
  GlossaryEntry,
  KnobDef,
  KnobKind,
  StyleOption,
};
```

## Common Pitfalls

1. **Missing `fbre/styles`** — preview renders unstyled. Import it once.
2. **Treating the editor as uncontrolled** — it has no internal theme state; if
   you don't feed `onChange` back into `theme`, nothing updates.
3. **Peer deps not installed** — `fiber-fbre` and `fiber-types` must be present
   in the consuming app; they are peers, not bundled.
4. **Expecting `theme` to be fully populated** — it's sparse. Read effective
   values from the preview/render, not from `theme` (unset knobs are absent).
5. **Wrong knob set for the preview** — exposing `success`/`warning` with a
   preview that has no callouts gives dead controls; use `FBTL_KNOBS` there.
6. **Still passing `mode`** — the prop and the `mode` half of the value were
   removed in 2.0 along with `FlowConfiguration.mode`. TypeScript flags the
   extra prop; the fix is to delete it, not to reintroduce the field.
