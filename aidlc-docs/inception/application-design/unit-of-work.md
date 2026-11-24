# Units of Work: Research Tracker

**Decisions applied**: `unit-of-work-plan.md` Part B3. Research Tracker is one deployable app (one FastAPI process per laptop), so each unit is a **module group inside one service**, owned by one person with one reviewer.

| Unit | Name | Folder name (docs and tests) | Owner | Reviewer |
|---|---|---|---|---|
| U0 | Foundation | `u0_foundation` | To be assigned (volunteers, Track C; or whole team, Track B) | Whole team (freeze review) |
| U1 | PDF and Retrieval | `u1_pdf_retrieval` | To be assigned | To be assigned |
| U2 | Paper Cards | `u2_paper_cards` | To be assigned | To be assigned |
| U3 | Comparison | `u3_comparison` | To be assigned | To be assigned |
| U4 | Summaries, Coverage, Export, Evaluation and Integration | `u4_summaries_integration` | To be assigned | To be assigned |
| U5 | Interface and Persistence | `u5_interface_persistence` | To be assigned | To be assigned |

**Assignment guidance** (story plan Q6):
- Five long-lived units for four people means one person owns two.
- U3 and U5 are the heaviest, so their owners take no second unit.
- U4 has grown since it now includes Evaluation and Integration (see the workload note under U4). Its owner should not take a second unit either.
- That leaves the lighter U1 or U2 as the natural pair for one person.
- The labeled test set workstream also needs an owner and a second labeler. It fits best with whoever owns U1 and U2, since U4's owner uses the labels and should not write them alone.

---

## U0: Foundation

- **Purpose**: the shared contracts and the thin end-to-end skeleton every other unit builds against. Short-lived: built first, reviewed and frozen by the whole team, then handed over.
- **Components and files**:
  - `research_tracker/models.py`: all shared data types
  - `research_tracker/settings.py`: first version
  - `research_tracker/model_client/` (`base.py`, `ollama.py`, `fake.py`): first working version
  - interface definitions (protocols) for `TrackerStore` and the Chroma `PassageIndex` used by `passages.py`
  - `tests/fakes/fake_store.py`: a minimal `FakeTrackerStore` with only the methods other units call, including the committed-only and relevant-only filters
  - `tests/contracts/test_tracker_store_contract.py`: one suite, run against `FakeTrackerStore` now and the real `TrackerStore` later
  - `tests/conftest.py` and `tests/fixtures/`: the fake model fixture, sample page texts, and later the two test-set papers
  - skeleton `research_tracker/processor.py` and `jobs.py`: every step present, each calling a stub that uses the fake model
  - skeleton `research_tracker/main.py`: an app that starts, binds to `127.0.0.1` and serves one placeholder page
  - project setup: `pyproject.toml` (Python 3.12, dependencies, Hypothesis and pytest as dev dependencies, a pytest `model` marker, the repository root on pytest's import path), `uv.lock`, `Modelfile`, `.env.example`, `.gitignore` (`.env`, `data/`, `eval-data/`), README setup section
- **Responsibilities**: make it possible for U1–U5 to start in parallel and test without Ollama or each other.
- **Provides**: every contract in `unit-of-work-dependency.md` marked "defined in U0."
- **Uses**: nothing.
- **Handover after the freeze**:

  | Item | Maintained by |
  |---|---|
  | `models.py` | Shared. Any change needs a review from every unit that uses the changed type |
  | Settings | U5 |
  | Model Client, including the Ollama checks for thinking mode (A-3), schema enforcement (Q6) and context size (A-4) | U2, the first and heaviest user of structured output |
  | `TrackerStore` interface, `FakeTrackerStore` and the contract tests | U5 |
  | `PassageIndex` interface | U1 |
  | Processor and Job Runner skeletons | U4 |
  | `main.py` | U5 |
  | Project setup files | U5 (the `pyproject.toml` dependency list is reviewed by everyone) |
- **AI-DLC**: one Functional Design (contracts only) and one Code Generation for the whole foundation.

## U1: PDF and Retrieval

- **Purpose**: turn an uploaded file into checked page text, chunks and embeddings, and own how passages are stored and searched.
- **Components and files**: PDF Intake (`pdf_intake.py`), Indexer (`indexer.py`), Storage's Chroma part (`storage/passages.py`).
- **Responsibilities**:
  - every FR-2.2 check during the upload request, plus duplicate detection
  - page-by-page extraction
  - chunking with page references, and embeddings
  - the in-memory search over the new paper's chunks
  - Chroma collections:
    - telemetry off
    - an embedding function that refuses to run, and only precomputed vectors
    - queries filtered by paper ID
- **Provides**: `check_upload`, `PdfRejected`, `chunk_pages`, `embed_chunks`, `embed_text`, `InMemoryPassages`, and the `PassageIndex` implementation (called only by `TrackerStore`).
- **Uses**: U0 (types, Settings, Model Client) and the `TrackerStore` interface (`paper_count`, `paper_exists`).
- **Functional Design must decide**:
  - the "too little text" threshold
  - chunk size and overlap
  - Chroma metadata fields
  - confirmation of the refusing embedding function on the installed Chroma version
  - property-based test properties for `chunk_pages`
- **Tests**: `tests/u1_pdf_retrieval/`.

## U2: Paper Cards

- **Purpose**: produce a cited card and check its quotes in code.
- **Components and files**: Card Writer (`cards.py`), Quote Checker (`quotes.py`). After the U0 freeze, also maintains the Model Client.
- **Responsibilities**:
  - the fixed card pipeline in batches
  - the card schema
  - quote normalization, minimum length, page check, and one replacement request
  - "Not verified" statuses and the card-level note
  - prompts for card writing and replacement (C-3)
  - the hard-case quote tests (NFR-4.6)
  - the Ollama behavior checks (A-3, A-4, Q6)
- **Provides**: `write_card`, `normalize`, `check_quote`, `check_with_replacement` (`check_quote` is also used by U4's Evaluation).
- **Uses**: U0 (types, Settings, Model Client).
- **Functional Design must decide**:
  - the card prompt sequence and batch size
  - the minimum quote length (default 8 words)
  - exact normalization rules
  - how thinking mode and JSON schemas interact
  - property-based test properties for `normalize` and `check_quote`
- **Tests**: `tests/u2_paper_cards/`.

## U3: Comparison

- **Purpose**: give every paper exactly one valid label with evidence.
- **Components and files**: Comparison Agent and Agent Tools (`agent/comparison.py`, `tools.py`, `validation.py`, `prompts.py`).
- **Responsibilities**:
  - the first-paper call (FR-6.0)
  - the seeded, capped loop with native tool calling
  - turn re-asks that count toward the cap
  - the final tool-free call, retried alone
  - validation of the label and its citations (retrieved-only, relevant-only, both-paper rule)
  - the three read-only tools, scoped to committed relevant papers
  - the shared relevance criteria
  - comparison prompts (C-3)
- **Provides**: `classify`, `ComparisonFailed`, and the tool contract (names, arguments and result fields). The tool contract is designed first in the parallel phase.
- **Uses**:
  - U0 (types, Settings, Model Client)
  - the `TrackerStore` interface (`list_sections`, `relevant_paper_ids`, `similar_cards`, `search_passages`, `get_paper`)
  - U1's `InMemoryPassages` and `embed_text`
- **Functional Design must decide**:
  - the model-call cap (6–8) and the seed count
  - tool argument limits
  - the exact validation rules
  - how re-ask messages are worded
  - how section names stay broad
- **Tests**: `tests/u3_comparison/`.

## U4: Summaries, Coverage, Export, Evaluation and Integration

- **Purpose**: keep the tracker's summaries and numbers right, run each paper end to end, and measure the app.
- **Components and files**:
  - Tracker Updater (`tracker_updater.py`), Coverage (`coverage.py`), Markdown Exporter (`export.py`)
  - Paper Processor (`processor.py`), Job Runner (`jobs.py`), step timing (`timing.py`)
  - Evaluation (`evaluate/`)
- **Responsibilities**:
  - section summary and overview rewrites
  - counts and year range
  - `tracker.md` rendering
  - the processing step order, step retries, progress, timing, commit-failure handling and failure cleanup
  - one-at-a-time background running, start-up recovery and notices
  - the evaluation run, report, quote checklist, stored-data check, timing summary and settings record
  - the label file format, agreed with the test set workstream
- **Provides**: `plan_update`, `compute_coverage`, `render_markdown`, `PaperProcessor.process`, `ProcessingFailed`, `JobRunner` (`submit`, `status`, `Busy`), and the evaluation command.
- **Uses**: U0, U1 (`check_upload`, Indexer), U2 (`write_card`, `check_quote`), U3 (`classify`), and the `TrackerStore` interface (including `commit`, `cleanup_uncommitted`, job records and notices).
- **Cross-unit review**: U5's owner reviews the commit-failure part of U4's Functional Design.
- **Functional Design must decide**:
  - the summary and overview prompts and batch sizes
  - whether the overview is rewritten for not-relevant papers
  - timing log format
  - the label file format
  - checklist format
  - property-based test properties for `compute_coverage` and `render_markdown`
- **Tests**: `tests/u4_summaries_integration/`, and the end-to-end fake-model test of the whole pipeline.
- **Workload note**: U4 has the most stories (see the story map). Most are small (pure functions) or concentrated in Evaluation. If the load proves too high, US-26 and US-27 (quote checklist and stored-data check) can move to U2's owner, who already owns quote checking.

## U5: Interface and Persistence

- **Purpose**: everything the student sees and everything kept on disk.
- **Components and files**:
  - Web (`web/routes.py`, `web/templates/`, `web/static/poll.js`, `main.py`)
  - Storage (`storage/db.py`, `schema.sql`, `files.py`, `store.py`)
  - after the U0 freeze, also Settings and the project setup files
- **Responsibilities**:
  - all HTTP routes and pages:
    - Jinja2 autoescape, `<details>` citations, the polling script
    - PDF by ID only
    - export download, notice dismissal
    - error-to-message mapping
  - the start-up order
  - SQLite with per-thread connections and `?` placeholders
  - the schema and version check
  - files by hash
  - the `TrackerStore` facade: committed-only reads, `commit`, `cleanup_uncommitted`, job records and notices
  - the real store passing the shared contract tests
- **Provides**: `TrackerStore` (real), the HTTP routes, `create_app`, Settings, and the SQLite schema. The schema is designed first in the parallel phase.
- **Uses**: U0, U1 (`PassageIndex`, inside `TrackerStore` only), U4 (`compute_coverage` and `render_markdown` for the tracker view and `tracker.md`; `JobRunner` from the upload and status routes), and U1 (`check_upload`) from the upload route.
- **Functional Design must decide**:
  - tables and columns
  - transaction boundaries for `commit`
  - the cleanup order
  - template structure
  - the polling interval
  - error codes and messages
  - the schema-version message
- **Tests**: `tests/u5_interface_persistence/`, plus running `tests/contracts/` against the real `TrackerStore`.

---

## Code Organization Strategy (greenfield)

```text
<repository root>/
    research_tracker/            # one package at the root (Q7: B); component modules per components.md
    tests/
        conftest.py              # shared fixtures: fake model, temp data folder, sample pages
        fixtures/                # sample text, two test-set papers' expected outputs
        fakes/fake_store.py      # FakeTrackerStore (U0, then U5)
        contracts/               # TrackerStore contract tests (U0, then U5)
        u0_foundation/
        u1_pdf_retrieval/
        u2_paper_cards/
        u3_comparison/
        u4_summaries_integration/
        u5_interface_persistence/
    pyproject.toml, uv.lock, Modelfile, .env.example, .gitignore, README.md
    aidlc-docs/                  # documentation only, never application code
```

- **Running the app and tools**:
  - `uv run uvicorn research_tracker.main:app` and `uv run python -m research_tracker.evaluate` work from the repository root
  - pytest gets the root on its import path through `pyproject.toml`
- **Model tests** (real Qwen3) are marked `@pytest.mark.model` and are skipped unless asked for (NFR-4.3). They live in each unit's test folder.

## Integration Approach

- **Contract-first** (Q4: A). U0 defines the types and interfaces and supplies two fakes, `FakeModelClient` and `FakeTrackerStore`. Every unit builds and tests against those.
- **No drift between the fake and real store**. The `tests/contracts/` suite runs against both. A behavior change in the interface updates the contract tests first.
- **Continuous integration through the skeleton** (Q2: B, Q8 fix). The skeleton Processor runs end to end from day one with the fake model. Each unit replaces its stub as it finishes. U4 keeps an end-to-end fake-model test green at all times.
- **When the real store replaces the fake**: as soon as U5's `TrackerStore` passes the contract tests, the end-to-end test switches to it.

## Running AI-DLC Across Owners (Q5: B)

- Each owner runs their unit's Construction stages (Functional Design, Code Generation) in their own session and git branch.
- Unit documents go in `aidlc-docs/construction/{unit-name}/` (`functional-design/`, `code/`, `plans/`), together with that unit's own `audit.md`.
- The main `aidlc-docs/audit.md` records only cross-unit decisions (contract changes, reviews across units) and stage approvals.
- `aidlc-state.md` has one progress line per unit, so branches edit different lines.
- Each unit's owner and reviewer approve that unit's stages (execution plan note). Contract changes need approval from every affected unit.
