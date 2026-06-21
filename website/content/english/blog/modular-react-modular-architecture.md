---
title: "modular-react: Modular Architecture for React Applications"
meta_title: "modular-react - Modular Architecture for React Router and TanStack Router"
description: "Introducing modular-react, a framework on top of React Router and TanStack Router that lets you build apps as self-contained, independently-owned feature modules — with optional Journeys for multi-module workflows and a build-time Catalog for discovery."
date: 2026-06-21T00:00:00Z
image: ""
categories: ["Typescript", "React"]
author: "kibertoad"
tags: ["react", "react-router", "tanstack-router", "architecture", "frontend", "typescript", "modular"]
draft: false
---

## Introduction

Most React applications start simple and then accumulate features until adding one stops being simple. Adding a new screen is rarely just "add a screen." You register a route in `App.tsx`, add a navigation entry to the sidebar config, register a command in the command palette, maybe wire up an auth guard, and update a feature-flag map somewhere. Every one of those files is shared with every other feature. Once a few teams are editing the same files, you get merge conflicts on every branch, nobody clearly owns anything, and removing a feature turns into a scavenger hunt for the fragments it left behind.

This was the motivation behind [modular-react](https://github.com/kibertoad/modular-react), a framework that sits on top of [React Router](https://reactrouter.com/) or [TanStack Router](https://tanstack.com/router) and lets you structure an application as a set of self-contained, independently-owned modules rather than one big shell that knows about everything. Each feature lives in a single `modules/<name>/` directory that declares everything it contributes — routes, navigation items, command palette entries, UI zone contributions, and its dependencies. You add a feature by creating a module and registering it once. You delete a feature by deleting a directory and removing one line. The shell never has to know about any specific module; it just registers them, and the runtime wires everything together.

## The Problem at Hand

Routers handle routing well. But a real application is more than a route table. A feature usually needs to:

- Register one or more **routes**
- Add itself to the **navigation** (sidebar, header, breadcrumbs)
- Contribute **commands** to a command palette
- Drop UI into shared **zones** (detail panels, toolbars, status bars)
- Respect **cross-cutting concerns** like auth and feature flags
- Declare **dependencies** on shared services or other features

In a router-only setup, none of that lives together. Routing is in one file, navigation in another, commands in a third, and the feature itself ends up scattered across the codebase. The shared files in the middle turn into a coordination bottleneck. One team and a small app can live with this. Several teams pushing features into the same shell cannot.

## Prior Alternatives

Teams have tried a few approaches to this, each with its own trade-offs:

- **Convention and discipline.** Agree on where things go and hope everyone follows the rules. It holds up right until someone is in a hurry, and the shared files are still there for everyone to edit regardless.
- **Micro-frontends.** Split the app into separately built and deployed applications stitched together at runtime (Module Federation, single-spa, iframes). This gives you real isolation and independent deploys, but it costs a lot: runtime integration complexity, version-skew headaches, duplicated dependencies, cross-app communication boilerplate, and a build pipeline that is much harder to debug. It is a heavy solution for what is usually an organizational problem rather than a deployment one.
- **Plugin systems rolled in-house.** Many large apps end up growing their own registry of "features" with ad-hoc lifecycle hooks. These tend to be untyped, undocumented, and specific to one codebase, so the patterns never transfer and the edges are never quite finished.

modular-react sits in between: the ownership and isolation of a plugin architecture, in a single application and a single build, with full TypeScript types end to end and no runtime federation machinery.

## The Core Model

The core idea is small and declarative. A module is a plain object that describes what it contributes. Here is a billing module:

```typescript
import { defineModule } from "@react-router-modules/core";

export default defineModule<AppDependencies, AppSlots>({
  id: "billing",
  version: "1.0.0",
  createRoutes: () => [{ path: "billing", Component: BillingPage }],
  navigation: [{ label: "Billing", to: "/billing", group: "finance" }],
  slots: {
    commands: [{ id: "export", label: "Export Invoices", onSelect: exportInvoices }],
  },
  dynamicSlots: (deps) => ({
    commands: deps.auth.user?.isAdmin
      ? [{ id: "void", label: "Void Invoice", onSelect: voidInvoice }]
      : [],
  }),
});
```

Everything that feature contributes is right there: its route, its navigation entry, its static command palette contributions, and — via `dynamicSlots` — contributions that depend on application state (here, only showing "Void Invoice" to admins).

The shell assembles modules through a typed registry. It never imports a `BillingPage` or knows that billing has an admin-only command; it just registers the module and resolves a manifest:

```typescript
// app/registry.ts
import { createRegistry } from "@react-router-modules/runtime";
import billingModule from "./modules/billing";

const registry = createRegistry<AppDependencies, AppSlots>({
  stores: { auth: authStore },
  services: { httpClient },
});
registry.register(billingModule);

export const manifest = registry.resolveManifest();

// app/root.tsx
import { Outlet } from "react-router";
import { manifest } from "./registry";
export default () => <manifest.Providers><Outlet /></manifest.Providers>;

// Recompute dynamic slot contributions when state changes
authStore.subscribe(manifest.recalculateSlots);
```

The same pattern works on TanStack Router by swapping the package and mounting the manifest's `Providers` in `__root.tsx`. Framework modes for both routers are supported (`@react-router/dev/vite` and TanStack Start with file-based routing), and legacy setups can drop down to `registry.resolve()` directly.

A few things make this more than just "objects in a folder":

- **Zone-based layouts (slots).** The app defines named zones — sidebars, headers, command palettes, detail panels — and modules fill them. Only the contributions relevant to the current context appear in each zone, and `recalculateSlots` keeps dynamic contributions in sync with state changes.
- **Typed dependencies.** Modules declare what stores and services they need; the registry provides them. The generics (`AppDependencies`, `AppSlots`) flow through, so a module's `dynamicSlots` callback sees a fully typed `deps`.
- **Real deletion.** Because a module owns everything it contributes, removing it is removing a directory plus one `register` call. There is no central file to clean up.

There is a CLI for scaffolding so you are not writing boilerplate by hand:

```bash
# Create a module with a route
react-router-modules create module billing --route billing

# Create a store
react-router-modules create store notifications

# Scaffold a typed multi-module workflow
react-router-modules create journey customer-onboarding \
  --modules profile,plan,billing [--persistence]
```

## Journeys: Multi-Module Workflows

Modules work well for features that stand on their own. The harder flows are the ones that run through several features in sequence: a customer onboarding flow that goes profile confirmation → plan selection → payment, where each step lives in a different module and state has to be carried through. Wiring these by hand usually means one module importing another, shared mutable state, and brittle conditional navigation, which is exactly the cross-module coupling modular-react is trying to avoid.

**Journeys** (`@modular-react/journeys`) handle exactly this case. A journey is a typed, serializable workflow that composes several modules into a stepped flow, while the modules themselves stay journey-unaware. A module just declares what input each entry point accepts and what outcomes (exits) it can emit; the journey owns the transitions between them and the shared state.

The roles are cleanly separated:

- A **module** declares **entry points** (with typed input) and **exit points** (with typed output), and stays decoupled from any journey logic.
- A **journey** owns the **transitions** between one module's exit and the next module's entry, and manages the accumulated state.
- The **shell** registers modules and journeys and mounts a `<JourneyOutlet>` to render the current step.

A module exposes its contract like this:

```ts
export const profileExits = {
  profileComplete: defineExit<{ customerId: string; hint: PlanHint }>(),
  cancelled: defineExit(),
} as const;

export default defineModule({
  id: "profile",
  version: "1.0.0",
  exitPoints: profileExits,
  entryPoints: {
    review: defineEntry({
      component: ReviewProfile,
      input: schema<{ customerId: string }>(),
    }),
  },
});
```

The component receives typed props with the `input` and an `exit(name, output)` callback. The journey then defines how those exits route forward:

```ts
export const customerOnboardingJourney = defineJourney<Modules, OnboardingState>()({
  id: "customer-onboarding",
  version: "1.0.0",
  initialState: ({ customerId }: { customerId: string }) => ({
    customerId,
    hint: null,
    selectedPlan: null,
  }),
  start: (s) => ({ module: "profile", entry: "review", input: { customerId: s.customerId } }),
  transitions: {
    profile: {
      review: {
        profileComplete: ({ output, state }) => ({
          state: { ...state, hint: output.hint },
          next: {
            module: "plan",
            entry: "choose",
            input: { customerId: state.customerId, hint: output.hint },
          },
        }),
      },
    },
  },
});
```

Transition handlers are pure, synchronous functions that return exactly one of three outcomes:

- `{ next: StepSpec }` — advance to the next step
- `{ complete: payload }` — terminal success
- `{ abort: reason }` — terminal failure

```ts
readyToBuy: ({ output }) => ({
  next: {
    module: "billing",
    entry: "collect",
    input: { customerId: output.customerId, amount: output.amount },
  },
}),
needsMoreDetails: ({ output }) => ({
  abort: { reason: "profile-incomplete", missing: output.missing },
}),
```

Because the entire flow is described by typed entry/exit contracts and pure transitions, the boundaries between modules are checked end to end at compile time, and the journey's state is serializable. Serializability is the part that pays off in practice: a user can reload mid-flow, or hand the flow off to another session, and pick up where they left off. You register a persistence adapter and the runtime handles the rest:

```ts
registry.registerJourney(customerOnboardingJourney, {
  persistence: defineJourneyPersistence<OnboardingInput, OnboardingState>({
    keyFor: ({ input }) => `journey:${input.customerId}:customer-onboarding`,
    load: (k) => backend.loadJourney(k),
    save: (k, b) => backend.saveJourney(k, b),
    remove: (k) => backend.deleteJourney(k),
  }),
});
```

Rendering a running journey is a single component, and starting one is a single typed call against a handle:

```ts
import { JourneyOutlet } from "@modular-react/journeys";

function TabContent({ tab }) {
  return (
    <JourneyOutlet
      instanceId={tab.instanceId}
      loadingFallback={<LoadingSpinner />}
      onFinished={(outcome) => workspace.closeTab(tab.tabId)}
    />
  );
}

// Elsewhere in the shell:
export const customerOnboardingHandle = defineJourneyHandle(customerOnboardingJourney);

const instanceId = manifest.journeys.start(customerOnboardingHandle, { customerId });
```

For flows where several modules need to be on screen at once rather than one-after-another (think an editor with a canvas, a source picker, and an inspector, each owned by a different team), there is a companion **Compositions** package, and the two interoperate: a composition zone can host a journey, and a journey step can render a composition.

## Catalog: Build-Time Discovery

The Catalog tackles a different scaling problem. Once you have dozens of modules from many teams, "is there already a module that does X?" becomes a question nobody can answer quickly, and people end up rebuilding things that already exist.

The **Catalog** (`@modular-react/catalog`) is a build-time discovery portal. It scans your monorepo, harvests every module and journey descriptor, and emits a static, deployable, searchable HTML/JS/CSS portal — no server-side runtime required. You can host it on S3 and CloudFront, GitHub Pages, or plain nginx.

Setup is a config file at the workspace root and one command:

```typescript
import { defineCatalogConfig } from "@modular-react/catalog";

export default defineCatalogConfig({
  out: "dist-catalog",
  title: "Acme Portal",
  roots: [
    { name: "modules", pattern: "packages/*/src/index.ts", resolver: "defaultExport" },
    { name: "journeys", pattern: "journeys/*/src/index.ts", resolver: "defaultExport" },
  ],
  theme: { brandName: "Acme Portal", primaryColor: "#0E7C66" },
});
```

```bash
pnpm exec modular-react-catalog build
# preview locally:
pnpm exec modular-react-catalog serve dist-catalog
```

It produces more than a list of names. Modules can carry a typed `meta` block — owning team, domain, tags, lifecycle status, and links to docs, source, Slack, or runbooks:

```typescript
export default defineModule({
  id: "billing",
  version: "1.2.0",
  meta: {
    name: "Billing",
    description: "Issues invoices and processes payments.",
    ownerTeam: "billing-platform",
    domain: "finance",
    tags: ["payments", "invoicing"],
    status: "stable",
    links: { docs: "https://internal/docs/billing", slack: "https://acme.slack.com/archives/CXYZ" },
  },
  // …routes, slots, navigation, etc.
});
```

The portal turns that metadata into **faceted browsing** with pivot pages — `/teams/$team`, `/domains/$domain`, `/tags/$tag` — so you can see everything a team owns, or everything in the "finance" domain, in one click, with all filter state driven by the URL.

The most useful piece is the **cross-reference graph**. The harvester doesn't just read descriptors; it statically parses journey source with `oxc-parser` and recovers transition destinations from the return statements of transition handlers. That lets the catalog pre-compute, at build time, indexes like:

- `journeysByModule` — which journeys route through each module
- `journeysByInvokedJourney` — which journeys invoke each journey
- `moduleEntryUsage` and `moduleExitUsage` — which journeys route into a given module entry, and where each exit leads

So before you delete or change a module, the catalog already shows you which workflows depend on it. There is also an `enrich` hook for injecting org-specific metadata (for example, inferring `ownerTeam` from CODEOWNERS), and an extension API for adding custom detail-page tabs and filter facets.

## Examples

The repository ships several runnable example apps, in both React Router and TanStack Router variants, each demonstrating a different pattern:

- **integration-manager** — three sibling modules (Contentful, Strapi, GitHub) that all render the same generic screen with different columns, buttons, and feature flags. A good illustration of shared screens with per-module configuration.
- **customer-onboarding-journey** — the multi-step enrollment flow used throughout this post, showing entry/exit contracts, branching, shared state, and persistence.
- **editor-composition** — an editing interface where canvas, source picker, and inspector panels are each owned by different modules and coordinated through the Compositions package.
- **remote-capabilities** — navigation and slots driven by backend-served JSON rather than hardcoded module source.
- **active-project-manifest** — the manifest switches dynamically when the user changes projects, with each project supplying its own configuration.

## Getting Started

Pick your router and scaffold a workspace:

```bash
# React Router
npx @react-router-modules/cli init my-app --scope @myorg --module dashboard

# TanStack Router
npx @tanstack-react-modules/cli init my-app --scope @myorg --module dashboard

cd my-app && pnpm install && pnpm dev
```

The framework targets React 19, Node 22+, and pnpm workspaces (Yarn Berry and Bun work with minor edits after scaffolding). The router integration packages (`@react-router-modules` v2.x and `@tanstack-react-modules` v1.x) are stable, as are `@modular-react/core`, `react`, and the catalog and testing packages; the `compositions` package is earlier (v0.1.x) and may still have breaking changes.

modular-react is most worthwhile when an application has outgrown a single hand-maintained shell: plugin-style apps, teams contributing independent features in parallel, and large codebases where `App.tsx` and the sidebar config have become merge-conflict magnets. For a small single-team app, a plain router is still the right call.

The project is available on [GitHub](https://github.com/kibertoad/modular-react). Give it a try, and if you run into issues or have suggestions, please [open an issue](https://github.com/kibertoad/modular-react/issues). Contributions are welcome!
