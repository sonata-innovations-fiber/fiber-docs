---
title: FBRE Integration Guide
applies-to:
  - "@sonata-innovations/fiber-fbre@^4.0"
read-when: "Embedding the render engine in a parent app: props and modes, onFlowComplete contract, confirmation screen, store access, events, theming, pre-populating data."
---

# FBRE — Render Engine

FBRE consumes Flow JSON, renders the form (conditions, validation, screen transitions), and returns collected data (`FlowData`) to the parent application.

For the Flow JSON format itself, see the [Flow JSON Quick Reference](../schema/flow-quick-reference.md). For theming in depth, see the [FBRE Theming Guide](../features/fbre-theming.md).

## Minimal Setup

```tsx
import { FBRE } from "@sonata-innovations/fiber-fbre";
import "@sonata-innovations/fiber-fbre/styles";  // MUST import separately

function App() {
  const handleComplete = (data: FlowData) => {
    console.log("Collected data:", data);
  };

  return (
    <FBRE
      flow={myFlow}
      onFlowComplete={handleComplete}
    />
  );
}
```

## Props Reference (Local Mode)

| Prop | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| `flow` | `Flow` | Yes | — | Flow JSON object to render |
| `onFlowComplete` | `(data: FlowData) => void \| ConfirmationResult \| Promise<void \| ConfirmationResult>` | Yes | — | Called when user completes the last screen. See [Confirmation Screen](#confirmation-screen) |
| `data` | `FlowData` | No | `undefined` | Pre-populated form data. **Not reactive** — applied only when the flow loads; changing `data` alone does nothing (see Pitfalls) |
| `screenIndex` | `number` | No | `0` | Controlled screen index |
| `themeDefaults` | `ThemeConfig` | No | — | Theme defaults merged *under* `flow.config.theme` — the flow wins where it sets a value |
| `theme` | `ThemeConfig` | No | — | Theme overrides merged *over* `flow.config.theme` — the prop wins |
| `navigation` | `NavigationConfig` | No | — | Override navigation settings (merged over `flow.config.navigation`) |
| `controls` | `ControlsConfig` | No | — | Override controls settings (merged over `flow.config.controls`) |
| `context` | `Record<string, string \| boolean \| number>` | No | — | External context for context conditions (reactive — updates trigger re-evaluation) |
| `storeRef` | `MutableRefObject<StoreApi<FBREStoreState> \| null>` | No | — | Exposes Zustand store for imperative access |
| `onScreenChange` | `(index: number, data: any) => void` | No | — | Called on screen navigation; at runtime `data` is the current `FlowData` (declared `any`) |

> **Watching screen validity.** There is no validity callback prop. Subscribe through `storeRef` instead — `storeRef.current.subscribe(...)` and read `getScreenValidity(index)`, or select `screenValidity` directly. (An `onScreenValidationChange` prop was accepted but never invoked in any released version; it has been removed.)

### Config group props and precedence

`navigation` and `controls` are shallow-merged **over** the corresponding `flow.config` group: the JSX prop wins.

`theme` has two props, because hosts have two different needs:

```
themeDefaults  <  flow.config.theme  <  theme
(house style)     (what the author chose)  (host override)
```

- **`themeDefaults`** — "use this unless the flow says otherwise." Reach for it when you render many flows and want a house style that individual flows can depart from.
- **`theme`** — "use this regardless of what the flow says." Reach for it when the embedding app owns the look, or when you are previewing a theme that isn't saved to the flow yet (this is what the [theme editor](theme-editor.md) does).

All merges are **key-level**, so each layer only displaces the keys it actually sets — a flow's `colorScheme` survives a `theme={{ color: "#f00" }}` that never mentions it.

```tsx
// The flow picks its own accent; everything else follows the host's house style.
<FBRE
  flow={flow}
  themeDefaults={{ colorScheme: "dark", surface: "#243244", radius: "4px" }}
  onFlowComplete={done}
/>
```

The props above apply to **local mode** (passing a `flow` object directly). FBRE also supports two additional modes:

## Remote Mode

Remote mode fetches the flow from a Fiber API server. Pass `flowId` and `apiEndpoint` instead of `flow`:

```tsx
<FBRE
  flowId="your-flow-id"
  apiEndpoint="https://your-fiber-server.com/api/v1"
  apiKey="optional-api-key"
  onFlowComplete={(data) => console.log(data)}
/>
```

| Prop | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| `flowId` | `string` | Yes | — | Flow ID to fetch from the API |
| `apiEndpoint` | `string` | Yes | — | Fiber API base URL |
| `apiKey` | `string` | No | — | API key for authentication |
| `data` | `FlowData` | No | `undefined` | Pre-populated form data (not reactive — see Pitfalls) |
| `screenIndex` | `number` | No | `0` | Controlled screen index |
| `themeDefaults` | `ThemeConfig` | No | — | Theme defaults merged *under* the fetched flow's `config.theme` |
| `theme` | `ThemeConfig` | No | — | Theme overrides merged *over* the fetched flow's `config.theme` |
| `navigation` | `NavigationConfig` | No | — | Override navigation settings |
| `controls` | `ControlsConfig` | No | — | Override controls settings |
| `storeRef` | `MutableRefObject<StoreApi<FBREStoreState> \| null>` | No | — | Exposes Zustand store |
| `onFlowComplete` | `(data: FlowData) => void \| ConfirmationResult \| Promise<void \| ConfirmationResult>` | Yes | — | Called on completion. See [Confirmation Screen](#confirmation-screen) |
| `context` | `Record<string, string \| boolean \| number>` | No | — | External context for context conditions (reactive) |
| `onScreenChange` | `(index: number, data: any) => void` | No | — | Called on screen navigation; `data` is the current `FlowData` at runtime |

## Server-Driven Mode

Server-driven mode delegates all logic (conditions, validation, screen transitions) to the server. The client renders one screen at a time:

```tsx
<FBRE
  sessionEndpoint="https://your-fiber-server.com/public/sessions"
  flowId="your-flow-id"
  apiKey="optional-api-key"
  onFlowComplete={(data) => console.log(data)}
/>
```

| Prop | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| `sessionEndpoint` | `string` | Yes | — | Session API endpoint |
| `flowId` | `string` | Yes | — | Flow ID to start a session with |
| `apiKey` | `string` | No | — | API key for authentication |
| `themeDefaults` | `ThemeConfig` | No | — | Theme defaults merged *under* the session's `config.theme` |
| `theme` | `ThemeConfig` | No | — | Theme overrides merged *over* the session's `config.theme` |
| `navigation` | `NavigationConfig` | No | — | Override navigation settings, e.g. the advance flags (merged over the session's `config.navigation`) |
| `context` | `Record<string, string \| boolean \| number>` | No | — | External context sent to the server at session start |
| `onFlowComplete` | `(data: any) => void` | Yes | — | Called on session completion |
| `onScreenChange` | `(screenNumber: number) => void` | No | — | Called on screen navigation (note: receives screen number, not index + data) |

> **Note:** Server-driven mode has a different `onScreenChange` signature — `(screenNumber: number) => void` — compared to local/remote mode's `(index: number, data: FlowData) => void`.

In server-driven mode, the `context` is sent to the server in the `POST /public/sessions` request body and stored on the session. The server uses it to evaluate cross-screen context conditions, and it is returned in each screen payload so the client can evaluate intra-screen context conditions locally.

On a successful completion, server-driven mode resets the submit button and renders a terminal [confirmation screen](#confirmation-screen) — the flow's configured `config.confirmation` message when present, otherwise a generic "Thank you". An explicit `config.confirmation.show: false` renders nothing. `onFlowComplete` still fires with the server result, so a parent that wants its own post-completion UI can unmount or replace the component from that handler.

## Focused Presentation

There is no presentation mode. The look is the **style**, and the advance behaviors are **navigation flags** — three ordinary settings you can mix freely.

```tsx
// Via flow config — the usual place
const flow = {
  ...myFlow,
  config: {
    ...myFlow.config,
    theme: { style: "centered-minimal" },
    navigation: { transition: "scaleFade", autoAdvance: true },
  },
};
<FBRE flow={flow} onFlowComplete={handleComplete} />

// Via JSX props — merged over the flow's own config
<FBRE
  flow={myFlow}
  theme={{ style: "centered-minimal" }}
  navigation={{ transition: "scaleFade", autoAdvance: true }}
  onFlowComplete={handleComplete}
/>

// Server-driven — the session carries theme and navigation, so a focused
// flow renders correctly with no extra props at all
<FBRE
  sessionEndpoint="https://api.example.com/api/v1/public/sessions"
  flowId="flow-id"
  onFlowComplete={handleComplete}
/>
```

Four of the ten styles — `centered-minimal`, `stacked-cards`, `soft-float`, `bold-statement` — form the **focused family**: they vertically center content in a narrow column, animate component entry, and enlarge tap targets and type. The other six are the **form family**. The vocabulary is flat: any style is valid on any flow, and FBRE derives the family itself, emitting it as `data-style-family` alongside `data-style`.

```ts
import { FOCUSED_STYLES, styleFamily } from "@sonata-innovations/fiber-fbre";

styleFamily("soft-float"); // "focused"
```

Auto-advance and Enter-to-advance are independent of the style:

| Setting | Default | Behavior |
| --- | --- | --- |
| `navigation.autoAdvance` | `false` | Advance ~500ms after a single-select choice (`radio`, `yesNo`, `cardSelect`, `dropDown`) |
| `navigation.advanceOnEnter` | `true` | Advance when Enter is pressed in an `inputText` / `inputNumber` |

Both only ever fire on a screen with **exactly one visible input**, never on the last screen, and never on an invalid screen. Enter-to-advance additionally stands down inside an open popup, so Enter in the colour picker's hex field commits the value instead of skipping the screen. FlowData output is unchanged by any of this.

See [Style Families](../features/style-families.md) for the full model.

> **Upgrading from FBRE 3.x.** `config.mode` and the `mode` prop are gone. A flow that had `mode: "conversational"` already carried a focused `theme.style`, so its look survives untouched; add `navigation: { autoAdvance: true }` to keep the auto-advance, which is now off by default. Enter-to-advance is now on for every flow, focused or not.

## Confirmation Screen

A terminal "thank you" screen shown after the flow is submitted. It is a flow-level setting (`config.confirmation`), not a `Screen`, so it is never edited in the builder's stage editor. The config-driven screen renders only when `show` is not `false` **and** `title` or `body` has content; when shown, it replaces the final screen and hides the navigation controls/stepper. Note the edge cases: a non-empty `ConfirmationResult` returned from `onFlowComplete` forces the confirmation to render **even when `config.confirmation.show` is `false`**. Server-driven mode renders a terminal confirmation too, but drives it from `config.confirmation` only (it does not honor a `ConfirmationResult` return, since `onFlowComplete` there receives the raw server result); when no confirmation is configured it falls back to a generic "Thank you", and an explicit `show: false` renders nothing. In local and remote modes there is no such fallback — configure a confirmation if you want the page to acknowledge the submit. A flow can only be submitted once: after a successful completion the controls are disabled even when no confirmation renders, and a rejected promise re-enables them for a retry. See [Confirmation Screen → Per-mode behavior](../features/confirmation-screen.md#per-mode-behavior).

```tsx
const flow = {
  ...myFlow,
  config: {
    ...myFlow.config,
    confirmation: {
      show: true,
      title: "Thank you!",
      // ${...} references resolve against collected fields, calculations, and context
      body: "We received your submission, ${email}.",
    },
  },
};

<FBRE flow={flow} onFlowComplete={handleComplete} />
```

**Deciding when to show it — the `onFlowComplete` contract.** The confirmation appears once `onFlowComplete` settles. The return value controls the behavior:

| Return | Behavior |
|--------|----------|
| `void` | Confirmation shows immediately (optimistic). |
| `Promise<void>` | Submit button stays in its in-flight (loading) state until the promise settles. On resolve → confirmation shows; on **reject** → a completion error renders and the confirmation is **not** shown. |
| `ConfirmationResult` (`{ title?, body? }`), sync or as the resolved value of a Promise | Overrides the configured message — use for content only known after submit, e.g. a server reference number. |

```tsx
// Wait for the server, then show a confirmation carrying the server's reference number
const handleComplete = async (data: FlowData): Promise<ConfirmationResult> => {
  const { referenceId } = await saveToBackend(data); // rejects → FBRE shows the error, no confirmation
  return { title: "All done!", body: `Your reference number is ${referenceId}.` };
};

<FBRE flow={flow} onFlowComplete={handleComplete} />
```

`ConfirmationConfig` and `ConfirmationResult` are exported from `@sonata-innovations/fiber-fbre`. See [FlowConfiguration → ConfirmationConfig](../schema/flow-schema.md#confirmationconfig) for the schema.

## Store Access Patterns

**Option A: `storeRef` prop (from outside FBRE)**
```tsx
import { useRef } from "react";
import { FBRE, type FBREStoreState } from "@sonata-innovations/fiber-fbre";
import type { StoreApi } from "zustand";

function App() {
  const storeRef = useRef<StoreApi<FBREStoreState> | null>(null);

  const readData = () => {
    const data = storeRef.current?.getState().getFlowData();
    console.log(data);
  };

  return <FBRE flow={myFlow} storeRef={storeRef} onFlowComplete={handleComplete} />;
}
```

**Option B: `useFBREStore` / `useFBREStoreApi` hooks (from inside FBRE children)**
```tsx
import { useFBREStore, useFBREStoreApi } from "@sonata-innovations/fiber-fbre";

// Reactive (re-renders on change):
const isValid = useFBREStore((s) => s.getScreenValidity(0));

// Non-reactive (read on demand):
const storeApi = useFBREStoreApi();
const data = storeApi.getState().getFlowData();
```

**Option C: `useFBREApi` hook (remote mode only)**
```tsx
import { useFBREApi } from "@sonata-innovations/fiber-fbre";

// Returns FBREApiContextValue | null (null outside remote mode)
const api = useFBREApi();
if (api) {
  console.log(api.config);         // { apiEndpoint, apiKey?, timeout? }
  console.log(api.flowId);         // string
  console.log(api.flowVersionId);  // string
  console.log(api.tenantId);       // string
}
```

## Consumer-Relevant Store Actions

| Action | Signature | Description |
|--------|-----------|-------------|
| `getFlowData` | `() => FlowData` | Returns all collected form data |
| `getScreenValidity` | `(index: number) => boolean` | Check if a screen passes validation |
| `evaluateFlowValidation` | `() => boolean` | Validate entire flow, returns `true` if all valid |
| `getScreenByIndex` | `(index: number) => FlowScreen \| null` | Get screen by position |
| `getMaxScreenIndex` | `() => number` | Returns the maximum screen **index** (`screen count − 1`); does not filter condition-hidden screens. The last screen is index `getMaxScreenIndex()` |
| `getValidationErrors` | `(uuid: string) => string[]` | Get validation errors for a component |
| `getCalculationResult` | `(uuid: string) => number \| null` | Result of a flow calculation by uuid |
| `updateComponentValue` | `(uuid: string, value: any) => void` | Programmatically set a component's value |
| `updateContext` | `(context: Record<string, string \| boolean \| number>) => void` | Replace the external context (normally driven by the `context` prop) |
| `loadFlow` | `(flow: Flow, data?: FlowData, context?: Record<string, string \| boolean \| number>) => void` | Load a new flow (usually handled by prop change) |

## Events

```tsx
import { addFBREEventListener, removeFBREEventListener } from "@sonata-innovations/fiber-fbre";

// File upload listener
const handler = (componentUuid: string, fileData: FileUploadData) => {
  console.log("File uploaded for component:", componentUuid, fileData);
};
addFBREEventListener("file-upload", handler);

// Cleanup
removeFBREEventListener("file-upload", handler);
```

`"file-upload"` is currently the only event type. The event bus is **module-global, not per-instance**: with multiple `<FBRE />` instances mounted, one listener receives file-upload events from all of them, distinguishable only by component UUID.

## Theming

Three ways to theme, from least to most control: pick a **color scheme** preset, set **palette knobs** on `theme`, or override the raw `--fbre-*` CSS custom properties. They layer — preset seeds the tokens, knobs override a subset inline, raw CSS wins last. The **[FBRE Theming Guide](../features/fbre-theming.md)** is the full reference (precedence model, derivation mechanics, complete token catalog); the most common tokens:

| Variable | Default (Light) | Default (Dark) | Knob | Description |
|----------|----------------|----------------|------|-------------|
| `--fbre-theme-color` | `#1976d2` | `#42a5f5` | `color` | Primary accent |
| `--fbre-bg` | `#fff` | `#121212` | `background` | Form ground |
| `--fbre-surface` | `#fff` | `#1e1e1e` | `surface` | Surfaces / input fills |
| `--fbre-input-bg` | `var(--fbre-surface)` | `var(--fbre-surface)` | (derived) | Input fill (resting) |
| `--fbre-text` | `#212121` | `#e0e0e0` | `text` | Primary text |
| `--fbre-text-secondary` | `#666` | `#9e9e9e` | (derived) | Secondary text |
| `--fbre-label` | `var(--fbre-text-secondary)` | (same) | (derived) | Field label |
| `--fbre-border` | `#ccc` | `#444` | `border` | Border color |
| `--fbre-error` | `#d32f2f` | `#ef5350` | `error` | Error color |
| `--fbre-success` | `#2d6a28` | `#a8e6a0` | `success` | Success color |
| `--fbre-warning` | `#7a5900` | `#ffe082` | `warning` | Warning color |
| `--fbre-radius` | `4px` | `4px` | `radius` | Border radius |
| `--fbre-font` | `"Segoe UI", system-ui, -apple-system, sans-serif` | (same) | `fontFamily` | Font family |
| `--fbre-transition-duration` | `250ms` | `250ms` | — | Screen transition timing |
| `--fbre-transition-easing` | `cubic-bezier(0.4, 0, 0.2, 1)` | (same) | — | Screen transition curve |

The color scheme is selected via the `theme` prop (`{ colorScheme: "dark" }`) or `flow.config.theme.colorScheme` (default `"light"`). The `theme` prop takes precedence over the flow; `themeDefaults` loses to it (see [Config group props and precedence](#config-group-props-and-precedence)). `colorScheme` seeds the built-in light/dark palette preset; it replaces the former `darkMode` boolean.

Palette knobs on `theme` (`color`, `background`, `surface`, `text`, `border`, `radius`, `fontFamily`, and the semantic `error`/`success`/`warning`) are applied automatically via inline style and override the corresponding `--fbre-*` CSS variables on top of the preset. `fontFamily` also accepts `{ family, src }` — see [Loading a brand font](../features/fbre-theming.md#loading-a-brand-font), which is the only form that works when the host page does not already load the font. Each derives its related tokens (e.g. `surface` also sets the input fills and hover/alt surfaces). Any subset may be set; unset knobs fall through to the preset. Example: `<FBRE theme={{ colorScheme: "dark", surface: "#243244", text: "#e8ede9" }} ... />`. For finer control you can still override the raw `--fbre-*` variables in your own CSS.

## Pre-populating Data

Pass the `data` prop with a `FlowData` object. UUIDs must match components in the flow.

```tsx
const prefilled: FlowData = {
  uuid: "flow-uuid-here",
  metadata: { name: "My Flow" },
  screens: [
    {
      uuid: "screen-1-uuid",
      label: "Screen 1",
      components: [
        { uuid: "comp-uuid-1", type: "inputText", value: "Pre-filled text" },
        { uuid: "comp-uuid-2", type: "checkbox", value: ["option1", "option3"] },
      ],
    },
  ],
};

<FBRE flow={myFlow} data={prefilled} onFlowComplete={handleComplete} />
```

## Integration Patterns

### External Navigation (Hiding Built-In Buttons)

Hide FBRE's built-in navigation with `controls={{ show: false }}` and drive `screenIndex` yourself:

```tsx
import { useRef, useState } from "react";
import { FBRE, type FBREStoreState } from "@sonata-innovations/fiber-fbre";
import type { StoreApi } from "zustand";

function ExternalNav() {
  const storeRef = useRef<StoreApi<FBREStoreState> | null>(null);
  const [screen, setScreen] = useState(0);

  const goNext = () => {
    const store = storeRef.current?.getState();
    if (!store) return;
    if (store.getScreenValidity(screen)) {
      // getMaxScreenIndex() returns the last screen's index (count - 1)
      setScreen((s) => Math.min(s + 1, store.getMaxScreenIndex()));
    }
  };

  return (
    <>
      <FBRE
        flow={myFlow}
        controls={{ show: false }}
        screenIndex={screen}
        storeRef={storeRef}
        onFlowComplete={handleComplete}
      />
      <button onClick={() => setScreen((s) => Math.max(0, s - 1))}>Back</button>
      <button onClick={goNext}>Next</button>
    </>
  );
}
```

### Multiple FBRE Instances

Each `<FBRE />` creates its own isolated store. No conflicts:

```tsx
<FBRE flow={flowA} onFlowComplete={handleA} />
<FBRE flow={flowB} onFlowComplete={handleB} />
```

### Mounting Inside a Shadow Root

Supported as of 3.4.1. FBRE declares no `:root`, `html`, `body`, or `*` rules and injects no styles of its own, so it carries nothing that depends on being in the main document.

The one thing the parent must do differently is **deliver the CSS into the root**. `import "@sonata-innovations/fiber-fbre/styles"` puts a stylesheet in `<head>`, which a shadow root does not inherit:

```tsx
// The `./styles` subpath — the exports map exposes only "." and "./styles",
// so a deep `dist/…` import will not resolve.
import css from "@sonata-innovations/fiber-fbre/styles?inline";

const sheet = new CSSStyleSheet();
sheet.replaceSync(css);              // build once, share across every root
root.adoptedStyleSheets = [sheet];   // root = element.attachShadow({ mode: "open" })
```

Everything else works unchanged:

- **Theming.** All `--fbre-*` tokens are declared on `.fbre-container`, which FBRE renders itself — so the override recipe in the [Theming Guide](../features/fbre-theming.md) applies verbatim inside a root. Nothing is read from `:root`.
- **Overlays.** Dropdowns and the date/colour pickers are positioned in-tree. The tooltip is the only portal, and it targets the nearest `.fbre-container`, so it stays inside the root.
- **Brand fonts.** `@font-face` is the one thing the `adoptedStyleSheets` recipe above cannot deliver — a face declared inside a shadow root's stylesheet is not registered at all. Give `theme.fontFamily` an object carrying the font's sources and FBRE registers the faces against the owning document itself. See [Loading a brand font](../features/fbre-theming.md#loading-a-brand-font).
- **Outside-click and Enter-to-advance** use `composedPath()`, so they survive event retargeting. Before 3.4.1 they did not: a `mousedown` inside an open panel read as a click outside it and closed it, which broke multi-select, the date picker's two-step range, and the colour picker's drag.

One behavioural difference to be aware of: overlay flip/shift detection walks up for a scrolling ancestor and cannot see past the shadow boundary, so it falls back to viewport bounds. Overlays still open and remain usable; placement near the edge of a scrolling container outside the root is just less precise.

### Reading Flow Data Mid-Flow

```tsx
const storeRef = useRef<StoreApi<FBREStoreState> | null>(null);

const checkProgress = () => {
  const data = storeRef.current?.getState().getFlowData();
  const isScreenValid = storeRef.current?.getState().getScreenValidity(0);
  console.log("Current data:", data, "Screen 0 valid:", isScreenValid);
};

<FBRE flow={myFlow} storeRef={storeRef} onFlowComplete={handleComplete} />
```

## Pitfalls

**MUST:**
- Import styles separately (`import "@sonata-innovations/fiber-fbre/styles"`) — styles are not bundled with the JS
- Deliver that CSS into the shadow root when mounting inside one — a `<head>` stylesheet does not reach it (see [Mounting Inside a Shadow Root](#mounting-inside-a-shadow-root))
- Match UUIDs exactly when pre-populating `FlowData`

**DO NOT:**
- Change `flow.uuid` unless you want a full re-load (triggers `loadFlow` via `useEffect` dependency)
- Expect a changed `data` prop to re-populate the form — `data` is applied only when the flow loads. To re-prefill, reload the flow (new `flow.uuid`) or call `loadFlow(flow, data)` via `storeRef`
- Set runtime-only fields (`value`, `valid`, `addedComponents`) in Flow JSON — these are managed by FBRE at runtime
- Expect display components (`header`, `text`, `divider`, `callout`, `table`) in FlowData output — they are excluded. `computed` **is** included (it carries a derived value), and flows with calculations also emit a top-level `calculations` array
- Expect hidden components or hidden screens (via conditions) in FlowData output — they are excluded
- Access `storeRef.current` before the provider has mounted — it will be `null`
- Use the `useFBREStore` hook outside the FBRE provider — it requires context

**Type imports:** `Flow`, `FlowData`, `ScreenData`, `ComponentData`, `FileUploadData`, `ThemeConfig`, `FontFamilyConfig`, and the condition/validation types are all re-exported from `@sonata-innovations/fiber-fbre` — no direct dependency on `fiber-types` is needed.

**Function exports:** `ensureFontLoaded(theme, doc?)` registers a theme's brand font in a document (FBRE calls it itself on mount; call it directly to register before first paint), and `fontFamilyStack(theme.fontFamily)` returns the CSS stack FBRE writes to `--fbre-font`.
