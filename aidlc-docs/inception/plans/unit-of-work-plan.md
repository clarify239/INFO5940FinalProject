# Unit of Work Plan: Research Tracker

Research Tracker is a single deployable app: one FastAPI process on each laptop. So the units here are **modules within one service**, not separately deployed services. A unit is a group of components and stories that one owner (plus a reviewer) designs, builds and tests in the Construction phase.

Answer the questions in Part B. Once the answers are checked and the plan is approved, I'll carry out Part C.

---

## Part A: Candidate Decompositions (for reference while answering)

**Inputs**:
- the 16 components in `application-design/components.md`
- the five ownership areas in `README.md` §4: PDF processing and retrieval; paper-card generation; comparison and classification; tracker summaries and coverage; interface and persistence
- a team of four

| Option | Units | Notes |
|---|---|---|
| **A: README areas + Foundation** | U0 Foundation; U1 PDF and Retrieval; U2 Paper Cards; U3 Comparison; U4 Summaries, Coverage and Export; U5 Interface and Persistence | Matches the README. Foundation holds what every unit needs first (shared types, Settings, Model Client and fake). Evaluation must be placed in one of the units |
| **B: Option A + separate Evaluation** | Option A plus U6 Evaluation | Evaluation has 6 stories and a test-set link, so it may deserve its own owner. Seven units for four people |
| **C: README areas only** | U1–U5, with the Foundation pieces spread across them | Fewest units, but every unit then waits on whichever unit owns `models.py` and the Model Client |

The components in each option:

| Component | A / B placement | C placement |
|---|---|---|
| `models.py`, Settings, Model Client (real and fake), project setup (`pyproject.toml`, Modelfile, `.env.example`) | U0 Foundation | Spread (for example U5) |
| PDF Intake, Indexer, Chroma part of Storage (`passages.py`) | U1 | U1 |
| Card Writer, Quote Checker | U2 | U2 |
| Comparison Agent, Agent Tools | U3 | U3 |
| Tracker Updater, Coverage, Markdown Exporter | U4 | U4 |
| Web, SQLite and file parts of Storage, `TrackerStore` | U5 | U5 |
| Paper Processor, Job Runner | See Question 2 | See Question 2 |
| Evaluation | See Question 1 | See Question 1 |

**Technical considerations checked**: all units run in one process on one laptop, with the same scaling and deployment needs. Units therefore differ only in ownership and build order, not in deployment.

---

## Part B: Planning Questions

Fill in the letter after each `[Answer]:` tag. If no option fits, choose the last option (Other) and describe what you want.

## Question 1: Unit grouping

Which decomposition should be used?

A) Option A: Foundation plus the five README areas, with Evaluation inside U4 (Summaries, Coverage and Export), since both check counts and the export

B) Option B: Foundation, the five README areas, and a separate U6 Evaluation unit

C) Option C: only the five README areas, with the Foundation pieces given to U5

X) Other (please describe after [Answer]: tag below)

[Answer]:

## Question 2: Where the Paper Processor and Job Runner go

The Paper Processor calls the Indexer, Card Writer, Agent, Tracker Updater and Storage. The Job Runner runs it in the background. Which unit should own them?

A) U5 Interface and Persistence, which already owns the web routes and Storage that start and save the work

B) U0 Foundation, as "integration," built as a skeleton early (with each step stubbed by the fake model) and filled in as units finish

C) A separate integration unit, built last

X) Other (please describe after [Answer]: tag below)

[Answer]:

## Question 3: Who owns Storage

Storage covers SQLite, Chroma and files behind one `TrackerStore` facade. The README puts "retrieval" with PDF processing and "persistence" with the interface.

A) Split: U1 owns `passages.py` (Chroma, including the rule that Chroma never embeds) and U5 owns `db.py`, `files.py`, `schema.sql` and the `TrackerStore` facade, including `commit` and cleanup

B) U5 owns all of Storage, and U1 only calls it

X) Other (please describe after [Answer]: tag below)

[Answer]:

## Question 4: Building units in parallel

Units depend on each other (for example U3's tools read from Storage). How should owners work before the units they depend on are finished?

A) Contract-first: Foundation is built first and includes the shared types, the `ModelClient` interface and fake, and a working in-memory `FakeTrackerStore` that follows the `TrackerStore` interface. Every other unit builds and tests against those, and the real Storage replaces the fake store when U5 is ready

B) Storage first: U5 builds the real `TrackerStore` early (with SQLite and Chroma in a temporary folder for tests), and other units test against it directly, with no fake store

X) Other (please describe after [Answer]: tag below)

[Answer]:

## Question 5: Running AI-DLC across four people

AI-DLC's Construction phase runs Functional Design and Code Generation unit by unit, and records progress in `aidlc-state.md` and `audit.md`. With four people working at once, these shared files may get conflicting edits.

A) Each owner runs their own unit's stages in their own session and branch. Unit documents go in `aidlc-docs/construction/{unit-name}/`. `aidlc-state.md` gets one line per unit, so edits rarely collide. Each owner appends to `audit.md` and resolves merge conflicts by keeping both entries

B) Same as A, but each unit also keeps its own `aidlc-docs/construction/{unit-name}/audit.md`, so the main `audit.md` only records cross-unit decisions and approvals

C) One person drives AI-DLC for all units in sequence, and the others implement from the approved plans

X) Other (please describe after [Answer]: tag below)

[Answer]:

## Question 6: Owners and reviewers

How should unit owners and reviewers be recorded?

A) Leave owner and reviewer fields as "to be assigned" in `unit-of-work.md`, for the team to fill in at the next meeting

B) I'll give the assignments now (write them after the [Answer]: tag, for example "U1: name, reviewer: name")

X) Other (please describe after [Answer]: tag below)

[Answer]:

## Question 7: Code and test layout

The technical document starts the app with `uv run uvicorn research_tracker.main:app`. How should folders be laid out?

A) One package in `src/research_tracker/` (uv's standard package layout), with the component modules from the application design. Tests go in `tests/{unit-name}/` (for example `tests/u2_paper_cards/`), so each owner has their own test folder. Shared fixtures (fake model, sample pages, the two test-set papers) go in `tests/conftest.py` and `tests/fixtures/`

B) Same as A, but the package sits at the repository root (`research_tracker/`) instead of under `src/`

C) One subpackage per unit (`src/research_tracker/u1_pdf/`, …) instead of per component

X) Other (please describe after [Answer]: tag below)

[Answer]:

## Question 8: Build order

In what order should the units go through Construction?

A) Foundation first (about 1–2 days, everyone helps or reviews). Then U1, U2, U3, U4 and U5 in parallel, against the Foundation contracts. Then integration through the Processor, then Evaluation. Units whose Functional Design affects others (U5 Storage schema, U3 tool results) finish their Functional Design first, so other units can rely on it

B) Strict dependency order, one unit at a time: Foundation → U5 → U1 → U2 → U3 → U4 → Evaluation

X) Other (please describe after [Answer]: tag below)

[Answer]:

---

## Part C: Generation Checklist

This runs after the answers are checked and the plan is approved.

- [ ] **1. Units** (`aidlc-docs/inception/application-design/unit-of-work.md`)
  - [ ] 1.1 Define each unit: purpose, components and modules owned, responsibilities, interfaces it provides and uses, owner and reviewer
  - [ ] 1.2 Record the code organization strategy (greenfield): package layout, test layout, shared fixtures
  - [ ] 1.3 Record the integration approach (contracts, fakes, when real parts replace fakes)
  - [ ] 1.4 Record how AI-DLC runs across owners (sessions, branches, state and audit files)
- [ ] **2. Dependencies** (`aidlc-docs/inception/application-design/unit-of-work-dependency.md`)
  - [ ] 2.1 Unit dependency matrix, with no loops
  - [ ] 2.2 Contracts each unit depends on (from component-methods.md), and who owns each contract
  - [ ] 2.3 Build order and parallel work, with the labeled test set workstream's dates
- [ ] **3. Story map** (`aidlc-docs/inception/application-design/unit-of-work-story-map.md`)
  - [ ] 3.1 Assign each of US-01 to US-29 to one primary unit, plus any supporting units
  - [ ] 3.2 Show the Demo path stories by unit
  - [ ] 3.3 Assign each shared constraint (C-1 to C-7) and each "Team practice" requirement to a unit
- [ ] **4. Check**
  - [ ] 4.1 Every story and every component belongs to exactly one primary unit
  - [ ] 4.2 Unit dependencies match the component dependencies (no unit relies on a component through another unit's internals)
  - [ ] 4.3 Update `aidlc-state.md` (including a per-unit progress list) and `audit.md`, and present for approval
