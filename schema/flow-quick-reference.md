---
title: Flow JSON Quick Reference
applies-to:
  - "@sonata-innovations/fiber-types@^2.2"
read-when: "Constructing or interpreting Flow JSON: structure, component types and value types, options, conditions, validation, widths, inline markup. For exhaustive per-property detail, use flow-schema.md instead."
---

# Flow JSON Quick Reference

A compact reference for constructing Flow JSON programmatically. The [Flow Schema](flow-schema.md) is the exhaustive per-property reference; the machine-validatable schema is [`flow-schema.json`](flow-schema.json).

## Schema

```
Flow
├── uuid: string                              # Unique flow identifier
├── metadata: { name?, description?, ... }    # Arbitrary key-value metadata
├── config?: FlowConfiguration                # Theme, mode, navigation, controls
├── calculations?: Calculation[]              # Named formulas over component values
└── screens: FlowScreen[]
    ├── uuid: string
    ├── label?: string
    ├── conditions?: FlowConditionConfig       # Show/hide this screen
    ├── nextButtonLabel?: string
    ├── backButtonLabel?: string
    └── components: Component[]
        ├── uuid: string
        ├── type: string                       # See type table below
        ├── properties: { label?, placeholder?, options?, validation?, width?, ... }
        ├── conditions?: FlowConditionConfig   # Show/hide this component
        └── components?: Component[]           # Group / repeater children only
```

## FlowConfiguration

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `mode` | `FlowModeType` | `"standard"` | `"standard"` or `"conversational"` — conversational mode centers content, auto-advances on selection, and Enter-advances on text inputs |
| `theme` | `ThemeConfig` | — | `{ color?, colorScheme?, style?, background?, surface?, text?, border?, radius?, fontFamily?, error?, success?, warning? }` — visual theme settings (palette knobs + preset). `fontFamily` is a CSS stack string **or** `{ family, src }` for a font FBRE loads itself |
| `navigation` | `NavigationConfig` | — | `{ transition?, allowInvalidTransition? }` — screen navigation settings |
| `controls` | `ControlsConfig` | — | `{ show?, layout?, showStepper?, stepperStyle? }` — navigation controls settings |
| `confirmation` | `ConfirmationConfig` | — | `{ show?, title?, body? }` — terminal thank-you screen (see [Confirmation Screen](../integration/fbre.md#confirmation-screen)) |
| `summary` | `boolean` | `false` | Show summary screen before completion |

## Component Type → Value Type Mapping

The registry has **30 component types**:

| Type | Category | Value in FlowData | Initial Value |
|------|----------|-------------------|---------------|
| `header` | Display | _(excluded)_ | — |
| `text` | Display | _(excluded)_ | — |
| `divider` | Display | _(excluded)_ | — |
| `callout` | Display | _(excluded)_ | — |
| `table` | Display | _(excluded)_ | — |
| `computed` | Display | `number` (derived) | `null` |
| `inputText` | Input | `string` | `null` |
| `inputTextArea` | Input | `string` | `null` |
| `inputNumber` | Input | `number` | `null` |
| `dropDown` | Selection | `string \| number` | `null` (or `defaultValue` if it matches an option) |
| `dropDownMulti` | Selection | `(string \| number)[]` | `[]` |
| `checkbox` | Selection | `(string \| number)[]` | `[]` |
| `radio` | Selection | `string \| number` | `null` (or `defaultValue` if it matches an option) |
| `cardSelect` | Selection | `string \| number` | `null` (or `defaultValue` if it matches an option) |
| `toggleSwitch` | Selection | `boolean` | `false` |
| `yesNo` | Selection | `string` | `null` (or `defaultValue` if it matches an option) |
| `confirm` | Selection | `true` when ticked; key absent after untick | `null` |
| `date` | Date/Time | `string` | `null` |
| `time` | Date/Time | `string` | `null` |
| `dateTime` | Date/Time | `string` | `null` |
| `dateRange` | Date/Time | `RangeValue` (`{ start, end }`) | `null` |
| `timeRange` | Date/Time | `RangeValue` (`{ start, end }`) | `null` |
| `dateTimeRange` | Date/Time | `RangeValue` (`{ start, end }`) | `null` |
| `fileUpload` | Interactive | `FileUploadData` | `null` |
| `rating` | Interactive | `number` | `null` |
| `slider` | Interactive | `number` | `initValue` or `min` or `0` |
| `colorPicker` | Interactive | `string` | `defaultValue` or `null` |
| `signature` | Interactive | `string` (typed name or `data:` image URL) | `null` |
| `group` | Container | container entry with nested `components[][]`; no own `value` | — |
| `repeater` | Container | container entry with nested `components[][]` (one inner array per iteration); no own `value` | — |

Notes:
- Display components and hidden (condition-suppressed) components/screens are **excluded** from FlowData; `computed` is included.
- Option-bearing types (`dropDown`, `radio`, etc.) never default to the first option — the initial value is `null` unless `properties.defaultValue` matches an option's `value`.
- Container entries (`group`, `repeater`) are emitted **with** their children nested under `components: ComponentData[][]`; the container itself carries no `value` key.

## Options Format

Selection components (`dropDown`, `dropDownMulti`, `checkbox`, `radio`) use:

```json
{
  "properties": {
    "options": [
      { "label": "Display Text", "value": "stored-value" },
      { "label": "Another Option", "value": "another" }
    ]
  }
}
```

Options **must** be `{ label, value }` objects. Bare strings are normalized automatically but should not be used when constructing flows.

## Conditions Format (FlowConditionConfig)

Applied to `component.conditions` or `screen.conditions`:

```json
{
  "action": "show",
  "when": {
    "logic": "and",
    "rules": [
      { "source": "other-component-uuid", "operator": "equals", "value": "yes" }
    ]
  }
}
```

- `action`: `"show"` or `"hide"`
- `logic`: `"and"` (all rules must match) or `"or"` (any rule matches)
- `source`: UUID of the component whose value is tested; rules may also set `sourceType: "context"` to test a key from FBRE's `context` prop instead of a component
- **19 operators**: `equals`, `notEquals`, `contains`, `notContains`, `startsWith`, `endsWith`, `isEmpty`, `isNotEmpty`, `greaterThan`, `greaterThanOrEqual`, `lessThan`, `lessThanOrEqual`, `isOneOf`, `isNotOneOf`, `includesAny`, `includesAll`, `includesNone`, `isTrue`, `isFalse`
- `value` field: required for most operators; omitted for `isEmpty`, `isNotEmpty`, `isTrue`, `isFalse`

## Validation Format (FlowValidationConfig)

Placed at `component.properties.validation`:

```json
{
  "validation": {
    "rules": [
      { "type": "required" },
      { "type": "email", "message": "Custom error message" },
      { "type": "minLength", "params": { "min": 5 } }
    ]
  }
}
```

**17 validators:**

| Type | Params | Description |
|------|--------|-------------|
| `required` | — | Field must have a value |
| `email` | — | Valid email format |
| `phone` | — | Valid phone format |
| `url` | — | Valid URL format |
| `minLength` | `{ min: number }` | Minimum character count |
| `maxLength` | `{ max: number }` | Maximum character count |
| `exactLength` | `{ length: number }` | Exact character count |
| `minValue` | `{ min: number }` | Minimum numeric value |
| `maxValue` | `{ max: number }` | Maximum numeric value |
| `pattern` | `{ regex: string }` | Regex pattern match — the param key is `regex`, **not** `pattern` (a `pattern` key is ignored and the rule silently always passes) |
| `minSelected` | `{ min: number }` | Minimum selected options (checkbox/multi-select) |
| `maxSelected` | `{ max: number }` | Maximum selected options |
| `fileType` | `{ types: string[] }` | Allowed MIME types |
| `fileSize` | `{ max: number }` | Max file size in bytes |
| `contains` | `{ text: string }` | Must contain substring |
| `excludes` | `{ text: string }` | Must not contain substring |
| `matchesField` | `{ field: string }` | Must match another component's value (by UUID) |

## Width Values (ComponentWidth)

Set on `component.properties.width`:

| Value | CSS Width |
|-------|-----------|
| `"full"` (default) | `100%` |
| `"half"` | `~50%` |
| `"third"` | `~33.33%` |
| `"two-thirds"` | `~66.67%` |
| `"quarter"` | `~25%` |
| `"three-quarters"` | `~75%` |

Fractional widths are gap-adjusted `calc()` values (e.g. half is `calc(50% - gap/2)`). There is no global full-width collapse breakpoint; a few controls compact themselves at narrow container widths.

## Inline Markup

`text` components and `detail` property fields support:
- `[b]bold[/b]` → **bold**
- `[i]italic[/i]` → *italic*
- `[l href="url"]link text[/l]` → hyperlink
- `[s size="sm|lg|xl"]sized text[/s]` → font-size span (12/18/24px)
- Newlines render as line breaks

Text-bearing fields (including the confirmation screen's `title`/`body`) also resolve `${uuid}` **references** against calculation results, component values, and FBRE's `context` prop, in that order.

## Rules for Constructed Flows

- Use `{ label, value }` objects for `options` arrays, not bare strings
- Do not set runtime-only fields (`value`, `valid`, `addedComponents`) in Flow JSON — these are managed by FBRE at runtime
- Every `uuid` must be unique within the flow; condition `source` and validation `matchesField` reference components by uuid
- Validate against [`flow-schema.json`](flow-schema.json) (`npm run validate -- <file>` in this repo)
