# Fiber Documentation

Fiber is a system for building and rendering data-collection forms. Builders (FBT, FBTL) author a JSON document called a Flow; the render engine (FBRE) renders it and returns collected data (FlowData). All libraries are standard React components published on npm.

## Quick start

Render a form in about two minutes. Install the render engine:

```bash
npm install @sonata-innovations/fiber-fbre
```

A **Flow** is plain JSON — screens of components, each with a uuid, a type, and properties. Note the third field: it declares a *condition*, so it appears only when the answer above it is "yes". That logic lives in the document, not in your components.

```json
{
  "uuid": "flow-quickstart",
  "metadata": {
    "name": "Contact us"
  },
  "screens": [
    {
      "uuid": "screen-about-you",
      "label": "About you",
      "components": [
        {
          "uuid": "comp-name",
          "type": "inputText",
          "properties": {
            "label": "Your name",
            "validation": {
              "rules": [
                {
                  "type": "required"
                }
              ]
            }
          }
        },
        {
          "uuid": "comp-newsletter",
          "type": "yesNo",
          "properties": {
            "label": "Subscribe to our newsletter?"
          }
        },
        {
          "uuid": "comp-email",
          "type": "inputText",
          "properties": {
            "label": "Email address",
            "inputType": "email",
            "validation": {
              "rules": [
                {
                  "type": "required"
                },
                {
                  "type": "email"
                }
              ]
            }
          },
          "conditions": {
            "action": "show",
            "when": {
              "logic": "and",
              "rules": [
                {
                  "source": "comp-newsletter",
                  "operator": "equals",
                  "value": "yes"
                }
              ]
            }
          }
        }
      ]
    }
  ]
}
```

Hand it to `<FBRE />`. When the user finishes, you get the collected **FlowData** back:

```tsx
import { FBRE } from "@sonata-innovations/fiber-fbre";
import "@sonata-innovations/fiber-fbre/styles"; // required — styles are not bundled with the JS
import flow from "./quickstart.flow.json";

export default function App() {
  return <FBRE flow={flow} onFlowComplete={(data) => console.log(data)} />;
}
```

That's the whole loop: **a Flow goes in, FlowData comes out.** Conditional logic, validation, and screen transitions are handled for you — the email field is required and format-checked, and it is skipped entirely when hidden.

This flow is [`examples/quickstart.flow.json`](examples/quickstart.flow.json), validated against the schema on every commit. See [all examples](examples/) for conditions, calculations, repeaters, file uploads, and theming.

Next: [Fiber Concepts](fiber-concepts.md) for the mental model, or the [FBRE integration guide](integration/fbre.md) for the full API. To let people *author* flows instead of hand-writing JSON, add a builder — [FBT](integration/fbt.md) for developers and form designers, [FBTL](integration/fbtl.md) for non-technical end users.

## Using these docs with an AI assistant

Point your assistant at [`llms.txt`](https://raw.githubusercontent.com/sonata-innovations/fiber-docs/main/llms.txt) — a titled, described index of every document, per the [llmstxt.org](https://llmstxt.org) convention — and it will fetch only the pages it needs. For a single paste into a context window, use [`llms-full.txt`](https://raw.githubusercontent.com/sonata-innovations/fiber-docs/main/llms-full.txt) (the entire corpus in one file).

Flow JSON is machine-validatable at a stable URL, with no install and no auth. Validate anything you generate rather than trusting it by inspection:

```bash
npx ajv-cli validate --spec=draft2020 \
  -s https://raw.githubusercontent.com/sonata-innovations/fiber-docs/main/schema/flow-schema.json \
  -d my-flow.json
```

## Packages

All packages are published to the public npm registry under the `@sonata-innovations` scope.

| Package | What it is |
|---|---|
| [`@sonata-innovations/fiber-fbre`](https://www.npmjs.com/package/@sonata-innovations/fiber-fbre) | Render engine — consumes a Flow, renders the form, returns FlowData |
| [`@sonata-innovations/fiber-fbt`](https://www.npmjs.com/package/@sonata-innovations/fiber-fbt) | Full drag-and-drop builder for power users |
| [`@sonata-innovations/fiber-fbtl`](https://www.npmjs.com/package/@sonata-innovations/fiber-fbtl) | Lite builder for non-technical end users |
| [`@sonata-innovations/fiber-theme-editor`](https://www.npmjs.com/package/@sonata-innovations/fiber-theme-editor) | Visual theme editor widget |
| [`@sonata-innovations/fiber-types`](https://www.npmjs.com/package/@sonata-innovations/fiber-types) | Schema types, and the canonical JSON Schema |
| [`@sonata-innovations/fiber-shared`](https://www.npmjs.com/package/@sonata-innovations/fiber-shared) | Condition, validation, markup, and formula engines |

Every package also ships these docs inside its own tarball, under `node_modules/<package>/docs/`, alongside an `AGENTS.md` routing guide. **Those copies always match the version you installed** — prefer them over this mirror once you have the package.

## Core concepts

- [Fiber Concepts](fiber-concepts.md) — First contact with Fiber: the Flow/Screen/Component model, FlowData, builders vs render engine, conditions/validation/calculations concepts.

## Integration guides

Embedding a Fiber package in a parent application.

- [FBRE Integration Guide](integration/fbre.md) — Embedding the render engine in a parent app: props and modes, onFlowComplete contract, confirmation screen, store access, events, theming, pre-populating data.
- [FBT Integration Guide](integration/fbt.md) — Embedding the full builder in a parent app: getting the Flow out, composable layout, custom presets and templates.
- [FBTL Integration Guide](integration/fbtl.md) — Embedding the lite builder in a parent app: controlled-component contract, scope, screen model, narrowing the palette, save lifecycle, normalizing generator-authored flows.
- [Theme Editor Integration Guide](integration/theme-editor.md) — Embedding the plug-and-play theming widget: controlled value contract, knob sets, defaults, modal pattern, passing the theme to FBRE/FBT/FBTL.

## Schema reference

The Flow (input) and FlowData (output) contracts, in prose and as JSON Schema.

- [FlowData Schema](schema/flow-data-schema.md) — Consuming FlowData output from FBRE: structure, per-type value shapes, exclusion rules, containers, calculations.
- [Flow JSON Quick Reference](schema/flow-quick-reference.md) — Constructing or interpreting Flow JSON: structure, component types and value types, options, conditions, validation, widths, inline markup. For exhaustive per-property detail, use flow-schema.md instead.
- [Flow JSON Schema Reference](schema/flow-schema.md) — Exhaustive per-property reference for Flow JSON: every component type, property, condition, validation rule, calculation, and config field. For a compact overview use flow-quick-reference.md.

## Feature guides

Deep dives on individual capabilities.

- [Confirmation ("Thank You") Screen](features/confirmation-screen.md) — Configuring the post-submit thank-you screen: config.confirmation, dynamic ${...} content, and the onFlowComplete return contract.
- [Custom Presets & Templates (FBT)](features/custom-presets-and-templates.md) — Extending FBT's pool with custom presets/templates: data-based vs factory definitions, icon catalog, collision rules, server-stored definitions.
- [FBRE Theming Guide](features/fbre-theming.md) — Theming FBRE-rendered forms: colorScheme presets, palette knobs, loading a brand font, raw --fbre-* token overrides, precedence model, full token catalog.
- [Style Families & Advance Behaviors](features/style-families.md) — Choosing a form style, understanding the focused presentation, or turning auto-advance / Enter-to-advance on and off. Also: migrating off config.mode.

## Example flows

Real Flow documents, each validated against the schema on every commit. Copy one and render it.

| Example | Screens | What it shows |
|---|---|---|
| [`conditions-full.flow.json`](examples/conditions-full.flow.json) | 9 | Conditions — Full Operator Demo |
| [`confirmation-demo.flow.json`](examples/confirmation-demo.flow.json) | 1 | Confirmation Demo |
| [`contact-us.flow.json`](examples/contact-us.flow.json) | 1 | Contact Us |
| [`conversational-onboarding.flow.json`](examples/conversational-onboarding.flow.json) | 9 | Conversational Onboarding |
| [`datetime-demo.flow.json`](examples/datetime-demo.flow.json) | 3 | Date/Time & New Components Demo |
| [`feedback.flow.json`](examples/feedback.flow.json) | 2 | Feedback |
| [`file-upload.flow.json`](examples/file-upload.flow.json) | 2 | File Upload |
| [`lead-generation.flow.json`](examples/lead-generation.flow.json) | 2 | Lead Generation |
| [`one-of-each.flow.json`](examples/one-of-each.flow.json) | 5 | One of Each |
| [`patient-intake.flow.json`](examples/patient-intake.flow.json) | 4 | Patient Intake Form |
| [`quickstart.flow.json`](examples/quickstart.flow.json) | 1 | Contact us |
| [`quote-builder.flow.json`](examples/quote-builder.flow.json) | 3 | Project Quote Builder |
| [`screen-transitions.flow.json`](examples/screen-transitions.flow.json) | 4 | Screen Transitions Demo |
| [`simple-thank-you.flow.json`](examples/simple-thank-you.flow.json) | 1 | Simple Thank You |
| [`theme-preview.flow.json`](examples/theme-preview.flow.json) | 1 | Theme Preview |
| [`theme-reference.flow.json`](examples/theme-reference.flow.json) | 5 | Theme Reference |

## Machine-readable schema

| File | Purpose |
|---|---|
| [`schema/flow-schema.json`](schema/flow-schema.json) | JSON Schema (draft 2020-12) for a Flow — the input to the render engine |
| [`schema/flow-data-schema.json`](schema/flow-data-schema.json) | JSON Schema for FlowData — the output |

## About this repository

This is a **generated mirror**, published from the `docs/` tree of the Fiber source repo and updated automatically on every change. Please don't open pull requests here — they can't be merged upstream. Corrections and questions are welcome as [issues](https://github.com/sonata-innovations/fiber-docs/issues).

## License

See [LICENSE](LICENSE).
