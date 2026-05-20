---
name: vue-optimize
description: Detection patterns and prioritization for Vue 3 Composition API performance and reuse issues. Use this skill when reviewing, refactoring, or editing .vue files, or when explicitly auditing a Vue component for optimization opportunities.
---

# Vue Component Optimization Audit

This skill provides a framework for spotting and prioritizing optimization opportunities in Vue 3 Composition API components. It complements `client/CLAUDE.md`, which documents the patterns themselves — this skill focuses on *what to look for*, *how to prioritize*, and *what to surface to the user*.

## How to use this skill

When working on a `.vue` file:
1. Scan the file against the detection patterns below.
2. Group findings into the four severity tiers (Critical, Performance, Reuse, Quality).
3. Surface findings to the user, smallest-blast-radius fixes first.

When invoked via `/vue-optimize`, follow the audit procedure in `.claude/commands/vue-optimize.md`.

## Severity tiers

**Critical** — likely bugs, fix before anything else
- `v-for` keyed on `index` (causes incorrect DOM reuse when list reorders)
- Direct prop mutation (breaks one-way data flow, throws in strict mode)
- Date math without `isNaN(date.getTime())` validation (NaN propagates)
- Deep watchers that fire on every nested mutation when a `computed` would suffice
- `setInterval`/`setTimeout` without `onBeforeUnmount` cleanup (memory leak)

**Performance** — measurable runtime cost
- Heavy computation in template expressions or methods that should be `computed`
- `v-if` toggled rapidly when `v-show` would be cheaper (and vice versa)
- Missing `key` on `<router-view>` when navigating between routes that reuse the component
- Large lists (>50 items) rendered without windowing
- Watchers triggering API calls without debouncing
- Components imported synchronously that could be `defineAsyncComponent`'d

**Reuse** — duplication and missing abstractions
- Two or more files with near-identical `setup()` blocks (extract composable)
- Same template block repeated across views (extract component)
- API logic inlined in components instead of `api.js`
- Filter state managed locally when a shared composable already exists (e.g. `useFilters`)

**Quality** — maintainability
- `setup()` body over ~150 lines (extract composable or split component)
- Template over ~100 lines (extract sub-components)
- Hardcoded strings for status/category/etc. (extract constants or use locales)
- Missing loading/error states on async data
- Mixing Options API and Composition API in the same file

## Detection patterns

Concrete grep-friendly cues for each smell:

### Critical
- `:key="index"` or `:key="(item, index)"` → v-for index key
- `props\.\w+\.` followed by `=` (assignment) → prop mutation
- `new Date(.*)\.get` without an `isNaN` guard nearby → unsafe date parse
- `watch\(.*\{\s*deep:\s*true` → deep watcher; consider `computed`
- `setInterval\|setTimeout` not paired with `onBeforeUnmount` → leak risk

### Performance
- Inline template expressions like `{{ items.filter(...).map(...) }}` → move to `computed`
- `v-show` on rarely-toggled content / `v-if` on frequently-toggled content → swap them
- `axios.` called directly in a component → move to `api.js`
- `<router-view>` without `:key="$route.fullPath"` on routes that should remount

### Reuse
- Multiple files importing the same group of refs together with the same helper functions → composable candidate
- Two views with similar table rendering (>20 lines of template) → extract a shared `<DataTable>`-style component
- Repeated `const formatDate = (d) => ...` or `const formatCurrency = ...` → move to a shared util

### Quality
- `setup()` body length: count lines between `setup(...)` and its `return` — flag if >150
- Inline `style="..."` on >5 elements → move to scoped CSS
- Magic numbers in `v-if` conditions → extract named constants

## Prioritization rules

1. **Always fix Critical first.** These are bugs in disguise.
2. **Performance issues with user-visible impact** (slow interactions, jank, large bundle) outrank Reuse.
3. **A Reuse refactor is only worth proposing when the duplicated pattern appears 3+ times** — two similar blocks may just be coincidence.
4. **Don't propose a refactor without an example call-site** showing how the new composable or component would be used.
5. **Respect existing project conventions.** If `client/CLAUDE.md` documents a pattern as intentional, do not flag deviations from generic Vue best practice that align with the project's own guidance.

## Output format

When reporting findings, group by severity tier and include:
- File path and line number
- One-sentence description of the issue
- Suggested fix in one or two lines (not a full implementation, unless the user asks)
- For Reuse findings, list every affected file
