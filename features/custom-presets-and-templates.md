---
title: Custom Presets & Templates (FBT)
applies-to:
  - "@sonata-innovations/fiber-fbt@^3.0"
  - "@sonata-innovations/fiber-types@^4.0"
read-when: "Extending FBT's pool with custom presets/templates: data-based vs factory definitions, icon catalog, collision rules, server-stored definitions."
---

# Custom Presets & Templates

Parent applications extend FBT's pool with custom **presets** and **templates** via the optional `customPresets` and `customTemplates` props. This is the deep-dive companion to the short "Custom Presets & Templates" section in `docs/integration/fbt.md`.

## Concepts

- A **preset** is one or more pre-configured components. Dropping it onto the stage batch-adds those components to the **current screen** (at the drop position if dropped over an existing component, otherwise appended).
- A **template** is a complete `Flow`. Dropping it **merges** into the current flow: its screens are **appended** to the existing screen list via `mergeFlow`, and the first appended screen becomes the active screen. Nothing is replaced — the existing screens, metadata, and config are untouched, and the template's own `metadata`/`config` are ignored by the merge (only its screens are ingested).

Whether a definition is treated as a preset or a template is determined by **which prop array it arrives in** (`customPresets` vs `customTemplates`), not by its `type` string.

## Two definition kinds, one set of props

Both props accept a union type:

```ts
// fbt/src/types/custom.ts
export type AnyPresetDefinition = CustomPresetDefinition | PresetData;
export type AnyTemplateDefinition = CustomTemplateDefinition | TemplateData;
```

- **Data-based** (`PresetData` / `TemplateData`, exported from `@sonata-innovations/fiber-types`) — plain serializable JSON. **Preferred**: it can be stored in a database, sent over the wire, and edited without code. FBT regenerates UUIDs automatically on every drop.
- **Factory-based** (`CustomPresetDefinition` / `CustomTemplateDefinition`, exported from `@sonata-innovations/fiber-fbt`) — a `factory()` function called on every drop. Kept for backward compatibility.

FBT distinguishes the two kinds structurally (`"factory" in definition`), so you can mix both kinds in the same array.

## Data-based definitions (primary)

### Shapes

```ts
// @sonata-innovations/fiber-types
export type PresetData = {
  type: string;       // unique identifier (see collision rules below)
  label: string;      // display name in the pool
  icon: string;       // key into FBT's ICON_MAP (see icon catalog below)
  components: Component[];
};

export type TemplateData = {
  type: string;
  label: string;
  icon: string;
  flow: Flow;
};
```

### UUID regeneration semantics

The UUIDs you write into a data-based definition are **placeholders**. On every drop, FBT calls `regenerateComponentUUIDs` (presets) or `regenerateFlowUUIDs` (templates) from `@sonata-innovations/fiber-types`:

- Every component gets a fresh UUID; `properties` are deep-cloned (`structuredClone`), and nested `components` arrays (groups) are regenerated recursively.
- For templates, the flow UUID and every screen UUID are also replaced.
- The original definition object is never mutated, so the same definition can be dropped any number of times without duplicate-UUID bugs.

This means placeholder UUIDs only need to be unique **within** the definition (so group nesting is unambiguous); any string works.

### Full preset example

`Component` is a discriminated union keyed on `type` — `properties` is checked against the shape for that `type` (e.g. `label` is required for `"inputText"`, `value` for `"header"`, `options` for `"dropDown"`). Group children live on the component's top-level `components` field, not inside `properties`.

```ts
import type { PresetData } from "@sonata-innovations/fiber-types";

const ssnPreset: PresetData = {
  type: "custom-ssn",
  label: "SSN",
  icon: "ssn",
  components: [
    {
      uuid: "ssn-field",
      type: "inputText",
      properties: {
        label: "Social Security Number",
        placeholder: "XXX-XX-XXXX",
        validation: {
          rules: [
            { type: "required" },
            // NOTE: the param key is `regex`, not `pattern`
            { type: "pattern", params: { regex: "^\\d{3}-\\d{2}-\\d{4}$" } },
          ],
        },
      },
    },
  ],
};

const emergencyContactPreset: PresetData = {
  type: "custom-emergencyContact",
  label: "Emergency Contact",
  icon: "emergencyContact",
  components: [
    {
      uuid: "ec-group",
      type: "group",
      properties: { label: "Emergency Contact", showLabel: true },
      components: [
        {
          uuid: "ec-name",
          type: "inputText",
          properties: {
            label: "Contact Name",
            width: "half",
            validation: { rules: [{ type: "required" }] },
          },
        },
        {
          uuid: "ec-phone",
          type: "inputText",
          properties: {
            label: "Contact Phone",
            placeholder: "(555) 555-5555",
            inputType: "tel",
            width: "half",
            validation: { rules: [{ type: "required" }, { type: "phone" }] },
          },
        },
        {
          uuid: "ec-relationship",
          type: "dropDown",
          properties: {
            label: "Relationship",
            options: [
              { label: "Spouse", value: "spouse" },
              { label: "Parent", value: "parent" },
              { label: "Sibling", value: "sibling" },
              { label: "Friend", value: "friend" },
              { label: "Other", value: "other" },
            ],
            validation: { rules: [{ type: "required" }] },
          },
        },
      ],
    },
  ],
};
```

### Full template example

```ts
import type { TemplateData } from "@sonata-innovations/fiber-types";

const onboardingTemplate: TemplateData = {
  type: "custom-employeeOnboarding",
  label: "Employee Onboarding",
  icon: "employeeOnboarding",
  flow: {
    uuid: "onboarding-flow",
    metadata: { name: "Employee Onboarding" },
    config: {},
    screens: [
      {
        uuid: "onboarding-screen-1",
        label: "Personal Info",
        components: [
          {
            uuid: "ob-header-1",
            type: "header",
            properties: { value: "Employee Information" },
          },
          {
            uuid: "ob-first-name",
            type: "inputText",
            properties: {
              label: "First Name",
              width: "half",
              validation: { rules: [{ type: "required" }] },
            },
          },
          {
            uuid: "ob-last-name",
            type: "inputText",
            properties: {
              label: "Last Name",
              width: "half",
              validation: { rules: [{ type: "required" }] },
            },
          },
          {
            uuid: "ob-email",
            type: "inputText",
            properties: {
              label: "Email Address",
              placeholder: "name@company.com",
              inputType: "email",
              validation: { rules: [{ type: "required" }, { type: "email" }] },
            },
          },
        ],
      },
      {
        uuid: "onboarding-screen-2",
        label: "Employment Details",
        components: [
          {
            uuid: "ob-header-2",
            type: "header",
            properties: { value: "Employment Details" },
          },
          {
            uuid: "ob-job-title",
            type: "inputText",
            properties: {
              label: "Job Title",
              validation: { rules: [{ type: "required" }] },
            },
          },
          {
            uuid: "ob-employment-type",
            type: "dropDown",
            properties: {
              label: "Employment Type",
              options: [
                { label: "Full-Time", value: "full-time" },
                { label: "Part-Time", value: "part-time" },
                { label: "Contract", value: "contract" },
                { label: "Intern", value: "intern" },
              ],
              validation: { rules: [{ type: "required" }] },
            },
          },
        ],
      },
    ],
  },
};
```

Remember: dropping this template does **not** overwrite the builder's current flow — its two screens are appended after the existing ones, and "Personal Info" becomes the active screen. The `metadata.name` above documents the template but is not merged into the current flow.

### Passing to FBT

```tsx
import { FBT } from "@sonata-innovations/fiber-fbt";
import "@sonata-innovations/fiber-fbt/styles";
import type { AnyPresetDefinition, AnyTemplateDefinition } from "@sonata-innovations/fiber-fbt";

const MY_PRESETS: AnyPresetDefinition[] = [ssnPreset, emergencyContactPreset];
const MY_TEMPLATES: AnyTemplateDefinition[] = [onboardingTemplate];

function App() {
  return (
    <FBT
      flow={existingFlow}
      onFlowChange={handleFlowChange}
      customPresets={MY_PRESETS}
      customTemplates={MY_TEMPLATES}
    />
  );
}
```

Both props are optional; if omitted, FBT shows only its built-in items. Keep the arrays referentially stable (module constants or memoized) — the context memoizes its lookup maps on the array identities.

## Factory-based definitions (back-compat)

```ts
// @sonata-innovations/fiber-fbt
export type CustomPresetDefinition = {
  type: string;
  label: string;
  icon: string;
  factory: () => Component[];
};

export type CustomTemplateDefinition = {
  type: string;
  label: string;
  icon: string;
  factory: () => Flow;
};
```

With a factory, **you** are responsible for returning fresh UUIDs on every call — FBT does not regenerate UUIDs for factory-based definitions. You would still use a factory when the definition must be computed at drop time (e.g. injecting the current date, user data, or environment-dependent options):

```ts
import type { CustomPresetDefinition } from "@sonata-innovations/fiber-fbt";

const visitDatePreset: CustomPresetDefinition = {
  type: "custom-visitDate",
  label: "Visit Date",
  icon: "date",
  factory: () => [
    {
      uuid: crypto.randomUUID(),
      type: "date",
      properties: {
        label: "Visit Date",
        min: new Date().toISOString().slice(0, 10), // today, computed at drop
        validation: { rules: [{ type: "required" }] },
      },
    },
  ],
};
```

For anything static, prefer `PresetData`/`TemplateData`.

## How resolution works

1. **Context injection** — the FBT provider builds `presetMap` / `templateMap` (`Map<string, () => Component[] | Flow>`) from the prop arrays, keyed by `type`. Factory definitions map to their `factory`; data definitions map to a wrapper that regenerates UUIDs. If two custom definitions share a `type`, the later one in the array wins (plain `Map.set` overwrite).
2. **Kind by prop array** — items from `customPresets` are tagged `kind: "preset"`, items from `customTemplates` are tagged `kind: "template"` when the pool renders them. The `type` string plays no part in this.
3. **Built-in-first lookup at drop** — the DnD handler resolves the dropped `type` against the **built-in** definitions first, and only falls back to the custom map if that returns nothing:

   ```ts
   // fbt/src/ui/dnd/dnd-wrapper.tsx
   const flow = createTemplate(item.type) ?? templateMap.get(item.type)?.();
   const components = createPreset(item.type) ?? presetMap.get(item.type)?.();
   ```

### Collision rules

There is **no required prefix** on custom `type` strings — FBT accepts any string. But because built-in lookup runs first, a custom definition whose `type` equals a built-in key is **silently shadowed**: your pool entry shows your label and icon, but dropping it produces the built-in content.

Built-in keys to avoid — presets: `preset-phone`, `preset-email`, `preset-date`, `preset-url`, `preset-fullName`, `preset-dollarAmount`, `preset-password`, `preset-addressBlock`, `preset-contactInfo`, `preset-nameFields`, `preset-agreement`; templates: `template-contactForm`, `template-signUp`, `template-patientIntake`, `template-feedbackSurvey`.

**Recommendation:** use a distinct namespace such as `custom-*`. No built-in uses it, and it matches what the Fiber server enforces for stored definitions (see below).

## Icon catalog

`icon` is a key into FBT's `ICON_MAP` (`fbt/src/lib/icons.tsx`) — 46 built-in SVG icons. Icon keys are their own namespace: they often mirror component types but are not the same set (e.g. the paragraph-text icon is `descriptionText` while the component type is `text`; `select` is the dropdown icon). If the key is unknown, the item renders with an empty icon slot — no error, graceful fallback.

### Display & structure

| Key | Glyph |
|-----|-------|
| `header` | Heading lines |
| `descriptionText` | Paragraph lines |
| `divider` | Horizontal rule |
| `callout` | Info box |
| `table` | Grid |

### Text & number inputs

| Key | Glyph |
|-----|-------|
| `textInput` | Single-line field |
| `textArea` | Multi-line field |
| `numberInput` | Field with `#` |
| `dollarAmount` | Field with `$` |
| `phone` | Phone handset/device |
| `email` | Envelope |
| `url` | Chain link |
| `password` | Padlock |

### Selection

| Key | Glyph |
|-----|-------|
| `select` | Dropdown |
| `multiSelect` | Checked list |
| `checkbox` | Checked box |
| `radio` | Radio dot |
| `toggleSwitch` | Switch |
| `yesNo` | Check/cross pair |
| `confirm` | Checked box (shares the `checkbox` glyph) |
| `cardSelect` | Card grid |

### Date & time

| Key | Glyph |
|-----|-------|
| `date` | Calendar |
| `time` | Clock |
| `dateTime` | Calendar + clock |
| `dateRange` | Two calendars |
| `timeRange` | Two clocks |
| `dateTimeRange` | Two calendar+clock pairs |

### Interactive & special

| Key | Glyph |
|-----|-------|
| `fileUpload` | Upload arrow |
| `rating` | Star |
| `slider` | Slider track |
| `colorPicker` | Color wheel |
| `signature` | Signature stroke |
| `repeater` | Stacked dashed rows |
| `calculated` | `fx` in a box |

### People & preset-flavored

| Key | Glyph |
|-----|-------|
| `fullName` | Person |
| `addressBlock` | Map pin |
| `contactInfo` | Contact card |
| `nameFields` | Paired fields |
| `agreement` | Document with check |
| `ssn` | Masked field (`***`) |
| `emergencyContact` | Person with plus |

### Template-flavored

| Key | Glyph |
|-----|-------|
| `signUpForm` | Form with avatar |
| `patientIntake` | Bulleted form |
| `feedbackSurvey` | Form with star |
| `contactForm` | Form lines |
| `employeeOnboarding` | Document lines |

## Server-stored definitions

The Fiber server (private package, not published) provides tenant-scoped storage for data-based definitions:

- CRUD endpoints at `/api/v1/presets` and `/api/v1/templates`, storing `PresetData` / `TemplateData` JSON per tenant.
- The server **rejects** any stored definition whose `type` does not start with `custom-` — this prefix guarantees no collision with built-in keys.
- The portal provides management pages for presets and templates, and its flow editor fetches the tenant's custom definitions on mount.

From FBT's perspective there is nothing special about server-stored definitions: the parent app fetches them and passes them through the same `customPresets` / `customTemplates` props as data-based definitions.

## UI behavior

- When either prop array is non-empty, the corresponding pool tab (Presets or Templates) appends a **CUSTOM** section after the built-in sections, containing one item per definition.
- Custom items carry a **star badge** in place of the colored accent that built-in preset/template items get; search results label them with a "Custom" badge.
- Items are drag-and-drop onto the stage. Icon resolution is `ICON_MAP[item.icon]` guarded by `{Icon && <Icon />}` — an unknown key renders nothing.
