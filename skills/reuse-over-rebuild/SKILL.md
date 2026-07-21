---
name: reuse-over-rebuild
description: Before building any infrastructure layer (caching, prefetching, URL state, search, forms, sync, agent loops, memory), find and adopt the ecosystem standard instead of hand-rolling it. Use when about to write cross-cutting plumbing, when reviewing code that reinvents a known category, or when asked what library to use for a capability.
---

# Reuse over rebuild

Build the product, not the plumbing. Every infrastructure category you are
about to hand-roll almost certainly has a standard: battle-tested,
documented, maintained by people whose whole job it is. Hand-rolled versions
start smaller but end up as miniature, worse copies of the standard with no
docs, no devtools, and no community fixing their edge cases.

The same posture most projects already take for UI (unstyled primitives like
Base UI plus composed components like shadcn/ui instead of inventing a
component library) applies to every other layer.

## The process

1. **Name the category before writing code.** "I need a Map of fetched
   records with a TTL" is not a task, it is the category *client
   server-state caching*. "I need to remember which page and filters the
   person was on" is *URL state*. "I need fuzzy find across documents" is
   *client search*. If you can name the category, someone has solved it.
2. **Search the ecosystem first.** Use the internet: npm download counts,
   the project's own docs, and at least one recent independent comparison.
   Prefer sources dated within the last year; this space moves, and training
   data is stale (packages get renamed, defaults change, new winners emerge).
3. **Evaluate against the project's rules, typically in this order:**
   - Open source and self-hostable if cost or lock-in matters; keep paid
     vendors behind interfaces and environment config.
   - Actively maintained and widely adopted (downloads, release cadence,
     open-issue health).
   - Size and scope fit: prefer the framework-agnostic core over the
     framework wrapper when the code lives behind an adapter.
   - Composability with what is already in the project.
4. **Adopt behind a seam.** Put the library inside an adapter the rest of
   the app already talks to (a platform bridge, an `adapters/` directory,
   thin `components/ui/` wrappers). The seam is what keeps a later swap
   cheap and keeps callers ignorant of the vendor.
5. **Record the decision.** Every new runtime dependency gets a line in the
   project's decision log: category, chosen standard, one-line evidence,
   and what it replaced. Deliberate non-adoptions are worth a line too,
   with the reason.
6. **Delete the hand-rolled version.** Do not keep both. Two
   implementations of one category is a bug.

## Category cheat sheet (snapshot, mid-2026)

Starting points, not verdicts: verify the load-bearing choice against live
docs and downloads at build time.

| Category | Look at first |
| --- | --- |
| Client server-state cache, dedupe, prefetch, invalidation | TanStack Query (framework-agnostic core available); SWR if you want tiny |
| URL as state (params, filters, position) | nuqs |
| Client cache persistence | TanStack Query persisters (mind stale-data risk on mutable records) |
| Client full-text or fuzzy search | MiniSearch, Orama, Fuse.js |
| Forms and validation | react-hook-form plus a schema library (zod, valibot) |
| Tables, virtual lists | TanStack Table, TanStack Virtual |
| Local-first sync (the "everything instant" endgame) | Zero, ElectricSQL, LiveStore |
| UI primitives and components | Base UI or Radix primitives; shadcn/ui on top |

## When custom code is right

Custom code earns its place only when the logic is the product or the
domain:

- Domain heuristics no library owns (usage-learning ranks, permission
  evaluation, business policy).
- Glue inside an adapter that maps a standard onto your contracts.
- Anything where the standard would replace less than roughly 50 lines of
  trivial code with no edge cases; do not add a dependency to avoid a
  helper function.

## The smell test

If you find yourself adding TTLs, dedupe, retries, ranking, diffing, or
invalidation to custom plumbing, stop: that is the signal you are
rebuilding a category. Go back to step 1.

## Worked example

A web workspace grew a hand-rolled data layer inside its API adapter: a
state cache with a timestamp, two Maps for file prefetching, manual
in-flight dedupe, and invalidation sprinkled through every write path.
It worked, and it was about 200 lines of a miniature TanStack Query.

Applying this skill: the category was *client server-state caching*; the
standard was TanStack Query; the framework-agnostic `@tanstack/query-core`
fit because the code lived behind an adapter the UI never sees. The swap
replaced the Maps with a `QueryClient` (staleTime for freshness,
`prefetchQuery` for warming, `removeQueries` for invalidation), kept the
domain heuristic (most-opened-pages ranking) as custom code, deleted the
hand-rolled version, and logged the decision plus two deliberate
deferrals (nuqs while the URL held only three params, and cache
persistence because persisted bodies risked stale-revision edits).
Behavior stayed identical; the bespoke machinery became configuration.
