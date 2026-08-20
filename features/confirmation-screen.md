---
title: Confirmation ("Thank You") Screen
applies-to:
  - "@sonata-innovations/fiber-fbre@^3.3"
  - "@sonata-innovations/fiber-fbt@^2.2"
  - "@sonata-innovations/fiber-shared@^1.0"
read-when: "Configuring the post-submit thank-you screen: config.confirmation, dynamic ${...} content, and the onFlowComplete return contract."
---

<!-- Spec: consumed by FBRE (fbre/src/ui/confirmation), FBT (editor-settings), fiber-types (ConfirmationConfig) -->

# Confirmation ("Thank You") Screen

A terminal screen shown **after** a form is submitted — a centered, formatted thank-you message. It is a flow-level setting, **not** a data-collection screen, so it never appears in the FBT/FBTL stage editor and never collects input.

- **Where it lives:** `flow.config.confirmation` (a serializable config group, alongside `theme` / `navigation` / `controls` / `summary`).
- **When it shows:** once the flow is submitted and `onFlowComplete` settles.
- **What it renders:** `title` + `body`, centered, with the controls and stepper hidden.

## Quick start

```ts
const flow = {
  ...myFlow,
  config: {
    ...myFlow.config,
    confirmation: {
      show: true,
      title: "Thank you!",
      body: "Your response has been submitted.",
    },
  },
};

<FBRE flow={flow} onFlowComplete={handleComplete} />;
```

In **local and remote modes** it renders **only** when `show` is not `false` **and** `title` or `body` has content. There is no built-in default copy — an enabled-but-empty confirmation shows nothing. Server-driven mode differs; see [Per-mode behavior](#per-mode-behavior).

### `ConfirmationConfig`

| Field   | Type      | Description                                                     |
| ------- | --------- | -------------------------------------------------------------- |
| `show`  | `boolean` | Explicit off-switch. Content presence is the positive gate.    |
| `title` | `string`  | Heading. Supports `${...}` references and text formatting.     |
| `body`  | `string`  | Message. Supports `${...}` references and text formatting.     |

## Dynamic content

`title` and `body` run through the same `${...}` reference markup as `text` / `header` / `callout` components. References resolve, in order, against:

1. **Calculations** — `${<calculationUuid>}`
2. **Collected field values** — `${<componentUuid>}` (what the user just entered)
3. **External context** — `${<contextKey>}` (values the parent passed via the `context` prop, by name)

```ts
// config.confirmation.body — mixes a collected field and a context value
"Thanks ${q_name}! A copy is on its way to ${q_email}. Our team at ${support_email} will follow up.";
```

```tsx
// q_name / q_email are component UUIDs; support_email comes from context
<FBRE flow={flow} context={{ support_email: "support@acme.com" }} onFlowComplete={handleComplete} />
```

> In FBT, the reference picker inserts field/calculation UUIDs for you. Context keys aren't in the picker yet — type `${key}` by hand for those. Resolved values are HTML-escaped (safe against injection).

### Building your own picker

Writing `"Thanks, ${...}!"` means knowing a component uuid, so a host app that offers its own picker ends up walking the flow to build the list. `resolvableReferences` does that walk once, in the order above, so a picker cannot drift from what the renderer will actually resolve:

```ts
import { resolvableReferences } from "@sonata-innovations/fiber-shared";

const refs = resolvableReferences(flow, {
  // Context keys can't be discovered from the flow — name the ones you pass.
  contextKeys: ["support_email"],
});
// [{ uuid, label, kind: "calculation" | "field" | "context",
//    componentType?, parentType?, screenUUID?, screenLabel? }, …]
```

Entries come back in resolution order — calculations, then fields in document order, then context keys — so the first match a picker offers is the one that wins at render time. Display-only components (headings, paragraphs, dividers) are omitted: they never contribute a value, so `${their-uuid}` resolves to nothing. Fields inside a group or repeater are included and carry `parentType`. Pass `exclude` to keep a field from being offered a reference to itself.

## Deciding *when* it shows: the `onFlowComplete` contract

The confirmation appears when `onFlowComplete` settles. Its return value drives the behavior:

```ts
onFlowComplete: (data: FlowData) =>
  void | ConfirmationResult | Promise<void | ConfirmationResult>;
```

| You return… | What happens |
| --- | --- |
| `void` | Confirmation shows immediately (optimistic). |
| `Promise<void>` | Submit button stays in its **in-flight / loading** state until the promise settles. Resolve → confirmation shows. **Reject → a completion error renders and the confirmation is _not_ shown.** |
| `ConfirmationResult` (`{ title?, body? }`), synchronously or as the resolved value of a Promise | **Overrides** the configured message. Use for content only known after submit — e.g. a server reference number. |

### Wait for the backend, then confirm with its reference number

```tsx
import type { FlowData, ConfirmationResult } from "@sonata-innovations/fiber-fbre";

const handleComplete = async (data: FlowData): Promise<ConfirmationResult> => {
  const { referenceId } = await saveToBackend(data); // rejects → FBRE shows the error, no confirmation
  return { title: "All done!", body: `Your reference number is ${referenceId}.` };
};

<FBRE flow={flow} onFlowComplete={handleComplete} />;
```

If you don't need a runtime override, just author the message in `config.confirmation` and return `void` (or a `Promise<void>` to get the in-flight state while your save runs).

## Authoring in FBT

Open **Flow Settings → Confirmation Screen**, toggle it on, and fill in the title and message. This writes `config.confirmation` — it is intentionally separate from the stage/question editor. The message fields accept `${...}` references.

## FBTL

FBTL doesn't edit this config, but it **carries it through untouched** on load and export. A flow authored with a confirmation in FBT (or by hand) keeps it when round-tripped through FBTL, and FBTL's live FBRE preview renders it.

## Try it

FBRE playground → **Confirmation Demo** fixture:

```bash
cd fbre && npm run dev:playground
```

Fill in name + email and submit: the demo simulates a ~1.2s server round-trip (showing the in-flight submit state), then renders the confirmation with the field values and a context value interpolated in.

## Exports & schema

- Types: `ConfirmationConfig`, `ConfirmationResult` — exported from `@sonata-innovations/fiber-fbre` (and `@sonata-innovations/fiber-types`).
- Reference list: `resolvableReferences`, and the `ResolvableReference` / `ReferenceKind` types — exported from `@sonata-innovations/fiber-shared`.
- Schema: [`flow-schema.md` → ConfirmationConfig](../schema/flow-schema.md#confirmationconfig).
- Integration reference: [FBRE Integration Guide → Confirmation Screen](../integration/fbre.md#confirmation-screen).

## Per-mode behavior

`<FBRE>` picks its mode from the props you pass, and the modes do not agree on what happens when a flow carries no confirmation:

| `config.confirmation` | Local / Remote | Server-driven |
| --- | --- | --- |
| `title` and/or `body`, `show` not `false` | Renders it | Renders it |
| Absent, or present but empty | Renders **nothing** — the final screen stays put, controls frozen | Falls back to a generic **"Thank you"** |
| `show: false` | Renders nothing | Renders nothing |

Server-driven mode falls back because the client there is frequently the entire page (the hosted renderer), with no parent UI to take over. In local and remote modes FBRE is embedded in an app that owns the page and receives `onFlowComplete`, so the library does not invent copy for someone else's layout — a parent that wants a terminal state either authors `config.confirmation` or returns a `ConfirmationResult`.

> If your form is a lead-capture or contact flow, configure a confirmation. Without one the visitor submits and the page does not visibly change, which reads as a failed submit.

## Edge cases

- A non-empty `ConfirmationResult` returned from `onFlowComplete` forces the confirmation to render even when `config.confirmation.show` is `false` (the override path bypasses the config gate). An empty object (`{}`) is ignored and does not count as an override.
- Server-driven mode drives the confirmation from `config.confirmation` alone — it does not honor a `ConfirmationResult` return, because `onFlowComplete` there receives the raw server result.
- A flow cannot be submitted twice. Once completion succeeds, local and remote modes disable the Back and Submit controls even when no confirmation renders. A **rejected** promise re-enables them so the visitor can retry.
