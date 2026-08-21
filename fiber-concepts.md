---
title: Fiber Concepts
applies-to:
  - "@sonata-innovations/fiber-types@^3.0"
  - "@sonata-innovations/fiber-shared@^1.2"
  - "@sonata-innovations/fiber-fbre@^4.0"
  - "@sonata-innovations/fiber-fbt@^3.0"
  - "@sonata-innovations/fiber-fbtl@^3.0"
  - "@sonata-innovations/fiber-theme-editor@^2.0"
read-when: "First contact with Fiber: the Flow/Screen/Component model, FlowData, builders vs render engine, conditions/validation/calculations concepts."
---

# Fiber Concepts

> A high-level conceptual overview of Fiber for AI assistants and newcomers.
> For integration code and API reference, see the [integration guides](integration/index.md).
> For the JSON schema specification, see [`flow-schema.md`](schema/flow-schema.md).

---

## 1. What is Fiber?

Fiber is a system for building and rendering data-collection forms. It splits authoring from rendering and offers two authoring surfaces targeted at different audiences:

- **FBT (Fiber Tool)** — A visual drag-and-drop builder for power users / form designers. Full feature surface: screens, multi-rule conditions, validation, calculations, reference markup, the complete 30-type component palette. See the [FBT integration guide](integration/fbt.md).
- **FBTL (Fiber Tool Lite)** — A stripped-down, end-user-friendly builder designed to be embedded in parent apps so non-technical end users (clinicians, teachers, etc.) can author their own forms. Seven question types + Information Screens + single-rule conditions. Preserves advanced properties from loaded flows without editing them. See the [FBTL integration guide](integration/fbtl.md).
- **FBRE (Fiber Render Engine)** — A render engine that consumes Flow JSON, renders the interactive form for end users, and outputs collected data ("FlowData") back to the parent application. See the [FBRE integration guide](integration/fbre.md).

A fourth library, the **Theme Editor**, is a plug-and-play widget that lets end users visually customize a flow's theme (color scheme, style, palette knobs) and emits the resulting `ThemeConfig`. See the [Theme Editor integration guide](integration/theme-editor.md).

**The problem it solves:** Both form designers and parent-app end users need to create multi-step forms without writing code. Developers need to embed those forms into their applications without building form infrastructure from scratch.

The libraries are standard React components — not iframes, not external services. They embed directly into a parent React application with full control over styling, data flow, and lifecycle. **FBT and FBTL produce the same Flow JSON schema**, so any Flow is portable between them (with FBTL preserving — but not editing — power-user features).

### Packages

| Package | What it is |
|---------|-----------|
| `@sonata-innovations/fiber-types` | Shared TypeScript types — the Flow/FlowData schema as code |
| `@sonata-innovations/fiber-shared` | Engine layer: condition evaluation, validation, markup conversion, formula evaluation. Used client-side (FBRE, FBT) **and** server-side, so the same rules evaluate identically anywhere |
| `@sonata-innovations/fiber-fbre` | The render engine |
| `@sonata-innovations/fiber-fbt` | The full builder |
| `@sonata-innovations/fiber-fbtl` | The lite builder |
| `@sonata-innovations/fiber-theme-editor` | The theming widget |

Three private (unpublished) packages — a Fastify **server**, an admin **portal**, and a **hosted** form-filling app — make up the hosted platform built on top of the libraries (see Section 13).

---

## 2. Core Data Model

### The Flow JSON

The Flow is the interchange format between the builders and the render engine. FBT and FBTL both produce it; FBRE consumes it. It is a plain JSON document that can be stored, transmitted, versioned, and inspected.

### Hierarchy

```
Flow
├── uuid            — Unique identifier for the entire flow
├── metadata        — Name, description, and arbitrary key-value pairs
├── config          — Runtime settings (mode, theme, navigation, controls, confirmation, summary)
├── calculations[]  — Optional: named formulas computed over component values
└── screens[]       — Ordered list of screens (pages)
    ├── uuid
    ├── label
    ├── conditions    — Optional: show/hide this screen based on rules
    └── components[]  — Ordered list of components on this screen
        ├── uuid
        ├── type          — One of 30 component types
        ├── properties    — Configuration (label, placeholder, options, validation, width, etc.)
        ├── conditions    — Optional: show/hide this component based on rules
        └── components[]  — Optional: child components (group and repeater types)
```

### Identity

Every entity (flow, screen, component, calculation) has a UUID. These UUIDs are the stable identity used for:
- Referencing components in conditions ("show this if *that component* equals X")
- Cross-field validation ("this field must match *that field*")
- Reference markup (`${uuid}` tokens resolving to live values)
- Pre-populating data (matching FlowData entries to their components)
- Tracking changes in the builder

### Flow Configuration

The optional `config` object controls runtime behavior, organized into semantic groups:

| Group | Setting | Purpose |
|-------|---------|---------|
| `theme` | `color` | Primary accent color (CSS value) |
| `theme` | `colorScheme` | Built-in palette preset: `"light"` (default) or `"dark"` (replaces the former `darkMode` boolean) |
| `theme` | `style` | Visual style (`"clean"`, `"outlined"`, …) |
| `theme` | `background` / `surface` / `text` / `border` | Palette knobs overriding the preset tokens |
| `theme` | `radius` / `fontFamily` | Corner radius and font family knobs. `fontFamily` also accepts `{ family, src }`, which FBRE loads itself rather than assuming the host page already has the font |
| `theme` | `error` / `success` / `warning` | Semantic state color knobs |
| `navigation` | `transition` | Screen transition animation type |
| `navigation` | `allowInvalidTransition` | Allow navigating past screens with validation errors |
| `navigation` | `autoAdvance` | Advance ~500ms after a single-select choice (default off) |
| `navigation` | `advanceOnEnter` | Advance on Enter in a single-line input (default on) |
| `controls` | `show` | Show/hide built-in next/back buttons |
| `controls` | `layout` | Controls layout (`"default"`, `"centered"`, `"inline-full"`, or `"stacked"`) |
| `controls` | `showStepper` | Show/hide the step indicator |
| `controls` | `stepperStyle` | Stepper variant (`"default"`, `"dots"`, `"pill"`, `"glow"`, `"bar"`, `"text"`) |
| `confirmation` | `show` / `title` / `body` | Terminal "thank you" screen shown after submission (see Section 13) |
| _(flat)_ | `summary` | Show a review screen before final submission |

---

## 3. Screens

Screens are the pages of a multi-step form. Users see one screen at a time and navigate between them with next/back buttons.

**Key behaviors:**
- Screens render in array order (first screen in the array is shown first)
- Navigation is linear by default — next goes forward, back goes backward
- Screens can be conditionally shown or hidden (see Conditions below) — hidden screens are automatically skipped during navigation
- Each screen can customize its next and back button labels
- Screen transitions can be animated (slide, fade, rise, etc.)

**Summary screen:** When enabled via `config.summary`, a review screen appears after the last screen showing all collected data before the user confirms submission.

---

## 4. Components

Components are the individual form elements within a screen. Fiber has **30 component types** across six categories. This section summarizes each category; the exhaustive per-type property reference lives in [`flow-schema.md`](schema/flow-schema.md).

| Category | Types | Purpose |
|----------|-------|---------|
| Display (6) | `header`, `text`, `divider`, `callout`, `table`, `computed` | Presentational content: headings, body text (with inline markup), separators, highlighted callouts, static tables, and calculated-value display |
| Text & Number (3) | `inputText`, `inputTextArea`, `inputNumber` | Free-form text and numeric entry |
| Selection (8) | `dropDown`, `dropDownMulti`, `checkbox`, `radio`, `toggleSwitch`, `yesNo`, `confirm`, `cardSelect` | Choosing from predefined options — single-select, multi-select, boolean toggles, an acknowledgment checkbox, and rich card-based selection |
| Date & Time (6) | `date`, `time`, `dateTime`, `dateRange`, `timeRange`, `dateTimeRange` | Point-in-time and range pickers; ranges produce `{ start, end }` values |
| Interactive (5) | `fileUpload`, `rating`, `slider`, `colorPicker`, `signature` | Richer input widgets: drag-and-drop file upload, star rating, numeric slider, color selection, and signature capture |
| Containers (2) | `group`, `repeater` | Hold child components (see below) |

**Display components** are presentational and are excluded from FlowData — with one exception: `computed` lives in the display category but *does* emit a value (the result of the calculation it displays).

**Options:** Selection components that use options store them as `OptionItem` objects: `{ label, value }` where `label` is what the user sees and `value` (a `string` or `number`) is what gets stored in FlowData. Each option may also carry `metadata` — an arbitrary `Record<string, string | number | boolean>` — for machine-readable data (scores, codes, flags) that calculations and parent applications can use.

### Group

The `group` type contains its own `components[]` array (containers are recursive — groups can nest). A group is a visual grouping with an optional label; it can be **collapsible** (expand/collapse by the user) and can force its border on or off (`showBorder`).

### Repeater

The `repeater` type is the repeatable container: the user can add, fill in, and remove multiple iterations of the repeater's child components (e.g., "Add another phone number"). Its properties include `minIterations` (the minimum number of iterations) and `initialData` (pre-populated iteration values). Groups and repeaters can nest inside each other, but repeaters cannot nest inside repeaters.

### Component Properties

Every component has a `properties` object. Some properties are common across types (like `label`, `width`, `validation`), while others are type-specific (like `options` for selection components, `min`/`max` for sliders, `dateFormat` for date pickers). The full per-type property tables are in [`flow-schema.md`](schema/flow-schema.md).

---

## 5. Layout System

Each component has an optional `width` property that controls how much horizontal space it occupies:

| Width | CSS | Visual |
|-------|-----|--------|
| `full` (default) | 100% | Entire row |
| `half` | 50% | Half row |
| `third` | 33.33% | One-third |
| `two-thirds` | 66.67% | Two-thirds |
| `quarter` | 25% | One-quarter |
| `three-quarters` | 75% | Three-quarters |

**Wrapping behavior:** Components flow left-to-right like text. When components don't fill a row (e.g., two `half` components), they sit side by side. When they exceed a row's width, they wrap to the next line.

**Responsive:** Fractional widths are gap-adjusted `calc()` values (e.g. half is `calc(50% - gap/2)`). There is no global full-width collapse breakpoint; a few individual controls (buttons, stepper, yes/no) compact themselves at narrow container widths via CSS container queries — responding to the FBRE container's size, not the viewport.

---

## 6. Condition System

Conditions control the visibility of components and screens based on the current values of other components in the flow — or on external **context** values supplied by the parent application.

### Structure

```
conditions: {
  action: "show" | "hide",
  when: {
    logic: "and" | "or",
    rules: [
      {
        source: "<component-uuid or context key>",
        sourceType: "component" | "context",   // optional, defaults to "component"
        operator: "<operator>",
        value: "<test-value>"
      }
    ]
  }
}
```

- **`action`** — What to do when the rules evaluate to true: `show` the target or `hide` it
- **`logic`** — How to combine multiple rules: `and` (all must match) or `or` (any must match)
- **`source`** — UUID of the component whose value is being tested, or (with `sourceType: "context"`) the key of an external context value
- **`sourceType`** — Where the source value comes from: `"component"` (default) tests another component's live value; `"context"` tests a value from FBRE's `context` prop, letting parent-supplied data (user role, tenant flags, etc.) drive visibility
- **`operator`** — One of 19 comparison operators
- **`value`** — The value to compare against (omitted for operators that don't need it)

### Operators (19)

| Category | Operators |
|----------|-----------|
| Equality | `equals`, `notEquals` |
| String | `contains`, `notContains`, `startsWith`, `endsWith` |
| Empty | `isEmpty`, `isNotEmpty` |
| Numeric | `greaterThan`, `greaterThanOrEqual`, `lessThan`, `lessThanOrEqual` |
| Set membership | `isOneOf`, `isNotOneOf` |
| Array | `includesAny`, `includesAll`, `includesNone` |
| Boolean | `isTrue`, `isFalse` |

### Runtime Effects

When a component or screen is hidden by a condition:
- It is **not rendered** in the UI
- Its value is **excluded from FlowData** output
- Its validation rules are **not enforced** (won't block form submission)
- If it's a screen, navigation automatically skips it

---

## 7. Validation System

Validation enforces rules on component values before the user can proceed to the next screen or submit the form.

### Structure

```
properties: {
  validation: {
    rules: [
      { type: "<validator>", params: { ... }, message: "Optional custom error" }
    ]
  }
}
```

Each rule has a `type` (the validator name), optional `params` (validator-specific configuration), and an optional `message` (custom error text that overrides the default).

### Validators (17)

| Validator | Params | Purpose |
|-----------|--------|---------|
| `required` | — | Field must have a non-empty value |
| `email` | — | Valid email format |
| `phone` | — | Valid phone format |
| `url` | — | Valid URL format |
| `minLength` | `{ min }` | Minimum character count |
| `maxLength` | `{ max }` | Maximum character count |
| `exactLength` | `{ length }` | Exact character count |
| `minValue` | `{ min }` | Minimum numeric value |
| `maxValue` | `{ max }` | Maximum numeric value |
| `pattern` | `{ pattern }` | Regex pattern match |
| `minSelected` | `{ min }` | Minimum selected options (checkbox/multi-select) |
| `maxSelected` | `{ max }` | Maximum selected options |
| `fileType` | `{ types[] }` | Allowed MIME types |
| `fileSize` | `{ max }` | Maximum file size in bytes |
| `contains` | `{ text }` | Must contain substring |
| `excludes` | `{ text }` | Must not contain substring |
| `matchesField` | `{ field }` | Must match another component's value (by UUID) |

### Validation Timing

- Validation runs when the user attempts to navigate forward (next button or submission)
- If `config.navigation.allowInvalidTransition` is false (default), the user cannot proceed past an invalid screen
- Components hidden by conditions are not validated
- `matchesField` enables cross-field validation (e.g., "confirm email must match email")

---

## 8. Calculations

Calculations are named formulas defined at the flow level (`Flow.calculations[]`) that compute numeric results from component values as the user fills in the form.

- Each calculation has a `uuid`, `label`, and a `formula` — an expression over component values referenced by UUID, supporting arithmetic and aggregate functions
- Optional formatting: `format` (`"number"`, `"currency"`, `"percentage"`), `decimalPlaces`, `currencySymbol`
- Results update live as the user types
- The `computed` display component shows a calculation's result inline in the form
- `${uuid}` reference markup (see Section 10) can interpolate calculation results into text
- Results are emitted in FlowData under a top-level `calculations[]` array (see Section 9)

The formula engine lives in `@sonata-innovations/fiber-shared`, so calculations evaluate identically in FBRE, in FBT's preview, and on the server.

---

## 9. Data Flow: End to End

### Authoring Phase (FBT or FBTL)

1. A form author opens **FBT** (power users) or **FBTL** (non-technical end users) in a parent application
2. They assemble the flow — FBT via multi-screen drag-and-drop with the full component palette; FBTL via a flat, one-question-per-screen list limited to seven question types plus Information Screens
3. The builder produces a Flow JSON object via the `onFlowChange` callback (FBT also exposes `exportFlow()` on the store)
4. The parent application saves the Flow JSON (to a database, file, API, etc.)

FBTL uses a **page-break model**: each stage card carries a break-after flag, so a freshly authored flow is one question per screen until the author merges cards onto a shared screen. Screen structure round-trips — a loaded multi-screen flow's boundaries become break flags rather than being flattened away — and the host can switch the builder to a single-screen model entirely (`options.screenModel`). A component-level condition is promoted to a screen-level condition only when its screen ends up holding exactly one component; otherwise it stays on the component as an in-place show/hide. Advanced properties FBTL cannot edit (calculations, reference markup, multi-rule conditions, and so on) are preserved untouched, and the resulting JSON remains fully interoperable with FBT.

### Rendering Phase (FBRE)

1. The parent application loads the saved Flow JSON and passes it to FBRE (as a prop, or via one of the remote modes — see Section 13)
2. FBRE renders the form screen by screen
3. The end user fills in the form, navigating between screens
4. On the final screen, the user submits the form
5. FBRE calls `onFlowComplete` with the collected FlowData

### FlowData Structure

FlowData mirrors the Flow hierarchy but contains only collected values:

```
FlowData
├── uuid            — Same as the flow's UUID
├── metadata        — Same as the flow's metadata
├── calculations[]  — Present when the flow defines calculations
│   ├── uuid            — Calculation UUID
│   ├── label           — Calculation label
│   ├── value           — Computed numeric result (or null)
│   └── formattedValue  — Formatted string (e.g., "$1,250.00"), when computed
└── screens[]
    ├── uuid      — Screen UUID
    ├── label     — Screen label
    └── components[]
        ├── uuid          — Component UUID
        ├── type          — Component type
        ├── label         — Component label
        ├── value         — The collected value (type varies by component)
        └── components[][] — Containers only: child data, one inner array per iteration
```

**Containers in FlowData:** `group` and `repeater` components appear as their own entries carrying a nested `components` field typed `ComponentData[][]` — an array of iterations, each an array of child component data. A group always has exactly one iteration; a repeater has one inner array per iteration the user filled in.

**What's excluded from FlowData:**
- Display components (`header`, `text`, `divider`, `callout`, `table`) — they don't collect data (`computed` *is* included, with its calculated value)
- Components hidden by conditions — their values are not relevant
- Screens hidden by conditions — all their components are excluded

The full output schema is documented in [`flow-data-schema.md`](schema/flow-data-schema.md).

---

## 10. Theming and Presentation

### CSS Custom Properties

FBRE uses CSS custom properties for theming. Parent applications can override these on the `.fbre-container` element:

- **Colors:** `--fbre-theme-color`, `--fbre-error`, `--fbre-text`, `--fbre-border`, `--fbre-bg`, `--fbre-surface` (and light/dark/secondary variants)
- **Sizing:** `--fbre-radius` (border radius)
- **Typography:** `--fbre-font` (font family)
- **Animation:** `--fbre-transition-duration`, `--fbre-transition-easing`

### Color Scheme

Selected via the `theme` prop on FBRE (`{ colorScheme: "dark" }`) or `config.theme.colorScheme` in the Flow JSON (default `"light"`). The prop takes precedence. The scheme seeds a built-in palette preset (all `--fbre-*` tokens shift to light- or dark-appropriate values); palette knobs and raw CSS-variable overrides then layer on top. Replaces the former `darkMode` boolean.

For the full theming model — palette knobs, the override precedence, and the complete `--fbre-*` token catalog — see the [FBRE Theming Guide](features/fbre-theming.md).

### Screen Transitions

Six animation types for transitions between screens:

| Type | Effect |
|------|--------|
| `none` | Instant switch (default) |
| `slide` | Horizontal slide left/right |
| `fade` | Cross-fade |
| `slideFade` | Slide combined with fade |
| `rise` | Vertical rise with fade |
| `scaleFade` | Scale up/down with fade |

Set via `config.navigation.transition` in the Flow JSON.

### Inline Markup

`text` components and `detail` property fields support lightweight inline formatting:
- `[b]bold[/b]` renders as **bold**
- `[i]italic[/i]` renders as *italic*
- `[s size="sm|lg|xl"]sized text[/s]` renders at a preset font size
- `[l href="url"]link text[/l]` renders as a hyperlink

The markup → HTML converter lives in `@sonata-innovations/fiber-shared`, shared by FBRE (render), FBT (preview), and the server.

### Reference Markup

`${...}` tokens interpolate live values into text. A token is resolved in order against: **calculations** (by UUID) → **component values** (by UUID) → **context** (by key, from FBRE's `context` prop). Unresolvable references render as empty. This powers dynamic text like "Your total is ${calc-uuid}" or "Welcome back, ${userName}".

Writing one means knowing a UUID, so builders offer a picker. `resolvableReferences(flow)` from `@sonata-innovations/fiber-shared` returns every reference a flow can resolve, in that same order — use it rather than re-walking the flow, so a picker cannot drift from what the renderer will actually resolve. See [Confirmation Screen → Building your own picker](features/confirmation-screen.md#building-your-own-picker).

---

## 11. Extension Points

### Custom Presets

Presets are pre-configured component templates that appear in FBT's component palette. They let teams create reusable field patterns (e.g., "Phone Number" = inputText with phone validation and placeholder).

A preset is **serializable data** — a `PresetData` object (`{ type, label, icon, components }`) where `components` is a plain array of `Component` JSON. Because presets are data, they can be stored anywhere JSON can: in code, in a file, or in a database. When a preset is dropped onto the stage, FBT regenerates every UUID in the copy, so the same preset can be used many times without ID collisions.

### Custom Templates

Templates are complete pre-built flows (multiple screens with components). They let teams provide starting points for common form patterns (e.g., "Employee Onboarding", "Patient Intake").

A template is likewise data — a `TemplateData` object (`{ type, label, icon, flow }`) holding a complete `Flow`. Loading a template replaces the current flow in FBT, with all UUIDs regenerated.

### Data-First Design

Presets and templates being plain data (rather than code) is what enables server-side storage: the hosted platform stores tenant-scoped custom presets and templates in the database and serves them to FBT at load time. Older factory-function definitions (functions returning fresh `Component`/`Flow` objects) are still accepted for back-compat, but data-based definitions are the current model.

See [`custom-presets-and-templates.md`](features/custom-presets-and-templates.md) for the full authoring guide.

---

## 12. Key Architectural Decisions

### Zustand Stores

FBT, FBTL, and FBRE all use Zustand for state management. Each component instance creates its own isolated store, so multiple instances on the same page don't conflict.

### Shared Engine Layer

Condition evaluation, validation, markup conversion, and formula evaluation live in `@sonata-innovations/fiber-shared` — a dependency of both the client libraries and the server. This is the key to the "same rules evaluate anywhere" mental model: a condition behaves identically in FBRE running in a browser and in the server evaluating a session screen-by-screen.

### Component Registry

FBRE uses a registry pattern to map component types to their React renderers. Each component type registers itself, and a dispatcher looks up the correct renderer at runtime. This makes the system extensible — adding a new component type means registering a new entry.

### Flattened Store Architecture

Components are stored in a flat `Record<string, Component>` map (keyed by UUID), not nested inside screens. Separate index maps (`screenComponents`, `groupComponents`) track parent-child relationships. This makes lookups O(1) and avoids deep cloning when updating nested structures.

### No Runtime Auto-Migration

There is no runtime auto-migration: legacy (pre-v2) flows — old condition/validation formats, group `layout` strings, deprecated component types — must be migrated to the current schema before being loaded.

### Container Queries

Where components adapt to narrow spaces (compact button/stepper sizing, yes/no stacking), they use CSS container queries rather than media queries. The layout responds to the FBRE container's width, not the browser viewport — critical for embedded use cases where the form may occupy a sidebar or modal.

---

## 13. Integration Modes and Platform

### FBRE's Three Integration Modes

FBRE accepts a flow through one of three mutually exclusive prop shapes:

1. **Local flow** — `<FBRE flow={flowJson} />`. The parent owns storage and passes the Flow JSON directly. The simplest and most common mode.
2. **Remote flow** — `<FBRE flowId="..." apiEndpoint="..." />`. FBRE fetches a published flow from a Fiber server and renders it client-side. All evaluation (conditions, validation, calculations) still happens in the browser.
3. **Server-driven** — `<FBRE flowId="..." sessionEndpoint="..." />`. FBRE starts a session and receives one screen at a time; the server evaluates conditions and validation between screens (using the same `fiber-shared` engines) and assembles the final FlowData. The full flow definition never reaches the client — useful for sensitive branching logic and server-side integrations between screens.

### Style Families

`theme.style` is one flat vocabulary of ten values, but four of them — `centered-minimal`, `stacked-cards`, `soft-float`, `bold-statement` — share a presentation treatment: a vertically centered narrow column, staggered component entry, enlarged tap targets and bolder type. That shared half is the **focused** family; the other six are the **form** family.

The family is derived from the style, never authored — it does not appear in Flow JSON. FBRE emits it as `data-style-family`, and `fiber-types` exports `styleFamily()` / `FOCUSED_STYLES` for builders grouping a style picker. Because it derives from `theme.style`, the presentation travels with the theme through every render path, server-driven included.

A focused style is a *look*. Pacing — how many questions land on a screen — is a property of the flow's structure (see FBTL's `screenModel`), and the two advance behaviors below are separate `navigation` flags. FBTL authors focused-styled flows by default.

### Advance Behaviors

`navigation.autoAdvance` (default `false`) advances ~500ms after a single-select choice; `navigation.advanceOnEnter` (default `true`) advances when Enter is pressed in a single-line input. Both only ever fire on a screen with exactly one visible input, never on the last screen, and never on an invalid screen. Neither is tied to a style — a `clean` form with one question per screen advances on Enter just like a focused one.

### Confirmation Screen

`config.confirmation` (`{ show, title, body }`) defines a terminal "thank you" screen shown after submission. Title and body support inline and reference markup, resolved against collected values and context. The parent can also drive it dynamically: `onFlowComplete` may return nothing (`void`), a `Promise` (the submit button stays in-flight until it settles), or a `ConfirmationResult` (directly or via the Promise) that overrides the configured message — e.g., to show a server-generated reference number. A rejected Promise surfaces as a completion error and the confirmation is not shown.

### Hosted Platform

The private packages (**server**, **portal**, **hosted**) form a hosted form service built on the public libraries: tenant-scoped flow storage and publishing, custom preset/template storage, file upload hosting, submission management, session-based server-driven rendering, and a hosted form-filling app. The libraries remain free and self-sufficient; the platform is the product layered on top.
