# skills

Agent skills by [Isaac Batho](https://github.com/IBatho). Skills are markdown instruction packages that teach coding agents (Claude Code, Codex, Cursor, and others) how to do a job well, installable with the [skills CLI](https://skills.sh/).

## Install

```bash
npx skills add IBatho/skills@base-ui-primitives
npx skills add IBatho/skills@desktop-to-web
npx skills add IBatho/skills@reuse-over-rebuild
```

Add `-g` to install globally (available to every project).

## Skills

| Skill | What it does |
|---|---|
| [`base-ui-primitives`](skills/base-ui-primitives/SKILL.md) | Build UI with [Base UI](https://base-ui.com) (`@base-ui/react`) unstyled primitives: anatomy, styling hooks, the `render` prop, events, a Radix-to-Base delta table, and a docs-on-demand workflow so agents fetch exact component docs instead of guessing from stale training data. |
| [`desktop-to-web`](skills/desktop-to-web/SKILL.md) | Port an existing desktop app (Electron, Tauri, or native `.dmg`/`.app`/`.exe`) to a web app: identify the stack inside the binary, pick the target web architecture, give every desktop capability an explicit fate via the [capability map](skills/desktop-to-web/references/capability-map.md) (keep / move server-side / degrade / cut), invert the security model, migrate user data, and ship. Pairs with `base-ui-primitives` for rebuilding the UI layer. |
| [`reuse-over-rebuild`](skills/reuse-over-rebuild/SKILL.md) | Stop agents from hand-rolling infrastructure that already has an ecosystem standard. A six-step process (name the category, search the ecosystem, evaluate, adopt behind a seam, log the decision, delete the hand-rolled version), a category cheat sheet (server-state caching, URL state, search, forms, sync, and more), a smell test for when custom plumbing is a rebuilt category, and the rules for when custom code IS right. |

## Design principles

- **Lean over encyclopedic.** A skill should teach the mental model and the traps, not mirror the whole API. Where docs are fetchable (Base UI serves every page as markdown, with an `llms.txt` index), the skill teaches the agent to pull the exact page it needs at build time. That keeps skills small and slow to rot.
- **Name the traps.** The highest-value lines are the ones that contradict stale training data (for example, `@base-ui-components/react` was renamed to `@base-ui/react`).

`base-ui-primitives` was written against Base UI v1.6.0 (July 2026). Because it defers to live docs for component specifics, minor releases should not break it.

`desktop-to-web` snapshots browser API support as of mid-2026 in its capability map and says so inline; verify load-bearing rows against MDN/caniuse at build time.

`reuse-over-rebuild` dates its category cheat sheet mid-2026 and tells the agent to re-verify choices against live downloads and docs at build time; the process itself does not rot.

## License

MIT. The `base-ui-primitives` skill is distilled from the official [Base UI documentation](https://base-ui.com) ([mui/base-ui](https://github.com/mui/base-ui), MIT); Base UI is built by the Base UI team at MUI. Repo structure inspired by [mattpocock/skills](https://github.com/mattpocock/skills). Drafted with Claude Code, curated by a human.
