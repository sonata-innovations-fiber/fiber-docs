---
title: FlowData Schema
applies-to:
  - "@sonata-innovations/fiber-types@^3.0"
  - "@sonata-innovations/fiber-fbre@^4.0"
read-when: "Consuming FlowData output from FBRE: structure, per-type value shapes, exclusion rules, containers, calculations."
---

# Fiber FlowData Schema

> Canonical source: [`flow-data-schema.json`](flow-data-schema.json)

FlowData is the output structure produced by FBRE (the render engine) when a form is completed. It contains the collected values for every visible, data-bearing component. The parent application receives FlowData via the `onFlowComplete` callback or the server-driven session completion endpoint.

> **In-memory vs serialized:** the object handed to `onFlowComplete` is a plain JavaScript object and may carry keys whose value is `undefined` (e.g. `value` on containers or an unticked `confirm`); once serialized with `JSON.stringify`, those keys are absent from the JSON entirely.

---

## FlowData (root)

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `uuid` | `string` | Yes | UUID of the flow this submission belongs to |
| `metadata` | `Record<string, string>` | Yes | Metadata from the original flow, passed through as-is |
| `screens` | [ScreenData](#screendata)[] | Yes | Ordered list of screens that were visible at completion time |
| `calculations` | [CalculationData](#calculationdata)[] | No | Evaluated calculation results. Present only when the flow defines calculations |

---

## ScreenData

Submitted data for a single screen.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `uuid` | `string` | Yes | UUID of the screen |
| `label` | `string` | No | Screen label. Present only when the screen has a non-empty label |
| `components` | [ComponentData](#componentdata)[] | Yes | Submitted data for each data-bearing component on this screen |

---

## ComponentData

Submitted data for a single component.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `uuid` | `string` | Yes | UUID of the component |
| `label` | `string` | No | Component label. Present only when the component has a non-empty label |
| `type` | `string` | Yes | Component type key (see [Value Types by Component](#value-types-by-component)) |
| `value` | *varies* | No | The collected value. Type depends on component type. Absent on containers (`group`, `repeater`) and on a ticked-then-unticked `confirm` (see [Assembly Rules](#assembly-rules)) |
| `components` | [ComponentData](#componentdata)[][] | No | Container iterations. Present on `group` and `repeater` components |

---

## Value Types by Component

The `value` field type depends on the component `type`. Display-only types (`header`, `text`, `divider`, `callout`, `table`) are never present in FlowData. Note that `computed` is **not** display-only — it appears in FlowData with its derived value.

| Component Type | Value Type | Example |
|----------------|-----------|---------|
| `inputText` | `string` | `"John Doe"` |
| `inputTextArea` | `string` | `"Long form text..."` |
| `inputNumber` | `number` | `42` |
| `dropDown` | `string` \| `number` | `"option-a"` |
| `dropDownMulti` | `string[]` \| `number[]` | `["opt-1", "opt-2"]` |
| `checkbox` | `string[]` \| `number[]` | `["check-a", "check-b"]` |
| `radio` | `string` \| `number` | `"radio-b"` |
| `rating` | `number` | `4` |
| `slider` | `number` | `75` |
| `toggleSwitch` | `boolean` | `true` |
| `fileUpload` | [FileUploadData](#fileuploaddata) | *(see below)* |
| `group` | *(no `value` key)* | children in `components` |
| `repeater` | *(no `value` key)* | iterations in `components` |
| `date` | `string` | `"2026-02-23"` |
| `time` | `string` | `"14:30"` |
| `dateTime` | `string` | `"2026-02-23T14:30"` |
| `dateRange` | [RangeValue](#rangevalue) | `{ "start": "2026-02-23", "end": "2026-02-28" }` |
| `timeRange` | [RangeValue](#rangevalue) | `{ "start": "09:00", "end": "17:00" }` |
| `dateTimeRange` | [RangeValue](#rangevalue) | `{ "start": "2026-02-23T09:00", "end": "2026-02-28T17:00" }` |
| `yesNo` | `string` | `"yes"` or `"no"` |
| `confirm` | `boolean` \| `null` \| *(absent)* | `true` when ticked; `null` when never touched; key absent when ticked then unticked |
| `cardSelect` | `string` \| `number` | `"premium-plan"` |
| `signature` | `string` | Typed name (`"Jane Smith"`) or drawn signature as a `data:image/png;base64,...` URL |
| `computed` | `number` \| `null` | `128.5` — derived value; `null` when the formula could not be evaluated |
| `colorPicker` | `string` | `"#FF5733"` |

> **Note:** `dropDown`, `radio`, `dropDownMulti`, `checkbox`, and `cardSelect` values match the `value` field of the selected `Option`(s) in the flow. When options use numeric values, the FlowData value will be `number` or `number[]` accordingly.

---

## Container Components (group and repeater)

Both container types — `group` and `repeater` — use the `components` field (a 2D array) to hold their children's data. Containers never have a `value` key: FBRE never assigns them a value, so after JSON serialization the key is absent (in the in-memory object it is `value: undefined`).

**Structure:** `components[iteration][child]`

- **Outer array** — one entry per iteration. A `group` has exactly one iteration; a `repeater` has one inner array per iteration the user added.
- **Inner array** — the child `ComponentData` entries for that iteration.

```
container.components = [
  [ /* iteration 0 */ child1, child2, child3 ],
  [ /* iteration 1 */ child1, child2, child3 ],
  ...
]
```

Display-only children (`header`, `text`, `divider`, `callout`, `table`) inside containers are excluded, just like at the screen level. Children hidden by conditions within an iteration are also excluded.

Nesting is expanded recursively: a `group` nested inside a repeater iteration carries its own `components` 2D array, following the same shape as a top-level container.

---

## FileUploadData

When no server-side storage is configured, files are returned as inline Base64; when S3 storage is configured, a reference to the stored file is returned instead. The two shapes are distinguished by the `mode` field: S3 references always carry `mode: "s3"`, while inline Base64 objects are emitted **without** a `mode` field.

### FileUploadBase64Data

Inline Base64-encoded file. This is the wire format FBRE actually emits — the `mode` field is not emitted (the TypeScript type allows an optional `"base64"` discriminant, but FBRE never sets it).

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `mode` | `"base64"` | No | Never emitted by FBRE; allowed by the TypeScript type as an optional discriminant |
| `name` | `string` | Yes | Original file name |
| `type` | `string` | Yes | MIME type (e.g. `"application/pdf"`, `"image/png"`) |
| `size` | `number` | Yes | File size in bytes |
| `lastModified` | `number` | Yes | Last modified timestamp (Unix milliseconds) |
| `lastModifiedDate` | `string` | No | Legacy `File.lastModifiedDate` as an ISO-8601 string, emitted only when the browser provides the property (deprecated File API) |
| `data` | `string` | Yes | Base64-encoded file content (data URL format) |

### FileUploadS3Data

Server-stored file reference.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `mode` | `"s3"` | Yes | Upload mode. Always `"s3"` for server-stored files |
| `fileId` | `string` | Yes | Server-assigned file identifier for retrieval |
| `name` | `string` | Yes | Original file name |
| `type` | `string` | Yes | MIME type of the file |
| `size` | `number` | Yes | File size in bytes |

---

## CalculationData

Evaluated result of a flow-level calculation.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `uuid` | `string` | Yes | UUID of the calculation |
| `label` | `string` | Yes | Human-readable label for the calculation |
| `value` | `number \| null` | Yes | Numeric result, or `null` if the formula could not be evaluated |
| `formattedValue` | `string` | No | Pre-formatted display string (e.g. `"$1,234.56"`, `"75.00%"`). Omitted when `value` is `null` |

---

## RangeValue

A start/end pair used by range components (`dateRange`, `timeRange`, `dateTimeRange`).

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `start` | `string` | Yes | Start value. Format matches the component type |
| `end` | `string` | Yes | End value. Same format as `start` |

### Value Formats by Range Type

| Component Type | `start` / `end` Format | Example |
|----------------|----------------------|---------|
| `dateRange` | `YYYY-MM-DD` | `"2026-02-23"` |
| `timeRange` | `HH:MM` (24h) | `"09:00"` |
| `dateTimeRange` | `YYYY-MM-DDTHH:MM` | `"2026-02-23T09:00"` |

---

## Assembly Rules

FlowData is assembled from the runtime state when the form is completed. The following rules govern what is included and excluded:

- **Display components excluded** — `header`, `text`, `divider`, `callout`, and `table` components are never included. They have no user-collected data. (`computed` is not in this set — it is included with its derived value.)
- **Hidden screens excluded** — Screens that are hidden by conditions at completion time are omitted entirely.
- **Hidden components excluded** — Components hidden by conditions at completion time are omitted, both at the screen level and inside container iterations.
- **Labels included only when non-empty** — The `label` field on `ScreenData` and `ComponentData` is present only when the source had a non-empty label string.
- **`components` field only on containers** — The `components` 2D array is present only on `group` and `repeater` entries. All other component types omit it.
- **`value` present on non-containers, with two exceptions** — Every non-container `ComponentData` entry has a `value` field (unanswered fields are `null`), except: containers (`group`, `repeater`) never have a `value` key, and a `confirm` that was ticked and then unticked has its `value` set to `undefined` in memory — so the key is absent after JSON serialization. A never-touched `confirm` is `null`.
- **In-memory vs serialized** — The object passed to `onFlowComplete` may carry `value: undefined` keys (containers, unticked `confirm`); serialized JSON omits those keys.
- **Screen order preserved** — Screens appear in the same order as the flow definition (minus hidden ones).
- **Component order preserved** — Components appear in the same order as they were defined on each screen.
- **Calculations included when present** — If the flow defines calculations (`flow.calculations`), the `calculations` array is included with each calculation's UUID, label, evaluated value, and (when the value is non-null) a pre-formatted display string. Omitted entirely when the flow has no calculations.
- **Nested containers expand recursively** — A `group` nested inside a repeater iteration carries its own `components` 2D array (see [Container Components](#container-components-group-and-repeater)).

---

## Complete Example

A realistic multi-screen FlowData submission:

```json
{
  "uuid": "flow-abc-123",
  "metadata": {
    "name": "Employee Onboarding",
    "description": "New hire information form"
  },
  "screens": [
    {
      "uuid": "screen-1",
      "label": "Personal Info",
      "components": [
        {
          "uuid": "comp-name",
          "label": "Full Name",
          "type": "inputText",
          "value": "Jane Smith"
        },
        {
          "uuid": "comp-dept",
          "label": "Department",
          "type": "dropDown",
          "value": "engineering"
        },
        {
          "uuid": "comp-skills",
          "label": "Skills",
          "type": "checkbox",
          "value": ["javascript", "python", "go"]
        },
        {
          "uuid": "comp-notifications",
          "label": "Enable Notifications",
          "type": "toggleSwitch",
          "value": true
        },
        {
          "uuid": "comp-start-date",
          "label": "Start Date",
          "type": "date",
          "value": "2026-03-15"
        },
        {
          "uuid": "comp-work-hours",
          "label": "Work Hours",
          "type": "timeRange",
          "value": { "start": "09:00", "end": "17:00" }
        },
        {
          "uuid": "comp-rating",
          "label": "Self-Assessed Skill Level",
          "type": "rating",
          "value": 4
        },
        {
          "uuid": "comp-experience",
          "label": "Years of Experience",
          "type": "slider",
          "value": 8
        },
        {
          "uuid": "comp-brand-color",
          "label": "Preferred Theme Color",
          "type": "colorPicker",
          "value": "#1976d2"
        },
        {
          "uuid": "comp-terms",
          "label": "I accept the terms",
          "type": "confirm",
          "value": true
        }
      ]
    },
    {
      "uuid": "screen-2",
      "label": "Emergency Contacts",
      "components": [
        {
          "uuid": "comp-has-contacts",
          "label": "Do you have an emergency contact?",
          "type": "yesNo",
          "value": "yes"
        },
        {
          "uuid": "comp-contacts",
          "label": "Contacts",
          "type": "repeater",
          "components": [
            [
              {
                "uuid": "comp-contact-name-0",
                "label": "Contact Name",
                "type": "inputText",
                "value": "John Smith"
              },
              {
                "uuid": "comp-contact-phone-0",
                "label": "Phone",
                "type": "inputText",
                "value": "+1-555-0100"
              }
            ],
            [
              {
                "uuid": "comp-contact-name-1",
                "label": "Contact Name",
                "type": "inputText",
                "value": "Mary Smith"
              },
              {
                "uuid": "comp-contact-phone-1",
                "label": "Phone",
                "type": "inputText",
                "value": "+1-555-0101"
              }
            ]
          ]
        }
      ]
    },
    {
      "uuid": "screen-3",
      "label": "Documents",
      "components": [
        {
          "uuid": "comp-resume",
          "label": "Resume",
          "type": "fileUpload",
          "value": {
            "mode": "s3",
            "fileId": "file-xyz-789",
            "name": "jane-smith-resume.pdf",
            "type": "application/pdf",
            "size": 245760
          }
        },
        {
          "uuid": "comp-signature",
          "label": "Signature",
          "type": "signature",
          "value": "data:image/png;base64,iVBORw0KGgo..."
        },
        {
          "uuid": "comp-score",
          "label": "Experience Score",
          "type": "computed",
          "value": 32
        }
      ]
    }
  ],
  "calculations": [
    {
      "uuid": "calc-total-exp",
      "label": "Total Experience Score",
      "value": 32,
      "formattedValue": "32.00"
    },
    {
      "uuid": "calc-bonus",
      "label": "Signing Bonus",
      "value": 5000,
      "formattedValue": "$5000.00"
    }
  ]
}
```

Note the `repeater` entry above: it has no `value` key — containers carry their data in the `components` 2D array only. A non-repeatable `group` looks the same but always has exactly one inner array.
