---
title: FBTL Integration Guide
applies-to:
  - "@sonata-innovations/fiber-fbtl@^3.0"
read-when: "Embedding the lite builder in a parent app: controlled-component contract, scope, screen model, narrowing the palette, save lifecycle, normalizing generator-authored flows."
---

# FBTL — Lite Builder UI

**Use FBTL when** the person authoring the flow is the end user of your app (a clinician, a teacher, a small-business admin) rather than a developer or dedicated form designer. FBTL hides the full component palette, multi-screen concept, multi-rule conditions, calculations, and reference markup behind a deliberately minimal creation UX. It emits the same Flow JSON [FBT](fbt.md) does, so [FBRE](fbre.md) renders an FBTL-authored flow identically to an FBT-authored one.

## Minimal Setup

```tsx
import { FBTL } from "@sonata-innovations/fiber-fbtl";
import "@sonata-innovations/fiber-fbtl/styles";  // MUST import separately
import type { Flow } from "@sonata-innovations/fiber-fbtl";

function App() {
  const [flow, setFlow] = useState<Flow>(initialFlow);
  return <FBTL flow={flow} onChange={setFlow} />;
}
```

FBTL is a **controlled** component. The parent always owns `flow` state; FBTL fires `onChange` on every mutation. There is no internal dirty tracking, debouncing, or import/export UI — parents choose their own persistence strategy.

> **Feedback-loop safety.** Treat the object passed to `onChange` like the value of a controlled `<input>`: store it and pass it straight back as the `flow` prop. The simple `const [flow, setFlow] = useState(...); <FBTL flow={flow} onChange={setFlow} />` pattern above is correct and loop-free. You only need to think about this if you route the flow through a layer that **re-creates the object** on the way back — a reducer/normalizer, a deep-clone, `setFlow({ ...next })`, a React Query cache, or a form library. As of **fbtl ≥ 2.1.2** FBTL detects a content-identical re-hydration and treats it as a no-op, so a re-created-but-equal object no longer causes an infinite render loop. Older versions (≤ 2.1.1) only short-circuit when you pass back the **exact same object reference**; on those, a normalizing parent re-hydrates → re-emits → re-renders without end. The fix either way: upgrade to ≥ 2.1.2, or ensure your state layer preserves the object identity FBTL handed you for an unchanged flow.
>
> Note this is unrelated to the renderer (`fiber-fbre`) and not a React 19 issue — FBTL is built and tested against React 18 and 19. If you saw a loop attributed to FBTL's drag-and-drop component or an "unstable Zustand selector," that was a misdiagnosis of this controlled-component echo; the `components` selector is reference-stable.

## Props Reference

**`<FBTL />` — zero-config convenience wrapper**

| Prop | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| `flow` | `Flow` | Yes | — | Current flow (controlled — parent owns state) |
| `onChange` | `(flow: Flow) => void` | Yes | — | Fires on every mutation |
| `storeRef` | `MutableRefObject<StoreApi<FBTLStoreState> \| null>` | No | — | Exposes the internal Zustand store (selection state, actions) to the parent |
| `theme` | `ThemeConfig` | No | — | Passed to the preview pane |
| `navigation` | `NavigationConfig` | No | — | Passed to the preview pane |
| `controls` | `ControlsConfig` | No | — | Passed to the preview pane |
| `options` | `FBTLOptions` | No | — | Builder-UI configuration that never touches the flow — see [Screen model](#screen-model) and [Narrowing the palette](#narrowing-the-palette) |
| `saveState` | `"idle" \| "saving" \| "saved" \| "error"` | No | `"idle"` | Drives the save-status pill |
| `onSaveRetry` | `() => void` | No | — | Retry button handler when `saveState === "error"` |
| `ctaPosition` | `"top" \| "bottom"` | No | `"bottom"` | Where the add-question CTA row renders |
| `stickyCta` | `boolean` | No | `true` | Keep the CTA row pinned while scrolling |
| `className` | `string` | No | — | Applied to the root container |
| `locale` | `string` | No | — | i18n locale (reserved; not yet used) |

**`<FBTLProvider />` — for composable layouts.** It does **not** take the full `<FBTL />` prop set — only:

| Prop | Type | Required |
|------|------|----------|
| `flow` | `Flow` | Yes |
| `onChange` | `(flow: Flow) => void` | No |
| `storeRef` | `MutableRefObject<StoreApi<FBTLStoreState> \| null>` | No |
| `options` | `FBTLOptions` | No |
| `children` | `ReactNode` | Yes |

The remaining `<FBTL />` props map to composable sub-components instead: `theme`/`navigation`/`controls` are props of `FBTLPreview`; `saveState`/`onSaveRetry` map to `FBTLSaveStatePill`'s `state`/`onRetry`.

## Scope — What FBTL Creates vs Preserves

| | Creates in FBTL | Preserves on load (read-only) |
|---|---|---|
| Question types | Short answer, Long answer, Pick one, Pick several, Confirmation (agree/opt-in checkbox), Date, Time | Full 30-type component registry |
| Screens | Page-break model: each card defaults to its own screen; consecutive cards can be merged onto a shared screen | Multi-screen structure round-trips (screen boundaries become break flags) |
| Conditions | Single-rule conditions between top-level questions | Multi-rule (AND/OR) conditions |
| Validation | Required toggle, Short Answer format sub-choice (Any / Email / Phone / URL / Number) | Character limits, regex, min/max, `matchesField`, custom validators |
| Groups | Information Screens (heading + optional paragraph, stored as a group of `[header, text]`) | Mixed-content groups render as read-only "Section" containers |
| Other | — | Calculations, reference markup, repeaters |

"Pick one" chooses its emitted component by display variant and option count: 2 options default to `yesNo` (two buttons), 3–4 to `cardSelect`, 5+ to `dropDown`; the author can also pick `radio`. "Date" likewise chooses between `date` and `dateTime` from its "Show as" sub-choice. Across the wizard, FBTL emits `inputText`, `inputTextArea`, `inputNumber` (Short answer with Number format), `yesNo`, `radio`, `cardSelect`, `dropDown`, `checkbox`, `dropDownMulti`, `confirm`, `date`, `dateTime`, `time`, and `group` (Information Screens) — useful when parsing FBTL-authored JSON.

Loaded components with preserved advanced properties show a "Has custom configuration" badge on the stage card. Clicking the badge discloses *what* is configured — "custom condition logic", "sets a minimum, a maximum", and so on — so an author handed a pre-built question can at least see what they are not allowed to change. **FBTL never strips data on save** — advanced properties are round-tripped unchanged. There is no cross-link to FBT in FBTL v1.

### Date and Time questions

The Date and Time tiles author the same constraints the renderer reads, and the editor works on any `date` / `time` / `dateTime` component in the flow — including ones FBTL did not create, which is the case for anything a generator wrote directly:

| Setting | Writes | Notes |
|---|---|---|
| Show as (Date tile) | the component `type` — `date` or `dateTime` | switching clears the keys the other shape does not use |
| Allowed dates | `min` / `max` | "Today or later" emits the relative token `"today"`, which FBRE resolves at render time — the constraint does not go stale |
| Date format | `dateFormat` | `MM/DD/YYYY`, `DD/MM/YYYY`, `YYYY-MM-DD` |
| Allowed times | `min`/`max` on `time`, `minTime`/`maxTime` on `dateTime` | a `dateTime` constrains its date and time halves independently |
| Time increments | `step` (seconds) | 15 minutes / 30 minutes / 1 hour, or unrestricted |

Range types (`dateRange`, `timeRange`, `dateTimeRange`) are still preserved-only — they have no palette tile, and the condition builder offers them presence operators only. That limit is in the shared condition engine, not the UI: a range answers with a `{ start, end }` object, which the comparison operators stringify to `"[object Object]"`.

### Narrowing the palette

`options` shapes the builder without touching the flow:

```ts
<FBTL
  flow={flow}
  onChange={setFlow}
  options={{
    screenModel: "single",
    allowedQuestionTypes: ["inputText", "pickOne", "date"],
    lockedQuestionTypes: { time: "Available on the Pro plan" },
  }}
/>
```

- `screenModel` — see [Screen model](#screen-model) below.
- `allowedQuestionTypes` — the palette shows only these tiles. Omit for all of them.
- `lockedQuestionTypes` — the tile renders disabled with the given reason as its description, so a capability-gated host can show that a type exists rather than hiding it. A locked tile is shown even when `allowedQuestionTypes` leaves it out.

Both are keyed by `QuestionType`, the palette's own vocabulary (`QUESTION_TYPES` is exported for building the list). These are *authoring tiles*, not FBRE component types — `"pickOne"` covers `yesNo` / `cardSelect` / `dropDown` / `radio`.

This replaces the `disabledQuestionTypes` prop removed in 2.2.0, which never had an effect.

### Screen model

FBTL's stage is paged: a divider between every card, and new cards default to starting a new screen. `options.screenModel` says whether that is what the form wants.

| Value | Stage | Emit |
|---|---|---|
| `"paged"` | break dividers between cards | screens follow the break flags |
| `"single"` | dividers hidden | the whole flow emits as one screen |
| `"auto"` (default) | resolves to one of the above | — |

`"auto"` picks `"single"` only for a flow carrying a **form-family style** that already fits on one screen, and `"paged"` for everything else. The style is the proxy for "this was authored as an ordinary form" — and since FBTL's own defaults fill in `centered-minimal` (a focused style), the form family only appears when the incoming flow chose it, which keeps the rule opt-in. The restriction is deliberate: a form-styled flow with six deliberate screens is a paged form, and collapsing it the first time the author edits anything would destroy structure the host never asked us to touch. A host that does want the collapse passes `"single"` outright.

**Nothing in `flow.config` decides this.** `theme.style` is presentational and `navigation.autoAdvance` / `advanceOnEnter` are behavioural; screens stay screens either way. "All on one page" is a statement about screen *count*, which is why it is a separate option. A host offering the author a "one question at a time / all on one page" choice sets both: the style and advance flags for the presentation, `screenModel` for the structure.

Switching the model is a rewrite, and an intentional one — it happens when the host changes the prop, not on load:

- → `"single"` collapses the break flags, so the next emit is one screen.
- → `"paged"` restores FBTL's one-card-per-screen default rather than leaving every card merged with no way to tell them apart.

One consequence worth knowing: on a single screen, no question is ever alone on its screen, so no condition is promoted to screen level — every condition becomes an in-place show/hide. FBTL says so on the conditional band (see the Emit Shape note below).

## Emit Shape (Page Breaks)

Screens in the emitted JSON are derived from **per-card break flags**, not a fixed one-question-per-screen rule:

- Every stage card carries a break-after flag. New cards default to break-after, so a freshly authored flow *is* one question per screen — until the author merges cards onto a shared screen (the flag is toggled via the divider between cards).
- Consecutive cards without a break between them emit onto the same `FlowScreen`.
- A conditional question is grouped onto its trigger's screen, so it appears as an inline reveal rather than a separately gated screen.
- Component-level conditions are promoted to screen-level `conditions` **only when the screen contains exactly one component**; on multi-component screens, conditions stay on the component. The two outcomes are visibly different to a visitor — a promoted condition skips the whole screen, a component-level one hides the question on a screen the visitor still reaches — so FBTL states which one the author is getting, on the conditional band and in the condition dialog. `getConditionBehavior` is exported if a host wants to say the same thing elsewhere.
- On load, a multi-screen flow's boundaries are recorded as break flags, so screen structure **round-trips**. One caveat: the first screen's uuid is stable, but **non-first screens' uuids and labels are regenerated on emit** — don't key external state to them.
- FBTL also force-fills missing config keys on load/emit: `theme.style: "centered-minimal"`, `navigation.transition: "slide"`, `navigation.autoAdvance: true`, `controls.showStepper: true`. A flow round-tripped through FBTL gains these defaults. (`mode` was one of them through 2.x; `FlowConfiguration.mode` no longer exists — see [Style Families](../features/style-families.md).)
- Because auto-advance only fires on a screen with exactly one visible input, merging two cards onto one screen silently turns it off there. The divider says so in its tooltip when `autoAdvance` is on.

## Normalizing Generator-Authored Flows (before loading into FBTL)

FBTL groups a conditional component with the question it depends on **only when the conditional immediately follows that trigger** in document order. Flows produced by an automated authoring step (e.g. an AI/LLM generator) frequently place a conditional *after* an unrelated non-conditional question, so FBTL cannot group them and the conditional becomes a detached, separately-gated downstream screen instead of an inline reveal.

Run such flows through `normalizeConditionalOrder` from `@sonata-innovations/fiber-types` **before** passing them to FBTL:

```ts
import { normalizeConditionalOrder } from "@sonata-innovations/fiber-types";

const flow = normalizeConditionalOrder(generatorOutput);
// ...then load `flow` into FBTL
```

It is a **pure structural reorder** — it returns a new `Flow` and never rewrites a condition, property, uuid, or label (component object references are reused):

- Each top-level conditional is moved to immediately follow the latest of its (resolved) trigger sources, so every dependency precedes it.
- A condition pointing at a question **nested inside a group/repeater** anchors on that question's top-level ancestor card.
- Dependency chains (A → C → D) are ordered trigger-before-dependent.
- Cycles and dangling/`context` sources are left in place (never relocated, never loops).
- Every empty screen is dropped; surviving screens keep their original order, uuids, labels, and screen-level conditions.
- If relocation drops the original first screen, its uuid is transplanted onto the new first screen so FBTL's stable round-trip `screenUUID` does not shift.

This is **not** a validator and does not gate or reject flows — pair it with your own validation if the generator can produce semantically broken conditions. It is safe to call unconditionally on every generator output, including flows that are already well-ordered (it is idempotent on those).

## Composable Layout

```tsx
import {
  FBTLProvider,
  FBTLDndZone,
  FBTLStage,
  FBTLPreview,
  FBTLWizardHost,
  FBTLUndoToast,
} from "@sonata-innovations/fiber-fbtl";
import "@sonata-innovations/fiber-fbtl/styles";

<FBTLProvider flow={flow} onChange={setFlow}>
  <div className="fbtl-container my-layout">
    <FBTLDndZone>
      <FBTLStage />
    </FBTLDndZone>
    <FBTLPreview />
    <FBTLWizardHost />
    <FBTLUndoToast />
  </div>
</FBTLProvider>
```

By default `FBTLStage` renders the add-question CTA inline. To reproduce `<FBTL />`'s sticky bottom CTA in a custom layout, pass `inlineCta={false}` and render `FBTLStageCtaRow` yourself where you want it.

## Styling

All visual values are `--fbtl-*` CSS custom properties. In the published package they ship bundled in `dist/fiber-fbtl.css` (the `./styles` export) — the token block sits at the top of that file; inspect `node_modules/@sonata-innovations/fiber-fbtl/dist/fiber-fbtl.css` for the full catalog. Override at any selector above the FBTL root:

```css
.my-app {
  --fbtl-primary: #4f46e5;
  --fbtl-primary-dark: #4338ca;
  --fbtl-radius-md: 8px;
}
```

Component styles use **only** token variables — no hardcoded colors or pixel values.

**The preview's styles are included.** FBTL's preview pane and the add-question wizard both mount a real `<FBRE>`, and FBRE ships its stylesheet separately from its JS. As of **fbtl 3.0.1** that stylesheet is bundled into `dist/fiber-fbtl.css`, so `import "@sonata-innovations/fiber-fbtl/styles"` is the only import you need — the preview renders styled out of the box. On **≤ 3.0.0** it was not: the builder chrome looked correct while the preview rendered as unstyled text, and the fix was to add `import "@sonata-innovations/fiber-fbre/styles"` yourself.

This does not change anything if you *also* render `<FBRE>` in your own app to publish the finished form — that instance is yours, and it still needs its own `import "@sonata-innovations/fiber-fbre/styles"` (see [FBRE](fbre.md)). Importing it alongside FBTL's stylesheet is harmless; the rules are identical and the browser dedupes the cascade.

## Save Lifecycle (Parent-Owned)

FBTL has no internal persistence. A typical debounced save:

```tsx
function App() {
  const [flow, setFlow] = useState<Flow>(initialFlow);
  const [saveState, setSaveState] = useState<"idle" | "saving" | "saved" | "error">("idle");

  const debouncedSave = useMemo(
    () => debounce(async (next: Flow) => {
      setSaveState("saving");
      try {
        await api.saveFlow(next);
        setSaveState("saved");
      } catch {
        setSaveState("error");
      }
    }, 800),
    []
  );

  const handleChange = (next: Flow) => {
    setFlow(next);
    debouncedSave(next);
  };

  return (
    <FBTL
      flow={flow}
      onChange={handleChange}
      saveState={saveState}
      onSaveRetry={() => debouncedSave(flow)}
    />
  );
}
```

## Integration Patterns

### Generator → Normalize → FBTL → FBRE Pipeline

When a flow is produced by an automated authoring step (AI/LLM) and then handed to an end user to review in FBTL before rendering, normalize it at the boundary so conditionals group correctly:

```tsx
import { normalizeConditionalOrder } from "@sonata-innovations/fiber-types";
import { FBTL } from "@sonata-innovations/fiber-fbtl";
import { FBRE } from "@sonata-innovations/fiber-fbre";
import "@sonata-innovations/fiber-fbtl/styles";
import "@sonata-innovations/fiber-fbre/styles";
import type { Flow, FlowData } from "@sonata-innovations/fiber-types";

function ReviewAndRun({ generated }: { generated: Flow }) {
  // Normalize ONCE, at ingest — before the end user ever sees it in FBTL.
  const [flow, setFlow] = useState<Flow>(() => normalizeConditionalOrder(generated));
  const [done, setDone] = useState<FlowData | null>(null);

  return done ? (
    <pre>{JSON.stringify(done, null, 2)}</pre>
  ) : flow ? (
    <>
      <FBTL flow={flow} onChange={setFlow} />
      <button onClick={() => setFlow(flow)}>Accept</button>
      <FBRE flow={flow} onFlowComplete={setDone} />
    </>
  ) : null;
}
```

Normalize at ingest, not on every `onChange` — re-running it after the user has begun editing in FBTL would reorder their in-progress work.

## Pitfalls

- **Uncontrolled usage is not supported.** `flow` is required; FBTL does not hold its own copy. If `onChange` doesn't round-trip back to `flow`, edits will appear to revert.
- **Do not mutate the `flow` prop in place.** Treat it as immutable — FBTL produces a new object on each `onChange`.
- **Non-first screen uuids/labels are regenerated on emit.** Screen *structure* round-trips (see Emit Shape), but only the first screen's uuid is stable. If external state keys off screen uuids, use FBT instead.
- **Advanced properties are preserved but not editable.** If an end user reports a card with a "Has custom configuration" badge and no way to change what it names, that's expected — the flow was authored (or modified) somewhere richer than FBTL. Clicking the badge says which settings those are; there is still no "Edit in FBT" affordance in v1.
- **Switching `screenModel` rewrites screen structure.** It is not a view toggle — going to `"single"` collapses the break flags and going back restores one card per screen. Change it when the author changes their mind, not per render.
- **Generator-authored flows may scatter conditionals.** If a conditional question shows up as its own gated screen instead of an inline reveal under its trigger, the source flow placed the conditional after an unrelated question. Run it through `normalizeConditionalOrder` (see *Normalizing Generator-Authored Flows*) before loading into FBTL.
- **Import styles separately** (`import "@sonata-innovations/fiber-fbtl/styles"`) — styles are not bundled with the JS. That one import covers the preview too from **3.0.1** on; on **≤ 3.0.0** an unstyled preview under a correct-looking builder means you also need `import "@sonata-innovations/fiber-fbre/styles"`.

## Public Exports

```ts
// Main component
import { FBTL } from "@sonata-innovations/fiber-fbtl";

// Composable sub-components (require FBTLProvider; FBTLStage also requires FBTLDndZone)
import {
  FBTLProvider,
  FBTLDndZone,
  FBTLStage,
  FBTLStageCtaRow,
  FBTLPreview,
  FBTLWizardHost,
  FBTLUndoToast,
  FBTLSaveStatePill,
  FBTLInvalidFlowBanner,
} from "@sonata-innovations/fiber-fbtl";

// Hooks — throw if used outside FBTLProvider
import { useFBTLStore, useFBTLStoreApi } from "@sonata-innovations/fiber-fbtl";

// Flatten/unflatten utilities (expose one-question-per-screen transform)
import {
  flattenFlow, unflattenFlow, componentsToScreens, partitionIntoScreens,
} from "@sonata-innovations/fiber-fbtl";

// Condition behaviour — skipped screen vs in-place hide, for the same
// labelling FBTL renders internally
import { getConditionBehavior, CONDITION_BEHAVIOR_LABELS } from "@sonata-innovations/fiber-fbtl";

// The add-question palette's vocabulary (for options.allowedQuestionTypes)
import { QUESTION_TYPES } from "@sonata-innovations/fiber-fbtl";

// Screen model resolution (same rule the store applies)
import { resolveScreenModel } from "@sonata-innovations/fiber-fbtl";

// Types
import type {
  FBTLProps, FBTLOptions, QuestionType, ConditionBehavior, ScreenModel, ScreenModelOption,
  SaveState, FBTLStoreState,
} from "@sonata-innovations/fiber-fbtl";
import type {
  Flow, FlowScreen, Component, FlowMetadata, FlowConfiguration, ComponentProperties,
  ThemeConfig, NavigationConfig, ControlsConfig,
} from "@sonata-innovations/fiber-fbtl";
```
