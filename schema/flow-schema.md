---
title: Flow JSON Schema Reference
applies-to:
  - "@sonata-innovations/fiber-types@^2.2"
read-when: "Exhaustive per-property reference for Flow JSON: every component type, property, condition, validation rule, calculation, and config field. For a compact overview use flow-quick-reference.md."
---

# Fiber JSON Schema

> Canonical source: [`flow-schema.json`](flow-schema.json)
>
> **TypeScript shape**: as of `@sonata-innovations/fiber-types` v2, `Component` is a discriminated union keyed on `type` — `component.properties` narrows to the matching `Component*Properties` shape after a `type` check. The Flow JSON described here is unchanged.

The Flow JSON is the interchange format between FBT (the builder UI) and FBRE (the render engine). FBT authors it; FBRE consumes it and renders the form.

---

## Flow (root)

| Field      | Type                                    | Required | Description                    |
| ---------- | --------------------------------------- | -------- | ------------------------------ |
| `uuid`     | `string`                                | Yes      | Unique identifier for the flow |
| `metadata` | [FlowMetadata](#flowmetadata)           | Yes      | Descriptive metadata           |
| `config`   | [FlowConfiguration](#flowconfiguration) | No       | Runtime configuration          |
| `screens`       | [FlowScreen](#flowscreen)[]             | Yes      | Ordered list of screens        |
| `calculations`  | [FlowCalculation](#flowcalculation)[]   | No       | Top-level calculations         |

---

## FlowMetadata

Descriptive metadata for the flow. Accepts additional string properties beyond the ones listed.

| Field         | Type     | Required | Description             |
| ------------- | -------- | -------- | ----------------------- |
| `name`        | `string` | No       | Name of the flow        |
| `description` | `string` | No       | Description of the flow |

---

## FlowConfiguration

Runtime configuration for the flow, organized into semantic groups.

| Field        | Type               | Required | Description                          |
| ------------ | ------------------ | -------- | ------------------------------------ |
| `mode`       | `FlowModeType`     | No       | Form presentation mode. See [Conversational Mode](#conversational-mode) |
| `theme`      | `ThemeConfig`      | No       | Visual theme settings                |
| `navigation` | `NavigationConfig` | No       | Screen navigation settings           |
| `controls`   | `ControlsConfig`   | No       | Navigation controls settings         |
| `summary`    | `boolean`          | No       | Show a summary screen before completion |
| `confirmation` | `ConfirmationConfig` | No     | Terminal thank-you screen shown after submission |

### ThemeConfig

The theme has two layers: `colorScheme` + `style` pick built-in presets (the seed palette and the shape), and the palette knobs override individual `--fbre-*` tokens on top. Any subset of knobs may be set; unset knobs fall through to the preset. Consumers needing finer control can still override the raw `--fbre-*` CSS custom properties directly.

| Field        | Type            | Required | Description                                                                 |
| ------------ | --------------- | -------- | --------------------------------------------------------------------------- |
| `color`      | `string`        | No       | Accent / primary color (`--fbre-theme-color`)                               |
| `colorScheme`| `"light" \| "dark"` | No   | Built-in palette preset that seeds the token layer. Default `"light"`. Replaces the former `darkMode` boolean |
| `style`      | `FlowStyleType` | No       | Visual style for form elements. See [Style Types](#style-types)             |
| `background` | `string`        | No       | Page/form ground (`--fbre-bg`)                                              |
| `surface`    | `string`        | No       | Raised surface: input fills, cards, popups (`--fbre-surface`)               |
| `text`       | `string`        | No       | Primary text color (`--fbre-text`); derives secondary/placeholder/label    |
| `border`     | `string`        | No       | Border/rule color (`--fbre-border`); derives hover/light/subtle            |
| `radius`     | `string`        | No       | Corner radius, any CSS length (`--fbre-radius`), e.g. `"3px"`               |
| `fontFamily` | `string \| FontFamilyConfig` | No | Font family (`--fbre-font`). See [FontFamilyConfig](#fontfamilyconfig)   |
| `error`      | `string`        | No       | Error state color (`--fbre-error`)                                          |
| `success`    | `string`        | No       | Success state color (`--fbre-success`)                                      |
| `warning`    | `string`        | No       | Warning state color (`--fbre-warning`)                                      |

#### FontFamilyConfig

A plain string is a CSS stack and nothing more: the form renders in that family only if the host page already loaded it, and falls back to `system-ui` otherwise — silently, and to something that looks approximately fine. Passing an object instead gives the renderer the sources, and FBRE registers the faces in the owning document itself.

That matters most where the host cannot patch around it. FBRE is designed to mount inside a shadow root, and `@font-face` rules declared inside a shadow root's stylesheet are never registered — faces resolve at document level only.

| Field    | Type               | Required | Description                                                        |
| -------- | ------------------ | -------- | ------------------------------------------------------------------ |
| `family` | `string`           | Yes      | CSS family name, used for both the `@font-face` and the token value |
| `src`    | `FontSource[]`     | No       | Shorthand for a single regular face. Merged with `faces`            |
| `faces`  | `FontFaceConfig[]` | No       | Additional faces — a second weight, an italic                       |
| `stack`  | `string`           | No       | Full stack written to `--fbre-font`. Defaults to the family plus a system fallback |

`FontSource` is `{ url, format? }`; `FontFaceConfig` is `{ src, weight?, style?, display?, unicodeRange? }`.

```json
{
  "fontFamily": {
    "family": "Brand Sans",
    "src": [{ "url": "https://cdn.example.com/brand.woff2", "format": "woff2" }],
    "faces": [
      { "src": [{ "url": "https://cdn.example.com/brand-bold.woff2", "format": "woff2" }], "weight": "700" }
    ]
  }
}
```

See [FBRE Theming Guide → Loading a brand font](../features/fbre-theming.md#loading-a-brand-font).

#### Style Types

**Standard mode styles** (6):

| Value | Description |
| --- | --- |
| `"clean"` | Bottom-border inputs with uppercase labels (default) |
| `"outlined"` | Full-border inputs with normal-case labels |
| `"refined-clean"` | Animated underline focus with left-accent groups |
| `"airy-clean"` | Spacious layout with tinted focus and pill buttons |
| `"soft-outlined"` | Full-border 8px radius with shadow-ring focus |
| `"defined-outlined"` | Filled-background inputs with top-accent groups |

**Conversational mode styles** (4):

| Value | Description |
| --- | --- |
| `"centered-minimal"` | Thin underline inputs, 1px bordered option cards (6px radius), uppercase 12px labels, theme-tinted hover/selected |
| `"stacked-cards"` | Filled background cards with left accent bar, keyboard shortcut badges (A, B, C, D) on options |
| `"soft-float"` | Pill-shaped options (24px radius) with shadow lift on hover, rounded inputs and buttons |
| `"bold-statement"` | 2px borders, 700-weight 24px headers, inverted selection (dark fill + white text), filled input backgrounds |

Each style has a default stepper visual (see `stepperStyle`). When switching form mode in FBT, the style auto-switches to the first style of the target mode.

### NavigationConfig

| Field                    | Type                   | Required | Description                                                                 |
| ------------------------ | ---------------------- | -------- | --------------------------------------------------------------------------- |
| `transition`             | `ScreenTransitionType` | No       | Screen transition animation type. See [Screen Transitions](#screen-transitions) |
| `allowInvalidTransition` | `boolean`              | No       | Allow navigating forward even when the screen has validation errors         |

### ControlsConfig

| Field        | Type                       | Required | Description                                                                                                           |
| ------------ | -------------------------- | -------- | --------------------------------------------------------------------------------------------------------------------- |
| `show`       | `boolean`                  | No       | Show built-in next/back navigation buttons. Default `true`                                                            |
| `layout`     | `"default"` \| `"centered"` \| `"inline-full"` \| `"stacked"` | No       | Layout for the navigation controls. `"default"` = side-by-side grid (back-left / stepper-center / next-right); single-button rows collapse to full-width. `"centered"` = stepper row above, buttons centered as a group. `"inline-full"` = stepper row above, buttons side-by-side at 50% each (full-width when solo). `"stacked"` = stepper, Back, Next, all stacked full-width |
| `showStepper`| `boolean`                  | No       | Show the step indicator dots in the controls bar. Default `true`                                                      |
| `stepperStyle`| `"default"` \| `"dots"` \| `"pill"` \| `"glow"` \| `"bar"` \| `"text"` | No | Style of the step indicator. `"default"` = use the form style's default stepper. `"dots"` = standard circles, scale on active. `"pill"` = active dot stretches to pill shape. `"glow"` = active dot gets a glow ring. `"bar"` = track + counter. `"text"` = "Step X of Y" above screen content. Default `"default"` |

### ConfirmationConfig

A terminal "thank you" screen shown after the flow is submitted (once `onFlowComplete` resolves). It is presentation-only — not a data-collection `Screen` — so it lives on the config. In local and remote modes the screen renders only when `show` is not `false` **and** `title` or `body` has content; server-driven mode falls back to a generic "Thank you" when nothing is configured, and renders nothing on an explicit `show: false` (see [Per-mode behavior](../features/confirmation-screen.md#per-mode-behavior)). When shown, it replaces the final screen and the navigation controls/stepper are hidden.

`title` and `body` support the same `${...}` reference markup as display components — references resolve against collected field values, calculations, and external `context` values (by name). A parent application can also override the configured message at runtime by returning (or resolving with) a `{ title?, body? }` object from `onFlowComplete` — useful for post-submit data such as a server-generated reference number.

| Field   | Type      | Required | Description                                                        |
| ------- | --------- | -------- | ------------------------------------------------------------------ |
| `show`  | `boolean` | No       | Explicit off-switch. Content presence is the positive gate         |
| `title` | `string`  | No       | Heading text. Supports `${...}` references and formatting          |
| `body`  | `string`  | No       | Body text. Supports `${...}` references and formatting             |

---

## Conversational Mode

Set `config.mode` to `"conversational"` to transform FBRE into a one-question-per-screen experience optimized for completion rates.

**`FlowModeType`**: `"standard"` | `"conversational"` (default: `"standard"`)

### Behaviors

| Behavior | Description |
| --- | --- |
| **Vertical centering** | Content is vertically and horizontally centered within the viewport |
| **Auto-advance** | Single-select components (`radio`, `yesNo`, `cardSelect`, `dropDown`) advance to the next screen ~500ms after selection. Multi-select (`checkbox`, `dropDownMulti`) does NOT auto-advance |
| **Enter-to-advance** | Pressing Enter on `inputText` / `inputNumber` advances to the next screen. `inputTextArea` is excluded (Enter inserts newlines) |
| **Animated entry** | Components fade + scale in with staggered delays on screen transitions. Respects `prefers-reduced-motion` |
| **Larger tap targets** | Yes/No buttons, option items, card-select cards, and input fields are enlarged for easier tapping |

### Conversational styles

Conversational mode has 4 dedicated styles (separate from the 6 standard styles):

| Style | Personality |
| --- | --- |
| `centered-minimal` | Thin underline inputs, bordered option cards, uppercase labels, theme-tinted hover/selected |
| `stacked-cards` | Filled background cards with left accent bar, keyboard shortcut badges (A, B, C, D) on options |
| `soft-float` | Pill-shaped options with shadow lift on hover, rounded inputs and buttons |
| `bold-statement` | Heavy borders, bold typography, inverted selection (dark fill + white text) |

When switching to conversational mode in FBT, the style auto-switches to `"centered-minimal"` and the transition to `"scaleFade"`.

### Guards

- Auto-advance does **not** fire on the last screen
- Auto-advance does **not** fire if the screen fails validation
- Auto-advance respects condition-hidden screens (skips them)
- Auto-advance does **not** fire during an active transition
- Enter-to-advance validates the screen before advancing

---

## FlowScreen

A single screen (page) in the flow.

| Field             | Type                                        | Required | Description                                                                  |
| ----------------- | ------------------------------------------- | -------- | ---------------------------------------------------------------------------- |
| `uuid`            | `string`                                    | Yes      | Unique identifier for the screen                                             |
| `label`           | `string`                                    | No       | Display label shown in navigation                                            |
| `components`      | [Component](#component)[]                   | Yes      | Components on this screen                                                    |
| `conditions`      | [FlowConditionConfig](#flowconditionconfig) | No       | Condition for showing/hiding the screen                                      |
| `nextButtonLabel` | `string`                                    | No       | Custom label for the Next/Done navigation button. Overrides the default text |
| `backButtonLabel` | `string`                                    | No       | Custom label for the Back navigation button. Overrides the default text      |
| `valid`           | `boolean`                                   | No       | _Runtime only._ Do not set in authored JSON                                  |

---

## Component

A single form component. Components are recursive — `group` components contain child components.

| Field             | Type                                        | Required | Description                                                    |
| ----------------- | ------------------------------------------- | -------- | -------------------------------------------------------------- |
| `uuid`            | `string`                                    | Yes      | Unique identifier for the component                            |
| `type`            | `string` (enum — see [Component Types](#component-types)) | Yes      | Component type key. In TypeScript, narrowing on this discriminator types `properties`. |
| `properties`      | [ComponentProperties](#componentproperties) | Yes      | Type-specific properties. TS-side: each `type` maps to a matching `Component*Properties` variant. |
| `conditions`      | [FlowConditionConfig](#flowconditionconfig) | No       | Condition for showing/hiding the component                     |
| `components`      | [Component](#component)[]                   | No       | Child component templates (used by `group` and `repeater`)     |
| `value`           | _any_                                       | No       | _Runtime only._ Current value                                  |
| `addedComponents` | [Component](#component)[][]                 | No       | _Runtime only._ Group/repeater iterations. Do not set in authored JSON |
| `valid`           | `boolean`                                   | No       | _Runtime only._ Do not set in authored JSON                    |
| `display`         | `object`                                    | No       | _Runtime only._ Display overrides. Do not set in authored JSON |

### Component Types

| Type            | Category    | Description                                                               |
| --------------- | ----------- | ------------------------------------------------------------------------- |
| `header`        | Display     | Section heading                                                           |
| `text`          | Display     | Static text block                                                         |
| `divider`       | Display     | Visual separator with optional label                                      |
| `callout`       | Display     | Styled alert/info box with variant coloring and optional icon             |
| `table`         | Display     | Static comparison/data table with optional column highlighting            |
| `inputText`     | Input       | Single-line text input                                                    |
| `inputTextArea` | Input       | Multi-line text input                                                     |
| `inputNumber`   | Input       | Numeric input (supports optional decimal restriction and start adornment) |
| `dropDown`      | Selection   | Single-select dropdown                                                    |
| `dropDownMulti` | Selection   | Multi-select dropdown                                                     |
| `checkbox`      | Selection   | Checkbox group                                                            |
| `radio`         | Selection   | Radio button group                                                        |
| `toggleSwitch`  | Selection   | Boolean toggle                                                            |
| `yesNo`         | Selection   | Two large tappable buttons for binary yes/no selection                    |
| `confirm`       | Selection   | Single checkbox for consent / opt-in (value is `true` when ticked, cleared when unticked) |
| `date`          | Date & Time | Calendar popup date picker                                                |
| `time`          | Date & Time | Hour/minute/AM-PM time selector                                           |
| `dateTime`      | Date & Time | Combined calendar + time picker                                           |
| `dateRange`     | Date & Time | Two calendar pickers for start/end dates                                  |
| `timeRange`     | Date & Time | Two time selectors for start/end times                                    |
| `dateTimeRange` | Date & Time | Two datetime pickers for start/end                                        |
| `fileUpload`    | Interactive | File upload                                                               |
| `rating`        | Interactive | Star rating                                                               |
| `slider`        | Interactive | Range slider                                                              |
| `colorPicker`   | Interactive | Saturation/hue picker with hex input and optional swatches                |
| `cardSelect`    | Selection   | Card-based single-select with optional price, features, and badge         |
| `group`         | Container   | Groups child components visually (fieldset + condition grouping)          |
| `repeater`      | Container   | Repeatable container — users can add/remove iterations of child components |
| `computed`      | Computed    | Displays a formula-evaluated result. Inside a repeater, evaluates per-iteration |
| `signature`     | Interactive | Captures a hand-drawn or typed signature. Stores base64 PNG (draw) or text (type) |

---

## ComponentProperties

Properties vary by component type. All fields are optional; which ones are relevant depends on the component `type`.

### Common Properties

| Field         | Type             | Applicable Types          | Description                                                                                                                          |
| ------------- | ---------------- | ------------------------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| `label`       | `string`         | All                       | Display label                                                                                                                        |
| `required`    | `boolean`        | All input/selection types | Field must have a value to pass validation                                                                                           |
| `placeholder` | `string`         | Text inputs               | Placeholder text when empty (dropdowns render a fixed built-in placeholder)                                                                                                          |
| `helperText`  | `string`         | All input/selection types | Instructional text below the field                                                                                                   |
| `tooltip`     | `string`         | All                       | Tooltip on hover/focus of info icon                                                                                                  |
| `detail`      | `string`         | All                       | Rich text displayed above the component. Supports markup (`[b]`, `[i]`, `[l]`, `[s size="sm|lg|xl"]`). When set, hides the label unless `showLabel` is true |
| `width`       | `ComponentWidth` | All                       | Component width for flow-based layout. See [Width](#width)                                                                           |

### Display Component Properties

| Field              | Type                                              | Applicable Types            | Description                                                        |
| ------------------ | ------------------------------------------------- | --------------------------- | ------------------------------------------------------------------ |
| `value`            | `string`                                          | `header`, `text`, `divider`, `callout` | Static content value (supports markup on text/callout) |
| `displayType`      | `boolean`                                         | `text`                      | When `true`, renders as display-only (no input)                    |
| `textAlign`        | `"left"` \| `"center"` \| `"right"`               | `divider`                   | Text alignment for the label                                       |
| `title`            | `string`                                          | `callout`                   | Bold header text for the callout                                   |
| `variant`          | `"info"` \| `"success"` \| `"warning"` \| `"neutral"` | `callout`              | Color scheme (default `"info"`)                                    |
| `icon`             | `string`                                          | `callout`                   | Icon key: `info`, `check`, `warning`, `question`, `lightbulb`, `megaphone`, `bell`, `shield`, `lock`, `heart`, `flag`, `bookmark`, `zap`, `pencil`, `star`, or `none`. Falls back to variant default if omitted |
| `label`            | `string`                                          | `table`                     | Optional title displayed above the table                           |
| `columns`          | `TableColumn[]`                                   | `table`                     | Column definitions: `{ label: string }[]`                          |
| `rows`             | `TableRow[]`                                      | `table`                     | Row definitions: `{ label: string, values: string[] }[]`           |
| `highlightColumn`  | `integer`                                         | `table`                     | 1-based column number to highlight (0 = none)                      |

### Computed Component Properties

| Field            | Type                                         | Description                                                                |
| ---------------- | -------------------------------------------- | -------------------------------------------------------------------------- |
| `label`          | `string`                                     | Display label for the computed value                                       |
| `formula`        | `string`                                     | Formula expression. References fields via `{uuid}`, supports `+`, `-`, `*`, `/`, `SUM()`, `COUNT()`, `AVG()`, `MIN()`, `MAX()`, `IF()`, comparison operators, and `.selectedOption.metadata.key` |
| `format`         | `"number"` \| `"currency"` \| `"percentage"` | Display format for the result (default `"number"`)                         |
| `decimalPlaces`  | `integer`                                    | Decimal places (0-10, default 2)                                           |
| `currencySymbol` | `string`                                     | Currency symbol when format is `"currency"` (default `"$"`)                |
| `showLabel`      | `boolean`                                    | Whether to display the label (default `true`)                              |
| `detail`         | `string`                                     | Rich text description displayed above the computed value                   |
| `width`          | `ComponentWidth`                             | Layout width                                                               |

Inside a repeater, formula references to sibling template UUIDs resolve to the current iteration's values. Aggregation functions (`SUM`, `COUNT`, `AVG`, `MIN`, `MAX`) still aggregate across all iterations. Outside a repeater, the computed component evaluates once using standard field resolution.

### Signature Properties

| Field       | Type                                   | Description                                                                          |
| ----------- | -------------------------------------- | ------------------------------------------------------------------------------------ |
| `label`     | `string`                               | Display label (default `"Signature"`)                                                |
| `showLabel` | `boolean`                              | Whether to display the label                                                         |
| `detail`    | `string`                               | Rich text description above the signature pad                                        |
| `mode`      | `"draw"` \| `"type"` \| `"both"`      | Draw = canvas pad, Type = typed name in script font, Both = toggle between modes     |
| `required`  | `boolean`                              | Whether a signature is required                                                      |
| `width`     | `ComponentWidth`                       | Layout width                                                                         |

Draw mode stores the signature as a base64 PNG data URL. Type mode stores the typed name as a plain string.

### Text Input Properties

| Field                | Type                                                        | Applicable Types                            | Description                                                                                                               |
| -------------------- | ----------------------------------------------------------- | ------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------- |
| `regex`              | `string`                                                    | `inputText`, `inputTextArea`                | Regular expression for validation                                                                                         |
| `maxlength`          | `integer`                                                   | `inputText`, `inputTextArea`                | Maximum character length                                                                                                  |
| `startAdornment`     | `string`                                                    | `inputText`, `inputNumber`                  | Text/symbol at the start of the field (e.g. `$`, `+1`)                                                                    |
| `decimalPlaces`      | `integer`                                                   | `inputNumber`                               | When set, restricts input to a fixed number of decimal places (0-10). Uses controlled text input with keystroke filtering |
| `min`                | `number`                                                    | `inputNumber`                               | Minimum allowed value. Sets HTML `min` attribute (constrains stepper arrows) and clamps value on blur                     |
| `max`                | `number`                                                    | `inputNumber`                               | Maximum allowed value. Sets HTML `max` attribute (constrains stepper arrows) and clamps value on blur                     |
| `inputType`          | `"text"` \| `"email"` \| `"tel"` \| `"url"` \| `"password"` | `inputText`                                 | HTML input type hint. Controls mobile keyboard behavior and input masking. Default `"text"`                               |
| `readOnly`           | `boolean`                                                   | `inputText`, `inputTextArea`, `inputNumber`, `dropDown`, `dropDownMulti`, `radio`, `checkbox`, `slider`, `date`, `time`, `dateTime` | When `true`, the field is non-editable with a muted background. Useful for locking pricing/values so users can see but not modify |
| `autocomplete`       | `string`                                                    | `inputText`, `inputTextArea`, `inputNumber` | HTML `autocomplete` attribute value for browser autofill (e.g. `"email"`, `"tel"`, `"given-name"`)                        |
| `showPasswordToggle` | `boolean`                                                   | `inputText`                                 | When `true` and `inputType` is `"password"`, shows an eye icon to toggle password visibility                              |

### Selection Component Properties

| Field          | Type                          | Applicable Types                                          | Description                                                                                                                                                                          |
| -------------- | ----------------------------- | --------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `options`      | [Option](#option)[]           | `dropDown`, `dropDownMulti`, `checkbox`, `radio`          | Available choices                                                                                                                                                                    |
| `defaultValue` | `string` \| `"yes"` \| `"no"` | `yesNo`, `radio`, `cardSelect`, `dropDown`                | Pre-selected option on first render. For `yesNo` use `"yes"` or `"no"`; for the others use the matching option `value`. Ignored when no option matches. Without a matching `defaultValue`, all of these start at `null` — there is no default-to-first-option behavior. |

### Slider Properties

| Field       | Type      | Description     |
| ----------- | --------- | --------------- |
| `min`       | `number`  | Minimum value   |
| `max`       | `number`  | Maximum value   |
| `steps`     | `number`  | Step increment  |
| `marks`     | `boolean` | Show tick marks |
| `initValue` | `number`  | Initial value   |

### Rating Properties

| Field       | Type     | Description                                                       |
| ----------- | -------- | ----------------------------------------------------------------- |
| `max`       | `number` | Maximum rating value                                              |
| `precision` | `number` | Rating precision (e.g. `0.5` for half-stars, `1` for whole stars) |
| `icon`      | `string` | Icon shape: `star`, `heart`, `thumbsUp`, `circle`, or `diamond` (default `"star"`) |

### File Upload Properties

| Field     | Type      | Description                                             |
| --------- | --------- | ------------------------------------------------------- |
| `accept`  | `string`  | Comma-separated file extensions (e.g. `.pdf,.jpg,.png`) |
| `maxSize` | `integer` | Maximum file size in bytes                              |

### Group Properties

| Field         | Type      | Description                                                                                                  |
| ------------- | --------- | ------------------------------------------------------------------------------------------------------------ |
| `showLabel`   | `boolean` | Show the component label even when detail text is set. For groups, shows the group label as a visible header |
| `collapsible` | `boolean` | Allow the group to be collapsed/expanded                                                                     |
| `showBorder`  | `boolean` | Draw the container border/frame around the group                                                             |

### Repeater Properties

| Field           | Type      | Description                                                                             |
| --------------- | --------- | --------------------------------------------------------------------------------------- |
| `showLabel`     | `boolean` | Show the repeater label as a visible header                                             |
| `collapsible`   | `boolean` | Allow each iteration to be collapsed/expanded                                           |
| `showBorder`    | `boolean` | Draw the container border/frame around the repeater                                     |
| `minIterations` | `integer` | Minimum number of rows (also initial count). Default `1`. Remove button hidden at minimum |
| `initialData`   | `array`   | Pre-populated data for rows. Each element maps template child UUIDs to initial values    |

Repeaters are containers that support multiple iterations — users can add and remove rows of child components. Groups are purely visual containers (fieldset + condition grouping).

#### Nesting Rules

- Groups inside repeaters: allowed
- Repeaters inside groups: allowed
- Repeaters inside repeaters: **not** allowed

#### Migration

Legacy `group` components with `repeatable: true` must be migrated to `type: "repeater"` before loading.

### Date & Time Properties

| Field              | Type         | Applicable Types                                 | Description                                                                                                                    |
| ------------------ | ------------ | ------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------ |
| `placeholder`      | `string`     | `date`, `time`, `dateTime`                       | Placeholder text when empty                                                                                                    |
| `placeholderStart` | `string`     | `dateRange`, `timeRange`, `dateTimeRange`        | Placeholder text for the start field                                                                                           |
| `placeholderEnd`   | `string`     | `dateRange`, `timeRange`, `dateTimeRange`        | Placeholder text for the end field                                                                                             |
| `min`              | `string`     | All date/time types                              | Minimum allowed value. Accepts ISO date (`YYYY-MM-DD`), relative expression (`today`, `today+7`, `today-3`), or time (`HH:MM`) |
| `max`              | `string`     | All date/time types                              | Maximum allowed value. Same formats as `min`                                                                                   |
| `dateFormat`       | `DateFormat` | `date`, `dateTime`, `dateRange`, `dateTimeRange` | Display format for dates. See [Date Format](#date-format). Default `"MM/DD/YYYY"`                                              |
| `minTime`          | `string`     | `dateTime`, `dateTimeRange`                      | Minimum allowed time (`HH:MM`, 24h). Enforced independently from `min` date                                                    |
| `maxTime`          | `string`     | `dateTime`, `dateTimeRange`                      | Maximum allowed time (`HH:MM`, 24h). Enforced independently from `max` date                                                    |
| `step`             | `number`     | `time`, `dateTime`, `timeRange`, `dateTimeRange` | Minute interval for time selection. Default `15`                                                                               |

#### Date Format

| Value          | Example                |
| -------------- | ---------------------- |
| `"MM/DD/YYYY"` | `02/17/2026` (default) |
| `"DD/MM/YYYY"` | `17/02/2026`           |
| `"YYYY-MM-DD"` | `2026-02-17`           |

#### Relative Date Constraints

The `min` and `max` properties on date-containing types accept relative expressions:

| Expression  | Meaning                                                 |
| ----------- | ------------------------------------------------------- |
| `"today"`   | Today's date                                            |
| `"today+N"` | N days from today (e.g. `"today+14"` = two weeks ahead) |
| `"today-N"` | N days before today (e.g. `"today-7"` = one week ago)   |

Fixed ISO dates (e.g. `"2026-01-01"`) continue to work as before. Relative expressions are resolved at render time in FBRE.

#### Value Formats

| Type            | Value Format         | Example                                                      |
| --------------- | -------------------- | ------------------------------------------------------------ |
| `date`          | `"YYYY-MM-DD"`       | `"2026-02-17"`                                               |
| `time`          | `"HH:MM"` (24h)      | `"14:30"`                                                    |
| `dateTime`      | `"YYYY-MM-DDTHH:MM"` | `"2026-02-17T14:30"`                                         |
| `dateRange`     | `{ start, end }`     | `{ "start": "2026-02-17", "end": "2026-02-20" }`             |
| `timeRange`     | `{ start, end }`     | `{ "start": "09:00", "end": "17:00" }`                       |
| `dateTimeRange` | `{ start, end }`     | `{ "start": "2026-02-17T09:00", "end": "2026-02-20T17:00" }` |

### Yes/No Properties

| Field      | Type     | Description                                                                                                                        |
| ---------- | -------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| `labelYes` | `string` | Custom label for the "Yes" button. Default `"Yes"`                                                                                 |
| `labelNo`  | `string` | Custom label for the "No" button. Default `"No"`                                                                                   |
| `layout`   | `string` | Button layout direction: `"horizontal"` (side-by-side) or `"vertical"` (stacked). Omit for responsive default (adapts to width) |

Value is `"yes"` or `"no"` (string).

### Color Picker Properties

| Field          | Type       | Description                                                       |
| -------------- | ---------- | ----------------------------------------------------------------- |
| `swatches`     | `string[]` | Preset color swatches (hex format, e.g. `["#FF0000", "#00FF00"]`) |
| `defaultValue` | `string`   | Default color value (hex format, e.g. `"#1976d2"`)                |

Value is a hex color string (e.g. `"#FF5733"`).

### Card Select Properties

| Field     | Type                                    | Description                                     |
| --------- | --------------------------------------- | ----------------------------------------------- |
| `options` | [CardSelectOption](#cardselectoption)[] | Array of card options to display                |
| `columns` | `integer`                               | Number of columns in the card grid. Default `2` |

Value is the `value` string of the selected card option.

#### CardSelectOption

| Field         | Type       | Required | Description                                                  |
| ------------- | ---------- | -------- | ------------------------------------------------------------ |
| `label`       | `string`   | Yes      | Display title for the card                                   |
| `value`       | `string`   | Yes      | Value stored when selected                                   |
| `description` | `string`   | No       | Short description below the title                            |
| `title`       | `string`   | No       | Heading text displayed prominently on the card               |
| `features`    | `string[]` | No       | Feature strings displayed as a checklist                     |
| `badge`       | `string`   | No       | Badge text shown as a pill above the card (e.g. `"Popular"`) |
| `metadata`    | `object`   | No       | Arbitrary key-value pairs for calculations                    |

#### Card Select Example

```json
{
  "type": "cardSelect",
  "properties": {
    "label": "Plan",
    "showLabel": false,
    "options": [
      {
        "label": "Pro",
        "value": "pro",
        "title": "$29/mo",
        "features": ["20 flows", "1,000 submissions/mo"]
      },
      {
        "label": "Business",
        "value": "business",
        "title": "$79/mo",
        "badge": "Popular",
        "features": ["100 flows", "5,000 submissions/mo"]
      }
    ]
  }
}
```

---

## Option

A selectable option used by `dropDown`, `dropDownMulti`, `checkbox`, and `radio` components.

| Field      | Type                                          | Required | Description                                                                                   |
| ---------- | --------------------------------------------- | -------- | --------------------------------------------------------------------------------------------- |
| `label`    | `string`                                      | Yes      | Display text                                                                                  |
| `value`    | `string` \| `number`                          | Yes      | Value stored when selected                                                                    |
| `metadata` | `Record<string, string \| number \| boolean>` | No       | Structured key-value metadata (e.g. price, weight, unit). Consumed by the calculations engine |

### Option Metadata Example

```json
{
  "type": "dropDown",
  "properties": {
    "label": "Material",
    "options": [
      {
        "label": "Standard Steel",
        "value": "steel",
        "metadata": { "pricePerUnit": 12.50, "unit": "sqft" }
      },
      {
        "label": "Premium Aluminum",
        "value": "aluminum",
        "metadata": { "pricePerUnit": 24.00, "unit": "sqft" }
      }
    ]
  }
}
```

---

## Width

Components use a flow-based layout system where each component declares its own width. Components flow left-to-right and wrap naturally, like words in a paragraph. This works at both the screen level and inside groups.

### ComponentWidth

| Value              | CSS Width | Description                       |
| ------------------ | --------- | --------------------------------- |
| `"full"`           | `100%`    | Full width (default when omitted) |
| `"three-quarters"` | `75%`     | Three quarters width              |
| `"two-thirds"`     | `66.67%`  | Two thirds width                  |
| `"half"`           | `50%`     | Half width                        |
| `"third"`          | `33.33%`  | One third width                   |
| `"quarter"`        | `25%`     | Quarter width                     |

### Width Example

```json
{
  "type": "group",
  "properties": { "label": "Name" },
  "components": [
    {
      "type": "inputText",
      "properties": { "label": "First Name", "width": "half" }
    },
    {
      "type": "inputText",
      "properties": { "label": "Last Name", "width": "half" }
    }
  ]
}
```

### Migration

Legacy group `layout` strings (e.g. `"2: 50,50"`) must be migrated to per-child `width` values before loading. The `layout` property is no longer supported.

---

## Validation

Components can have validation rules defined in `properties.validation`. This replaces the legacy `required` and `regex` flat properties. Legacy formats must be migrated before loading.

### FlowValidationConfig

| Field   | Type                                | Required | Description                      |
| ------- | ----------------------------------- | -------- | -------------------------------- |
| `rules` | [ValidationRule](#validationrule)[] | Yes      | Ordered list of validation rules |

### ValidationRule

| Field     | Type     | Required | Description                                                     |
| --------- | -------- | -------- | --------------------------------------------------------------- |
| `type`    | `string` | Yes      | Rule type (see [Validation Rule Types](#validation-rule-types)) |
| `params`  | `object` | No       | Type-specific parameters                                        |
| `message` | `string` | No       | Custom error message (default message used when omitted)        |

### Validation Rule Types

| Type           | Params                | Default Message                            | Applicable Types                            |
| -------------- | --------------------- | ------------------------------------------ | ------------------------------------------- |
| `required`     | —                     | "This field is required"                   | All input/selection types                   |
| `email`        | —                     | "Please enter a valid email address"       | `inputText`                                 |
| `phone`        | —                     | "Please enter a valid phone number"        | `inputText`                                 |
| `url`          | —                     | "Please enter a valid URL"                 | `inputText`                                 |
| `minLength`    | `{ min: number }`     | "Must be at least N characters"            | `inputText`, `inputTextArea`                |
| `maxLength`    | `{ max: number }`     | "Must be no more than N characters"        | `inputText`, `inputTextArea`                |
| `exactLength`  | `{ length: number }`  | "Must be exactly N characters"             | `inputText`, `inputTextArea`                |
| `minValue`     | `{ min: number }`     | "Must be at least N"                       | `inputNumber`                               |
| `maxValue`     | `{ max: number }`     | "Must be no more than N"                   | `inputNumber`                               |
| `pattern`      | `{ regex: string }`   | "Value does not match the required format" | `inputText`, `inputTextArea`                |
| `minSelected`  | `{ min: number }`     | "Select at least N options"                | `dropDownMulti`, `checkbox`                 |
| `maxSelected`  | `{ max: number }`     | "Select no more than N options"            | `dropDownMulti`, `checkbox`                 |
| `fileType`     | `{ types: string[] }` | "File type is not allowed"                 | `fileUpload`                                |
| `fileSize`     | `{ max: number }`     | "File must be smaller than N"              | `fileUpload`                                |
| `contains`     | `{ text: string }`    | "Must contain \"text\""                    | `inputText`, `inputTextArea`                |
| `excludes`     | `{ text: string }`    | "Must not contain \"text\""                | `inputText`, `inputTextArea`                |
| `matchesField` | `{ field: string }`   | "Fields must match"                        | `inputText`, `inputTextArea`, `inputNumber` |

### Validation Example

```json
{
  "validation": {
    "rules": [
      { "type": "required" },
      { "type": "email" },
      {
        "type": "minLength",
        "params": { "min": 5 },
        "message": "Email must be at least 5 characters"
      }
    ]
  }
}
```

### Error Display

When validation fails, the first failing error message is shown below the field. If multiple rules fail, an "(and N more)" indicator is appended.

### Migration

Legacy `required` and `regex` properties must be migrated to `FlowValidationConfig` before loading.:

- `{ required: true }` → `{ validation: { rules: [{ type: "required" }] } }`
- `{ regex: "..." }` → `{ validation: { rules: [{ type: "pattern", params: { regex: "..." } }] } }`

---

## Screen Transitions

Animated transitions between screens can be enabled via `config.navigation.transition`. When absent or set to `"none"`, screen changes are instant (zero overhead).

### ScreenTransitionType

| Value         | Effect                             | Direction-aware? |
| ------------- | ---------------------------------- | ---------------- |
| `"none"`      | Instant swap (default)             | —                |
| `"slide"`     | Full horizontal slide left/right   | Yes              |
| `"fade"`      | Crossfade between screens          | No               |
| `"slideFade"` | 30px horizontal slide with opacity | Yes              |
| `"rise"`      | Vertical rise up / sink down       | Yes              |
| `"scaleFade"` | Scale 0.95→1 / 1→1.05 with opacity | No               |

Direction-aware transitions reverse their animation when navigating backward.

### Example

```json
{
  "config": {
    "navigation": { "transition": "slideFade" }
  }
}
```

### CSS Custom Properties

| Property                     | Default                        | Description            |
| ---------------------------- | ------------------------------ | ---------------------- |
| `--fbre-transition-duration` | `250ms`                        | Animation duration     |
| `--fbre-transition-easing`   | `cubic-bezier(0.4, 0, 0.2, 1)` | Animation easing curve |

`@media (prefers-reduced-motion: reduce)` sets the duration to `0ms` automatically.

---

## Conditions

Conditions allow components and screens to be shown or hidden based on the runtime value of other components.

### FlowConditionConfig

| Field    | Type                              | Required | Description                                                             |
| -------- | --------------------------------- | -------- | ----------------------------------------------------------------------- |
| `action` | `"show"` \| `"hide"`              | Yes      | `show` = visible when condition met; `hide` = hidden when condition met |
| `when`   | [ConditionGroup](#conditiongroup) | Yes      | The condition group to evaluate                                         |

### ConditionGroup

| Field   | Type                              | Required | Description                                           |
| ------- | --------------------------------- | -------- | ----------------------------------------------------- |
| `logic` | `"and"` \| `"or"`                 | Yes      | `and` = all rules must match; `or` = any rule matches |
| `rules` | [ConditionRule](#conditionrule)[] | Yes      | One or more rules (minimum 1)                         |

### ConditionRule

| Field        | Type                                                          | Required | Description                                                                                                                         |
| ------------ | ------------------------------------------------------------- | -------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| `source`     | `string`                                                      | Yes      | UUID of the source component, or a context key when `sourceType` is `"context"`                                                     |
| `sourceType` | `"component"` \| `"context"`                                  | No       | Where to resolve the source. `"component"` (default) looks up a component UUID. `"context"` looks up a key in FBRE's `context` prop |
| `operator`   | `string`                                                      | Yes      | Comparison operator (see [Operators](#operators))                                                                                   |
| `value`      | `string` \| `number` \| `boolean` \| `string[]` \| `number[]` | No       | Comparison value. Omitted for unary operators                                                                                       |

#### External Context Conditions

When `sourceType` is `"context"`, the rule evaluates against a value from FBRE's `context` prop rather than a component UUID. This allows conditions based on external application state.

```jsx
<FBRE
  flow={flow}
  context={{ isInvite: true, userTier: "pro" }}
  onFlowComplete={handleComplete}
/>
```

```json
{
  "conditions": {
    "action": "hide",
    "when": {
      "logic": "and",
      "rules": [
        { "source": "isInvite", "sourceType": "context", "operator": "isTrue" }
      ]
    }
  }
}
```

### Operators

| Operator             | Category | Expects Value | Description                                               |
| -------------------- | -------- | ------------- | --------------------------------------------------------- |
| `equals`             | Equality | Yes           | Exact match                                               |
| `notEquals`          | Equality | Yes           | Not an exact match                                        |
| `contains`           | String   | Yes           | Source contains the value substring                       |
| `notContains`        | String   | Yes           | Source does not contain the value substring               |
| `startsWith`         | String   | Yes           | Source starts with the value                              |
| `endsWith`           | String   | Yes           | Source ends with the value                                |
| `isEmpty`            | Presence | No            | Source has no value                                       |
| `isNotEmpty`         | Presence | No            | Source has a value                                        |
| `greaterThan`        | Numeric  | Yes           | Source > value                                            |
| `greaterThanOrEqual` | Numeric  | Yes           | Source >= value                                           |
| `lessThan`           | Numeric  | Yes           | Source < value                                            |
| `lessThanOrEqual`    | Numeric  | Yes           | Source <= value                                           |
| `isOneOf`            | Set      | Yes (array)   | Source value is in the provided array                     |
| `isNotOneOf`         | Set      | Yes (array)   | Source value is not in the provided array                 |
| `includesAny`        | Set      | Yes (array)   | Source (multi-value) includes any of the provided values  |
| `includesAll`        | Set      | Yes (array)   | Source (multi-value) includes all of the provided values  |
| `includesNone`       | Set      | Yes (array)   | Source (multi-value) includes none of the provided values |
| `isTrue`             | Boolean  | No            | Source is truthy                                          |
| `isFalse`            | Boolean  | No            | Source is falsy                                           |

---

## FlowCalculation

Top-level calculations are global formulas not tied to any screen. They compute values reactively from component inputs, option metadata, and other calculations.

| Field            | Type                | Required | Description                                                    |
| ---------------- | ------------------- | -------- | -------------------------------------------------------------- |
| `uuid`           | `string`            | Yes      | Unique identifier for the calculation                          |
| `label`          | `string`            | Yes      | Human-readable label                                           |
| `formula`        | `string`            | Yes      | Formula expression (see syntax below)                          |
| `format`         | `CalculationFormat` | No       | Display format: `"number"`, `"currency"`, or `"percentage"`    |
| `decimalPlaces`  | `integer`           | No       | Number of decimal places (0–10)                                |
| `currencySymbol` | `string`            | No       | Currency symbol when format is `"currency"` (defaults to `$`)  |

### Formula Syntax

| Syntax                                        | Description                                                     |
| --------------------------------------------- | --------------------------------------------------------------- |
| `{uuid}`                                      | Component value reference                                       |
| `{uuid}.selectedOption.metadata.key`          | Selected option's metadata value                                |
| `SUM({uuid})`                                 | Sum across repeater iterations                                  |
| `COUNT({uuid})`                               | Count of repeater iterations                                    |
| `AVG({uuid})`                                 | Average across repeater iterations                              |
| `MIN({uuid})`                                 | Minimum across repeater iterations                              |
| `MAX({uuid})`                                 | Maximum across repeater iterations                              |
| `MIN(expr, expr, ...)`                        | Scalar minimum of N expressions (e.g. discount capping)         |
| `MAX(expr, expr, ...)`                        | Scalar maximum of N expressions (e.g. minimum charge)           |
| `IF(cond, then, else)`                        | Conditional — returns `then` when `cond != 0`, else `else`      |
| `> < >= <= == !=`                             | Comparison operators (return `1` for true, `0` for false)       |
| `+ - * /`                                     | Arithmetic operators                                            |
| `( )`                                         | Grouping / precedence                                           |
| Numeric literals                              | Constants (e.g. `0.08`, `100`)                                  |

Formulas return `null` when any referenced field is empty or unresolvable. Calculations can reference other calculations by UUID; evaluation follows topological order.

### Example

```json
{
  "calculations": [
    {
      "uuid": "calc-subtotal",
      "label": "Subtotal",
      "formula": "SUM({line-total-uuid})",
      "format": "currency",
      "decimalPlaces": 2,
      "currencySymbol": "$"
    },
    {
      "uuid": "calc-tax",
      "label": "Tax Amount",
      "formula": "{calc-subtotal} * 0.08",
      "format": "currency",
      "decimalPlaces": 2,
      "currencySymbol": "$"
    }
  ]
}
```

---

## Reference Markup

Text properties can include `${...}` references that resolve at render time. Resolution tries, in order: calculation results, component values, then keys from FBRE's `context` prop. References are replaced with the resolved value when the form is displayed.

### Syntax

| Pattern | Resolves To | Example |
| --- | --- | --- |
| `${calculation-uuid}` | Formatted calculation result | `${calc-456}` → `"$1,247.50"` |
| `${component-uuid}` | Component's current value | `${abc-123}` → `"John Smith"` |
| `${context-key}` | Value from the FBRE `context` prop | `${accountName}` → `"Acme Inc."` |

### Supported Surfaces

References are supported in the following component properties:

| Surface | Property | Resolution |
| --- | --- | --- |
| Display Text | `value` | HTML (within markup) |
| Header | `value` | Plain text |
| Callout | `title`, `value` | title: plain text, value: HTML |
| Field labels | `label` | Plain text |
| Field descriptions | `detail` | HTML (within markup) |
| Helper text | `helperText` | Plain text |
| Placeholders | `placeholder` | Plain text |
| Screen labels | `label` | Plain text |

### Example

```json
{
  "type": "text",
  "properties": {
    "value": "Your total is ${calc-total} for ${service-type-uuid} service."
  }
}
```
