---
title: Fiber Integration — Start Here
applies-to:
  - "@sonata-innovations/fiber-fbre@^4.0"
  - "@sonata-innovations/fiber-fbt@^3.0"
  - "@sonata-innovations/fiber-fbtl@^3.0"
  - "@sonata-innovations/fiber-theme-editor@^2.0"
read-when: "Choosing which Fiber packages to install, or looking for the right integration guide."
---

# Fiber Integration — Start Here

> For AI assistants and developers integrating Fiber libraries into a parent application.
> This page routes you to the right guide; each package has its own.

## Which package do I need?

| | FBT (Builder) | FBTL (Lite Builder) | FBRE (Render Engine) |
|---|---|---|---|
| Package | `@sonata-innovations/fiber-fbt` | `@sonata-innovations/fiber-fbtl` | `@sonata-innovations/fiber-fbre` |
| Import | `import { FBT } from "@sonata-innovations/fiber-fbt"` | `import { FBTL } from "@sonata-innovations/fiber-fbtl"` | `import { FBRE } from "@sonata-innovations/fiber-fbre"` |
| Styles | `import "@sonata-innovations/fiber-fbt/styles"` | `import "@sonata-innovations/fiber-fbtl/styles"` | `import "@sonata-innovations/fiber-fbre/styles"` |
| Peer deps | `react ^18.0.0 \|\| ^19.0.0`, `react-dom ^18.0.0 \|\| ^19.0.0` | same | same |
| Audience | Power users / form designers | Non-technical end users embedded in parent apps | End users filling in the form |
| Purpose | Full-featured form builder → outputs Flow JSON | Stripped-down builder → outputs Flow JSON (same schema) | Consumes Flow JSON → renders form → outputs FlowData |

**Choosing a builder:** FBT exposes the full feature surface (screens, multi-rule conditions, calculations, reference markup, the full 30-type component palette). FBTL exposes seven question types + Information Screens + single-rule conditions, designed for non-technical end users, and the host can narrow that palette further. Both produce the same Flow schema, so a flow authored in either is renderable by FBRE and portable between them. FBTL preserves advanced properties on loaded flows without editing them (surfaced via a "Has custom configuration" badge that discloses which settings it cannot touch).

## Installation

All `@sonata-innovations/*` packages are published to the **public npm registry**. No special registry configuration is needed.

```bash
# FBRE only (render engine)
npm install @sonata-innovations/fiber-fbre

# FBT only (full-featured builder)
npm install @sonata-innovations/fiber-fbt

# FBTL only (lite builder for end users)
npm install @sonata-innovations/fiber-fbtl

# FBT + FBRE (author + render)
npm install @sonata-innovations/fiber-fbt @sonata-innovations/fiber-fbre

# FBTL + FBRE (author + render, end-user authoring)
npm install @sonata-innovations/fiber-fbtl @sonata-innovations/fiber-fbre
```

Peer dependencies (`react`, `react-dom`) must already be in your project. Transitive dependencies (`zustand`, `@sonata-innovations/fiber-types`, `@sonata-innovations/fiber-shared`, etc.) are installed automatically.

## Guides

| Guide | Read when |
|-------|-----------|
| [FBRE — Render Engine](fbre.md) | Rendering flows: props and integration modes (local / remote / server-driven), `onFlowComplete`, confirmation screen, store access, theming, pre-population |
| [FBT — Builder UI](fbt.md) | Embedding the full builder: getting the Flow out, composable layout, custom presets and templates |
| [FBTL — Lite Builder UI](fbtl.md) | Embedding the lite builder: controlled-component contract, scope, save lifecycle, normalizing generator-authored flows |
| [Theme Editor](theme-editor.md) | Embedding the plug-and-play theming widget |
| [Style Families](../features/style-families.md) | Choosing a style, the focused presentation, and the auto-advance / Enter-to-advance flags |
| [Flow JSON Quick Reference](../schema/flow-quick-reference.md) | Constructing or interpreting Flow JSON: component types, conditions, validation, options, widths |
| [Flow Schema (full)](../schema/flow-schema.md) | Complete schema reference for every component type and property |
| [Fiber Concepts](../fiber-concepts.md) | Conceptual model: Flow, Screen, Component, FlowData, condition and validation systems |

## Where these docs live

Every guide above ships **inside its npm package**, under `node_modules/<package>/docs/`, alongside an `AGENTS.md` routing guide at the package root. Those copies are version-matched to the code you installed — prefer them once you have the package. Schema docs ship in `@sonata-innovations/fiber-types`, a dependency of every other Fiber package.

The same docs are browsable at **[github.com/sonata-innovations/fiber-docs](https://github.com/sonata-innovations/fiber-docs)**, which also publishes an [`llms.txt`](https://raw.githubusercontent.com/sonata-innovations/fiber-docs/main/llms.txt) index and an [`llms-full.txt`](https://raw.githubusercontent.com/sonata-innovations/fiber-docs/main/llms-full.txt) single-file bundle for AI assistants.

The Flow schema is machine-validatable at a stable URL, with no install and no auth:

```bash
npx ajv-cli validate --spec=draft2020 \
  -s https://raw.githubusercontent.com/sonata-innovations/fiber-docs/main/schema/flow-schema.json \
  -d my-flow.json
```
