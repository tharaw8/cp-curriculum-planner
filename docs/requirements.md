# Requirements - CP Curriculum Planner

Self-service study planner for Computer Engineering students at Chulalongkorn
University, curriculum revision 2566 (138 credits).

This document supersedes the earlier `docs/PROMPT.md`. It records what the
product must do and which decisions are settled. It does not describe the
current state of `src/` -- nothing already built is guaranteed correct.

---

## 1. Product

A student loads the app, gets a board of academic years and terms, drags courses
into the terms they intend to take them in, enters grades as they earn them, and
sees GPA, credits, graduation progress, probation status and honors eligibility
update live.

Capabilities:

- Build a personalized study plan.
- Drag and drop courses between terms.
- Semester GPA, cumulative GPA (GPAX), accumulated credits.
- Validate graduation requirements.
- Detect probation status.
- Predict honors eligibility.
- Reverse-calculate the minimum average grade needed to reach a target GPA.
- Export and import plans as JSON.

Students may take courses in any order. The app must not enforce the official
curriculum sequence.

## 2. Architecture (locked)

- Client-side only. No backend, no database, no authentication, no API.
- Persistence is `localStorage`, plus JSON export/import.
- Deployed as a static site (Vercel).
- Desktop only. Mobile is not a target.
- English UI only.
- Master curriculum data stays in JSON, separate from application logic.
- Business logic lives in an engine layer with no React imports. UI components
  stay presentational and receive computed values as props.

## 3. Tech stack and dependency budget

TypeScript + React + Vite + Tailwind CSS.

Dependencies are kept deliberately small. Current runtime deps are `react`,
`react-dom`, `tailwindcss`, `clsx`. Only one addition is planned:

- `dnd-kit` -- drag and drop. Worth the weight; hand-rolling this is a mistake.

Explicitly rejected, with the replacement to use instead:

| Rejected    | Use instead                                        |
| ----------- | -------------------------------------------------- |
| `zustand`   | `useReducer` + Context                              |
| `zod`       | hand-written validators in the validation layer     |
| `uuid`      | `crypto.randomUUID()`                               |
| `recharts`  | nothing; progress bars in plain CSS are sufficient  |

Anything beyond this list needs a deliberate decision, not a reflex `npm i`.

## 4. Curriculum data

Two files under `docs/`, both hand-maintained:

- `courses.json` -- the 36 hardcoded courses (90 of 138 credits). Keyed by
  course code. Sole source of truth for prerequisites.
- `requirements.json` -- credit requirements per category, honors thresholds,
  probation and retirement rules. Authoritative for all credit totals.

`outline-1024x576.png` is a reference diagram of the suggested 4-year plan. It
is a reading aid, not a source of truth -- see the discrepancies in section 11.

### Course record

```ts
{
  code: string;
  name_en: string;
  credits: number;
  counts_gpa: boolean;
  grading_type?: "S_U";   // present only when counts_gpa is false
  category: CategoryId;
  prerequisites: string[];
  default_term: string;   // "y1-t1" .. "y4-t2"
}
```

`requirements.json` uses `name_th` for category names while `courses.json` uses
`name_en` for course names. This asymmetry is intentional and stays as is.

### Categories

`requirements.json` is the authority. The eight categories sum to exactly 138.

| id                        | credits | type                   |
| ------------------------- | ------- | ---------------------- |
| `gen_ed`                  | 30      | min_credits_from_pool  |
| `science_math`            | 21      | fixed_list             |
| `eng_foundation`          | 11      | fixed_list             |
| `major_core`              | 40      | fixed_list             |
| `major_elective_required` | 6       | choose_n_from_pool     |
| `skill_21st_century`      | 6       | choose_n_from_pool     |
| `major_elective`          | 18      | choose_n_from_pool     |
| `free_elective`           | 6       | any_course             |

`major_elective` accepts overflow from `major_elective_required`.

### Core principle

**A course may satisfy only one category.** The same course must never be
counted twice across categories. This is the single most important invariant in
the engine.

## 5. Course cards

Hardcoded course:

```ts
{
  instanceId: string;
  courseCode: string;
  categoryId: string;
  letterGrade: string | null;
  isCustom: false;
}
```

User-defined course:

```ts
{
  instanceId: string;
  courseCode: string;
  categoryId: string;
  letterGrade: string | null;
  isCustom: true;
  customName: string;
  customCredits: number;
}
```

## 6. Plan structure

```
StudentPlan
  years[]
    terms[]
      courses[]
```

## 7. Layout

Four regions: header, kanban board, footer catalog, modal layer.

**Header** shows cumulative GPA, semester GPA, total credits, credit progress,
honors status, probation status.

**Board** is Year -> Term -> Course card. Each year holds Term 1, Term 2 and an
optional Summer. Every term is a drop target.

**Footer catalog** holds two kinds of block:

- *Hardcoded course blocks* -- exist exactly once, disappear once placed, return
  when deleted from the board, cannot be duplicated.
- *Pool blocks* (major electives, free electives, 21st-century skills, general
  education) -- unlimited copies, always visible, require user input.

### Term credit limits

- Regular term: 22 credits.
- Summer term: 9 credits.

Exceeding the limit shows a warning. The drop is still allowed.

### Summer terms

Hidden by default. Dragging a card into the gap between two academic years
reveals that year's summer term. For the final year, the summer term appears at
the far right of the board.

### Set-as-default

Applies a predefined track to one term.

- Term-level only.
- Requires overwrite confirmation.
- Excludes elective courses.
- Not available on summer terms.

## 8. Grading and calculations

```
A = 4.0   B+ = 3.5   B = 3.0   C+ = 2.5
C = 2.0   D+ = 1.5   D = 1.0   F  = 0.0
```

`W`, `S` and `U` are excluded: they contribute neither credits nor grade points.

GPA is `point_sum / credit_sum`. Round half-up to two decimal places.

### Honors calculator

```
remaining_points_needed =
  (target_gpa * total_countable_credits) - (current_gpa * completed_countable_credits)

required_average =
  remaining_points_needed / remaining_countable_credits
```

Handle: division by zero, impossible targets, already-completed curriculum.

Honors thresholds and probation/retirement rules live in `requirements.json`.
Two retirement rules there carry `interpretation_note` fields flagging readings
that were guessed and need confirmation against the official regulations.

### Prerequisites

Prerequisites are **advisory**. Show a warning on a course placed before its
prerequisite; never block the drop. This is the resolution of a contradiction in
the old PROMPT.md, which said sequence must not be enforced while the data
carried prerequisite edges.

## 9. Persistence and import/export

Persist to `localStorage` after every modification: student plan, preferences,
probation history, unlocked summer terms.

Export format:

```ts
{
  exportSchemaVersion: string;
  exportedAt: string;
  studentPlan: StudentPlan;
}
```

Import validation order: parse JSON -> validate schema version -> validate
structure -> ignore unknown entries -> show confirmation dialog -> overwrite.

## 10. Engine modules

Separate from UI. At minimum:

```
calculateGpa
calculateCredits
validateGraduation
calculateHonors
calculateProbation
validateCreditLimit
```

## 11. Known data caveats

Recorded deliberately so they are not rediscovered as bugs.

1. **`outline-1024x576.png` legend says `แกนระดับสาขาวิชา (39)`.** That is a
   typo in the diagram. Counting the 17 `major_core` courses in `courses.json`
   gives 40, which is what `requirements.json` states and what makes the
   category totals sum to 138.

2. **The image is titled "(2022)".** The curriculum is revision 2566. The title
   is stale or mistaken; `requirements.json` is correct.

3. **`2302127` (General Chemistry) has `default_term: "y1-t1"` while the outline
   places it in 1/2.** Intentional -- it reflects a different track.
   `default_term` only seeds the Set-as-default feature and is not
   authoritative; students reposition courses freely.

4. **Pool courses have no catalog and will not get one.** There are several
   hundred electives, offerings change year to year, and courses are retired and
   reintroduced. Users type the name and credits by hand. Do not build an
   autocomplete against a static list.

5. **The official department page's own credit breakdown does not reconcile.**
   It lists engineering fundamentals 15 and core 45 which sums to 141, not the
   138 it also states. `requirements.json` is internally consistent with
   `courses.json` and is what the engine uses.

6. **`prerequisites.json` was deleted.** It duplicated `courses.json` and had
   only 5 of the 11 courses that carry prerequisites.

7. **`2110471` (Computer Networks) previously listed `2110263` (Digital Logic
   Laboratory I) as a prerequisite.** Removed -- it was copied from `2110366`.
   Not yet re-verified against the real curriculum.
