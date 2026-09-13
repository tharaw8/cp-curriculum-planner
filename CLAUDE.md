# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run dev       # Vite dev server
npm run build     # tsc -b && vite build  -- type errors fail the build
npm run lint      # eslint .
npm run preview   # serve the production build
```

There is no test runner configured. Do not invent one or reference a test
command that does not exist; if tests are wanted, that is a decision to raise
first.

`npm run build` type-checks the whole project before bundling, so it is the real
verification step -- `npm run dev` will happily run code that does not compile.

## Where the project actually is

**`src/App.tsx` renders `PreferencesPage`, not the planner.** `PreferencesPage`
is a component playground: a sidebar of component names and a preview pane. It
exists to review the design system, and it is the only thing that runs today.

Everything under `src/components/` is presentational -- props in, markup out. No
state management, no drag and drop, no persistence, and no engine layer exists
yet. The JSON data in `docs/` is not imported by any code.

Treat the existing components as a first draft, not as settled architecture.
They predate the current spec and the owner has explicitly said they are not
guaranteed correct.

## Read this first

`docs/requirements.md` is the specification: product scope, locked architecture
decisions, business rules, and a "Known data caveats" section recording
discrepancies that are deliberate rather than bugs. Read it before changing
behavior or curriculum data. It supersedes `docs/PROMPT.md`, which was deleted.

## Domain rules that drive the design

The app is a client-only study planner for Chulalongkorn CP students
(curriculum 2566, 138 credits). No backend, no auth; `localStorage` plus JSON
export/import is the whole persistence story.

Three rules shape most of the logic:

- **A course may satisfy only one category.** Never double-count a course across
  the eight credit categories. This is the central invariant of the engine.
- **Course order is never enforced.** Students may place any course in any term.
  Prerequisites are advisory -- warn, never block.
- **Credit limits warn but never block.** 22 credits for a regular term, 9 for
  summer; over-limit drops are still allowed.

Business logic belongs in an engine layer with no React imports, consuming the
JSON data and returning plain values. Components receive computed results as
props. Do not push GPA, credit, honors or probation math into components, and do
not hardcode curriculum data in them.

## Curriculum data

`docs/courses.json` and `docs/requirements.json` are hand-maintained.

- `courses.json` -- 36 hardcoded courses, 90 of the 138 credits, keyed by course
  code. Sole source of truth for prerequisites.
- `requirements.json` -- authoritative for every credit total, plus honors,
  probation and retirement rules.

The remaining 48 credits are electives the user types in by hand. There is
deliberately no catalog of elective courses and there will not be one -- see
caveat 4 in `docs/requirements.md` for why. Do not build autocomplete against a
static elective list.

`outline-1024x576.png` is a reference diagram, not a source of truth. Where it
disagrees with `requirements.json`, the JSON wins; the specific known
disagreements are already catalogued in the spec.

## Dependency budget

This is a hard constraint, not a preference. Runtime deps are `react`,
`react-dom`, `tailwindcss`, `clsx`. Only `dnd-kit` is approved to be added.

Use `useReducer` + Context rather than Zustand, hand-written validators rather
than zod, `crypto.randomUUID()` rather than uuid, and CSS progress bars rather
than a charting library. Reaching for a package outside this list needs a
decision from the owner first.

## Component conventions

Each component is a folder holding the implementation plus an `index.ts` barrel
that re-exports it; imports go through the barrel.

Styling is Tailwind v4 via `@tailwindcss/vite`. There is no `tailwind.config.js`
-- `src/index.css` is a single `@import "tailwindcss"`. The house pattern is a
`Record<Variant, string>` lookup table of class strings merged with `clsx`, and
components spread `...props` onto the root element after extending the matching
`HTMLAttributes` type. Follow it.

## Known trap

`CourseCard` maps categories to badge colors using the keys `major`,
`elective`, `general`, `free`, but the real category ids in `courses.json` are
`major_core`, `gen_ed`, `science_math`, `eng_foundation`,
`major_elective_required`, `skill_21st_century`, `major_elective` and
`free_elective`. Every badge silently falls through to `neutral` on real data.
The playground previews hide this because they pass mock values that happen to
match. Fix the mapping when wiring real data in.

## Environment

Windows. The Bash tool runs Git Bash; PowerShell is also available. Prefer
forward slashes in paths.
