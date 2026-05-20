# Vue Component Optimization Audit

Run a focused audit of Vue 3 components for performance and code-reuse opportunities. Complements the broader `/optimize` command, which targets the whole codebase.

## Usage

- `/vue-optimize` — scan all `.vue` files under `client/src`
- `/vue-optimize <path>` — scan a single file or a directory

## Procedure

1. **Resolve the target.** If the user provided a path argument, use it. Otherwise default to `client/src/views/*.vue` and `client/src/components/*.vue`.

2. **Apply the detection patterns** from the `vue-optimize` skill (`.claude/skills/vue-optimize/SKILL.md`) to each file:
   - Critical bugs (v-for index keys, prop mutation, unsafe date parsing, deep watchers, leaked timers)
   - Performance smells (template computations, v-show/v-if misuse, missing debouncing, sync imports of large components)
   - Reuse opportunities (duplicated setup logic, repeated template blocks, inlined API calls)
   - Quality concerns (long setup blocks, long templates, missing loading/error states)

3. **For Reuse findings, cross-reference at least two files** before reporting — a pattern that appears once is not a duplication.

4. **Skip findings** that are already documented as intentional in `CLAUDE.md` or `client/CLAUDE.md`.

5. **Produce a report** with this structure:

   ```
   # Vue Optimization Report

   Scanned <N> files in <path>.

   ## Critical (<N> findings)
   - <file>:<line> — <issue>
     Fix: <one-line suggestion>

   ## Performance (<N> findings)
   - <file>:<line> — <issue>
     Fix: <one-line suggestion>

   ## Reuse (<N> findings)
   - <issue>
     Affects: <file1>:<line>, <file2>:<line>, ...
     Fix: <one-line suggestion>

   ## Quality (<N> findings)
   - <file>:<line> — <issue>
     Fix: <one-line suggestion>

   ## Suggested order of work
   1. <top critical finding>
   2. <top performance finding>
   3. <top reuse finding>
   ```

6. **Do not edit any files.** This command produces an audit, not a refactor. After the report, ask the user which findings (if any) to act on.

## What to omit

- Style nits that don't affect runtime behavior or reuse
- "Could be more idiomatic" findings with no measurable benefit
- Findings under `node_modules`, `dist`, or `.git`
- Suggestions that contradict guidance in `CLAUDE.md` or `client/CLAUDE.md`
