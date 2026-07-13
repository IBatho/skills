---
name: base-ui-primitives
description: "Build UI with Base UI (@base-ui/react) unstyled primitives. Use when implementing components with Base UI, styling headless primitives, composing with the render prop, or porting Radix UI patterns to Base UI. Covers anatomy, styling hooks, events, and how to fetch exact per-component docs on demand."
---

# Base UI Primitives

Base UI is an unstyled ("headless") React component library from the creators of Radix, Floating UI, and Material UI. It provides behavior and accessibility (WAI-ARIA, keyboard nav, focus management); you provide 100% of the styling. Stable since v1.0 (Dec 2025), currently v1.6.x, React 17+.

## Package name (critical, common trap)

The package is `@base-ui/react`. It was previously published as `@base-ui-components/react` and has been RENAMED. Training data often has the old name. Always use:

```bash
npm i @base-ui/react
```

```tsx
import { Popover } from '@base-ui/react/popover';   // per-component subpath import
import { Menu } from '@base-ui/react/menu';
```

Single package, tree-shakable. If docs conflict with prior knowledge, trust the docs (fetch them, see "Docs on demand" below).

## Platform

Base UI targets the web platform: browsers plus any embedded webview (Electron, Tauri). The same components serve a web app and a desktop app built on web tech. It is NOT for React Native or native toolkits (SwiftUI/AppKit).

## One-time app setup

1. Portal stacking: wrap app content in a root element with `isolation: isolate` so popups (rendered in portals) always sit above page content without z-index wars.
2. iOS 26+ Safari: add `body { position: relative }` so `position: absolute` backdrops cover the visual viewport.

## Anatomy: namespaced parts

Every component is a namespace of parts. Popups follow this nesting:

```tsx
<Popover.Root>                            {/* state owner, renders nothing */}
  <Popover.Trigger />                     {/* renders <button> by default */}
  <Popover.Portal>
    <Popover.Positioner sideOffset={8}>   {/* positioning props live HERE */}
      <Popover.Popup>                     {/* the styled surface */}
        <Popover.Arrow />
        <Popover.Title />
        <Popover.Description />
      </Popover.Popup>
    </Popover.Positioner>
  </Popover.Portal>
</Popover.Root>
```

Key difference from Radix: there is an explicit `Positioner` part between Portal and Popup. Positioning props (`sideOffset`, `side`, `align`, anchoring) go on the Positioner, not on the popup/content.

## Styling hooks (in order of preference)

1. `className` as a string, or as a function of state:
   `className={(state) => state.checked ? 'on' : 'off'}`
2. Data attributes for state, in plain CSS or Tailwind:
   `[data-checked]`, `[data-unchecked]`, `[data-disabled]`, `[data-highlighted]`, `[data-popup-open]`, `[data-side=...]`
   Tailwind: `data-popup-open:bg-neutral-100`, `data-highlighted:bg-neutral-950`
3. CSS variables exposed per component for dynamic values:
   `var(--available-height)`, `var(--anchor-width)`, `var(--transform-origin)`
4. `style` prop also accepts a function of state.

No CSS is bundled. Works with Tailwind (docs examples are Tailwind v4), CSS Modules, plain CSS, or CSS-in-JS (wrap parts with `styled()`).

## Animation

Popups expose `[data-starting-style]` and `[data-ending-style]` for CSS enter/exit transitions:

```css
.Popup {
  transition: scale 0.1s, opacity 0.1s;
}
.Popup[data-starting-style], .Popup[data-ending-style] {
  scale: 0.95;
  opacity: 0;
}
```

Tailwind: `transition data-starting-style:scale-95 data-starting-style:opacity-0 data-ending-style:scale-95 data-ending-style:opacity-0`.

## Composition: the `render` prop (NOT `asChild`)

Base UI has no `asChild`. To merge a part into your own component or change its element:

```tsx
<Menu.Trigger render={<MyButton size="md" />}>Open menu</Menu.Trigger>
<Menu.Item render={<a href="/settings" />}>Settings</Menu.Item>
```

- The custom component must forward `ref` and spread received props onto its DOM node.
- Nest `render` props to stack behaviors (e.g. a button that is Tooltip.Trigger + Dialog.Trigger + Menu.Trigger).
- Performance-sensitive or state-dependent content: pass a function
  `render={(props, state) => <span {...props}>{state.checked ? <On/> : <Off/>}</span>}`
- `useRender` hook and `mergeProps` utility exist for building your own render-prop components.

## State and events

- Uncontrolled by default; control via value prop + change handler (`open`/`onOpenChange`, `value`/`onValueChange`).
- Change handlers receive `(value, eventDetails)`. `eventDetails` has `reason` (why it changed, e.g. `'trigger-press'`, `'escape-key'`), `event` (native DOM event), `cancel()` (veto the state change while staying uncontrolled), and `allowPropagation()`.
- `event.preventBaseUIHandler()` on a React event stops Base UI's own handling (escape hatch).

```tsx
<Tooltip.Root onOpenChange={(open, details) => {
  if (details.reason === 'trigger-press') details.cancel();
}}>
```

## Docs on demand (use this instead of guessing)

Every docs page is fetchable as raw markdown. Before implementing a component you have not used in this session, fetch its page for the exact part list, props, data attributes, and CSS variables:

- Index of all pages: `https://base-ui.com/llms.txt`
- Per component: `https://base-ui.com/react/components/<slug>.md` (e.g. `dialog.md`, `select.md`, `number-field.md`)
- Handbook: `https://base-ui.com/react/handbook/styling.md`, `composition.md`, `animation.md`, `customization.md`, `forms.md`, `typescript.md`

## Component inventory (v1.6)

Accordion, Alert Dialog, Autocomplete, Avatar, Button, Checkbox, Checkbox Group, Collapsible, Combobox, Context Menu, Dialog, Drawer (swipe-to-dismiss), Field, Fieldset, Form, Input, Menu, Menubar, Meter, Navigation Menu, Number Field, OTP Field, Popover, Preview Card, Progress, Radio, Scroll Area, Select, Separator, Slider, Switch, Tabs, Toast, Toggle, Toggle Group, Toolbar, Tooltip. Utilities: CSP Provider, Direction Provider (RTL), mergeProps, useRender.

Forms: `Field`/`Fieldset`/`Form` provide labeling, validation, and consolidated error handling around the form controls; fetch `forms.md` when building forms.

## Coming from Radix? The delta

| Radix | Base UI |
|---|---|
| `radix-ui` or `@radix-ui/react-*` packages | `@base-ui/react/<component>` subpaths |
| `asChild` prop | `render` prop (element or function) |
| `Portal > Content` (positioning props on Content) | `Portal > Positioner > Popup` (positioning props on Positioner) |
| `data-state="open"` etc. | Dedicated attributes: `data-popup-open`, `data-checked`, `data-starting-style`/`data-ending-style` |
| `onOpenChange(open)` | `onOpenChange(open, eventDetails)` with `reason`/`cancel()` |

Component and part names are otherwise mostly parallel (Root/Trigger/Portal/Popup/Item). For migrating existing Radix or shadcn code, the `shadcn/ui@migrate-radix-to-base` skill has exact per-component mapping tables; shadcn also serves Base-based variants of its registry styles (`base-<style>`).

## Ecosystem context

- Floating UI is the positioning engine already inside Base UI's Positioner parts; only reach for it directly when building fully custom floating elements without Base UI.
- Material UI is the same company's pre-styled (Material Design) line; the opposite philosophy. Do not mix design systems casually.
- Base UI is the successor project of the Radix team (Colm Tuite, Jenna Smith) with Floating UI (James Nelson) and Material UI maintainers; prefer it over Radix for new code.

---

*Distilled from the official [Base UI documentation](https://base-ui.com) ([mui/base-ui](https://github.com/mui/base-ui), MIT), written against v1.6.0 (July 2026). Maintained at [IBatho/skills](https://github.com/IBatho/skills).*
