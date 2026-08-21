---
title: FBT Integration Guide
applies-to:
  - "@sonata-innovations/fiber-fbt@^3.0"
read-when: "Embedding the full builder in a parent app: getting the Flow out, composable layout, custom presets and templates."
---

# FBT — Builder UI

FBT is the full-featured drag-and-drop form builder for power users and form designers. It outputs Flow JSON that [FBRE](fbre.md) renders. For the lite, end-user-facing builder, see [FBTL](fbtl.md).

## Minimal Setup

```tsx
import { FBT } from "@sonata-innovations/fiber-fbt";
import "@sonata-innovations/fiber-fbt/styles";  // MUST import separately

function App() {
  return (
    <FBT
      onFlowChange={(flow) => saveFlow(flow)}
    />
  );
}
```

## Props Reference

**`<FBT />` — zero-config convenience wrapper**

| Prop | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| `flow` | `Flow` | No | — | Initial flow to load |
| `onFlowChange` | `(flow: Flow) => void` | No | — | Called on every change |
| `themeColor` | `string` | No | `#1976d2` | Accent color |
| `storeRef` | `MutableRefObject<StoreApi<FBTStoreState> \| null>` | No | — | Exposes store |
| `customPresets` | `AnyPresetDefinition[]` | No | — | Additional presets (factory-based or data-based) |
| `customTemplates` | `AnyTemplateDefinition[]` | No | — | Additional templates (factory-based or data-based) |

**`<FBTProvider />` — for composable layouts (same props + `children`)**

| Prop | Type | Required |
|------|------|----------|
| `children` | `ReactNode` | Yes |
| _(all FBT props above)_ | | |

## Getting the Flow Out

**Option A: `onFlowChange` callback (recommended)**
```tsx
<FBT onFlowChange={(flow) => saveToBackend(flow)} />
```
Fires on every change. The `flow` is a clean, serializable `Flow` object.

**Option B: `storeRef` + `exportFlow()` (on demand)**
```tsx
const storeRef = useRef<StoreApi<FBTStoreState> | null>(null);

const handleSave = () => {
  const flow = storeRef.current?.getState().exportFlow();
  if (flow) saveToBackend(flow);
};

<FBT storeRef={storeRef} />
```

`storeRef` also exposes the other programmatic actions: `loadFlow(flow)`, `resetFlow()`, `mergeFlow(flow)` (append screens), `updateConfig(partialConfig)` (merge a `Partial<FlowConfiguration>`), and `updateMetadata(key, value)`.

## Composable Layout

Use `<FBTProvider>` with individual sub-components for custom arrangements:

```tsx
import {
  FBTProvider, FBTDndZone, FBTPool, FBTStage,
  FBTEditor, FBTScreenBar, FBTToolbar,
  FBTPreviewPanel, FBTPreviewFloat, FBTHelpModal,
  FBTConfirmDeleteDialog,
} from "@sonata-innovations/fiber-fbt";
import "@sonata-innovations/fiber-fbt/styles";

function CustomBuilder() {
  return (
    <FBTProvider onFlowChange={handleChange}>
      <FBTDndZone>
        <FBTPool />
        <FBTScreenBar />
        <FBTToolbar />
        <FBTStage />
        <FBTEditor />
        <FBTPreviewPanel />
        <FBTPreviewFloat />
      </FBTDndZone>
      <FBTHelpModal />
      <FBTConfirmDeleteDialog />
    </FBTProvider>
  );
}
```

**Sub-component list:**

| Component | Description | Requires `FBTDndZone`? |
|-----------|-------------|----------------------|
| `FBTPool` | Left panel — components, presets, templates | Yes |
| `FBTStage` | Center — drop zone for form components | Yes |
| `FBTEditor` | Right panel — property editor for selected component | No |
| `FBTScreenBar` | Screen tabs / navigation | No |
| `FBTToolbar` | Undo/redo, import/export, validate, help | No |
| `FBTPreviewPanel` | Live FBRE preview (inline) | No |
| `FBTPreviewFloat` | Floating preview overlay | No |
| `FBTHelpModal` | Keyboard shortcuts modal | No |
| `FBTConfirmDeleteDialog` | Confirmation prompt shown when deleting a component other conditions depend on | No |
| `FBTDndZone` | Drag-and-drop context wrapper | — |
| `FBTDefaultLayout` | The standard 3-column layout (used by `<FBT />`) | No — it renders its **own** `FBTDndZone`; do not nest it inside another one |

> **Render `FBTConfirmDeleteDialog` in custom layouts.** Deleting a component that other components' conditions depend on sets a pending-deletion state and waits for confirmation. `FBTDefaultLayout` renders the dialog for you; a fully custom layout must render it somewhere inside `FBTProvider`, or those deletes will appear to hang.

## Custom Presets & Templates

Definitions come in two kinds, and both are accepted by the same props:

- **Data-based** (`PresetData` / `TemplateData`, exported from `@sonata-innovations/fiber-types`): plain serializable JSON — `{ type, label, icon, components }` for presets, `{ type, label, icon, flow }` for templates. UUIDs are regenerated automatically on every drop, so the same definition can be dropped repeatedly (and stored server-side). Prefer this kind.
- **Factory-based** (`CustomPresetDefinition` / `CustomTemplateDefinition`): a `factory()` that returns fresh `Component[]` / `Flow` per drop. Kept for backward compatibility.

```tsx
import type { AnyPresetDefinition, AnyTemplateDefinition } from "@sonata-innovations/fiber-fbt";

const myPresets: AnyPresetDefinition[] = [
  {
    type: "custom-ssn",          // any unique string — see collision warning below
    label: "SSN",
    icon: "textInput",           // use any built-in icon name
    factory: () => [             // returns Component[]
      {
        uuid: crypto.randomUUID(),
        type: "inputText",
        properties: {
          label: "Social Security Number",
          placeholder: "XXX-XX-XXXX",
          // NOTE: the param key is `regex`, not `pattern`
          validation: { rules: [{ type: "pattern", params: { regex: "^\\d{3}-\\d{2}-\\d{4}$" } }] },
        },
      },
    ],
  },
];

const myTemplates: AnyTemplateDefinition[] = [
  {
    type: "custom-onboarding",
    label: "Employee Onboarding",
    icon: "contactInfo",
    factory: () => ({             // returns Flow
      uuid: crypto.randomUUID(),
      metadata: { name: "Onboarding" },
      screens: [/* ... */],
    }),
  },
];

<FBT customPresets={myPresets} customTemplates={myTemplates} />
```

> **Collision warning.** There is no required prefix for custom `type` strings, but built-in lookup runs **before** the custom map — a custom definition whose `type` collides with a built-in key is shadowed by the built-in and never used. FBT logs a `console.warn` at registration when it detects such a collision, but it does not rename or override for you. Pick a distinct namespace such as `custom-*` (which is also what the Fiber server enforces for stored definitions).

Whether a definition is treated as a preset or a template is determined by **which prop array it arrives in**, not by its `type` string. A template drop **merges** — it appends the template's screens to the current flow (nothing is replaced) and focuses the first appended screen.

## Public Exports

```ts
// Components
import { FBT, FBTProvider, FBTDefaultLayout } from "@sonata-innovations/fiber-fbt";
import { FBTPool, FBTStage, FBTEditor, FBTScreenBar, FBTToolbar } from "@sonata-innovations/fiber-fbt";
import { FBTPreviewPanel, FBTPreviewFloat, FBTHelpModal, FBTDndZone } from "@sonata-innovations/fiber-fbt";

// Hooks
import { useFBTStore, useFBTStoreApi } from "@sonata-innovations/fiber-fbt";

// Types
import type { FBTStoreState } from "@sonata-innovations/fiber-fbt";
import type { FBTProviderProps } from "@sonata-innovations/fiber-fbt";
import type { CustomPresetDefinition, CustomTemplateDefinition, AnyPresetDefinition, AnyTemplateDefinition } from "@sonata-innovations/fiber-fbt";
import type { Flow, FlowScreen, Component, FlowMetadata, FlowConfiguration, ComponentProperties } from "@sonata-innovations/fiber-fbt";

// Data-based definition shapes come from fiber-types, not fiber-fbt:
import type { PresetData, TemplateData } from "@sonata-innovations/fiber-types";
```

`FlowData` is **not** exported from `fiber-fbt` — import it from `@sonata-innovations/fiber-fbre` (or `fiber-types`).

## Integration Patterns

### FBT → Save → FBRE Pipeline

```tsx
import { useRef, useState } from "react";
import { FBT } from "@sonata-innovations/fiber-fbt";
import { FBRE } from "@sonata-innovations/fiber-fbre";
import "@sonata-innovations/fiber-fbt/styles";
import "@sonata-innovations/fiber-fbre/styles";
import type { Flow } from "@sonata-innovations/fiber-fbt";
import type { FlowData } from "@sonata-innovations/fiber-fbre"; // FlowData is NOT exported from fiber-fbt

function App() {
  const [savedFlow, setSavedFlow] = useState<Flow | null>(null);
  const [mode, setMode] = useState<"build" | "fill">("build");

  return mode === "build" ? (
    <>
      <FBT onFlowChange={(flow) => setSavedFlow(flow)} />
      <button onClick={() => setMode("fill")} disabled={!savedFlow}>
        Preview as Form
      </button>
    </>
  ) : (
    <FBRE
      flow={savedFlow!}
      onFlowComplete={(data: FlowData) => {
        console.log("Collected:", data);
        setMode("build");
      }}
    />
  );
}
```

## Pitfalls

**MUST:**
- Import styles separately (`import "@sonata-innovations/fiber-fbt/styles"`) — styles are not bundled with the JS
- Wrap `FBTPool` and `FBTStage` in `<FBTDndZone>` when using composable layout
- Render `<FBTConfirmDeleteDialog />` inside `FBTProvider` when using composable layout — without it, deleting a condition-dependent component hangs on a prompt that never appears

**DO NOT:**
- Use the `useFBTStore` hook outside `FBTProvider` — it requires context
- Access `storeRef.current` before the provider has mounted — it will be `null`
- Key long-lived UI to builder state across a `flow` prop change expecting it to survive — a new `flow` reference whose content **differs** from the current builder state re-runs the internal load and resets selection/undo. Echoing `onFlowChange` output straight back into `flow` is safe: FBT now has a content-identity guard (like FBTL ≥ 2.1.2) that treats a content-identical reload as a no-op, so a round-tripping parent (save → re-fetch → set `flow`) no longer loops. Still treat `flow` as *initial* state and drive post-mount changes through the builder UI or `storeRef` actions
