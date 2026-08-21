---
title: Style Families & Advance Behaviors
applies-to:
  - "@sonata-innovations/fiber-fbre@^4.0"
  - "@sonata-innovations/fiber-types@^3.0"
  - "@sonata-innovations/fiber-fbt@^3.0"
  - "@sonata-innovations/fiber-fbtl@^3.0"
  - "@sonata-innovations/fiber-theme-editor@^2.0"
read-when: "Choosing a form style, understanding the focused presentation, or turning auto-advance / Enter-to-advance on and off. Also: migrating off config.mode."
---

# Style Families & Advance Behaviors

A Fiber form's presentation is decided by one setting — `config.theme.style` —
and its two advance behaviors by two more — `config.navigation.autoAdvance` and
`config.navigation.advanceOnEnter`. That is the whole model. There is no
presentation mode.

> **Migrating from an earlier version?** `config.mode` (`"standard"` /
> `"conversational"`) was removed. Jump to [Migrating off `config.mode`](#migrating-off-configmode).

## The ten styles are one flat vocabulary

Any style is valid on any flow. Four of them share an extra presentation
treatment, and that shared half is called a **family**:

| Family | Styles |
| --- | --- |
| `"form"` | `clean`, `outlined`, `refined-clean`, `airy-clean`, `soft-outlined`, `defined-outlined` |
| `"focused"` | `centered-minimal`, `stacked-cards`, `soft-float`, `bold-statement` |

On top of whichever of the four you pick, the focused family adds:

| Treatment | What it does |
| --- | --- |
| **Vertical centering** | Content sits centered in a narrow column, falling back to top-aligned and scrollable when a screen is taller than the viewport |
| **Animated entry** | Components fade and scale in with staggered delays. Respects `prefers-reduced-motion` |
| **Larger tap targets** | Option items, Yes/No buttons, card-select cards and inputs are enlarged |
| **Bolder type** | Headers, labels, prompts and inputs step up in size |

Each of the four styles then applies its own look — underline inputs, filled
cards, pill options, heavy borders — exactly as each of the six form styles
does.

### The family is derived, never authored

It does not appear in Flow JSON. FBRE computes it from `theme.style` and emits
it on the container as `data-style-family`, next to `data-style`:

```html
<div class="fbre-container" data-style="soft-float" data-style-family="focused">
```

That is what lets the ~32 shared rules live under one selector instead of a
repeated four-way list. If you write custom CSS against a focused form, target
`[data-style-family="focused"]` for anything that should apply to all four and
`[data-style="…"]` for one style.

`fiber-types` (and `fiber-fbre`, which re-exports it) publishes the derivation
so a builder can group its own style picker from one list:

```ts
import { FOCUSED_STYLES, styleFamily } from "@sonata-innovations/fiber-types";

styleFamily("bold-statement"); // "focused"
styleFamily("clean");          // "form"
styleFamily(undefined);        // "form"  — the `clean` default

FOCUSED_STYLES.has("soft-float"); // true
```

### Why this matters for server-driven rendering

The session protocol sends `config.theme`. Because presentation is *derived
from* `theme.style`, a focused-styled flow renders identically whether it is
rendered locally, fetched remotely, or served one screen at a time by a
session — no extra field has to be carried, and none can be forgotten.

## Advance behaviors

Auto-advance and Enter-to-advance are independent `navigation` flags. Neither is
tied to a style: a `clean` form with one question per screen advances on Enter,
and a `bold-statement` flow only auto-advances if it asks to.

| Flag | Default | Behavior |
| --- | --- | --- |
| `navigation.autoAdvance` | `false` | Advance to the next screen ~500ms after a single-select choice — `radio`, `yesNo`, `cardSelect`, `dropDown`. Multi-select (`checkbox`, `dropDownMulti`) never auto-advances |
| `navigation.advanceOnEnter` | `true` | Advance when Enter is pressed in an `inputText` or `inputNumber`. `inputTextArea` is excluded — Enter inserts a newline there |

```jsonc
{
  "config": {
    "theme": { "style": "centered-minimal" },
    "navigation": {
      "transition": "scaleFade",
      "autoAdvance": true,
      "advanceOnEnter": true
    }
  }
}
```

The defaults are deliberately asymmetric. A form that moves without a click is
surprising, and shifting content and focus ~500ms after a selection carries a
real accessibility cost — so `autoAdvance` is opt-in. Enter-to-advance is an
ordinary form convention, so it is on.

### Guards

Both behaviors share the same guards. None of them are configurable — they are
correctness guards, not policy:

- Neither fires on a screen with **more than one visible input**. A shared
  screen behaves like a normal form, so components after the first are never
  skipped. (Conditionally hidden inputs don't count toward the total.)
- Neither fires on the last screen — so Enter can never submit the form.
- Neither fires when the screen fails validation.
- Both respect condition-hidden screens, skipping to the next reachable one.
- Auto-advance does not fire during an active screen transition.
- Enter-to-advance stands down inside an open popup (`.fbre-popup`), so Enter
  there commits the value rather than skipping the screen. Today this affects
  one component: the colour picker's hex field is the only `<input>` rendered
  inside an overlay. The date/time pickers and both dropdowns use buttons and
  `role="option"` elements, which Enter-to-advance already ignores.

The single-visible-input guard is worth designing around: turning `autoAdvance`
on for a flow whose screens hold several questions each does nothing at all.
Both builders say so — FBT warns on a multi-input screen tab, FBTL notes it on
the divider that merges two cards onto one screen.

## In the builders

Neither builder stores a mode. Each offers a preset that writes the settings the
focused look is made of, after which every one of them stays individually
editable:

- **FBT** — a *Focused presentation* preset in the Appearance section writes
  `theme.style: "centered-minimal"` and `navigation: { transition: "scaleFade",
  autoAdvance: true }`. The style picker lists all ten styles, captioned *Form*
  and *Focused*.
- **FBTL** — its default config already uses `centered-minimal` with
  `autoAdvance: true`. Pacing (one question per screen vs all on one page) is a
  separate concern there: see `screenModel` in the
  [FBTL Integration Guide](../integration/fbtl.md#screen-model).
- **Theme Editor** — the style dropdown lists all ten, grouped under *Form* and
  *Focused* headings. Its value is `{ theme }`.

## Migrating off `config.mode`

`FlowConfiguration.mode`, the `FlowModeType` type, and FBRE's `mode` prop are
gone. Each of `mode`'s jobs moved somewhere it belongs:

| Was | Now |
| --- | --- |
| `mode: "conversational"` for the centered, animated presentation | the focused **style** already on the flow — nothing to set |
| `mode: "conversational"` for auto-advance | `navigation.autoAdvance: true` |
| `mode: "conversational"` for Enter-to-advance | `navigation.advanceOnEnter` — on by default now, for every flow |
| `mode` deciding which styles were legal | nothing — the vocabulary is flat |

### Updating a stored flow

A flow that had `mode: "conversational"` already carried a focused
`theme.style`, so **its look survives untouched**. The only behavior that
changes is auto-advance, which is now off unless asked for:

```jsonc
// before
{ "config": {
    "mode": "conversational",
    "theme": { "style": "centered-minimal" },
    "navigation": { "transition": "scaleFade" }
} }

// after
{ "config": {
    "theme": { "style": "centered-minimal" },
    "navigation": { "transition": "scaleFade", "autoAdvance": true }
} }
```

A flow that had `mode: "standard"` (or no `mode` at all) needs only the field
deleted. It gains Enter-to-advance on its one-question screens, which was
previously withheld for no reason anyone could see.

`mode` is not read anywhere any more, so leaving a stale `"mode"` key in stored
JSON is inert rather than harmful — but the schema no longer describes it, and
`npm run validate` will not vouch for it.

### Updating code

```diff
-<FBRE flow={flow} mode="conversational" onFlowComplete={done} />
+<FBRE
+  flow={flow}
+  theme={{ style: "centered-minimal" }}
+  navigation={{ autoAdvance: true }}
+  onFlowComplete={done}
+/>
```

```diff
-<ThemeEditor mode={value.mode} theme={value.theme} onChange={setValue} />
+<ThemeEditor theme={value.theme} onChange={setValue} />
```

The theme editor's `STANDARD_STYLE_OPTIONS`, `CONVERSATIONAL_STYLE_OPTIONS`,
`styleOptionsForMode` and `reconcileStyle` exports are replaced by a single
`STYLE_OPTIONS` array; use `styleFamily()` from `fiber-types` if you need the
partition.

## Related

- [Flow Schema → Style Families](../schema/flow-schema.md#style-families)
- [FBRE Theming Guide](fbre-theming.md) — palette tokens, brand fonts, and how a
  style's preset interacts with the knobs
- [FBRE Integration Guide → Focused Presentation](../integration/fbre.md#focused-presentation)
