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

[Answer]: A. Six units for four people is already a stretch, and seven (option B) would leave several people owning two units each. U4 is the lightest unit: Coverage and the Exporter are pure functions, and the Tracker Updater is a couple of model calls. Evaluation mostly re-checks counts and the export anyway, so U4 has room for it. Option C would make every unit wait on whoever owns models.py and the Model Client.

## Question 2: Where the Paper Processor and Job Runner go

The Paper Processor calls the Indexer, Card Writer, Agent, Tracker Updater and Storage. The Job Runner runs it in the background. Which unit should own them?

A) U5 Interface and Persistence, which already owns the web routes and Storage that start and save the work

B) U0 Foundation, as "integration," built as a skeleton early (with each step stubbed by the fake model) and filled in as units finish

C) A separate integration unit, built last

X) Other (please describe after [Answer]: tag below)

[Answer]: B. It matches the earlier execution-plan point about integrating early: a thin end-to-end version exists from the start, and each unit replaces its stub as it finishes. U5 is already the heaviest unit (Web, SQLite, files, commit, cleanup), so option A would overload it. Option C is the "integrate at the end" risk we wanted to avoid. One gap: Foundation is a 1–2 day group effort, so the skeleton needs a named owner afterward, or nobody owns integration once Foundation is done.

## Question 3: Who owns Storage

Storage covers SQLite, Chroma and files behind one `TrackerStore` facade. The README puts "retrieval" with PDF processing and "persistence" with the interface.

A) Split: U1 owns `passages.py` (Chroma, including the rule that Chroma never embeds) and U5 owns `db.py`, `files.py`, `schema.sql` and the `TrackerStore` facade, including `commit` and cleanup

B) U5 owns all of Storage, and U1 only calls it

X) Other (please describe after [Answer]: tag below)

[Answer]: A. Search quality depends on how passages are chunked and embedded, so U1 should own both sides of retrieval, as the README says. It also takes some load off U5. To keep this clean, only TrackerStore should call passages.py. Other units go through the facade, which keeps the "only Storage imports chromadb" rule true.

## Question 4: Building units in parallel

Units depend on each other (for example U3's tools read from Storage). How should owners work before the units they depend on are finished?

A) Contract-first: Foundation is built first and includes the shared types, the `ModelClient` interface and fake, and a working in-memory `FakeTrackerStore` that follows the `TrackerStore` interface. Every other unit builds and tests against those, and the real Storage replaces the fake store when U5 is ready

B) Storage first: U5 builds the real `TrackerStore` early (with SQLite and Chroma in a temporary folder for tests), and other units test against it directly, with no fake store

X) Other (please describe after [Answer]: tag below)

[Answer]: A, with one addition. U5 can't finish real Storage in a day or two, and with option B everyone would be blocked waiting on it. The risk with a fake store is that it slowly stops behaving like the real one, for example by forgetting the "committed only" or "relevant only" filter. Addition: write one set of contract tests and run it against both FakeTrackerStore and the real TrackerStore, so the two can't drift apart. Also keep the fake small: only the methods other units actually call.

## Question 5: Running AI-DLC across four people

AI-DLC's Construction phase runs Functional Design and Code Generation unit by unit, and records progress in `aidlc-state.md` and `audit.md`. With four people working at once, these shared files may get conflicting edits.

A) Each owner runs their own unit's stages in their own session and branch. Unit documents go in `aidlc-docs/construction/{unit-name}/`. `aidlc-state.md` gets one line per unit, so edits rarely collide. Each owner appends to `audit.md` and resolves merge conflicts by keeping both entries

B) Same as A, but each unit also keeps its own `aidlc-docs/construction/{unit-name}/audit.md`, so the main `audit.md` only records cross-unit decisions and approvals

C) One person drives AI-DLC for all units in sequence, and the others implement from the approved plans

X) Other (please describe after [Answer]: tag below)

[Answer]: B. When several branches each add entries to the end of the same audit.md, nearly every merge conflicts. A per-unit audit file avoids that, and the main file stays readable for decisions that span units. Option C makes one person the bottleneck for every approval, which defeats the plan to let each unit's owner and reviewer approve their own unit.

## Question 6: Owners and reviewers

How should unit owners and reviewers be recorded?

A) Leave owner and reviewer fields as "to be assigned" in `unit-of-work.md`, for the team to fill in at the next meeting

B) I'll give the assignments now (write them after the [Answer]: tag, for example "U1: name, reviewer: name")

X) Other (please describe after [Answer]: tag below)

[Answer]: B if the team has already decided, otherwise A. That's a team decision, not mine. One thing to keep in mind when assigning: U0 is shared, but that still leaves five units for four people. A workable split gives U4 (Evaluation included) to the person who also owns the integration skeleton from Q2. U3 (the agent) and U5 (web and storage) are the heaviest, so their owners shouldn't also get a second unit.

## Question 7: Code and test layout

The technical document starts the app with `uv run uvicorn research_tracker.main:app`. How should folders be laid out?

A) One package in `src/research_tracker/` (uv's standard package layout), with the component modules from the application design. Tests go in `tests/{unit-name}/` (for example `tests/u2_paper_cards/`), so each owner has their own test folder. Shared fixtures (fake model, sample pages, the two test-set papers) go in `tests/conftest.py` and `tests/fixtures/`

B) Same as A, but the package sits at the repository root (`research_tracker/`) instead of under `src/`

C) One subpackage per unit (`src/research_tracker/u1_pdf/`, …) instead of per component

X) Other (please describe after [Answer]: tag below)

[Answer]: B. The technical document's uv run uvicorn research_tracker.main:app and the execution plan's "application code at the repository root" both assume the package sits at the root. The src/ layout in option A works too, but it needs the package set up for installation, and two documents would have to change. The per-unit test folders and shared fixtures from option A still apply.

## Question 8: Build order

In what order should the units go through Construction?

A) Foundation first (about 1–2 days, everyone helps or reviews). Then U1, U2, U3, U4 and U5 in parallel, against the Foundation contracts. Then integration through the Processor, then Evaluation. Units whose Functional Design affects others (U5 Storage schema, U3 tool results) finish their Functional Design first, so other units can rely on it

B) Strict dependency order, one unit at a time: Foundation → U5 → U1 → U2 → U3 → U4 → Evaluation

X) Other (please describe after [Answer]: tag below)

[Answer]: A, with a wording fix. Parallel work against the contracts is the only way to fit three weeks, and having U5's schema and U3's tool results designed first is the right priority. The fix: "then integration through the Processor" contradicts Q2 B. With a skeleton in Foundation, integration happens all along, not as a separate step afterward. Step 2.3 should also put real dates on this, since Foundation falls in Thanksgiving week.

---

## Part B2: Follow-up Questions

## Follow-up 1: Who owns the Processor and Job Runner after Foundation (Q2 + Q6)

Q2 puts the Paper Processor and Job Runner skeleton in U0 Foundation, and notes it needs a named owner once the group effort ends. Q6 suggests that the U4 owner also owns "the integration skeleton." It is not yet clear which unit the code formally belongs to after Foundation. That decides whose Functional Design and Code Generation covers the Processor's retry, commit-failure and progress rules, and whose test folder holds those tests.

Where should the Processor and Job Runner live once Foundation is finished?

A) They move into U4 (which becomes "Summaries, Coverage, Export, Evaluation and Integration"). U0 builds the first skeleton; U4's owner then owns the code, its design rules and its tests. U4 already calls the Processor directly from Evaluation

B) They stay in U0, which becomes a long-lived unit with one named owner (who may also own U4). U0 gets its own Functional Design for the Processor and Job Runner

X) Other (please describe after [Answer]: tag below)

[Answer]: A. Moving them into U4 gives integration a clear owner without adding a sixth long-lived unit, and U4's Evaluation already calls the Processor directly, so its owner exercises the Processor in every evaluation run. U4 was the lightest unit, so it has room. Because the commit-failure rule depends on U5's TrackerStore.commit, U5's owner reviews that part of U4's Functional Design.

## Follow-up 2: Foundation dates around Thanksgiving (Q8)

Q8 asks for real dates, and Foundation falls in Thanksgiving week (Thursday, November 26, 2026). The parallel unit work needs Foundation's contracts (shared types, `ModelClient` and fake, `FakeTrackerStore`, contract tests, Processor skeleton) before it can start.

When should Foundation be built?

A) Before the break: Nov 24–25, with the team working on it remotely. Parallel units start Nov 27 (or right after the break), and contracts are frozen by Nov 25

B) After the break: Nov 30 – Dec 1. Parallel units run Dec 2–9, integration and evaluation Dec 9–13, buffer Dec 14–16. Over the break, only optional work happens (test set selection, reading, Functional Design drafts)

C) Split: one or two people build the Foundation contracts over the break (Nov 25–29), and the whole team reviews and freezes them on Nov 30

X) Other (please describe after [Answer]: tag below)

[Answer]: C if one or two people volunteer, otherwise B. Most of the Foundation contract is already written in component-methods.md, so building it is about a day or two for one person, and a whole-team review on Nov 30 still lets everyone agree before freezing. This starts parallel work on Dec 1 instead of Dec 2. A is too rushed, since Units Generation isn't approved yet. Test-set selection happens over the break under either option.

---

## Part B3: Decisions Applied

- **Units** (Q1: A, follow-up 1: A):
  - **U0 Foundation**: shared types (`models.py`), Settings, `ModelClient` and fake, minimal `FakeTrackerStore`, `TrackerStore` contract tests, project setup, and the first Processor and Job Runner skeleton
  - **U1 PDF and Retrieval**
  - **U2 Paper Cards**
  - **U3 Comparison**
  - **U4 Summaries, Coverage, Export, Evaluation and Integration**: after Foundation, owns the Paper Processor and Job Runner code, their Functional Design and their tests
  - **U5 Interface and Persistence**
- **Reviews across units**:
  - U5's owner reviews the commit-failure part of U4's Functional Design (follow-up 1)
  - U5's Storage schema and U3's tool results finish Functional Design first (Q8)
- **Storage ownership** (Q3: A): U1 owns `passages.py`. U5 owns `db.py`, `files.py`, `schema.sql` and `TrackerStore`. Only `TrackerStore` calls `passages.py`
- **Parallel work** (Q4: A):
  - contract-first against U0
  - one `TrackerStore` contract test suite runs against both `FakeTrackerStore` and the real store
  - the fake implements only the methods other units call
- **AI-DLC across owners** (Q5: B):
  - each owner works in their own session and branch
  - unit documents and a per-unit `audit.md` go in `aidlc-docs/construction/{unit-name}/`
  - the main `audit.md` records cross-unit decisions and approvals
  - `aidlc-state.md` has one progress line per unit
- **Owners** (Q6, read as A): "to be assigned" by the team. Guidance: U4 (with Evaluation and Integration) goes to someone without a second unit, and the U3 and U5 owners take no second unit
- **Layout** (Q7: B):
  - the `research_tracker/` package sits at the repository root, with component modules
  - tests go in `tests/{unit-name}/`, shared fixtures in `tests/conftest.py` and `tests/fixtures/`, and contract tests in `tests/contracts/`
  - pytest gets the repository root on its import path
- **Build order** (Q8: A, follow-up 2), with integration running continuously through the skeleton:

  | Dates (2026) | Track C (one or two volunteers build Foundation) | Track B (no volunteers) |
  |---|---|---|
  | Nov 25–29 | Volunteers build U0; the team selects test-set papers | Optional only: test-set selection, Functional Design drafts |
  | Nov 30 | Whole team reviews and freezes U0 contracts | U0 built together |
  | Dec 1 | Parallel units start | U0 finished and frozen |
  | Dec 2 | — | Parallel units start |
  | Dec 9–13 | Integration checks, evaluation runs, tuning | Same |
  | Dec 14–16 | Quote checklist, demo rehearsal, buffer | Same |

  The team picks the track before the break, by Nov 25. Either way, U5 (schema) and U3 (tool results) do Functional Design first in the parallel phase.

---

## Part C: Generation Checklist

This runs after the answers are checked and the plan is approved.

- [x] **1. Units** (`aidlc-docs/inception/application-design/unit-of-work.md`)
  - [x] 1.1 Define each unit: purpose, components and modules owned, responsibilities, interfaces it provides and uses, owner and reviewer
  - [x] 1.2 Record the code organization strategy (greenfield): package layout, test layout, shared fixtures
  - [x] 1.3 Record the integration approach (contracts, fakes, when real parts replace fakes)
  - [x] 1.4 Record how AI-DLC runs across owners (sessions, branches, state and audit files)
- [x] **2. Dependencies** (`aidlc-docs/inception/application-design/unit-of-work-dependency.md`)
  - [x] 2.1 Unit dependency matrix, with no loops
  - [x] 2.2 Contracts each unit depends on (from component-methods.md), and who owns each contract
  - [x] 2.3 Build order and parallel work, with the labeled test set workstream's dates
- [x] **3. Story map** (`aidlc-docs/inception/application-design/unit-of-work-story-map.md`)
  - [x] 3.1 Assign each of US-01 to US-29 to one primary unit, plus any supporting units
  - [x] 3.2 Show the Demo path stories by unit
  - [x] 3.3 Assign each shared constraint (C-1 to C-7) and each "Team practice" requirement to a unit
- [x] **4. Check**
  - [x] 4.1 Every story and every component belongs to exactly one primary unit
  - [x] 4.2 Unit dependencies match the component dependencies (no unit relies on a component through another unit's internals)
  - [x] 4.3 Update `aidlc-state.md` (including a per-unit progress list) and `audit.md`, and present for approval
